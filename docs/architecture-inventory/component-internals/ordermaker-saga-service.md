# Component internals — `ordermaker-saga-service`

| | |
| --- | --- |
| **Component** | `ordermaker-saga-service` |
| **Source repository** | `hianshul100_Pacco.Services.OrderMaker` |
| **Scoped path** | `.` (whole repository; single project `src/Pacco.Services.OrderMaker`) |
| **Base ref** | `feature/12998/aidlc` (HEAD `2d9ae68` — *"Added security package"*) |
| **Batch** | 4 of 7 |
| **Status** | New artifact. Deepens, and in two places **corrects**, `../patterns/orchestration/saga-process-manager.md`. |
| **Grounding** | Every claim below is tied to a file path and, where useful, a line number. Mechanisms that live in NuGet packages not present in this workspace are marked `[convey]`, `[chronicle]` or `[framework]` and stated as inference. |

> **Scope of verifiability.** The repository contains **32 C# files**, one `.csproj`, four `appsettings*.json`, a `launchSettings.json`, a `Dockerfile`, a `.travis.yml`, four shell scripts, a `.sln`, a `.rest` file, a README and one certificate asset. **All of them were read in full**, so the concept list in §2 is complete by exhaustion rather than by sampling — if a mechanism is not named here, it is not in this repository.
>
> Two dependency families are *not* in the workspace and are therefore reasoned about from their call sites plus documented behaviour: **Convey `0.4.*`** (18 packages) and **Chronicle_ `3.2.1`**. Claims that rest on them carry `[convey]` / `[chronicle]` and are restated as assumptions in §8.1.
>
> Cross-repository claims (Orders, Availability, Vehicles, the platform compose file) were verified by reading those repositories directly; those are cited with their own repo-relative paths.

---

## Contents

1. [Purpose & boundary](#1-purpose--boundary)
2. [Core concepts](#2-core-concepts)
3. [Concept-by-concept model](#3-concept-by-concept-model)
4. [Primary control flows](#4-primary-control-flows)
5. [Persistence & schema evolution](#5-persistence--schema-evolution)
6. [Surface → internals map](#6-surface--internals-map)
7. [Change/extension guide](#7-changeextension-guide)
8. [Assumptions, Blockers & Open Questions](#8-assumptions-blockers--open-questions)

---

## 1. Purpose & boundary

### 1.1 What it is

`ordermaker-saga-service` is Pacco's **single orchestration component**. Every other service in the
platform reacts to events it happens to be subscribed to; this one is the only place where a
*multi-service business transaction is driven from a single point of control*.

Its entire purpose is one workflow, encoded in one class — `AIOrderMakingSaga`
(`src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`, 154 lines). Given a customer, a parcel
and an order id, it:

1. asks `orders-service` to create the order (`AIOrderMakingSaga.cs:63`),
2. adds every known parcel to it (`:74`),
3. **synchronously** picks a vehicle from `vehicles-service` (`:93`),
4. **synchronously** computes a delivery date from `availability-service` (`:98`),
5. assigns that vehicle and date to the order (`:103`),
6. reserves the vehicle in `availability-service` (`:113`),
7. and, when the order comes back approved, announces completion and closes the saga (`:124`, `:131`).

The name is a joke the code makes about itself: the "AI" that chooses the best vehicle is
`Items.FirstOrDefault()` with the comment `// typical AI in a startup`
(`Services/Clients/VehiclesServiceClient.cs:24`). The banner served at `GET /` reads
`"Welcome to Pacco uber AI order maker Service!"` (`Program.cs:25`). **This is a demonstration of the
saga pattern, not a production ordering engine** — a judgement grounded in §3.13 (state is lost on
restart), §3.12 (the completion predicate is inverted), §3.24 (a wired subscription reaches no saga
action) and §3.39 (no tests at all).

### 1.2 What it is not

| Not | Evidence |
| --- | --- |
| Not a source of truth for orders | It owns no order aggregate; `orders-service` does. This component holds only `AIMakingOrderData` (`Sagas/AIMakingOrderData.cs`) — seven fields that exist to drive the next step. |
| Not a persistent workflow engine | `builder.Services.AddChronicle()` (`Extensions.cs:43`) is called with **no persistence package**. See §3.13. |
| Not a query surface | `Convey.CQRS.Queries` is referenced (`.csproj:13`) but **no `IQuery` type exists** in the repository and no `Get<TQuery,TResult>` endpoint is mapped. Its only use is `PagedResult<T>` in `VehiclesServiceClient.cs:23`. |
| Not the API the front-end calls | `Pacco.Web` and `api-gateway` route order creation to `orders-service`. Nothing in the platform points a user at port 5015 except this repository's own `.rest` file. See §3.38. |
| Not a replacement for the choreography | The plain choreographed flow (`POST /orders` on `orders-service`) still exists and is the one the rest of the platform uses. Both paths publish the **same** commands to the **same** exchange, so both can run at once. See §3.19 and Q3. |
| Not secured | `AddSecurity()` is registered (`Extensions.cs:41`) but `UseAuthentication()`/`UseAuthorization()` are **never called** (`Extensions.cs:51-65`, `Program.cs:22-26`). See §3.33. |
| Not traced | Every sibling service calls `.UseJaeger()`; this one does not, and does not reference the package. See §3.35. |

### 1.3 Position in the platform

```
                  POST /orders  {parcelId, customerId}     ← unauthenticated (§3.33)
                            │
                  ┌─────────▼──────────────┐
                  │ ordermaker-service     │  :5015 / :80
                  │  AIOrderMakingSaga     │  in-memory saga state (§3.13)
                  └───┬────────────┬───────┘
      publishes (AMQP)│            │reads (HTTP, via Fabio) (§3.26)
                      │            ├──────────────► vehicles-service   GET /vehicles      (§3.27)
                      │            └──────────────► availability-service GET /resources/{id} (§3.28)
                      │
      exchange "orders"│                         exchange "availability"
   CreateOrder ────────┤                         ReserveResource ──────┐
   AddParcelToOrder ───┤                                                │
   AssignVehicleToOrder┤                                                │
   CancelOrder ────────┘                                                │
                      ▼                                                 ▼
             ┌──────────────────┐                          ┌──────────────────────┐
             │ orders-service   │                          │ availability-service │
             └────────┬─────────┘                          └──────────┬───────────┘
   OrderCreated,      │                                               │ ResourceReserved
   ParcelAddedToOrder,│                                               │
   VehicleAssignedTo… │◄──────────── ResourceReserved ────────────────┘
   OrderApproved      │              (Orders approves the order)
                      ▼
             back to ordermaker-service (5 subscriptions, §3.23)
```

The loop is closed by `orders-service`: its `ResourceReservedHandler`
(`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Events/External/Handlers/ResourceReservedHandler.cs:24-35`)
calls `order.Approve()` and publishes `OrderApproved`, which is the saga's terminal trigger. **The
saga never sends `ApproveOrder` itself**, even though it defines that command (§3.22).

| Direction | Peer | Mechanism |
| --- | --- | --- |
| inbound | any HTTP client | `POST /orders` (§3.5) |
| inbound | `orders-service` | AMQP events `OrderCreated`, `ParcelAddedToOrder`, `VehicleAssignedToOrder`, `OrderApproved` (§3.23) |
| inbound | `availability-service` | AMQP event `ResourceReserved` — **subscribed but inert** (§3.24) |
| outbound | `orders-service` | AMQP commands `CreateOrder`, `AddParcelToOrder`, `AssignVehicleToOrder`, `CancelOrder` (§3.19) |
| outbound | `availability-service` | AMQP command `ReserveResource` (§3.19, §3.21) |
| outbound | `vehicles-service` | HTTP `GET /vehicles` (§3.27) |
| outbound | `availability-service` | HTTP `GET /resources/{resourceId}` (§3.28) |
| outbound | `operations-service` | AMQP event `MakeOrderCompleted` on exchange `ordermaker` — subscribed generically (`Pacco.Services.Operations.Api/messages.json:75-83`) but **dropped on arrival** because the correlation id is empty (§3.18, §3.31) |
| infra | Consul, Fabio, RabbitMQ, Redis, Seq, Prometheus | §3.25, §3.26, §3.34, §3.35, §3.36 |

---

## 2. Core concepts

Forty concepts, in the order the code reaches them. Each has a full entry in §3.

| # | Concept | Where it lives |
| --- | --- | --- |
| 3.1 | Service identity & `app` options | `appsettings.json:2-6` |
| 3.2 | Hosting & composition root | `Program.cs:15-29` |
| 3.3 | `AddInfrastructure` — the registration graph | `Extensions.cs:26-49` |
| 3.4 | `UseApp` — the middleware & subscription pipeline | `Extensions.cs:51-65` |
| 3.5 | `MakeOrder` — the only inbound contract | `Commands/MakeOrder.cs` |
| 3.6 | HTTP surface & `UseDispatcherEndpoints` | `Program.cs:24-26` |
| 3.7 | In-memory command dispatch & response semantics | `Extensions.cs:35-36` |
| 3.8 | `AIOrderMakingHandler` — the funnel into Chronicle | `Handlers/AIOrderMakingHandler.cs` |
| 3.9 | `ISagaCoordinator.ProcessAsync` and `SagaContext.Empty` | `Handlers/AIOrderMakingHandler.cs:25-41` |
| 3.10 | `AIOrderMakingSaga` — type shape & action set | `Sagas/AIOrderMakingSaga.cs:16-22` |
| 3.11 | `ResolveId` — the saga correlation key | `Sagas/AIOrderMakingSaga.cs:45-54` |
| 3.12 | `AIMakingOrderData` — saga state | `Sagas/AIMakingOrderData.cs` |
| 3.13 | `AllPackagesAddedToOrder` — the completion predicate | `Sagas/AIMakingOrderData.cs:16` |
| 3.14 | Saga persistence: in-memory only | `Extensions.cs:43` |
| 3.15 | Saga lifecycle, states and `CompleteAsync` | `Sagas/AIOrderMakingSaga.cs:131` |
| 3.16 | Compensation model | `Sagas/AIOrderMakingSaga.cs:134-152` |
| 3.17 | The `Saga` header | `Sagas/AIOrderMakingSaga.cs:23,67` |
| 3.18 | `ICorrelationContextAccessor` and the constructor overwrite | `Sagas/AIOrderMakingSaga.cs:39-42` |
| 3.19 | Local `CorrelationContext` type | `CorrelationContext.cs` |
| 3.20 | Outbound command contracts & exchange routing | `Commands/External/*.cs` |
| 3.21 | `AddParcelToOrder` — the dropped `customerId` | `Commands/External/AddParcelToOrder.cs` |
| 3.22 | `ReserveResource` — positional drift | `Commands/External/ReserveResource.cs` |
| 3.23 | `ApproveOrder` — the command nobody sends | `Commands/External/ApproveOrder.cs` |
| 3.24 | Inbound event contracts & `SubscribeEvent` wiring | `Events/External/*.cs`, `Extensions.cs:58-62` |
| 3.25 | `ResourceReserved` — subscribed but inert | `Handlers/AIOrderMakingHandler.cs:210` |
| 3.26 | RabbitMQ topology & conventions | `appsettings.json:89-130` |
| 3.27 | Synchronous HTTP reads & service resolution | `Services/Clients/*.cs`, `appsettings.json:23-34` |
| 3.28 | `VehiclesServiceClient.GetBestAsync` — the "AI" | `Services/Clients/VehiclesServiceClient.cs:21-31` |
| 3.29 | `ResourceReservationsService.GetBestAsync` — the scheduler | `Services/ResourceReservationsService.cs:18-44` |
| 3.30 | Local DTOs and their upstream coupling | `DTO/*.cs` |
| 3.31 | `MakeOrderCompleted` — terminal event with no consumer | `Events/MakeOrderCompleted.cs` |
| 3.32 | `MakeOrderRejected` — the unused rejection convention | `Events/Rejected/MakeOrderRejected.cs` |
| 3.33 | Error handling & `ExceptionToResponseMapper` | `ExceptionToResponseMapper.cs` |
| 3.34 | Security posture | `Extensions.cs:41`, `appsettings.json:71-79` |
| 3.35 | Redis: registered, never used | `Extensions.cs:37`, `appsettings.json:131-134` |
| 3.36 | Observability: logging, metrics, no tracing | `appsettings.json:35-88` |
| 3.37 | Service discovery & health | `appsettings.json:7-22` |
| 3.38 | Configuration layering & environments | `appsettings.{local,docker,development}.json` |
| 3.39 | Deployment & topology | `Dockerfile`, `hianshul100_Pacco/compose/services.yml:78-85` |
| 3.40 | Build, CI and the absence of tests | `.travis.yml`, `scripts/*.sh`, `.sln` |

Two cross-cutting properties are treated inside the entries above rather than as separate concepts:
**concurrency/ordering** (§3.9, §3.13, §3.14) and **the package graph, including unused references**
(§3.3, §3.35).

---

## 3. Concept-by-concept model

Each entry follows the same six headings: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement** (explicitly labelling whether a violation *fails loudly*
or *fails silently*), **Extension procedure**, **Failure modes**.

### 3.1 Service identity & `app` options

**Definition.** The logical name the platform uses for this process: `ordermaker-service`. It is the
Consul service name, the Fabio route key, the RabbitMQ connection name, the queue-template prefix
and the compose service name — one string reused in five roles.

**Representation & storage.** `appsettings.json:2-6`:

```json
"app": { "name": "Pacco Order Maker Service", "service": "ordermaker-service", "version": "1" }
```

Bound by Convey into `AppOptions` `[convey]`. The same literal appears at `appsettings.json:10`
(consul.service), `:21` (fabio.service), `:90` (rabbitMq.connectionName), `:123` (queue.template) and
in `hianshul100_Pacco/compose/services.yml:78`. `appsettings.docker.json:3` overrides only the
display `name` to `"Pacco OrderMaker Service"` — the `service` key is inherited, so the identity is
stable across environments.

**Lifecycle.** Read once at host build (`Program.cs:16`, `CreateDefaultBuilder` loads
`appsettings.json` then `appsettings.{ASPNETCORE_ENVIRONMENT}.json` `[framework]`). Never re-read;
there is no `IOptionsMonitor` consumer anywhere in the repository.

**Invariants & enforcement.**
- The five occurrences must agree. **Nothing enforces this** — a mismatch between
  `consul.service` and `httpClient.services` entries in *other* services would break routing to this
  one at runtime with a 404 from Fabio. **Fails silently at build, loudly at runtime.**
- `version` is a string `"1"` and is used nowhere in code — it is inert metadata.

**Extension procedure.** To rename the service you must change `appsettings.json:4,10,21,90,123`,
`appsettings.docker.json` (consul.address at `:10`), `compose/services.yml:78-85` in
`hianshul100_Pacco`, the Docker repository name in `scripts/dockerize.sh:16`, and any
`httpClient.services` entry pointing here — there are **none today**, which is itself a finding
(§1.2: no service calls this one).

**Failure modes.** A Consul registration under a stale name leaves the process healthy but
unreachable through Fabio. Because nothing calls this service, that failure is currently invisible.

---

### 3.2 Hosting & composition root

**Definition.** The single `Program.Main` that builds the web host, registers everything and maps
two endpoints. There is no `Startup` class; the whole composition is nine fluent lines.

**Representation & storage.** `Program.cs:15-29`:

```csharp
public static async Task Main(string[] args)
    => await WebHost.CreateDefaultBuilder(args)
        .ConfigureServices(services => services
            .AddConvey().AddWebApi().AddInfrastructure().Build())     // :17-21
        .Configure(app => app
            .UseApp()                                                  // :23
            .UseDispatcherEndpoints(endpoints => endpoints
                .Get("", ctx => ctx.Response.WriteAsync("Welcome to Pacco uber AI order maker Service!"))
                .Post<MakeOrder>("orders")))                           // :24-26
        .UseLogging()                                                  // :27
        .Build().RunAsync();
```

**Lifecycle.** `AddConvey()` creates the `IConveyBuilder`; every `Add*` extension appends a
registration; `.Build()` (`:21`) executes the deferred registrations `[convey]`. `UseLogging()`
(`:27`) installs Serilog from the `logger` section (§3.36) at the **host** level, so it also captures
startup logs.

**Invariants & enforcement.**
- `.Build()` must be the last call in the `ConfigureServices` chain. Omitting it silently skips
  every deferred registration `[convey]` and the app starts with a half-empty container —
  **fails loudly** only when the first resolution throws.
- `UseApp()` must precede `UseDispatcherEndpoints`, because `UseApp` installs the error handler and
  the RabbitMQ subscriptions. Ordering is positional and unchecked. **Fails silently** if inverted:
  exceptions bypass `ExceptionToResponseMapper` and surface as raw 500s.

**Extension procedure.** New middleware goes in `Extensions.UseApp` (§3.4), not here; new endpoints
go in the `UseDispatcherEndpoints` lambda; new registrations go in `Extensions.AddInfrastructure`
(§3.3). `Program.cs` should not need to change for anything except a new endpoint.

**Failure modes.** Compared with every sibling service, one call is conspicuously absent:
`.UseJaeger()` (see `hianshul100_Pacco.Services.Orders/.../Infrastructure/Extensions.cs:89`). Its
absence is what makes this the only service whose work is invisible in the trace graph (§3.36).

---

### 3.3 `AddInfrastructure` — the registration graph

**Definition.** The one place where the container is populated. Fourteen Convey extensions plus four
manual registrations.

**Representation & storage.** `Extensions.cs:26-49`:

| Line | Call | What it brings |
| --- | --- | --- |
| 29 | `AddErrorHandler<ExceptionToResponseMapper>()` | binds §3.33 |
| 30 | `AddHttpClient()` | `IHttpClient` + `HttpClientOptions` (§3.27) |
| 31 | `AddConsul()` | registration + `ping` health endpoint (§3.37) |
| 32 | `AddFabio()` | load-balanced HTTP resolution (§3.27) |
| 33 | `AddCommandHandlers()` | assembly-scans `ICommandHandler<>` → finds `AIOrderMakingHandler` |
| 34 | `AddEventHandlers()` | assembly-scans `IEventHandler<>` → same class, five interfaces |
| 35 | `AddInMemoryCommandDispatcher()` | `ICommandDispatcher` (§3.7) |
| 36 | `AddInMemoryEventDispatcher()` | `IEventDispatcher` |
| 37 | `AddRedis()` | `IDistributedCache` — **never injected anywhere** (§3.35) |
| 38 | `AddMetrics()` | App.Metrics + Prometheus scrape endpoint (§3.36) |
| 39 | `AddRabbitMq()` | `IBusPublisher`, `IBusSubscriber`, conventions (§3.26) |
| 40 | `AddWebApiSwaggerDocs()` | `/docs` (§3.6) |
| 41 | `AddSecurity()` | JWT handler + `AddCertificates` (§3.34) |
| 43 | `builder.Services.AddChronicle()` | saga coordinator, **in-memory** (§3.14) |
| 44 | `AddTransient<IAvailabilityServiceClient, AvailabilityServiceClient>()` | §3.27 |
| 45 | `AddTransient<IVehiclesServiceClient, VehiclesServiceClient>()` | §3.27 |
| 46 | `AddTransient<IResourceReservationsService, ResourceReservationsService>()` | §3.29 |

Note the change of receiver at `:43`: `AddChronicle()` extends `IServiceCollection`, not
`IConveyBuilder`, so it is called on `builder.Services` and its return value is discarded. The saga
class itself is **not** registered here — Chronicle discovers `Saga<TData>` implementations by
assembly scan `[chronicle]` (A2 in §8.1).

**Lifecycle.** Executed once, during `ConfigureServices`. All four manual registrations are
`Transient`, so a new `AvailabilityServiceClient`, `VehiclesServiceClient` and
`ResourceReservationsService` are constructed per resolution — and, because `AIOrderMakingSaga`
takes them as constructor parameters, **per saga message** (§3.10).

**Invariants & enforcement.**
- `AddCommandHandlers()` and `AddEventHandlers()` scan the entry assembly. A handler in another
  assembly would be silently missed — not a risk here (one project).
- `AddRabbitMq()` must be registered for `IBusPublisher` to resolve. The saga takes `IBusPublisher`
  in its constructor (`Sagas/AIOrderMakingSaga.cs:31`), so removing `:39` breaks saga construction
  — **fails loudly**, at the first message, not at startup.
- Three referenced packages are never used: `Convey.MessageBrokers.Outbox` (`.csproj:20` — no
  `AddMessageOutbox`, no `UseMessageOutbox` anywhere), `Convey.CQRS.Queries` (`.csproj:13` — used
  only for the `PagedResult<T>` type), and `Convey.Persistence.Redis`'s runtime effect (§3.35).
  **Fails silently**: they inflate the image and imply capabilities the service does not have.

**Extension procedure.** Add the `Add*` call in the chain at `:28-41` and, if it has a middleware
counterpart, the matching `Use*` in `UseApp` (§3.4). For a new collaborator, register it at `:44-46`
and inject it into the saga constructor.

**Failure modes.** The **most consequential omission is the outbox**. Every `PublishAsync` in the
saga (§3.20) is a bare AMQP publish with no transactional guarantee; if the process dies between
mutating `Data` and publishing, the step is lost with no retry. `Convey.MessageBrokers.Outbox` is
sitting in the `.csproj` unused — the fix is one registration away and was evidently intended. This
is baseline gap **G3**.

---

### 3.4 `UseApp` — the middleware & subscription pipeline

**Definition.** The request pipeline plus the five AMQP subscriptions that make this service
reactive.

**Representation & storage.** `Extensions.cs:51-65`:

```csharp
app.UseErrorHandler()      // :53  → ExceptionToResponseMapper (§3.33)
   .UseSwaggerDocs()       // :54  → /docs
   .UseConvey()            // :55  → correlation/context middleware [convey]
   .UseMetrics()           // :56  → /metrics
   .UseRabbitMq()          // :57  → starts IBusSubscriber
   .SubscribeEvent<OrderApproved>()          // :58
   .SubscribeEvent<OrderCreated>()           // :59
   .SubscribeEvent<ParcelAddedToOrder>()     // :60
   .SubscribeEvent<ResourceReserved>()       // :61   ← inert (§3.25)
   .SubscribeEvent<VehicleAssignedToOrder>() // :62
```

Each `SubscribeEvent<T>` declares a queue named by the `queue.template`
(`appsettings.json:123`, `"ordermaker-service/{{exchange}}.{{message}}"`) and binds it to the
exchange named by `T`'s `[Message]` attribute with a routing key derived from the type name in
snake_case (`appsettings.json:93`, `conventionsCasing: "snakeCase"`) `[convey]`. So:

| Subscription | Exchange | Queue |
| --- | --- | --- |
| `OrderApproved` | `orders` | `ordermaker-service/orders.order_approved` |
| `OrderCreated` | `orders` | `ordermaker-service/orders.order_created` |
| `ParcelAddedToOrder` | `orders` | `ordermaker-service/orders.parcel_added_to_order` |
| `ResourceReserved` | `availability` | `ordermaker-service/availability.resource_reserved` |
| `VehicleAssignedToOrder` | `orders` | `ordermaker-service/orders.vehicle_assigned_to_order` |

Queue names are derived, not literal — the table is inference from the template plus the casing
convention `[convey]` (A3 in §8.1).

**Lifecycle.** Subscriptions are declared at startup and last for the process lifetime; queues are
`durable: true, autoDelete: false` (`appsettings.json:119-122`), so **messages accumulate while the
service is down and are delivered on restart** — into a saga store that no longer has the matching
state (§3.14). That combination is the single most important operational fact about this service.

**Invariants & enforcement.**
- Every subscribed event must have an `IEventHandler<T>`; otherwise resolution fails when the
  message arrives. All five do (`Handlers/AIOrderMakingHandler.cs:11-16`). **Fails loudly**, per
  message.
- The converse is *not* enforced: an event can be subscribed and handled and still reach no saga
  action. `ResourceReserved` is exactly that (§3.25). **Fails silently.**
- `UseRabbitMq()` must precede the `SubscribeEvent` calls; they are extensions on the value it
  returns, so the compiler enforces this one.

**Extension procedure.** To react to a new external event: create the contract under
`Events/External/` with the right `[Message]` exchange (§3.24), add
`IEventHandler<T>` to `AIOrderMakingHandler` with a `ProcessAsync` body, add `ISagaAction<T>` to the
saga with both `HandleAsync` and `CompensateAsync`, add a `ResolveId` arm (§3.11), and add
`.SubscribeEvent<T>()` here. **Five edits; skipping any one of the last four fails silently.**

**Failure modes.** No `SubscribeCommand<T>` is registered, so this service cannot be driven by AMQP
commands — only by HTTP. Any plan to trigger the saga from another service must add both a
subscription here and a publisher there.

---

### 3.5 `MakeOrder` — the only inbound contract

**Definition.** The command that starts a saga. It is the sole thing a caller can ask this service
to do.

**Representation & storage.** `Commands/MakeOrder.cs`:

```csharp
public class MakeOrder : ICommand
{
    public Guid OrderId { get; }
    public Guid CustomerId { get; }
    public Guid ParcelId { get; }

    public MakeOrder(Guid orderId, Guid customerId, Guid parcelId)
    {
        OrderId = orderId == Guid.Empty ? Guid.NewGuid() : orderId;   // :14
        CustomerId = customerId;
        ParcelId = parcelId;
    }
}
```

Get-only properties with a constructor — the Convey/Pacco house style. Deserialised from the
request body by the dispatcher endpoint (§3.6) using Newtonsoft `[convey]`, which matches JSON
properties to constructor parameters by name, case-insensitively.

**Lifecycle.** Constructed per HTTP request; passed to `ICommandDispatcher`; forwarded verbatim into
Chronicle (§3.9); its `OrderId` becomes the `SagaId` (§3.11) and is copied into `Data.OrderId`
(`Sagas/AIOrderMakingSaga.cs:60`). It is not stored anywhere else.

**Invariants & enforcement.**
- **`OrderId` is auto-generated when empty** (`:14`). This is the single most useful behaviour of
  the type: the `.rest` sample (`Pacco.Services.OrderMaker.rest:9-12`) posts only `parcelId` and
  `customerId`, so `orderId` deserialises to `Guid.Empty` and a fresh id is minted. **The caller
  never learns that id** — the response body is empty (§3.6). *Fails silently* in the sense that the
  client has no handle on the workflow it just started.
- `CustomerId` and `ParcelId` have **no validation at all**. `Guid.Empty` for either is accepted and
  propagates: `CreateOrder` is published with an empty customer id, and `orders-service` will reject
  it downstream. **Fails loudly, but in another service and asynchronously** — the HTTP caller has
  already received 2xx.
- Exactly **one** `ParcelId` can be supplied. `Data.ParcelIds` is a list (`AIMakingOrderData.cs:12`)
  but the entry point adds exactly one element (`AIOrderMakingSaga.cs:59`). The multi-parcel
  machinery in `HandleAsync(OrderCreated)` (§3.13) therefore always iterates a single item.

**Extension procedure.** To accept several parcels, change `ParcelId` to
`IEnumerable<Guid> ParcelIds`, replace `Data.ParcelIds.Add(message.ParcelId)`
(`AIOrderMakingSaga.cs:59`) with `AddRange`, and fix §3.13's predicate — which is the point at which
the inverted predicate stops being harmless.

**Failure modes.** Because `MakeOrder` carries no idempotency token beyond `OrderId`, posting the
same body twice with an empty `orderId` starts **two independent sagas** creating two orders for the
same parcel. Supplying an explicit `orderId` twice re-enters the same saga id — see §3.11 for what
Chronicle does with that.

---

### 3.6 HTTP surface & `UseDispatcherEndpoints`

**Definition.** Two routes, mapped inline. There are no MVC controllers in the repository.

**Representation & storage.** `Program.cs:24-26`:

| Method & path | Binding | Behaviour |
| --- | --- | --- |
| `GET /` | inline lambda | writes `"Welcome to Pacco uber AI order maker Service!"` |
| `POST /orders` | `.Post<MakeOrder>("orders")` | deserialises the body into `MakeOrder`, resolves `ICommandDispatcher`, dispatches, returns **`202 Accepted` with an empty body** `[convey]` |

Two more endpoints exist by virtue of registrations rather than mapping: `/ping` (Consul health,
`appsettings.json:14`), `/metrics` (Prometheus, `appsettings.json:83`) and `/docs` (Swagger,
`appsettings.json:141`). All three are excluded from request logging (`appsettings.json:37`) —
except `/docs`, which is not.

**Lifecycle.** Routes are registered once at startup and are static thereafter.

**Invariants & enforcement.**
- `.Post<MakeOrder>("orders")` has **no `afterDispatch` callback**. Compare `availability-service`,
  which returns a `Location` header:
  `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/Program.cs:41-42`
  uses `afterDispatch: (cmd, ctx) => ctx.Response.Created(...)`. Here the caller gets no id, no
  location, and no way to poll. **Fails silently** — the API looks like it worked.
- **The platform's status channel is wired but unusable from here.** `operations-service` *does*
  subscribe to this service's events — `Pacco.Services.Operations.Api/messages.json:75-83` declares
  exchange `ordermaker` with event `make_order_completed` and rejected event `make_order_rejected`.
  But its `GenericEventHandler` returns immediately when the AMQP correlation id is blank
  (`Pacco.Services.Operations.Api/Handlers/GenericEventHandler.cs:31-35`), and this service always
  publishes with a blank correlation context (§3.18). Additionally, this service never builds a
  `Correlation-Context` from the inbound HTTP request — contrast
  `hianshul100_Pacco.Services.Orders/.../Infrastructure/Extensions.cs:113-116`. **So a caller of
  `POST /orders` on this service has no status channel at all**, even though the plumbing for one
  exists on both sides. See §3.31.
- Swagger is enabled in every environment including `docker` (`appsettings.json:136`,
  not overridden in `appsettings.docker.json`). **Fails silently** as an exposure concern.

**Extension procedure.** Add a line to the lambda at `Program.cs:24-26`. For a read endpoint you
would need `Get<TQuery, TResult>` and an `IQueryHandler` — `Convey.CQRS.Queries` is already
referenced (`.csproj:13`), so only the handler is missing.

**Failure modes.** `POST /orders` with a malformed body produces a deserialisation exception, which
`ExceptionToResponseMapper` flattens to `400 {"code":"error","reason":"There was an error."}`
(§3.33) — indistinguishable from a business failure.

---

### 3.7 In-memory command dispatch & response semantics

**Definition.** How a `MakeOrder` gets from the HTTP endpoint to `AIOrderMakingHandler`.

**Representation & storage.** `Extensions.cs:35-36` registers
`AddInMemoryCommandDispatcher()` and `AddInMemoryEventDispatcher()`. The in-memory dispatcher
resolves `ICommandHandler<MakeOrder>` from the container and awaits `HandleAsync` **in the request
thread** `[convey]`.

**Lifecycle.** Per request: endpoint → `ICommandDispatcher.SendAsync` → handler → coordinator →
saga → `PublishAsync`. The HTTP response is not written until the saga's first `HandleAsync`
returns.

**Invariants & enforcement.**
- **The `202 Accepted` is a lie about asynchrony.** Because dispatch is in-process and awaited, the
  request does not complete until `CreateOrder` has been published. Any exception thrown inside
  `HandleAsync(MakeOrder)` — including a Chronicle failure — surfaces as an HTTP error. *This is the
  only place in the whole workflow where a failure is visible to the caller.* Every subsequent step
  runs on a RabbitMQ consumer thread where nobody is listening (§3.16).
- The event dispatcher is registered but **never used**: no code resolves `IEventDispatcher`.
  Inbound events arrive via the RabbitMQ subscriber, not the in-memory dispatcher. **Fails
  silently** — dead registration.

**Extension procedure.** If the first saga step ever becomes slow (it publishes one message today,
so it is not), move dispatch off the request thread with Convey's outbox or a background queue. Do
not simply fire-and-forget the task: the current shape is the only error reporting the caller has.

**Failure modes.** If RabbitMQ is unavailable, `PublishAsync` throws inside the request and the
caller sees a 400 (§3.33). Nothing retries. The saga has already mutated `Data` (`:59-61`) before the
publish at `:63`, so an in-memory saga instance exists with state but no message ever sent — a
**stuck saga**, invisible, that will never receive `OrderCreated`.

---

### 3.8 `AIOrderMakingHandler` — the funnel into Chronicle

**Definition.** One class implementing six handler interfaces, each body identical: hand the message
to the saga coordinator.

**Representation & storage.** `Handlers/AIOrderMakingHandler.cs`:

```csharp
public class AIOrderMakingHandler :
    ICommandHandler<MakeOrder>,               // :11
    IEventHandler<OrderApproved>,             // :12
    IEventHandler<OrderCreated>,              // :13
    IEventHandler<ParcelAddedToOrder>,        // :14
    IEventHandler<VehicleAssignedToOrder>,    // :15
    IEventHandler<ResourceReserved>           // :16
{
    private readonly ISagaCoordinator _coordinator;
    public Task HandleAsync(MakeOrder command) => _coordinator.ProcessAsync(command, SagaContext.Empty);  // :25-26
    // …five more, byte-identical except for the parameter type (:28-41)
}
```

**Lifecycle.** Resolved per message by the dispatcher / subscriber. Holds no state.

**Invariants & enforcement.**
- The six interfaces here and the five subscriptions in `UseApp` (§3.4) plus the one HTTP route
  (§3.6) must line up. They do — but **the count is 6 here and 5 saga actions in
  `AIOrderMakingSaga`** (§3.10). `ResourceReserved` is the sixth. **Fails silently** (§3.25).
- Every body passes `SagaContext.Empty`. Consequences in §3.9.
- The class is `public` and non-sealed, unlike the `internal sealed` style used for handlers in
  `orders-service` and `availability-service`. Cosmetic; no enforcement.

**Extension procedure.** Add the interface, add the one-line body, and — critically — add the
matching `ISagaAction<T>` to the saga. The compiler will not remind you.

**Failure modes.** This class is the *only* thing standing between the message bus and Chronicle. It
does no filtering, no validation and no logging, so a message that reaches a saga action the saga
does not implement is dropped inside Chronicle with no trace here.

---

### 3.9 `ISagaCoordinator.ProcessAsync` and `SagaContext.Empty`

**Definition.** Chronicle's entry point: resolve the saga id from the message, load-or-create the
saga's state, invoke the matching `HandleAsync`, persist the state, and on exception invoke
`CompensateAsync` for the actions already executed `[chronicle]`.

**Representation & storage.** Called six times, always as
`_coordinator.ProcessAsync(message, SagaContext.Empty)`
(`Handlers/AIOrderMakingHandler.cs:26,29,32,35,38,41`). `SagaContext.Empty` is Chronicle's
null-object context — no metadata, no originator, no user data `[chronicle]`.

**Lifecycle.** Per message. The coordinator:
1. calls `ResolveId(message, context)` (§3.11),
2. looks the saga up in the saga log/state store (in-memory, §3.14),
3. for `ISagaStartAction<T>` creates a new instance if none exists; for `ISagaAction<T>` requires an
   existing one,
4. constructs `AIOrderMakingSaga` from the container — **a new instance per message**, which is why
   the constructor's side effect (§3.18) runs on every message,
5. awaits `HandleAsync`,
6. persists `Data`,
7. on exception, walks back through the completed actions calling `CompensateAsync` (§3.16).

**Invariants & enforcement.**
- **`SagaContext.Empty` discards the incoming message context.** The AMQP `message_context` header
  (`appsettings.json:127`) and the `span_context` header (`:129`) are available to the subscriber but
  are not threaded into the saga context. Combined with §3.18, the result is that **nothing about the
  original request survives into the saga.** *Fails silently* — logs and messages are simply
  uncorrelated.
- If no saga exists for a resolved id and the message is a non-start action, Chronicle's behaviour is
  to do nothing `[chronicle]` (A4 in §8.1). This is what happens to every in-flight order after a
  restart (§3.14). **Fails silently.**
- Nothing serialises concurrent `ProcessAsync` calls for the same saga id at the application level.
  With one parcel per order the flow is naturally sequential, but with several parcels
  `HandleAsync(ParcelAddedToOrder)` can run concurrently on multiple consumer threads, and
  `Data.AddedParcelIds.Add` (`AIOrderMakingSaga.cs:86`) mutates a plain `List<Guid>` — **not
  thread-safe**. Whether Chronicle locks per saga id is `[chronicle]`-dependent and is recorded as
  A5 in §8.1. If it does not, the failure is a corrupted list or a lost update: **fails silently**.

**Extension procedure.** To carry context, replace `SagaContext.Empty` with a populated
`ISagaContext` built from the message properties, and stop overwriting the correlation context in the
saga constructor (§3.18). Both changes are needed; either alone accomplishes nothing.

**Failure modes.** An exception anywhere in a saga action triggers Chronicle's compensation walk.
Since four of the five `CompensateAsync` methods are `Task.CompletedTask` (§3.16), that walk mostly
does nothing, and the exception then propagates back to the RabbitMQ subscriber, where Convey's
handler either nacks or logs-and-drops `[convey]` (A6 in §8.1).

---

### 3.10 `AIOrderMakingSaga` — type shape & action set

**Definition.** The process manager. One class, five action interfaces, five collaborators.

**Representation & storage.** `Sagas/AIOrderMakingSaga.cs:16-43`:

```csharp
public class AIOrderMakingSaga : Saga<AIMakingOrderData>,
    ISagaStartAction<MakeOrder>,               // :17  ← the only start action
    ISagaAction<OrderCreated>,                 // :18
    ISagaAction<ParcelAddedToOrder>,           // :19
    ISagaAction<VehicleAssignedToOrder>,       // :20
    ISagaAction<OrderApproved>                 // :21
{
    private const string SagaHeader = "Saga";                          // :23
    private readonly IResourceReservationsService _resourceReservationsService;
    private readonly IVehiclesServiceClient _vehiclesServiceClient;
    private readonly IBusPublisher _publisher;
    private readonly ICorrelationContextAccessor _accessor;
    private readonly ILogger<AIOrderMakingSaga> _logger;
    // ctor :30-43 — assigns all five, then overwrites _accessor.CorrelationContext (§3.18)
}
```

`Saga<AIMakingOrderData>` gives the class a `Data` property of that type and the `CompleteAsync()` /
`RejectAsync()` state transitions `[chronicle]`.

**Lifecycle.** Instantiated by Chronicle **per message** from the DI container (§3.9). All five
dependencies are transient or singleton-per-container; none is scoped to a saga. The instance is
discarded after each `HandleAsync`; only `Data` persists between messages.

**Invariants & enforcement.**
- **Exactly one `ISagaStartAction`.** `MakeOrder` is it. Any other message arriving for an unknown
  saga id is dropped (§3.9). Enforced by Chronicle's interface model, not by this code.
- Five `ISagaAction<T>` implementations means five `HandleAsync` **and** five `CompensateAsync`
  methods — the interface forces the pair, which is why the four no-op compensations exist
  (§3.16). **The compiler forces the method to exist but cannot force it to do anything.**
- The action set (5) is smaller than the handler set (6) (§3.8). `ResourceReserved` has no action.
  **Fails silently** (§3.25).
- The class is stateless apart from `Data`; every field is `readonly` and assigned in the
  constructor.

**Extension procedure.** Adding a step means: new `ISagaAction<T>`, new `ResolveId` arm (§3.11), new
`HandleAsync`, new `CompensateAsync`, new field in `AIMakingOrderData` if the step produces data, new
subscription (§3.4) and new handler interface (§3.8). Seven coordinated edits.

**Failure modes.** Because the instance is per-message, anything the constructor does runs on every
message — see §3.18 for why that matters here.

---

### 3.11 `ResolveId` — the saga correlation key

**Definition.** The function that maps an arbitrary message onto the saga instance it belongs to.
This is the mechanism that makes five different message types converge on one workflow.

**Representation & storage.** `Sagas/AIOrderMakingSaga.cs:45-54`:

```csharp
public override SagaId ResolveId(object message, ISagaContext context)
    => message switch
    {
        MakeOrder m              => (SagaId) m.OrderId.ToString(),   // :48
        OrderCreated m           => (SagaId) m.OrderId.ToString(),   // :49
        ParcelAddedToOrder m     => (SagaId) m.OrderId.ToString(),   // :50
        VehicleAssignedToOrder m => (SagaId) m.OrderId.ToString(),   // :51
        OrderApproved m          => m.OrderId.ToString(),            // :52  ← no cast, implicit
        _                        => base.ResolveId(message, context) // :53
    };
```

**The saga id is the order id**, stringified. `SagaId` is a Chronicle value type with an implicit
conversion from `string` `[chronicle]`, which is why `:52` compiles without the explicit cast the
other four arms use — a cosmetic inconsistency, not a behavioural one.

**Lifecycle.** Called by the coordinator before every action, on a freshly constructed saga instance.
Pure; touches no state.

**Invariants & enforcement.**
- **`Guid.ToString()` is the canonical form.** `Guid.ToString()` produces lowercase "D" format
  (`xxxxxxxx-xxxx-…`) on every platform `[framework]`, and every arm uses the same call, so ids are
  consistent. If any future arm used `ToString("N")` or an upper-cased value, that message would
  resolve to a **different saga id** and be dropped as unknown. **Fails silently.**
- The `_ =>` arm delegates to `base.ResolveId`, whose default is derived from the message type
  `[chronicle]`. `ResourceReserved` falls into this arm — and it has no `OrderId` at all
  (`Events/External/ResourceReserved.cs:10-11`: `ResourceId`, `DateTime`). Even if an
  `ISagaAction<ResourceReserved>` were added, **it could not be correlated to the order** without a
  lookup, because the event does not carry the order id. See §3.25 and Q4.
- Order id uniqueness across sagas is assumed, not enforced. Re-posting `MakeOrder` with an
  already-used `OrderId` re-enters `ISagaStartAction` for an existing saga id; Chronicle's behaviour
  there (create-new vs. reject vs. reuse) is `[chronicle]`-dependent (A7, §8.1).

**Extension procedure.** Every new message type needs an arm here. Forgetting it sends the message to
`base.ResolveId` and, almost certainly, to a saga id that does not exist. **Fails silently** — the
single most likely mistake when extending this class.

**Failure modes.** The correlation key is a business identifier chosen by the caller (§3.5), not an
opaque workflow id. Two concurrent sagas cannot exist for one order — which is correct — but it also
means a caller who supplies a colliding `orderId` can interfere with an in-flight workflow.

---

### 3.12 `AIMakingOrderData` — saga state

**Definition.** The seven fields that constitute everything the workflow remembers.

**Representation & storage.** `Sagas/AIMakingOrderData.cs:7-17`:

| Field | Type | Written at | Read at |
| --- | --- | --- | --- |
| `OrderId` | `Guid` | `Saga:60` | `:63`, `:74`, `:104` |
| `CustomerId` | `Guid` | `Saga:61` | `:74`, `:113` |
| `VehicleId` | `Guid` | `Saga:94` | `:98`, `:104`, `:113` |
| `ReservationDate` | `DateTime` | `Saga:99` | `:101`, `:104`, `:114` |
| `ReservationPriority` | `int` | `Saga:100` | `:114` |
| `ParcelIds` | `List<Guid>` (initialised) | `Saga:59` | `:73`, predicate |
| `AddedParcelIds` | `List<Guid>` (initialised) | `Saga:86` | predicate |
| `AllPackagesAddedToOrder` | `bool` (computed) | — | `Saga:87` |

All setters are public; both lists are initialised inline, so `Data` is usable immediately after
construction without a null check — which is why `Data.ParcelIds.Add(...)` at `Saga:59` is safe on a
brand-new saga.

**Lifecycle.** Created by Chronicle when the start action fires; mutated in place by each
`HandleAsync`; persisted after each action into the in-memory store (§3.14); discarded on
`CompleteAsync()` or process exit.

**Invariants & enforcement.**
- `VehicleId`, `ReservationDate` and `ReservationPriority` are meaningless until
  `HandleAsync(ParcelAddedToOrder)` has run past `:87`. Nothing marks them as unset —
  `Guid.Empty`/`DateTime.MinValue`/`0` are indistinguishable from real values. If
  `HandleAsync(VehicleAssignedToOrder)` somehow ran first, it would publish `ReserveResource` with
  `ResourceId = Guid.Empty` and `DateTime = 0001-01-01`. **Fails silently here, loudly in
  `availability-service`** (`ResourceNotFoundException`,
  `hianshul100_Pacco.Services.Availability/.../Commands/Handlers/ReserveResourceHandler.cs:35-39`).
- `ReservationDate` is a bare `DateTime` with **no `Kind` discipline**. It originates either from
  `DateTime.UtcNow.AddDays(1)` (`ResourceReservationsService.cs:34` — `Kind=Utc`) or from a
  deserialised upstream value (`Kind` depends on the JSON, typically `Utc` or `Unspecified`). It is
  then serialised into `AssignVehicleToOrder` and `ReserveResource` and compared upstream against
  stored reservation dates. **Fails silently** if a `Kind` mismatch shifts the date by the local
  offset — see Q5.
- Nothing prevents `AddedParcelIds` from receiving a parcel id that is not in `ParcelIds`; see §3.13.

**Extension procedure.** Add a property; it is picked up automatically because the whole object is
persisted as a unit. **But see §5.2**: with the in-memory store there is no schema-migration problem
today, and adding a persistent store later creates one immediately.

**Failure modes.** The type has no `SagaId`, no timestamps and no step marker, so an operator
inspecting a stuck saga cannot tell *which step* it is stuck on except by inferring from which
fields are still zero.

---

### 3.13 `AllPackagesAddedToOrder` — the completion predicate

**Definition.** The gate that decides whether all parcels have been added and the workflow may move
on to vehicle selection. **It is inverted, and it is the clearest latent defect in the component.**

**Representation & storage.** `Sagas/AIMakingOrderData.cs:16`:

```csharp
public bool AllPackagesAddedToOrder => AddedParcelIds.Any() && AddedParcelIds.All(ParcelIds.Contains);
```

Read once, at `Sagas/AIOrderMakingSaga.cs:87`:

```csharp
Data.AddedParcelIds.Add(message.ParcelId);   // :86
if (!Data.AllPackagesAddedToOrder) { return; }  // :87-90
```

**Lifecycle.** Evaluated on every `ParcelAddedToOrder` event, immediately after appending the newly
confirmed parcel.

**Invariants & enforcement.**
- The predicate asks *"is every parcel I have confirmed a member of the set I requested?"* The
  question it must ask is *"has every parcel I requested been confirmed?"* — i.e.
  `ParcelIds.All(AddedParcelIds.Contains)`. **The operands are the wrong way round.**
- With one parcel (the only shape the entry point can produce, §3.5) the two formulations coincide:
  after the single `ParcelAddedToOrder`, `AddedParcelIds` has one element which is indeed in
  `ParcelIds`, and the requested set is also fully covered. **The bug is invisible today.**
- With *n* parcels it fires on the **first** event: `AddedParcelIds = {p1}`, all of which are in
  `ParcelIds` ⇒ `true`. The saga selects a vehicle and publishes `AssignVehicleToOrder` while
  parcels 2..n are still being added. It then continues to receive the remaining
  `ParcelAddedToOrder` events, each of which **re-runs the whole vehicle-selection block**
  (`:92-109`) and publishes another `AssignVehicleToOrder` and, downstream, another
  `ReserveResource`. **Fails silently, and duplicates real reservations in `availability-service`.**
- `AddedParcelIds.Any()` is the only thing preventing a vacuous `true` on an empty list — a correct
  guard for the wrong predicate.

**Extension procedure.** The fix is one line:

```csharp
public bool AllPackagesAddedToOrder => ParcelIds.Any() && ParcelIds.All(AddedParcelIds.Contains);
```

It must land **together with** multi-parcel support in `MakeOrder` (§3.5), because until then it
changes nothing. Add an idempotency guard at `:87` as well — e.g. `if (Data.VehicleId != Guid.Empty)
return;` — so that a redelivered `ParcelAddedToOrder` cannot re-run the selection block.

**Failure modes.** In its current single-parcel shape the predicate is also the only thing standing
between a **redelivered** `ParcelAddedToOrder` and a duplicate vehicle assignment: a redelivery adds
the same id again, the predicate is still `true`, and the block at `:92-109` runs a second time.
Redelivery is a normal AMQP occurrence (§3.26), so this is reachable **today**, with one parcel.
This is the most important operational risk in the component.

---

### 3.14 Saga persistence: in-memory only

**Definition.** Where `AIMakingOrderData` lives between messages.

**Representation & storage.** `Extensions.cs:43` is the whole story:

```csharp
builder.Services.AddChronicle();
```

There is **no** `AddChronicle(o => o.UseMongo(...))`, no Chronicle persistence package in
`Pacco.Services.OrderMaker.csproj` (the 19 `PackageReference` entries at `:9-27` are Chronicle_
itself plus 18 Convey packages), and no MongoDB/EF/Redis saga configuration in any `appsettings*`
file. Chronicle's default `ISagaStateRepository` / `ISagaLog` are in-process dictionaries
`[chronicle]` (A8, §8.1).

Note the near-miss: `AddRedis()` **is** registered (`Extensions.cs:37`) with an instance prefix
`"ordermaker:"` (`appsettings.json:133`), so a durable store is *connected* but not wired to
Chronicle.

**Lifecycle.** State is created on `MakeOrder`, updated after each action, and **destroyed when the
process exits**. There is no snapshot, no recovery and no eviction — a saga that never receives
`OrderApproved` occupies memory until restart.

**Invariants & enforcement.**
- **A restart loses every in-flight workflow.** Meanwhile the queues are durable and non-auto-delete
  (`appsettings.json:119-122`), so the events that would have advanced those sagas are redelivered
  after the restart and resolve to saga ids that no longer exist — dropped by Chronicle with no log
  line from this code (§3.9). **Fails silently, and permanently: the order is left half-built in
  `orders-service` with no compensation.**
- **Horizontal scaling is unsafe.** Two replicas of `ordermaker-service` share the durable queues as
  competing consumers, so `OrderCreated` may land on replica A while `ParcelAddedToOrder` lands on
  replica B, which has no state for that saga id. `compose/services.yml:78-85` declares a single
  replica, so this is latent rather than active. **Fails silently.**
- There is **no timeout**: a saga whose `OrderApproved` never arrives stays `Pending` forever.
  Chronicle offers no built-in timeout in this configuration and none is coded here.

**Extension procedure.** Add a Chronicle persistence package (the ecosystem provides Mongo and Redis
providers `[chronicle]`), configure it in the `AddChronicle` call at `:43`, and add its connection
settings to all four `appsettings*` files. Then — and only then — §5.2's schema-evolution rules
begin to apply. Adding persistence **without** adding an idempotency guard (§3.13) makes redelivery
worse, not better, because redelivered messages will now find live state.

**Failure modes.** This is baseline gap **G3** and the reason `patterns/orchestration/saga-process-manager.md`
describes the pattern as "demonstrated, not production-ready". A deployment that restarts under load
silently abandons every order mid-flight; the customer sees an order that was created, has a parcel,
and will never be approved.

---

### 3.15 Saga lifecycle, states and `CompleteAsync`

**Definition.** The state machine Chronicle maintains around `AIMakingOrderData`, and the one place
this code drives it.

**Representation & storage.** Chronicle's `SagaStates` enum is referenced three ways in this
repository, all as header values (§3.17): `SagaStates.Pending` (`AIOrderMakingSaga.cs:67,78,108,118`),
`SagaStates.Completed` (`:128`) and `SagaStates.Rejected` (`:145`). The only transition the code
actually performs is:

```csharp
await CompleteAsync();   // AIOrderMakingSaga.cs:131, in HandleAsync(OrderApproved)
```

`RejectAsync()` is **never called**. Nothing in this repository moves a saga to `Rejected`; the
`Rejected` header value at `:145` is stamped on the compensating `CancelOrder` while the saga itself
remains `Pending`.

**Lifecycle.**

```
   (no saga)
       │  MakeOrder                       ISagaStartAction
       ▼
   Pending ──OrderCreated──► Pending ──ParcelAddedToOrder──► Pending
       │                                        │  (predicate §3.13)
       │                                        ▼
       │                        Pending ──VehicleAssignedToOrder──► Pending
       │                                        │
       │                                        ▼ (ResourceReserved → orders-service, §1.3)
       │                        Pending ──OrderApproved──► CompleteAsync() ──► Completed
       │
       └── any exception ──► Chronicle compensation walk (§3.16) ──► saga stays Pending
```

**Invariants & enforcement.**
- **There is exactly one terminal transition and it is on the happy path.** A saga that fails, times
  out, or is compensated never leaves `Pending`. **Fails silently** — an operator cannot distinguish
  "in progress" from "dead" by state.
- `CompleteAsync()` is awaited *after* `MakeOrderCompleted` is published (`:124` then `:131`). If the
  publish throws, the saga is not completed and — because compensation for `OrderApproved` is a no-op
  (§3.16) — nothing happens at all. The order is approved upstream; only this service's bookkeeping
  is wrong. **Fails silently.**
- The `Saga` header values are *strings derived from* `SagaStates`, not the state itself. They can
  and do disagree with the saga's real state (the `Rejected` case at `:145`). **Fails silently.**

**Extension procedure.** To introduce failure handling, call `RejectAsync()` in a `catch` around the
body of each `HandleAsync`, publish `MakeOrderRejected` (§3.32 — the type already exists and is
unused), and stamp `SagaStates.Rejected`. Chronicle's compensation walk will still run; the two
mechanisms are independent.

**Failure modes.** Because `Rejected` is unreachable, the `MakeOrderRejected` contract (§3.32) has no
producer, and the platform has no way to learn that an order-making workflow failed.

---

### 3.16 Compensation model

**Definition.** The five `CompensateAsync` methods Chronicle invokes, in reverse order, when an
action throws.

**Representation & storage.** `Sagas/AIOrderMakingSaga.cs:134-152`:

| Method | Line | Body |
| --- | --- | --- |
| `CompensateAsync(MakeOrder)` | 134-135 | `Task.CompletedTask` |
| `CompensateAsync(OrderCreated)` | 137-138 | `Task.CompletedTask` |
| `CompensateAsync(ParcelAddedToOrder)` | 140-146 | publishes `CancelOrder(message.OrderId, "Because I'm saga")` with header `Saga=Rejected` |
| `CompensateAsync(VehicleAssignedToOrder)` | 148-149 | `Task.CompletedTask` |
| `CompensateAsync(OrderApproved)` | 151-152 | `Task.CompletedTask` |

**One of five compensations does anything.** It is also the one placed on the step most likely to
throw, since `HandleAsync(ParcelAddedToOrder)` is the only action that makes synchronous HTTP calls
(`:93`, `:98`).

**Lifecycle.** Invoked by Chronicle during the compensation walk (§3.9). The saga instance is
reconstructed for the walk, so §3.18's constructor side effect applies here too.

**Invariants & enforcement.**
- **The most dangerous gap is `CompensateAsync(VehicleAssignedToOrder)`.** That action publishes
  `ReserveResource` (`:113`), which causes `availability-service` to create a real reservation. Its
  compensation does nothing, so **a failure after the reservation leaves the vehicle booked
  indefinitely.** `availability-service` exposes the inverse command —
  `ReleaseResourceReservation`, routed at
  `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/Program.cs:45` — so
  the compensating action exists in the platform and is simply not used here. **Fails silently:
  capacity leaks with no error anywhere.**
- `CancelOrder`'s reason is the hard-coded string `"Because I'm saga"` (`:141`). It is stored on the
  order and surfaced to whoever reads it. **Fails silently** as a user-experience defect.
- `CompensateAsync(ParcelAddedToOrder)` uses `message.OrderId`, not `Data.OrderId` — the only place
  in the class that prefers the message over saga state. They are equal by construction (§3.11), so
  this is stylistic.
- **Compensation is not itself compensated.** If the `CancelOrder` publish throws, the walk fails and
  nothing retries.
- **Publishing inside compensation cannot be rolled back.** There is no outbox (§3.3), so a
  `CancelOrder` published for a saga that then succeeds on retry would cancel a live order.

**Extension procedure.** For each action that causes an external effect, write the inverse:
`CompensateAsync(VehicleAssignedToOrder)` should publish `ReleaseResourceReservation(Data.VehicleId,
Data.ReservationDate)` — note that this requires adding that command contract to
`Commands/External/` with `[Message("availability")]`. `CompensateAsync(OrderCreated)` should
publish `CancelOrder` (or `DeleteOrder`, which `orders-service` also subscribes to:
`hianshul100_Pacco.Services.Orders/.../Infrastructure/Extensions.cs:96`). Take the cancellation
reason from the exception rather than a literal.

**Failure modes.** The net effect today: **a saga that fails after step 6 leaves an order created,
parcelled, vehicle-assigned and a resource reserved, with no cleanup and no notification.** The only
recovery is manual.

---

### 3.17 The `Saga` header

**Definition.** A single AMQP header, `"Saga"`, stamped on every message this component publishes,
carrying the saga's nominal state as a string.

**Representation & storage.** Declared once (`Sagas/AIOrderMakingSaga.cs:23`,
`private const string SagaHeader = "Saga"`) and used in all six publishes:

| Publish | Line | Header value |
| --- | --- | --- |
| `CreateOrder` | 63-68 | `Pending` |
| `AddParcelToOrder` | 74-79 | `Pending` |
| `AssignVehicleToOrder` | 103-109 | `Pending` |
| `ReserveResource` | 113-119 | `Pending` |
| `MakeOrderCompleted` | 124-129 | `Completed` |
| `CancelOrder` (compensation) | 141-146 | `Rejected` |

**Lifecycle — and this is the interesting part.** The header does not stop at the first hop.
`orders-service` reads it back off the incoming message and forwards it onto the events it publishes:

```csharp
internal static IDictionary<string, object> GetHeadersToForward(this IMessageProperties messageProperties)
{
    const string sagaHeader = "Saga";
    if (messageProperties?.Headers is null || !messageProperties.Headers.TryGetValue(sagaHeader, out var saga))
        return null;
    return saga is null ? null : new Dictionary<string, object> { [sagaHeader] = saga };
}
```
— `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:118-132`.

So a `CreateOrder` stamped `Saga=Pending` produces an `OrderCreated` that is *also* stamped
`Saga=Pending`, and that is how the platform distinguishes a saga-driven flow from an ordinary
choreographed one. **This header is the entire mechanism by which orchestration and choreography
coexist on the same exchange** (§3.20, Q3).

**Its real consumer is `operations-service`.** That service parses the same literal into an
`OperationState`:

```csharp
public static OperationState? GetSagaState(this IMessageProperties messageProperties)
{
    const string sagaHeader = "Saga";
    if (messageProperties?.Headers is null || !messageProperties.Headers.TryGetValue(sagaHeader, out var saga))
        return null;
    …
}
```
— `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Handlers/Extensions.cs:9-14`,
used at `Handlers/GenericCommandHandler.cs:40`, `GenericEventHandler.cs:38` and
`GenericRejectedEventHandler.cs:40`. Without the header a command defaults to `Pending` and an event
to `Completed`; **with** it, an intermediate command in a saga is correctly shown as still pending
rather than done. That is the header's purpose: it stops `operations-service` from telling a user
their request finished when only the first of six steps has. See §3.31 for why this service's own
messages never reach that code path.

**Invariants & enforcement.**
- The literal `"Saga"` is duplicated in two repositories with no shared constant. Changing it here
  silently severs the forwarding in `orders-service`. **Fails silently** — the flow still works,
  because nothing in *this* service reads the header back.
- **This service never reads its own header.** `AIOrderMakingSaga` inspects no message properties;
  correlation is by `ResolveId` alone (§3.11). The header is therefore write-only from this side:
  it exists for other services' benefit.
- The header does **not** gate anything in `availability-service`, which has no equivalent
  forwarding helper — so `ReserveResource` carries `Saga=Pending` and it is dropped at that boundary.
  Consequently `ResourceReserved` arrives back at this service **without** the header, and
  `orders-service` sees an un-stamped `ResourceReserved` when it approves the order — meaning the
  final `OrderApproved` may not be stamped either. **Fails silently**; the header chain breaks at
  the availability hop. Whether `OrderApproved` carries it depends on `orders-service`'s
  `IEventMapper`/`IMessageBroker` plumbing (A9, §8.1).

**Extension procedure.** If you add a service to the saga, give it the same `GetHeadersToForward`
helper, or the chain breaks there too. Better: promote the header name and the forwarding helper into
a shared package.

**Failure modes.** The header is untyped (`object`) and its value is a `ToString()` of an enum. A
consumer that string-compares against `"pending"` rather than `"Pending"` fails silently.

---

### 3.18 `ICorrelationContextAccessor` and the constructor overwrite

**Definition.** The message-context object attached to every published message — and the three lines
that make it empty.

**Representation & storage.** `Sagas/AIOrderMakingSaga.cs:39-42`, at the end of the constructor:

```csharp
_accessor.CorrelationContext = new CorrelationContext
{
    User = new CorrelationContext.UserContext()
};
```

That value is then passed as `messageContext:` on all six publishes (`:64, :75, :105, :115, :125,
:142`).

**Lifecycle.** The saga is constructed **per message** (§3.9, §3.10), so this assignment runs on
every message the service handles. Whatever Convey's `UseConvey()` middleware
(`Extensions.cs:55`) or the RabbitMQ subscriber had placed in the accessor is **discarded and
replaced** with a blank object.

**Invariants & enforcement.**
- The replacement object has `CorrelationId = null`, `SpanContext = null`, `TraceId = null`,
  `ResourceId = null`, `ConnectionId = null`, `Name = null`, `CreatedAt = default` and a `User` whose
  `Id` is `null` and `IsAuthenticated` is `false` (`CorrelationContext.cs:8-23`). **Every message
  this service publishes therefore carries an empty `message_context` header**
  (`appsettings.json:125-128`).
- **This has a concrete, checkable downstream effect.** `availability-service`'s
  `ReserveResourceHandler` guards on identity:

  ```csharp
  var identity = _appContext.Identity;
  if (identity.IsAuthenticated && identity.Id != command.CustomerId && !identity.IsAdmin)
      throw new UnauthorizedResourceAccessException(command.ResourceId, identity.Id);
  ```
  — `hianshul100_Pacco.Services.Availability/.../Commands/Handlers/ReserveResourceHandler.cs:29-33`.

  Because `IsAuthenticated` is `false`, the guard is **skipped entirely**. The saga's
  `ReserveResource` is accepted regardless of who the customer is. The empty context is thus not
  merely a telemetry loss — it is an **authorization bypass by omission**. **Fails silently, and
  permissively.** This is the finding that most needs a decision (B2, §8.2).
- Correlation ids and span contexts do not propagate. Combined with the absence of Jaeger (§3.36),
  **a saga-driven order is untraceable end to end.**

> **Reconciliation with an earlier artifact.** `../patterns/orchestration/saga-process-manager.md`
> states that the saga "propagates the correlation context onto every published message". That is
> true of the *call shape* — `messageContext:` is passed on all six publishes — but **not of the
> content**, because the constructor overwrites the accessor first. The pattern document should be
> read as describing the intent; this section describes the behaviour. Recorded as Q6 in §8.3.

**Extension procedure.** Delete `:39-42`. The accessor will then hold whatever the inbound pipeline
put there (an HTTP-derived context for `MakeOrder`, an AMQP-derived one for the events). Expect two
consequences, both desirable and both breaking: `availability-service`'s identity guard starts
firing, so the saga must run under a principal permitted to reserve on the customer's behalf (an
admin token, or the customer's own); and correlation ids start appearing in Seq. Do not delete these
lines without deciding the identity question first.

**Failure modes.** If the accessor is null at construction — it cannot be, it is a registered
dependency — the saga would fail to construct. The real failure mode is the opposite: everything
works, quietly, with no identity.

---

### 3.19 Local `CorrelationContext` type

**Definition.** A DTO defining the shape of the `message_context` header for this service.

**Representation & storage.** `CorrelationContext.cs:6-24` — eight scalar properties plus a nested
`UserContext` (`Id`, `IsAuthenticated`, `Role`, `Claims`). It is a **hand-copied duplicate** of the
same type in every other Pacco service (compare
`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/CorrelationContext.cs`).

**Lifecycle.** Instantiated once per saga construction (§3.18), serialised into the AMQP header by
Convey `[convey]`, deserialised by consuming services into *their* copy of the type.

**Invariants & enforcement.**
- **The contract is the JSON shape, not the C# type.** Property names must match across all
  services, case-insensitively. Adding a property here is safe; renaming one breaks correlation for
  that field only, **silently**.
- This service **only writes** the type. It never deserialises an inbound `message_context` — there
  is no `GetCorrelationContext` helper as in `orders-service`
  (`.../Infrastructure/Extensions.cs:113-116`). One more reason inbound context is lost (§3.18).
- `Claims` is `IDictionary<string,string>` here; if another service declares it differently, the
  round-trip degrades silently.

**Extension procedure.** Any change must be replicated across all thirteen services or accepted as a
one-way field. This is the argument for a shared contracts package (Q7).

**Failure modes.** Because the object published is always blank (§3.18), no field of this type is
exercised in practice. It is a correct implementation of a mechanism that is disabled.

---

### 3.20 Outbound command contracts & exchange routing

**Definition.** The six `Commands/External/*.cs` types — this service's outbound vocabulary — and
the exchanges they are routed to.

**Representation & storage.**

| Type | File | `[Message]` exchange | Published at | Consumed by |
| --- | --- | --- | --- | --- |
| `CreateOrder` | `Commands/External/CreateOrder.cs:7` | `orders` | `Saga:63` | `orders-service` (`SubscribeCommand<CreateOrder>`, Extensions.cs:95) |
| `AddParcelToOrder` | `Commands/External/AddParcelToOrder.cs:7` | `orders` | `Saga:74` | `orders-service` (Extensions.cs:98) |
| `AssignVehicleToOrder` | `Commands/External/AssignVehicleToOrder.cs:7` | `orders` | `Saga:103` | `orders-service` (Extensions.cs:100) |
| `CancelOrder` | `Commands/External/CancelOrder.cs:7` | `orders` | `Saga:141` (compensation) | `orders-service` (Extensions.cs:96) |
| `ReserveResource` | `Commands/External/ReserveResource.cs:7` | `availability` | `Saga:113` | `availability-service` |
| `ApproveOrder` | `Commands/External/ApproveOrder.cs:7` | `orders` | **never** | — (§3.23) |

All are `ICommand` with get-only properties and an explicit constructor. The `[Message("…")]`
attribute names the **exchange**; the routing key is the type name in snake_case
(`appsettings.json:93`) `[convey]`.

**Lifecycle.** Constructed inside a saga action, handed to `IBusPublisher.PublishAsync` with a
message context (§3.18) and a `Saga` header (§3.17), serialised by Newtonsoft and published. No
persistence, no outbox, no confirm handling in this code.

**Invariants & enforcement.**
- **The wire contract is the JSON property set, and it is duplicated by hand.** These types are
  local re-declarations of types that live in `orders-service` and `availability-service` and are
  marked `[Contract]` there (e.g.
  `hianshul100_Pacco.Services.Orders/.../Application/Commands/CreateOrder.cs:6`). That attribute
  feeds `UsePublicContracts<ContractAttribute>()`
  (`hianshul100_Pacco.Services.Orders/.../Infrastructure/Extensions.cs:91`), which publishes the
  expected shapes at a `/_contracts` endpoint — **a drift-detection facility this service does not
  consume.** Two of the six have already drifted (§3.21, §3.22). **Fails silently.**
- `CreateOrder` here repeats the empty-guid guard of its upstream twin
  (`Commands/External/CreateOrder.cs:15` vs. Orders' `:14`) — harmless, since `Data.OrderId` is
  always non-empty by then (§3.5), but it means a bug in `ResolveId` could produce an order whose id
  differs from the saga id.
- **Publishing is fire-and-forget.** `PublishAsync` returns when the message is handed to the
  RabbitMQ client. There is no publisher confirm, no retry and no outbox (§3.3). A broker outage
  loses the step. **Fails loudly for `MakeOrder`** (the exception reaches the HTTP caller, §3.7) and
  **silently for every other step** (the exception reaches a consumer thread).
- `Task.WhenAll` at `Saga:81` publishes all parcel commands concurrently; ordering among them is not
  guaranteed and does not matter.

**Extension procedure.** New outbound command: create the type under `Commands/External/` with the
correct `[Message]` exchange, verify the property names against the consuming service's `[Contract]`
type **by reading that file**, and publish it with the same `messageContext`/`headers` pair used by
its five siblings. Copying an existing publish block verbatim is the safest approach — the six blocks
are already identical apart from the payload.

**Failure modes.** The exchange name is a string in an attribute; a typo creates a new exchange
(`exchange.declare: true`, `appsettings.json:112`) and the message vanishes into it with **no
error**. This is the highest-frequency silent failure available when extending the component.

---

### 3.21 `AddParcelToOrder` — the dropped `customerId`

**Definition.** A concrete, verified contract drift between this service's outbound command and the
consumer's declared contract.

**Representation & storage.** Local (`Commands/External/AddParcelToOrder.cs:10-19`):

```csharp
public Guid OrderId { get; }
public Guid ParcelId { get; }
public Guid CustomerId { get; }
public AddParcelToOrder(Guid orderId, Guid parcelId, Guid customerId)
```

Consumer (`hianshul100_Pacco.Services.Orders/.../Application/Commands/AddParcelToOrder.cs:6-16`):

```csharp
[Contract]
public class AddParcelToOrder : ICommand
{
    public Guid OrderId { get; }
    public Guid ParcelId { get; }
    public AddParcelToOrder(Guid orderId, Guid parcelId)
}
```

The saga populates all three (`Saga:74`: `new AddParcelToOrder(Data.OrderId, id, Data.CustomerId)`).

**Lifecycle.** The extra `customerId` is serialised onto the wire and **discarded during
deserialisation** by `orders-service`, because Newtonsoft ignores JSON properties with no
constructor parameter or settable property by default `[newtonsoft]`.

**Invariants & enforcement.**
- **The message is wire-compatible and the drift is invisible.** No error, no warning, no log line.
  **Fails silently.**
- The consequence is not currently harmful: `orders-service` resolves the customer from the order
  aggregate it already holds. But it means the saga *believes* it is passing an authorization-
  relevant field that is being thrown away — precisely the belief that makes §3.18's bypass easy to
  overlook.
- Direction matters: **extra** fields are safe, **missing** fields deserialise to `default`. The
  same drift in the opposite direction would silently send `Guid.Empty`.

**Extension procedure.** Either drop `CustomerId` from the local type and from the `Saga:74` call, or
add it to the `[Contract]` type in `orders-service`. Do not leave the asymmetry: it is the kind of
detail a later change will read as intent.

**Failure modes.** If `orders-service` ever adds a `CustomerId` **constructor parameter** with a
different meaning, this message will suddenly start populating it. A silent drift becomes a silent
behaviour change.

---

### 3.22 `ReserveResource` — positional drift

**Definition.** The same class of drift as §3.21, but on constructor **parameter order** rather than
field set — and the one where the compiler is least help.

**Representation & storage.** Local (`Commands/External/ReserveResource.cs:10-21`):

```csharp
public Guid ResourceId { get; }
public Guid CustomerId { get; }
public DateTime DateTime { get; }
public int Priority { get; }
public ReserveResource(Guid resourceId, Guid customerId, DateTime dateTime, int priority)
```

Consumer (`hianshul100_Pacco.Services.Availability/.../Application/Commands/ReserveResource.cs:6-15`):

```csharp
[Contract]
public class ReserveResource : ICommand
{
    public Guid ResourceId { get; }
    public DateTime DateTime { get; }
    public int Priority { get; }
    public Guid CustomerId { get; }
    public ReserveResource(Guid resourceId, DateTime dateTime, int priority, Guid customerId)
}
```

Call site: `Saga:113-114` — `new ReserveResource(Data.VehicleId, Data.CustomerId,
Data.ReservationDate, Data.ReservationPriority)`.

**Lifecycle.** Serialised **by property name**, so the wire JSON has the same four keys in both
declarations and the message deserialises correctly. **The drift is real but currently harmless.**

**Invariants & enforcement.**
- **JSON is name-based; C# constructors are position-based.** The two orderings coexist safely only
  while the property *names* and *types* match. `CustomerId` and `ResourceId` are both `Guid`: if
  anyone ever "fixes" the local constructor to match the remote signature without also fixing the
  call site at `:113`, the arguments will bind positionally to the wrong properties and **a customer
  id will be sent as a resource id**. It compiles. It runs. It reserves the wrong thing — or, more
  likely, throws `ResourceNotFoundException` upstream. **The trap fails silently at compile time and
  loudly at runtime, in another service.**
- This is the strongest argument in the repository for shared contract packages (Q7).

**Extension procedure.** If you touch this type, align it field-for-field *and* position-for-position
with the `availability-service` declaration, and update `Saga:113` in the same commit. Prefer named
arguments at the call site (`new ReserveResource(resourceId: …, dateTime: …, priority: …,
customerId: …)`) — that makes the drift a compile error instead of a runtime one.

**Failure modes.** As above. Note also that `ReserveResource` carries `Saga=Pending`
(`Saga:116-119`) but `availability-service` has no header-forwarding helper (§3.17), so the saga
marker dies at this hop.

---

### 3.23 `ApproveOrder` — the command nobody sends

**Definition.** A fully-formed outbound contract with no publisher.

**Representation & storage.** `Commands/External/ApproveOrder.cs` — 17 lines, `[Message("orders")]`,
`OrderId`, matching `orders-service`'s `[Contract] ApproveOrder`
(`hianshul100_Pacco.Services.Orders/.../Application/Commands/ApproveOrder.cs`) exactly. Verified by
search: the identifier `ApproveOrder` appears in this repository **only** in that file — it is not
constructed, published or referenced anywhere in `Sagas/`, `Handlers/` or `Extensions.cs`.

**Lifecycle.** None. The type is compiled and shipped and never instantiated.

**Invariants & enforcement.**
- **It marks a design the code did not take.** The saga could approve the order itself after
  `ResourceReserved`; instead it relies on `orders-service` to approve autonomously in its
  `ResourceReservedHandler` (§1.3). That is choreography inside an orchestration, and it is why
  `ISagaAction<ResourceReserved>` is missing (§3.25): the saga does not need `ResourceReserved`
  *because someone else acts on it*.
- Its presence implies a capability the service does not have. **Fails silently** as documentation
  drift.

**Extension procedure.** Two coherent options, and the choice is a real architectural decision (Q4):
either **delete the file**, accepting that approval is choreographed; or **complete the
orchestration** — add `ISagaAction<ResourceReserved>`, publish `ApproveOrder` from it, and remove the
autonomous approval from `orders-service`'s `ResourceReservedHandler`. The second requires a change
in a repository this component does not own, and would break the non-saga order flow, which relies
on that same handler.

**Failure modes.** None at runtime. The risk is a future engineer wiring up the publish without
removing `orders-service`'s autonomous approval, producing **double approval** of every order.

---

### 3.24 Inbound event contracts & `SubscribeEvent` wiring

**Definition.** The five `Events/External/*.cs` types this service listens for, and their local
re-declarations.

**Representation & storage.**

| Local type | `[Message]` | Fields | Upstream declaration |
| --- | --- | --- | --- |
| `OrderCreated` | `orders` | `OrderId` | `Orders/.../Events/OrderCreated.cs` `[Contract]` — **identical** |
| `ParcelAddedToOrder` | `orders` | `OrderId`, `ParcelId` | `Orders/.../Events/ParcelAddedToOrder.cs` `[Contract]` — **identical** |
| `VehicleAssignedToOrder` | `orders` | `OrderId`, `VehicleId` | `Orders/.../Events/VehicleAssignedToOrder.cs` — identical, but **not** marked `[Contract]` upstream |
| `OrderApproved` | `orders` | `OrderId` | `Orders/.../Events/OrderApproved.cs` `[Contract]` — **identical** |
| `ResourceReserved` | `availability` | `ResourceId`, `DateTime` | `Availability/.../Events/ResourceReserved.cs` `[Contract]` — **identical** |

Verified field-by-field against the upstream files. Unlike the outbound commands (§3.21, §3.22), the
inbound events have **not** drifted.

**Lifecycle.** RabbitMQ delivers → Convey deserialises into the local type → `AIOrderMakingHandler`
(§3.8) → `ISagaCoordinator` (§3.9) → `ResolveId` (§3.11) → saga action.

**Invariants & enforcement.**
- **Only `OrderId` matters.** Four of the five events are correlated by `OrderId` and only
  `ParcelAddedToOrder.ParcelId` and `VehicleAssignedToOrder.VehicleId` are otherwise read — and
  `VehicleId` is *not* read: `HandleAsync(VehicleAssignedToOrder)` uses `Data.VehicleId`, not
  `message.VehicleId` (`Saga:113`). **If `orders-service` assigned a different vehicle than the one
  requested, this saga would reserve the one it asked for, not the one that was assigned.** Fails
  silently. Low risk today (Orders assigns what it is told) but a real coupling assumption.
- `VehicleAssignedToOrder` lacks `[Contract]` upstream, so it is **not** advertised at Orders'
  `/_contracts` endpoint and drift in it would be undetectable even by a service that did check.
- A field added upstream is ignored here (Newtonsoft default `[newtonsoft]`); a field removed
  upstream deserialises to `default` here. Both **fail silently**; the second is dangerous because
  `Guid.Empty` would resolve to a nonexistent saga id.

**Extension procedure.** See §3.4 — five coordinated edits. Always copy the upstream declaration
verbatim, including property order, and check whether it carries `[Contract]`.

**Failure modes.** Events arrive on durable queues, so redelivery is expected. **No inbound event is
idempotent in this saga**: `ParcelAddedToOrder` re-runs vehicle selection (§3.13),
`VehicleAssignedToOrder` re-publishes `ReserveResource` (double reservation), `OrderApproved`
re-publishes `MakeOrderCompleted` and calls `CompleteAsync()` twice. Only `OrderCreated` is benign,
and only because `AddParcelToOrder` is idempotent upstream.

---

### 3.25 `ResourceReserved` — subscribed but inert

**Definition.** The one wiring path in this component that goes nowhere.

**Representation & storage.** Three of the four required pieces exist:

| Piece | Present? | Evidence |
| --- | --- | --- |
| Contract type | ✅ | `Events/External/ResourceReserved.cs` |
| AMQP subscription | ✅ | `Extensions.cs:61` |
| `IEventHandler<ResourceReserved>` | ✅ | `Handlers/AIOrderMakingHandler.cs:16,40-41` |
| `ISagaAction<ResourceReserved>` | ❌ | absent from `Sagas/AIOrderMakingSaga.cs:16-21` |

**Lifecycle.** A `ResourceReserved` message is delivered, deserialised, handed to
`_coordinator.ProcessAsync` (`Handlers/AIOrderMakingHandler.cs:41`), and Chronicle finds no action
for the type. It is **discarded with no exception and no log line** `[chronicle]` (A4, §8.1).

**Invariants & enforcement.**
- The queue `ordermaker-service/availability.resource_reserved` is declared, bound and consumed, so
  the message is **acked** and does not accumulate. From the broker's point of view everything is
  healthy. **Fails silently in the strongest sense: the mechanism reports success.**
- Even if the action were added, **it could not be correlated**: `ResourceReserved` carries
  `ResourceId` and `DateTime` and no order id (`Events/External/ResourceReserved.cs:10-11`), so
  `ResolveId` falls to `base.ResolveId` (§3.11). Correlating would require either a reverse lookup
  (`Data.VehicleId` + `Data.ReservationDate` → saga) — which Chronicle's id-based model does not
  offer — or a new field on the upstream event.
- Note that `orders-service` solves exactly this problem with a repository query:
  `_orderRepository.GetAsync(@event.ResourceId, @event.DateTime)`
  (`Orders/.../Events/External/Handlers/ResourceReservedHandler.cs:26`). It can, because it has a
  database. This service cannot, because it has no store (§3.14).

**Extension procedure.** Three options, in increasing cost:
1. **Remove the subscription and the handler interface** (`Extensions.cs:61`,
   `Handlers/AIOrderMakingHandler.cs:16,40-41`). Zero behaviour change; the wiring stops lying.
2. **Add `ISagaAction<ResourceReserved>` and a `ResolveId` arm**, having first added `OrderId` to the
   upstream `ResourceReserved` contract in `availability-service`. This is the step needed before
   `ApproveOrder` (§3.23) could ever be published by this saga.
3. **Give the saga a persistent store keyed on `(VehicleId, ReservationDate)`** — heavy, and
   duplicates what `orders-service` already does.

Option 1 is the honest default until Q4 is decided.

**Failure modes.** None operationally — this is a documentation and intent defect. Its cost is that
it makes the saga *look* like it closes the loop on reservations when it does not; §3.16's missing
compensation is easy to overlook for exactly this reason.

---

### 3.26 RabbitMQ topology & conventions

**Definition.** The broker configuration that determines exchange names, routing keys, queue names
and redelivery behaviour.

**Representation & storage.** `appsettings.json:89-130`:

| Key | Value | Consequence |
| --- | --- | --- |
| `connectionName` | `ordermaker-service` | visible in the RabbitMQ management UI |
| `retries` / `retryInterval` | `3` / `2` | **connection** retries, not message retries |
| `conventionsCasing` | `snakeCase` | `MakeOrderCompleted` → routing key `make_order_completed` |
| `exchange.name` | `ordermaker` | **default** exchange for types with no `[Message]` |
| `exchange.type` | `topic` | routing-key matching |
| `exchange.durable` / `autoDelete` | `true` / `false` | survives broker restart |
| `queue.template` | `ordermaker-service/{{exchange}}.{{message}}` | one queue per (exchange, message) pair |
| `queue.durable` / `exclusive` / `autoDelete` | `true` / `false` / `false` | **messages persist while the service is down** |
| `context.enabled` / `context.header` | `true` / `message_context` | where §3.18's blank object lands |
| `spanContextHeader` | `span_context` | read by other services; never populated here |
| `username` / `password` | `guest` / `guest` | default credentials, in source control |

`appsettings.docker.json:59-61` overrides only `hostnames` to `["rabbitmq"]`; everything else is
inherited, **including the `guest`/`guest` credentials**.

**Lifecycle.** Exchanges and queues are declared at startup (`declare: true` on both) and are
durable. Consumption begins when `UseRabbitMq()` runs (`Extensions.cs:57`).

**Invariants & enforcement.**
- **Durable queues + in-memory sagas is the core contradiction of this component** (§3.14). The
  broker guarantees delivery to a service that cannot remember what to do with it.
- The `ordermaker` exchange is used **only** for outbound `MakeOrderCompleted` and (potentially)
  `MakeOrderRejected` — every other publish carries an explicit `[Message]` naming a foreign
  exchange. Nothing is consumed from `ordermaker` by this service.
- `conventionsCasing: snakeCase` must match every peer's setting or routing keys will not bind.
  All Pacco services set it identically. **Fails silently** if one diverges: the queue exists, binds
  to a key nobody publishes, and stays empty forever.
- **There is no dead-letter configuration and no `requeue` policy** in any `appsettings*` file. What
  happens to a message whose handler throws is entirely Convey's default `[convey]` (A6, §8.1) —
  most likely logged and dropped, since no DLX is declared. **A poison message is lost, not
  parked.**

**Extension procedure.** To add a DLX, set the exchange/queue arguments in the `rabbitMq` section
(Convey supports `deadLetter` options `[convey]`) in **all four** `appsettings*` files. To publish to
a new exchange, use `[Message("name")]` — no configuration change needed.

**Failure modes.** Credentials `guest`/`guest` appear in `appsettings.json:97-98` and are inherited
by the docker profile. In the compose topology RabbitMQ is not exposed outside the `pacco` network
(`hianshul100_Pacco/compose/`), which limits the exposure, but the credentials are still committed.
Recorded as B3 (§8.2).

---

### 3.27 Synchronous HTTP reads & service resolution

**Definition.** The two blocking HTTP calls the saga makes mid-workflow, and how their target
addresses are resolved.

**Representation & storage.** Two thin clients, both `Transient` (`Extensions.cs:44-45`):

```csharp
// Services/Clients/AvailabilityServiceClient.cs:13-20
public AvailabilityServiceClient(IHttpClient httpClient, HttpClientOptions options)
{ _httpClient = httpClient; _url = options.Services["availability"]; }        // :16
public Task<ResourceDto> GetResourceReservationsAsync(Guid resourceId)
    => _httpClient.GetAsync<ResourceDto>($"{_url}/resources/{resourceId}");   // :20

// Services/Clients/VehiclesServiceClient.cs:15-19
_url = options.Services["vehicles"];                                          // :18
```

Resolution is environment-dependent (`appsettings.json:23-34` vs `appsettings.local.json`):

| Environment | `httpClient.type` | `services.availability` | `services.vehicles` |
| --- | --- | --- | --- |
| default / `development` | `""` (direct) | `availability-service` | `vehicles-service` |
| `local` | `""` (inherited) | `http://localhost:5001` | `http://localhost:5009` |
| `docker` | `fabio` | `availability-service` | `vehicles-service` (inherited) |

Under `type: "fabio"` Convey rewrites `availability-service` into a request through the Fabio load
balancer at `http://fabio:9999` (`appsettings.docker.json:19`) `[convey]`. Under the **default**
profile `type` is `""` and the service names are bare hostnames — which resolve only if DNS provides
them. `retries: 3` (`appsettings.json:25`) applies Polly-style retries inside `IHttpClient`
`[convey]`.

**Lifecycle.** Called exactly twice per saga, both inside `HandleAsync(ParcelAddedToOrder)`
(`Saga:93`, `:98`), sequentially, on a RabbitMQ consumer thread.

**Invariants & enforcement.**
- **The dictionary lookups at `AvailabilityServiceClient.cs:16` and `VehiclesServiceClient.cs:18`
  are unguarded.** A missing `httpClient.services` key throws `KeyNotFoundException` **at client
  construction**, which — because the clients are transient and constructed as saga dependencies —
  means **at saga construction**, i.e. on every message, before any handler runs. **Fails loudly**,
  and usefully: a misconfiguration is immediately obvious. This is the best failure mode in the
  component.
- **These calls block a saga step.** The saga is not a background worker: `HandleAsync` runs on the
  consumer thread, so a slow `vehicles-service` stalls the consumer and, with a prefetch of 1,
  the whole subscription.
- **There is no timeout configured anywhere.** No `httpClient.timeout` key exists in any
  `appsettings*` file, so `HttpClient`'s default 100 seconds applies `[framework]`. Two sequential
  calls means a worst case near **200 seconds** per `ParcelAddedToOrder`. **Fails silently as a
  latency problem**, loudly as a `TaskCanceledException` at the end.
- **No circuit breaker.** Convey's `retries: 3` will re-attempt a failing dependency three times per
  saga step, multiplying load on an already-struggling service.
- `requestMasking.enabled: true` (`appsettings.json:30-33`) masks request bodies in logs — but these
  are GETs with no bodies, so it has no effect here.

**Extension procedure.** Add a `timeout` to the `httpClient` section in all four `appsettings*`
files before adding any further synchronous call. Better: replace both reads with a
request/response message pair, or pre-fetch the vehicle before the saga starts. Any new client
follows the same three-part shape: interface, implementation reading `options.Services[key]`, and a
`AddTransient` line at `Extensions.cs:44-46`.

**Failure modes.** **This is the architectural tension at the centre of the component**: a saga that
exists to coordinate asynchronously makes two synchronous calls in the middle of its most important
step, with no timeout and no breaker. An exception from either propagates to Chronicle and triggers
the compensation walk — which, for `ParcelAddedToOrder`, does the one thing it can: cancels the order
(§3.16). So a `vehicles-service` outage cancels customer orders. That is arguably correct, and
certainly undocumented.

---

### 3.28 `VehiclesServiceClient.GetBestAsync` — the "AI"

**Definition.** Vehicle selection. The component's namesake feature.

**Representation & storage.** `Services/Clients/VehiclesServiceClient.cs:21-31`:

```csharp
public async Task<VehicleDto> GetBestAsync()
{
    var vehicles = await _httpClient.GetAsync<PagedResult<VehicleDto>>($"{_url}/vehicles");   // :23
    var bestVehicle = vehicles?.Items?.FirstOrDefault();  // typical AI in a startup           // :24
    if (bestVehicle is null)
        throw new InvalidOperationException("The best vehicle was not found.");                // :27
    return bestVehicle;
}
```

**Lifecycle.** One call per saga, at `Saga:93`. The result's `Id` is stored in `Data.VehicleId`
(`:94`) and its `PricePerService` is logged (`:95`) and then discarded.

**Invariants & enforcement.**
- **"Best" means "first page, first row."** `GET /vehicles` returns a `PagedResult<VehicleDto>`
  whose ordering is whatever `vehicles-service` chooses; there is no `orderBy`, no filter on
  capacity, availability, location or price, and no page parameter. **Every saga picks the same
  vehicle**, which then accumulates one reservation per order (§3.29 turns that into an ever-later
  delivery date).
- The null-coalescing chain `vehicles?.Items?.FirstOrDefault()` tolerates a null response *and* a
  null `Items`, then converts both to the same exception. **Fails loudly** — but with a message
  (`"The best vehicle was not found."`) that cannot distinguish "no vehicles exist" from
  "`vehicles-service` returned garbage". It is also mapped to a generic 400 if it ever reached HTTP
  (§3.33), which it cannot, since this runs on a consumer thread.
- `PricePerService` is `decimal` (`DTO/VehicleDto.cs:10`) and is logged with a `$` suffix
  (`Saga:95`) with no currency or culture handling. Cosmetic.

**Extension procedure.** Replace the body with a real selection: request a filtered page
(`{_url}/vehicles?…`), or move selection into `vehicles-service` behind a purpose-built endpoint —
the better option, since only that service knows about availability. Keep the exception on
"nothing suitable"; it is the saga's only failure signal at this step.

**Failure modes.** If `vehicles-service` returns an empty page — a normal condition, not an error —
every saga throws at `:27`, compensates, and cancels the order with the reason `"Because I'm saga"`
(§3.16). The customer is told nothing useful.

---

### 3.29 `ResourceReservationsService.GetBestAsync` — the scheduler

**Definition.** Delivery-date selection: given a vehicle, decide when it can serve this order and at
what priority.

**Representation & storage.** `Services/ResourceReservationsService.cs:18-44`:

```csharp
var resource = await _availabilityServiceClient.GetResourceReservationsAsync(resourceId);   // :20
if (resource is null)
    throw new InvalidOperationException($"Resource with id: '{resourceId}' was not found."); // :23

var latestReservation = resource.Reservations.Any()
    ? resource.Reservations.OrderBy(r => r.DateTime).Last()                                  // :27
    : null;

if (latestReservation is null)
    return new ReservationDto { DateTime = DateTime.UtcNow.AddDays(1), Priority = 0 };        // :32-36

return new ReservationDto
{
    DateTime = latestReservation.DateTime.AddDays(1),                                         // :41
    Priority = latestReservation.Priority + 1                                                 // :42
};
```

The rule: **queue behind the last existing reservation, one day later, one priority higher.**

**Lifecycle.** One call per saga, at `Saga:98`, immediately after vehicle selection. Both outputs
land in `Data` (`:99-100`) and are used twice each: in `AssignVehicleToOrder` (`:104`) and
`ReserveResource` (`:114`).

**Invariants & enforcement.**
- **The date is unbounded and monotonically increasing.** Because §3.28 always picks the same
  vehicle, the *n*-th order is scheduled *n* days out with priority *n*. Nothing caps either. There
  is no business calendar, no working-hours logic, no capacity model — one reservation per vehicle
  per day, forever. **Fails silently**: the workflow succeeds and the customer gets a delivery date
  in 2027.
- `resource.Reservations.Any()` will **throw `NullReferenceException`** if the upstream
  `ResourceDto.Reservations` is null. `availability-service` populates it from the aggregate
  (`Availability/.../Application/DTO/ResourceDto.cs:9`), but a resource with no reservations could
  serialise it as `null` rather than `[]` depending on the mapper. `:21`'s null check guards the
  *outer* object only. **Fails loudly, at the wrong layer, with an unhelpful message.**
- `OrderBy(...).Last()` is O(n log n) where `Max()` would do, and materialises the whole list. Fine
  at demo scale.
- **`Kind` is not normalised.** `DateTime.UtcNow.AddDays(1)` yields `Kind=Utc`; `latestReservation.
  DateTime.AddDays(1)` yields whatever Newtonsoft produced when deserialising the upstream JSON —
  typically `Local` or `Utc` depending on `DateTimeZoneHandling` `[newtonsoft]`. The two branches can
  therefore differ by the host's UTC offset, and the value is round-tripped back to
  `availability-service` as a reservation key. **Fails silently**; a container in a non-UTC timezone
  could reserve the wrong day. Recorded as Q5.
- The `Priority` semantics are defined by `availability-service`'s `Reservation` value object
  (`Availability/.../Core/ValueObjects/Reservation.cs`), which this service does not read. It
  increments the number without knowing what it means.

**Extension procedure.** The whole method is a placeholder; replacing it is expected. A real
implementation belongs in `availability-service` behind an endpoint like
`GET /resources/{id}/next-available`, which would also fix the null-`Reservations` hazard and the
`Kind` problem. If it stays here, at minimum: normalise to `DateTime.SpecifyKind(..., Utc)`, guard
`Reservations` against null, and cap the horizon.

**Failure modes.** Combined with §3.28, this is the component's only piece of business logic and it
is deliberately naive. The comment at `VehiclesServiceClient.cs:24` is the repository's own
acknowledgement. Do not let a later phase mistake this for a specification.

---

### 3.30 Local DTOs and their upstream coupling

**Definition.** The three read-model types this service deserialises HTTP responses into.

**Representation & storage.**

| Local DTO | Fields | Upstream source | Drift |
| --- | --- | --- | --- |
| `DTO/VehicleDto.cs` | `Id`, `Brand`, `Model`, `PricePerService` | `vehicles-service` `GET /vehicles` | only `Id` and `PricePerService` are read |
| `DTO/ResourceDto.cs` | `Id`, `Reservations` | `availability-service` `GET /resources/{id}` — which also returns `Tags` (`Availability/.../DTO/ResourceDto.cs:9`) | **`Tags` is omitted here**; harmless, since extra JSON is ignored `[newtonsoft]` |
| `DTO/ReservationDto.cs` | `DateTime`, `Priority` | `Availability/.../DTO/ReservationDto.cs` — **identical** | none |

All three have public setters (unlike the message contracts, which are get-only) because they are
deserialisation targets, not constructor-bound contracts.

**Lifecycle.** Constructed by Newtonsoft inside `IHttpClient.GetAsync<T>`; read immediately; never
stored. `ReservationDto` is also used as the *return* type of `IResourceReservationsService`
(§3.29), so one type serves both as a wire DTO and as an internal result — a small conflation that
means a change to the upstream shape also changes an internal signature.

**Invariants & enforcement.**
- **Deliberate under-specification is safe in this direction.** Omitting `Tags` costs nothing.
  Omitting a field the code *needs* would produce `null`/`default` with no error. **Fails silently.**
- `VehicleDto.PricePerService` is `decimal`; if `vehicles-service` ever emits it as a string, the
  deserialisation throws. Currently it is a `decimal` upstream too.
- Nothing validates that `ResourceDto.Id` matches the id that was requested.

**Extension procedure.** Add only the fields you consume. When a DTO is used as an internal return
type (as `ReservationDto` is), consider splitting it, so the wire shape and the internal shape can
evolve separately.

**Failure modes.** These are the only types in the repository that would break *loudly* on upstream
change (via a type mismatch); every other cross-service coupling breaks silently.

---

### 3.31 `MakeOrderCompleted` — the terminal event that never lands

**Definition.** The only event this service publishes on its own exchange, announcing that a saga
finished.

**Representation & storage.** `Events/MakeOrderCompleted.cs` — `IEvent`, one field `OrderId`, and
**no `[Message]` attribute**, so it routes to the default exchange `ordermaker`
(`appsettings.json:116`) with routing key `make_order_completed`. Published at `Saga:124-129` with
header `Saga=Completed`, immediately before `CompleteAsync()`.

**Lifecycle — the full round trip.**

```
Saga:124  publish MakeOrderCompleted → exchange "ordermaker", key "make_order_completed"
                                        message_context = blank object (§3.18)
   │
   ▼
operations-service subscribes generically:
   Pacco.Services.Operations.Api/messages.json:75-83 → { "exchange": "ordermaker",
                                                          "events": ["make_order_completed"] }
   │
   ▼
GenericEventHandler<T>.HandleAsync                Operations.Api/Handlers/GenericEventHandler.cs:29
   var correlationId = messageProperties?.CorrelationId;                              // :31
   if (string.IsNullOrWhiteSpace(correlationId)) { return; }                          // :32-35  ◄── DROPPED
```

**The subscription exists on both sides and the message is discarded on arrival**, because this
service publishes with an empty correlation context (§3.18) and therefore no AMQP correlation id.
Nothing logs the drop.

**Invariants & enforcement.**
- **Fails silently, at the far end, in another repository.** From this service's perspective the
  publish succeeded; from `operations-service`'s perspective nothing arrived worth recording; from
  the user's perspective the order simply never shows a completion.
- The header `Saga=Completed` would have mapped to `OperationState.Completed`
  (`Operations.Api/Handlers/Extensions.cs:9-14`, §3.17) — so the *design* is complete and correct;
  only §3.18 breaks it. **Fixing §3.18 fixes this too.** That is the single highest-value change in
  the component.
- No service other than `operations-service` references the type (verified by workspace-wide search:
  the only non-OrderMaker hits are the two strings in `messages.json`).

**Extension procedure.** Do not add consumers for this event; fix the correlation context instead
(§3.18) and the existing consumer starts working. If you do want a business consumer, give the type
an explicit `[Message("ordermaker")]` so the routing is not implicit.

**Failure modes.** As above. Note also that `CompleteAsync()` runs *after* this publish (§3.15), so a
publish failure leaves the saga permanently `Pending`.

---

### 3.32 `MakeOrderRejected` — the unused rejection convention

**Definition.** Pacco's `IRejectedEvent` convention: a typed failure notification carrying a code
and a reason, which `operations-service` turns into a rejected operation shown to the user.

**Representation & storage.** `Events/Rejected/MakeOrderRejected.cs`:

```csharp
public class MakeOrderRejected : IRejectedEvent
{
    public Guid OrderId { get; }
    public string Reason { get; }
    public string Code { get; }
}
```

No `[Message]`, so it would route to exchange `ordermaker`, key `make_order_rejected` — **which
`operations-service` already subscribes to** (`Operations.Api/messages.json:80-82`, handled by
`GenericRejectedEventHandler` which defaults the state to `OperationState.Rejected`,
`Operations.Api/Handlers/GenericRejectedEventHandler.cs:40-42`).

**Lifecycle.** None. Verified by search: the identifier appears only in its own declaration file. The
type is compiled and never constructed.

**Invariants & enforcement.**
- **The platform is fully prepared to display a failed order-making workflow, and this service never
  tells it about one.** The consumer exists, the queue would exist, the state mapping exists. The
  producer is missing. **Fails silently** — failures are simply invisible.
- Together with §3.15 (`RejectAsync()` never called) and §3.16 (four no-op compensations), this
  completes the picture: **the component has no failure path at all.** Every error either reaches the
  HTTP caller as a generic 400 (only for `MakeOrder`, §3.7) or vanishes on a consumer thread.

**Extension procedure.** Wrap each saga action body in a `try/catch`, and on failure publish
`new MakeOrderRejected(Data.OrderId, ex.Message, "make_order_failed")` with header
`Saga=Rejected`, then call `RejectAsync()`. Use a stable `Code` — `operations-service` surfaces it
verbatim. Note that the reason string reaches the end user, so do not pass raw exception text; map
known exceptions (`InvalidOperationException` from §3.28/§3.29) to friendly codes.

**Failure modes.** None today. The cost is that **an operator cannot tell a stuck saga from a failed
one from a completed one**, because none of the three produces a distinguishable signal.

---

### 3.33 Error handling & `ExceptionToResponseMapper`

**Definition.** The HTTP error contract. One class, one arm.

**Representation & storage.** `ExceptionToResponseMapper.cs:9-14`:

```csharp
public ExceptionResponse Map(Exception exception)
    => exception switch
    {
        _ => new ExceptionResponse(new { code = "error", reason = "There was an error." },
                                   HttpStatusCode.BadRequest)
    };
```

Registered at `Extensions.cs:29`, activated by `UseErrorHandler()` at `Extensions.cs:53`.

**Lifecycle.** Invoked by Convey's error-handling middleware for any unhandled exception on an HTTP
request `[convey]`.

**Invariants & enforcement.**
- **Every exception becomes `400 {"code":"error","reason":"There was an error."}`.** A malformed
  body, a missing RabbitMQ connection, a DI resolution failure and a business rule violation are
  indistinguishable to the caller. **Fails loudly but uninformatively.**
- **`400` is wrong for infrastructure failures.** A broker outage is a `503`, not a client error.
  Retrying clients will not retry a 400.
- The switch expression with a single discard arm is the shape used when a service has typed domain
  exceptions to add later; compare `orders-service`, whose mapper enumerates real exception types.
  **This service has no custom exception types at all** — it throws bare
  `InvalidOperationException` in two places (`VehiclesServiceClient.cs:27`,
  `ResourceReservationsService.cs:23`) — so there is nothing to enumerate.
- The mapper covers **only HTTP**. Exceptions on RabbitMQ consumer threads never reach it (§3.26),
  which is where all but the first saga step run.

**Extension procedure.** Define exception types (e.g. `VehicleNotFoundException`,
`ResourceUnavailableException`) in a `Exceptions/` folder, throw them from the two clients, and add
arms mapping each to a stable `code` and an appropriate status. Do this together with §3.32, since
the same taxonomy feeds both the HTTP response and the rejected event.

**Failure modes.** The one place this matters today is a bad `POST /orders` body: the caller sees a
400 with no indication of which field was wrong.

---

### 3.34 Security posture

**Definition.** What this service does and does not enforce about who is calling it.

**Representation & storage.** Registration exists; enforcement does not.

| Piece | Present? | Evidence |
| --- | --- | --- |
| `AddSecurity()` | ✅ | `Extensions.cs:41` — registers JWT handling, certificates, `IAppContext` `[convey]` |
| `jwt` configuration | ✅ | `appsettings.json:71-79` — `validIssuer: "pacco"`, `validateIssuer/Lifetime: true`, `validateAudience: false`, certificate at `certs/localhost.cer` |
| Certificate asset | ✅ | `src/Pacco.Services.OrderMaker/certs/localhost.cer` (1 115 bytes), copied to publish output by `.csproj:31` |
| `UseAuthentication()` | ❌ | absent from `Extensions.cs:51-65` and `Program.cs:22-26` |
| `UseAuthorization()` | ❌ | absent |
| `.RequireAuthorization()` / `auth: true` on the endpoint | ❌ | `Program.cs:26` is a bare `.Post<MakeOrder>("orders")` |
| `IAppContext` / identity check in a handler | ❌ | no reference anywhere in the repository |
| `swagger.includeSecurity` | ✅ (`true`, `appsettings.json:142`) | documents a security scheme the service does not enforce |
| Vault | ❌ | no `.UseVault()`, unlike `availability-service` (`Api/Program.cs:48`) |

`appsettings.local.json` blanks the certificate location (`"location": ""`), so local runs have no
certificate at all.

**Lifecycle.** The JWT machinery is constructed at startup and never invoked, because no middleware
challenges a request.

**Invariants & enforcement.**
- **`POST /orders` is unauthenticated.** Anyone who can reach port 5015 (or port 80 inside the
  compose network) can start a saga for **any** `customerId` and **any** `parcelId`. **Fails
  silently — the request succeeds.**
- The saga then acts on that customer's behalf against `availability-service`, whose identity guard
  is skipped because the correlation context is blank (§3.18). **The two gaps compose into a
  complete bypass:** an unauthenticated caller can reserve platform capacity in another customer's
  name. This is the component's most serious finding (B1, §8.2).
- The service is **not** behind the API gateway. `hianshul100_Pacco.APIGateway`'s configuration does
  not route to `ordermaker-service`, and `compose/services.yml:78-85` publishes `5015:80` directly to
  the host. So the gateway's authentication is not in the path either.
- `validateAudience: false` (`appsettings.json:76`) would, if authentication were enabled, accept a
  token minted for any Pacco audience — consistent with the other services, and moot here.

**Extension procedure.** Three steps, in order: (1) add `.UseAuthentication()` between `UseConvey()`
and `UseMetrics()` in `Extensions.cs:55-56`; (2) mark the endpoint as authenticated
(`.Post<MakeOrder>("orders", auth: true)` `[convey]`); (3) resolve the caller's identity and either
validate `MakeOrder.CustomerId` against it or overwrite it. Then remove the constructor overwrite in
§3.18 so the identity propagates onto the published commands — **all four steps are needed; any
three leave the bypass open.**

**Failure modes.** Because nothing calls this service today (§1.3), the exposure is theoretical in
the demo topology and immediate in any deployment that makes port 5015 reachable.

---

### 3.35 Redis: registered, never used

**Definition.** A configured, connected datastore with no consumer.

**Representation & storage.** `Extensions.cs:37` (`.AddRedis()`) and `appsettings.json:131-134`
(`connectionString: "localhost"`, `instance: "ordermaker:"`; `appsettings.docker.json:64` overrides
the host to `redis`). Verified by search: `IDistributedCache` appears nowhere in the repository, and
the only `Redis` references are the `using` at `Extensions.cs:12` and the registration itself.

**Lifecycle.** `AddRedis()` opens a connection multiplexer at startup `[convey]`. It is held open for
the process lifetime and never used.

**Invariants & enforcement.**
- The service **will fail to start or log connection errors if Redis is unavailable**, despite not
  needing it. Whether this is fatal depends on Convey's lazy-connection behaviour `[convey]` (A10,
  §8.1). Either way it is a spurious dependency: `compose/services.yml` does not list a `depends_on`
  for this service, so startup ordering is not guaranteed.
- The `instance: "ordermaker:"` prefix reserves a key namespace that is never written.
- **This is the obvious home for saga persistence** (§3.14): the store is already configured and
  connected. That is what makes the gap notable rather than merely untidy.

**Extension procedure.** Either delete `Extensions.cs:37`, the `using` at `:12`, the
`Convey.Persistence.Redis` reference (`.csproj:23`) and the `redis` section from all four
`appsettings*` files — or use it, by backing Chronicle's saga store with it.

**Failure modes.** A spurious hard dependency on an infrastructure component the service does not
need, which will cause an outage it should have been immune to.

---

### 3.36 Observability: logging, metrics, no tracing

**Definition.** What an operator can see.

**Representation & storage.**

| Facility | Configuration | Status |
| --- | --- | --- |
| Serilog | `appsettings.json:35-70`, installed by `.UseLogging()` (`Program.cs:27`) | ✅ |
| Console sink | `:52-54` `enabled: true` | ✅ |
| Rolling file sink | `:59-63` `logs/logs.txt`, daily | ✅ (disabled in `local`) |
| Seq | `:64-68` `http://localhost:5341`, `apiKey: "secret"` → `http://seq:5341` in docker | ✅ |
| ELK | `:55-58` `enabled: false` | ❌ |
| Path exclusions | `:37` `["/", "/ping", "/metrics"]` | ✅ — but **not** `/docs` |
| Property masking | `:38-51` 12 sensitive property names redacted | ✅ |
| App.Metrics / Prometheus | `:80-88`, `.AddMetrics()` / `.UseMetrics()` | ✅ (both disabled in `local`) |
| **Jaeger / distributed tracing** | — | ❌ **absent entirely** |

Application-level logging is five `_logger.LogInformation` calls, all in the saga:

| Line | Message |
| --- | --- |
| `Saga:58` | `Started a saga for order: '{OrderId}'.` |
| `Saga:92` | `Searching for a vehicle...` |
| `Saga:95` | `Found a vehicle with id: '{id}' for {price}$.` |
| `Saga:97` | `Reserving a date for vehicle: '{id}'...` |
| `Saga:101` | `Reserved a date: {date} for vehicle: '{id}'.` |
| `Saga:123` | `Completed a saga for order: '{OrderId}'.` |

**Lifecycle.** Emitted synchronously during saga actions; shipped to console, file and Seq.

**Invariants & enforcement.**
- **All six use string interpolation, not structured templates.** `$"Started a saga for order:
  '{message.OrderId}'."` produces a rendered string with no `OrderId` property, so **Seq cannot
  filter or group by order id.** Compare the platform norm of `LogInformation("… {OrderId}",
  orderId)`. **Fails silently**: the logs look fine and are unqueryable.
- **No log line marks a failure, a compensation or a drop.** Nothing logs in `CompensateAsync`
  (§3.16), nothing logs when `AllPackagesAddedToOrder` returns false (`Saga:87-90`), nothing logs
  when an event reaches no saga action (§3.25). **The three most important events in the component
  are silent.**
- **This is the only Pacco service without Jaeger.** Every sibling calls `.UseJaeger()` (e.g.
  `Orders/.../Infrastructure/Extensions.cs:89`); this one does not, and does not reference the
  package. Combined with the blank correlation context (§3.18) and unstructured logs, **a saga is
  unobservable end to end**: there is no trace, no correlation id, and no queryable order id.
- `logger.seq.apiKey: "secret"` is a committed placeholder credential (`appsettings.json:67`).
  Recorded with B3.

**Extension procedure.** Highest value first: (1) convert the six log calls to structured templates
— a five-minute change that makes Seq useful; (2) add `LogWarning` to each `CompensateAsync` and to
the early return at `Saga:87-90`; (3) add `Convey.Tracing.Jaeger` and `.UseJaeger()` in
`Extensions.cs`; (4) fix §3.18 so spans link.

**Failure modes.** As it stands, diagnosing a stuck saga means reading console output and inferring
state from which of the six messages appeared.

---

### 3.37 Service discovery & health

**Definition.** How the platform finds this service, and how it decides the process is alive.

**Representation & storage.** `appsettings.json:7-22`:

```json
"consul": { "enabled": true, "url": "http://localhost:8500", "service": "ordermaker-service",
            "address": "localhost", "port": "5015", "pingEnabled": true, "pingEndpoint": "ping",
            "pingInterval": 3, "removeAfterInterval": 3 },
"fabio":  { "enabled": true, "url": "http://localhost:9999", "service": "ordermaker-service" }
```

Registered by `.AddConsul()` / `.AddFabio()` (`Extensions.cs:31-32`). Docker overrides:
`consul.url → http://consul:8500`, `consul.address → ordermaker-service`, `consul.port → "80"`,
`fabio.url → http://fabio:9999` (`appsettings.docker.json:8-11,19`). Local disables both
(`appsettings.local.json:3-7`).

**Lifecycle.** Registration happens at startup and deregistration on graceful shutdown `[convey]`.
Consul polls `GET /ping` every 3 seconds and removes the service 3 intervals after it stops
responding.

**Invariants & enforcement.**
- **The health check is liveness only.** `/ping` returns 200 as long as the HTTP pipeline is up. It
  says nothing about RabbitMQ connectivity, Redis, or whether any saga is progressing. A service
  whose broker connection has died stays "healthy" in Consul indefinitely. **Fails silently.**
- `consul.port` is a **string** (`"5015"`, `"80"`) — matching the other services; the type is
  Convey's, not a mistake here.
- **The registration is write-only in practice.** No service resolves `ordermaker-service` through
  Consul or Fabio, because nothing calls it (§1.3). The registration exists for symmetry.
- Fabio is used in the *outbound* direction under the docker profile (`httpClient.type: "fabio"`,
  §3.27) — that is where these settings actually matter.

**Extension procedure.** If a real health endpoint is wanted, add a readiness check that verifies the
RabbitMQ connection and (once §3.14 is fixed) the saga store, and point `pingEndpoint` at it.

**Failure modes.** A 3-second interval with removal after 3 intervals means a ~9-second detection
window — fine. The risk is the opposite: a process that is up but not consuming stays registered.

---

### 3.38 Configuration layering & environments

**Definition.** Four `appsettings` files and how they compose.

**Representation & storage.**

| File | Size | Role |
| --- | --- | --- |
| `appsettings.json` | 144 lines | the full baseline; **also the effective config for the `development` environment** |
| `appsettings.local.json` | partial | disables Consul, Fabio, metrics, file/Seq logging; points HTTP clients at `localhost:5001`/`5009`; blanks the JWT certificate; sets log level `verbose` |
| `appsettings.docker.json` | partial | container hostnames (`consul`, `fabio`, `rabbitmq`, `redis`, `seq`, `influx`, `elk`), `httpClient.type: "fabio"`, `consul.port: "80"`, `metrics.env: "docker"`, file logging off |
| `appsettings.development.json` | **`{}` — empty object** | placeholder |

Environment selection: `scripts/start.sh:2` and `launchSettings.json:14,22` export
`ASPNETCORE_ENVIRONMENT=local`; `Dockerfile:10` sets it to `docker`.

**Lifecycle.** `CreateDefaultBuilder` loads `appsettings.json` then
`appsettings.{environment}.json`, the latter overriding key-by-key `[framework]`, then environment
variables, then command-line arguments.

**Invariants & enforcement.**
- **The default (no `ASPNETCORE_ENVIRONMENT`) and `Development` profiles are the same thing**, and
  that thing points at `localhost:8500`, `localhost:9999`, `localhost:5341` and bare hostnames
  `availability-service`/`vehicles-service` with `httpClient.type: ""`. **This combination works in
  no environment**: on a developer machine the infrastructure hosts are right but the service
  hostnames do not resolve; in a container the reverse. Running without setting the environment
  variable therefore fails at the first HTTP call, **loudly but confusingly**.
  `scripts/start.sh` and `launchSettings.json` both set `local`, so the trap is only sprung by
  `dotnet run` invoked directly — which is exactly what `README.md:21` tells you to do.
- **Overrides are per-key, not per-section**, and JSON arrays are replaced wholesale. This matters
  for `rabbitMq.hostnames` (docker replaces `["localhost"]` with `["rabbitmq"]` — correct) and would
  matter for `logger.excludePaths` if it were ever overridden.
- **`local` blanks the JWT certificate path** (`"location": ""`), so security cannot be enabled
  locally even if §3.34 were fixed, without also supplying a certificate.
- Credentials in `appsettings.json`: `rabbitMq.username/password` = `guest`/`guest` (`:97-98`) and
  `logger.seq.apiKey` = `"secret"` (`:67`). Committed. Recorded as B3.

**Extension procedure.** Add new keys to `appsettings.json` first (the baseline), then override in
`local` and `docker` as needed. **Do not put anything in `appsettings.development.json`** — it is
empty by convention, and the default profile is what `development` resolves to.

**Failure modes.** The empty `development` file is a trap: an engineer who adds an override there
will find it silently ignored in every environment that matters, because nothing runs under
`Development`.

---

### 3.39 Deployment & topology

**Definition.** How the service is packaged and where it runs.

**Representation & storage.**

`Dockerfile` (11 lines):
```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build
WORKDIR /app
COPY . .
RUN dotnet publish src/Pacco.Services.OrderMaker -c release -o out    # :4
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
COPY --from=build /app/out .
ENV ASPNETCORE_URLS http://*:80                                       # :9
ENV ASPNETCORE_ENVIRONMENT docker                                     # :10
ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll                       # :11
```

`hianshul100_Pacco/compose/services.yml:78-85`:
```yaml
ordermaker-service:
  image: devmentors/pacco.services.ordermaker
  container_name: ordermaker-service
  restart: unless-stopped
  ports: [ "5015:80" ]
  networks: [ pacco ]
```

**Lifecycle.** Built by Travis on `master`/`develop` (§3.40), pushed as
`$DOCKER_USERNAME/pacco.services.ordermaker` with tags `latest`/`dev` plus a build number
(`scripts/dockerize.sh:16-21`), pulled by compose.

**Invariants & enforcement.**
- **`restart: unless-stopped` plus in-memory sagas is an active hazard.** Docker will restart this
  container on crash; every restart silently abandons every in-flight saga (§3.14) while the durable
  queues redeliver their events into a service that has forgotten them. **Fails silently.**
- **No `depends_on`.** Unlike `operations-service` (which lists eight,
  `compose/services.yml:51-67`), this service declares no ordering, so it may start before RabbitMQ,
  Consul or Redis. `rabbitMq.retries: 3` with a 2-second interval gives ~6 seconds of tolerance —
  usually enough, not guaranteed.
- **The port is published to the host** (`5015:80`) while the service is unauthenticated (§3.34).
  Combined, these mean the saga trigger is reachable from outside the compose network.
- Single replica; no `deploy.replicas`, no scaling config. Consistent with §3.14's constraint, but
  by accident rather than by declaration — **nothing records that this service must not be scaled.**
- `COPY . .` at `:3` copies the whole repository including `.git`; `.dockerignore` exists at the
  repository root and mitigates this. Build-time only.
- The published output includes `certs/localhost.cer` (`.csproj:31`), which is a public certificate,
  not a key — safe to ship.

**Extension procedure.** Before scaling this service, fix §3.14. Before exposing it, fix §3.34. Add
a `depends_on: [rabbitmq, consul, redis]` to the compose entry — note this requires editing
`hianshul100_Pacco`, a different repository.

**Failure modes.** The dangerous combination is: durable queues + restart-on-crash + in-memory state
+ no dead-lettering. A crash loop would consume and discard every queued saga event permanently.

---

### 3.40 Build, CI and the absence of tests

**Definition.** How the repository is built and validated.

**Representation & storage.**

| Artifact | Content |
| --- | --- |
| `Pacco.Services.OrderMaker.sln` | one project, GUID `{F2CBF6B7-501A-474D-B7DA-042C72823DE8}`, nested under a `src` solution folder (`:6-9,37`); Debug/Release × AnyCPU/x64/x86 all mapped to `Any CPU` (`:23-34`) |
| `scripts/build.sh` | `dotnet build -c release` |
| `scripts/test.sh` | `dotnet test` |
| `scripts/start.sh` | `ASPNETCORE_ENVIRONMENT=local; cd src/…; dotnet run` |
| `scripts/dockerize.sh` | Travis-branch-driven tagging and push |
| `.travis.yml` | `dotnet: 3.1.100`; branches `master`, `develop`; `script: build.sh, test.sh`; `after_success: dockerize.sh` |
| **test projects** | **none** — the solution contains exactly one project, and no `*.Tests` directory exists |

**Lifecycle.** Travis builds on pushes to `master`/`develop` only. The current branch —
`feature/12998/aidlc` — is **not** built by CI.

**Invariants & enforcement.**
- **`./scripts/test.sh` runs `dotnet test` against a solution with no test project and exits 0**
  `[framework]` (A11, §8.1). CI is therefore green by construction. **Fails silently: the pipeline
  reports success for an unverified build.**
- **Nothing in this component is tested.** Not the inverted predicate (§3.13), not the contract
  drifts (§3.21, §3.22), not `ResolveId`, not the compensation paths. Every finding in this document
  would have been caught by one unit test of `AIMakingOrderData` or one integration test of the
  saga. This is baseline gap **G5**.
- `dotnet: 3.1.100` pins the SDK; `netcoreapp3.1` is out of support (A12, §8.1).
- The `.sln` has complete `Build.0` rows for all six configurations (`:23-34`), so the project is
  never silently excluded — contrast the risk noted in `operations-grpc-client.md` §3.22.

**Extension procedure.** The highest-value first test is a pure unit test of
`AIMakingOrderData.AllPackagesAddedToOrder` with two parcels — it fails immediately and proves
§3.13. Second: a saga test with a fake `IBusPublisher` asserting the message sequence. Add the test
project to the `.sln` (six `ProjectConfigurationPlatforms` rows) or `dotnet test` will not find it.

**Failure modes.** A green pipeline that verifies nothing is worse than no pipeline, because it
creates the impression of coverage.

---

## 4. Primary control flows

### 4.1 Startup

```
dotnet Pacco.Services.OrderMaker.dll                        Dockerfile:11
  └─ Program.Main                                           Program.cs:15
       ├─ CreateDefaultBuilder → load appsettings.json + appsettings.{env}.json   (§3.38)
       ├─ AddConvey().AddWebApi().AddInfrastructure().Build()                     (§3.3)
       │     ├─ Redis multiplexer opens                     Extensions.cs:37      (§3.35)
       │     ├─ RabbitMQ connection opens (3 retries × 2 s) Extensions.cs:39      (§3.26)
       │     └─ Chronicle in-memory saga store created      Extensions.cs:43      (§3.14)
       ├─ UseApp()                                          Extensions.cs:51
       │     ├─ error handler, swagger, convey, metrics
       │     └─ declare exchange "ordermaker" + 5 durable queues, begin consuming (§3.4)
       ├─ UseDispatcherEndpoints → GET "/", POST "/orders"  Program.cs:24-26      (§3.6)
       ├─ Consul registration + /ping health check          Extensions.cs:31      (§3.37)
       └─ Kestrel binds http://*:80 (docker) / :5015 (local)
```

Note what is *not* here: no saga state is loaded, because there is none to load.

### 4.2 The happy path, end to end

```
HTTP  POST /orders {parcelId, customerId}                                   (§3.6)
  │   ← 202 Accepted, empty body, no order id                              (§3.5)
  ▼
MakeOrder → ICommandDispatcher → AIOrderMakingHandler.HandleAsync           (§3.7, §3.8)
  └─ ISagaCoordinator.ProcessAsync(cmd, SagaContext.Empty)                  (§3.9)
       ├─ ResolveId → SagaId = OrderId.ToString()                           (§3.11)
       ├─ create AIMakingOrderData                                          (§3.12)
       └─ HandleAsync(MakeOrder)                            Saga:56-69
            ├─ log "Started a saga for order: …"            :58
            ├─ Data.ParcelIds.Add / OrderId / CustomerId    :59-61
            └─ publish CreateOrder                          :63   ▶ exchange "orders", Saga=Pending
                                                                    ▲ HTTP response returns here
                    ┌───────────────────────────────────────────────┘
                    ▼
        orders-service: creates the order, publishes OrderCreated (Saga header forwarded, §3.17)
                    │
                    ▼
HandleAsync(OrderCreated)                                    Saga:71-82
  └─ for each ParcelId (exactly one, §3.5): publish AddParcelToOrder   :74  ▶ "orders", Saga=Pending
                    │                                                       (customerId dropped, §3.21)
                    ▼
        orders-service: adds the parcel, publishes ParcelAddedToOrder
                    │
                    ▼
HandleAsync(ParcelAddedToOrder)                              Saga:84-110
  ├─ Data.AddedParcelIds.Add(message.ParcelId)               :86
  ├─ if (!AllPackagesAddedToOrder) return                    :87-90   ← inverted predicate (§3.13)
  ├─ HTTP GET {vehicles}/vehicles → Items.First()            :93      ← blocking, no timeout (§3.27, §3.28)
  ├─ HTTP GET {availability}/resources/{vehicleId}           :98      ← blocking (§3.29)
  │     → latest reservation + 1 day, priority + 1
  └─ publish AssignVehicleToOrder                            :103     ▶ "orders", Saga=Pending
                    │
                    ▼
        orders-service: assigns, publishes VehicleAssignedToOrder
                    │
                    ▼
HandleAsync(VehicleAssignedToOrder)                          Saga:112-119
  └─ publish ReserveResource(VehicleId, CustomerId, Date, Priority)  :113
                                          ▶ exchange "availability", Saga=Pending  (positional drift §3.22)
                    │
                    ▼
        availability-service: identity guard SKIPPED (context blank, §3.18)
                              creates the reservation, publishes ResourceReserved
                    │                                     │
                    │ (Saga header NOT forwarded here, §3.17)
                    ├─────────────────────────────────────┴──────────────┐
                    ▼                                                    ▼
        orders-service.ResourceReservedHandler                 ordermaker: HandleAsync?
          GetAsync(ResourceId, DateTime) → order.Approve()       ✗ no ISagaAction<ResourceReserved>
          publishes OrderApproved                                   message acked and dropped (§3.25)
                    │
                    ▼
HandleAsync(OrderApproved)                                   Saga:121-132
  ├─ log "Completed a saga for order: …"                     :123
  ├─ publish MakeOrderCompleted                              :124   ▶ "ordermaker", Saga=Completed
  │       └─ operations-service GenericEventHandler: correlationId blank → return (§3.31)  ✗ dropped
  └─ await CompleteAsync()                                   :131   → saga state Completed, data discarded
```

**Seven messages published, two synchronous HTTP calls, five inbound events, one of which is
discarded — and the terminal notification never reaches its consumer.** The workflow completes
correctly in `orders-service` and `availability-service`; only its observability and its final
announcement are broken.

### 4.3 The failure path

```
exception anywhere in a saga action
  └─ Chronicle compensation walk, reverse order                      (§3.9, §3.16)
       ├─ CompensateAsync(OrderApproved)          → Task.CompletedTask        ✗
       ├─ CompensateAsync(VehicleAssignedToOrder) → Task.CompletedTask        ✗ reservation NOT released
       ├─ CompensateAsync(ParcelAddedToOrder)     → publish CancelOrder       ✓ "Because I'm saga"
       ├─ CompensateAsync(OrderCreated)           → Task.CompletedTask        ✗
       └─ CompensateAsync(MakeOrder)              → Task.CompletedTask        ✗
  └─ saga state remains Pending — RejectAsync() is never called      (§3.15)
  └─ MakeOrderRejected is never published                            (§3.32)
  └─ nothing is logged                                               (§3.36)
  └─ exception propagates to the RabbitMQ subscriber → Convey default, no DLX (§3.26)
```

The single realistic trigger is `vehicles-service` or `availability-service` being unreachable during
`HandleAsync(ParcelAddedToOrder)`, since that is the only action with an outbound HTTP dependency. In
that case the compensation *does* fire usefully and cancels the order. **Every other failure point
compensates into silence.**

### 4.4 Restart

```
process exits (crash, deploy, docker restart)
  ├─ Chronicle's in-memory store is lost                             (§3.14)
  └─ 5 durable queues retain undelivered events                      (§3.26)
process starts
  ├─ queues redeliver OrderCreated / ParcelAddedToOrder / …
  ├─ ResolveId → a SagaId with no state                              (§3.11)
  └─ Chronicle finds no saga for a non-start action → drop, silently (§3.9)
result: order exists upstream, half-built, permanently. No compensation, no alert.
```

This is the flow that most needs to be understood before this component is deployed anywhere real.

---

## 5. Persistence & schema evolution

### 5.1 What is stored, and where

| Store | Owner | Contents | Durability |
| --- | --- | --- | --- |
| Chronicle in-memory saga store | this service | `AIMakingOrderData` per in-flight saga, keyed by `SagaId` = order id | **process lifetime only** (§3.14) |
| Redis (`ordermaker:` prefix) | this service | **nothing** — connected, never written (§3.35) | n/a |
| RabbitMQ durable queues | broker | undelivered inbound events (§3.26) | survives restart |
| MongoDB `orders` | `orders-service` | the actual order aggregate | durable |
| MongoDB `availability` | `availability-service` | resources and reservations | durable |
| Redis `requests:{id}` | `operations-service` | request status, 300 s sliding expiry — **never written for this service's flows** (§3.31) | volatile |

**This component owns no durable state.** Every consequence of a saga lives in another service's
database. That is the correct shape for a process manager; the defect is that its *own* state is not
durable either.

### 5.2 Schema evolution of the saga state

Today `AIMakingOrderData` has no schema-evolution problem, because it is never serialised across a
process boundary or a version boundary. **This changes the moment a persistence provider is added
(§3.14)**, at which point the following rules apply:

| Change to `AIMakingOrderData` | Safe with a persisted store? | Notes |
| --- | --- | --- |
| Add a property | yes | Old records deserialise it as `default`. Guard against `Guid.Empty` / `null` in the actions that read it. |
| Remove a property | yes for reads, **lossy** | In-flight sagas lose the value mid-workflow. |
| Rename a property | **no** | In-flight sagas silently lose the value. Requires a migration or a `[JsonProperty]` alias. |
| Change a type (`Guid` → `string`) | **no** | Deserialisation of in-flight records throws, or silently defaults, depending on the provider. |
| Change `List<Guid>` to `HashSet<Guid>` | usually yes | JSON array either way; would also fix §3.13's duplicate-add hazard. |
| Change the meaning of `AllPackagesAddedToOrder` | **behavioural, not schema** | It is a computed property, never persisted — so the §3.13 fix applies immediately to in-flight sagas. |

**The general rule for a stateful workflow: never deploy a state-shape change while sagas are in
flight.** With no persistence there is nothing to migrate but everything to lose; with persistence
there is something to migrate and no migration mechanism. Either way, **drain before deploying** —
which, given that a saga can be pending indefinitely (§3.15), is not currently achievable. Recorded
as Q8.

### 5.3 Schema evolution of the message contracts

This is the evolution problem the component actually has today. Contracts are duplicated by hand
across repositories (§3.20, §3.24); the wire format is name-based JSON.

| Change | Effect on this service | Detection |
| --- | --- | --- |
| Peer **adds** a field to an event it publishes | ignored here `[newtonsoft]` | none needed |
| Peer **removes** a field this service reads | deserialises to `default` — e.g. `OrderId = Guid.Empty` → saga id mismatch → message dropped | **none. Fails silently.** |
| Peer **renames** a field | same as removal | **none** |
| Peer **changes** a field's type | deserialisation exception | loud, on a consumer thread |
| This service adds a field to a command | ignored by the peer (§3.21) | none needed |
| This service renames a field on a command | peer receives `default` | **none. Fails silently.** |
| Peer changes a **constructor parameter order** | no wire effect; a C# trap (§3.22) | **none** |

`orders-service` publishes its expected shapes via `UsePublicContracts<ContractAttribute>()`
(`Orders/.../Infrastructure/Extensions.cs:91`). **This service does not consume that endpoint**, and
two of its six outbound commands have already drifted. A CI check that fetches `/_contracts` from
each peer and compares against the local types would catch every row marked "fails silently" above —
recorded as Q7.

### 5.4 Idempotency and replay

**No message handled by this service is idempotent** (§3.24). There is no processed-message ledger,
no version field on `AIMakingOrderData`, and no guard on any handler. Because RabbitMQ redelivery is
normal, the following are all reachable today:

| Redelivered | Consequence |
| --- | --- |
| `MakeOrder` (HTTP retry with the same explicit `orderId`) | re-enters the start action for an existing saga id; Chronicle's behaviour is `[chronicle]`-dependent (A7) |
| `OrderCreated` | duplicate `AddParcelToOrder` — harmless, `orders-service` is idempotent here |
| `ParcelAddedToOrder` | **re-runs vehicle selection and re-publishes `AssignVehicleToOrder`** (§3.13) |
| `VehicleAssignedToOrder` | **duplicate `ReserveResource` → a second real reservation** |
| `OrderApproved` | duplicate `MakeOrderCompleted` and a second `CompleteAsync()` |

The cheapest general fix is a guard at the top of each action keyed on the state that action
produces — e.g. `if (Data.VehicleId != Guid.Empty) return;` in `HandleAsync(ParcelAddedToOrder)`.
That is a one-line change per action and closes the two rows in bold.

---

## 6. Surface → internals map

### 6.1 Inbound surfaces

| Surface | Entry point | Internal path | Mutating? |
| --- | --- | --- | --- |
| `POST /orders` | `Program.cs:26` | `MakeOrder` → `ICommandDispatcher` → `AIOrderMakingHandler:26` → `ISagaCoordinator` → `AIOrderMakingSaga.HandleAsync(MakeOrder)` → publish `CreateOrder` | **yes** — creates a real order in `orders-service` and, eventually, a real reservation. Unauthenticated (§3.34). |
| `GET /` | `Program.cs:25` | writes a literal string | no |
| `GET /ping` | `.AddConsul()` | Convey health endpoint | no |
| `GET /metrics` | `.AddMetrics()` | App.Metrics/Prometheus | no |
| `GET /docs` | `.AddWebApiSwaggerDocs()` | Swagger UI; enabled in **all** environments | no |
| AMQP `orders.order_created` | `Extensions.cs:59` | → `AIOrderMakingHandler:32` → `HandleAsync(OrderCreated)` → publish `AddParcelToOrder` | yes (downstream) |
| AMQP `orders.parcel_added_to_order` | `Extensions.cs:60` | → `:35` → `HandleAsync(ParcelAddedToOrder)` → **2 HTTP calls** → publish `AssignVehicleToOrder` | yes (downstream) |
| AMQP `orders.vehicle_assigned_to_order` | `Extensions.cs:62` | → `:38` → publish `ReserveResource` | yes (downstream) |
| AMQP `orders.order_approved` | `Extensions.cs:58` | → `:29` → publish `MakeOrderCompleted`, `CompleteAsync()` | terminal |
| AMQP `availability.resource_reserved` | `Extensions.cs:61` | → `:41` → coordinator → **no action; dropped** (§3.25) | **no** |

### 6.2 Outbound surfaces

| Target | Mechanism | Trigger | Failure handling |
| --- | --- | --- | --- |
| exchange `orders` / `create_order` | `IBusPublisher` | `Saga:63` | exception reaches HTTP caller as 400 (§3.7) |
| exchange `orders` / `add_parcel_to_order` | `IBusPublisher` | `Saga:74` (×`ParcelIds`) | none — consumer thread |
| exchange `orders` / `assign_vehicle_to_order` | `IBusPublisher` | `Saga:103` | none |
| exchange `orders` / `cancel_order` | `IBusPublisher` | `Saga:141` (compensation only) | none; compensation is not compensated |
| exchange `availability` / `reserve_resource` | `IBusPublisher` | `Saga:113` | none — **and never released** (§3.16) |
| exchange `ordermaker` / `make_order_completed` | `IBusPublisher` | `Saga:124` | none; **dropped by the consumer** (§3.31) |
| `GET {vehicles}/vehicles` | `IHttpClient` | `Saga:93` | throws → compensation → order cancelled |
| `GET {availability}/resources/{id}` | `IHttpClient` | `Saga:98` | throws → compensation → order cancelled |
| Consul registration | `.AddConsul()` | startup | Convey internal |
| Seq / console / file logs | Serilog | throughout | Convey internal |

### 6.3 Configuration surfaces

| Key | Consumed by | Effect if wrong |
| --- | --- | --- |
| `httpClient.services.availability` / `.vehicles` | `Services/Clients/*.cs:16,18` | **`KeyNotFoundException` at saga construction** — loud, immediate (§3.27) |
| `httpClient.type` | Convey | `""` = direct hostname, `"fabio"` = load-balanced. Wrong value ⇒ connection refused (§3.27) |
| `rabbitMq.exchange.name` | default exchange for `MakeOrderCompleted`/`MakeOrderRejected` | messages route to an exchange nobody consumes (§3.26) |
| `rabbitMq.conventionsCasing` | routing keys | queues bind to keys nobody publishes — **silent** |
| `rabbitMq.queue.template` | queue names | new queues declared; old ones orphaned with unconsumed messages |
| `consul.*` / `fabio.*` | discovery | only affects inbound resolution, which nothing uses (§3.37) |
| `redis.*` | nothing (§3.35) | a spurious startup dependency |
| `jwt.*` | nothing, because authentication is not enabled (§3.34) | none today |
| `logger.*` | Serilog | log destinations |
| `ASPNETCORE_ENVIRONMENT` | config layering | unset ⇒ a profile that works nowhere (§3.38) |

### 6.4 What the component exposes to other code

Nothing. It publishes no library, and its `Commands/External` and `Events/External` types are
private re-declarations rather than a shared contract package (§3.20, Q7).

---

## 7. Change/extension guide

Ordered by likely need. Each entry names the files, the order of operations, and what silently
accepts an incomplete change.

### 7.1 Add a step to the saga

1. Define or copy the contract into `Commands/External/` (outbound) or `Events/External/` (inbound),
   with the correct `[Message("exchange")]`. **Verify property names against the peer's declaration
   by reading that file** (§3.20).
2. Add the field(s) the step produces to `AIMakingOrderData`.
3. Add `ISagaAction<TEvent>` to the `AIOrderMakingSaga` declaration (`Sagas/AIOrderMakingSaga.cs:16-21`).
4. Add a `ResolveId` arm (`:45-54`) — **the message must carry the order id** (§3.11).
5. Implement `HandleAsync`, copying the publish block shape from `:103-109` (message context +
   `Saga` header).
6. Implement `CompensateAsync` with the **inverse effect**, not `Task.CompletedTask` (§3.16).
7. Add `IEventHandler<TEvent>` + a one-line body to `AIOrderMakingHandler`.
8. Add `.SubscribeEvent<TEvent>()` to `Extensions.UseApp` (`Extensions.cs:58-62`).

*Silently accepted if you skip:* step 4 (message resolves to an unknown saga, dropped), step 6
(compilation succeeds, compensation does nothing), step 8 (the queue is never declared, messages
never arrive), or step 3 while doing step 7 (exactly the `ResourceReserved` situation, §3.25).
*Loud if you skip:* step 7 while doing step 8 — handler resolution fails on the first message.

### 7.2 Support multiple parcels per order

1. `Commands/MakeOrder.cs`: `Guid ParcelId` → `IEnumerable<Guid> ParcelIds`.
2. `Sagas/AIOrderMakingSaga.cs:59`: `Data.ParcelIds.Add(...)` → `AddRange`.
3. **`Sagas/AIMakingOrderData.cs:16`: fix the predicate to `ParcelIds.Any() &&
   ParcelIds.All(AddedParcelIds.Contains)`** (§3.13).
4. Add an idempotency guard at `Saga:87`: `if (Data.VehicleId != Guid.Empty) return;`.

*Silently accepted if you skip step 3:* the saga assigns a vehicle after the **first** parcel and
then re-assigns — and re-reserves — once per remaining parcel. This is the most likely way to break
the component while appearing to extend it.

### 7.3 Make the saga durable

1. Add a Chronicle persistence package to `.csproj` and configure it at `Extensions.cs:43`
   (`AddChronicle(o => …)`).
2. Add its connection settings to **all four** `appsettings*.json` files — remembering that
   `appsettings.development.json` is empty by design (§3.38).
3. Add idempotency guards **first** (§5.4): with a durable store, redelivered messages now find live
   state and will re-execute steps that previously fell into the void.
4. Decide the migration policy for `AIMakingOrderData` (§5.2) before the first shape change.
5. Only then consider more than one replica (§3.14).

*Silently accepted if you skip step 3:* the change makes redelivery worse, not better.

### 7.4 Fix correlation, tracing and the identity bypass

Do these together; individually they accomplish little.

1. **Delete `Sagas/AIOrderMakingSaga.cs:39-42`** (§3.18).
2. Replace `SagaContext.Empty` in `Handlers/AIOrderMakingHandler.cs` with a context built from
   `IMessagePropertiesAccessor`, so inbound AMQP context flows into the saga (§3.9).
3. Build a `CorrelationContext` from the HTTP request for the `MakeOrder` path, mirroring
   `Orders/.../Infrastructure/Extensions.cs:113-116`.
4. Add `Convey.Tracing.Jaeger` + `.UseJaeger()` (§3.36).
5. Convert the six `LogInformation` calls to structured templates (§3.36).

*Expected consequences, both desirable:* `operations-service` starts recording these operations
(§3.31), and `availability-service`'s identity guard starts firing (§3.18) — which means step 6:
decide what principal the saga acts as. **Do not do 1–5 without deciding 6**, or reservations will
begin failing with `UnauthorizedResourceAccessException`.

### 7.5 Secure the endpoint

See §3.34's four-step procedure. Note that step 3 (validating `MakeOrder.CustomerId` against the
caller) is the only one that closes the "start a saga for someone else" hole; the other three are
prerequisites.

### 7.6 Add failure reporting

1. Wrap each `HandleAsync` body in `try/catch`.
2. On catch: publish `MakeOrderRejected(Data.OrderId, friendlyReason, stableCode)` with header
   `Saga=Rejected`, then `await RejectAsync()`, then rethrow (so Chronicle still compensates).
3. Define exception types and map them in `ExceptionToResponseMapper` (§3.33).
4. Log at `LogWarning`/`LogError` in every `CompensateAsync` (§3.36).

Everything downstream already works: `operations-service` subscribes to `make_order_rejected`
(§3.32).

### 7.7 Release reservations on failure

1. Add `ReleaseResourceReservation` to `Commands/External/` with `[Message("availability")]`,
   matching `availability-service`'s declaration (routed at
   `Availability/.../Api/Program.cs:45`).
2. Publish it from `CompensateAsync(VehicleAssignedToOrder)` using `Data.VehicleId` and
   `Data.ReservationDate`.
3. Guard against compensating a reservation that was never made (`Data.ReservationDate == default`).

This is the single highest-value correctness fix after §3.13.

### 7.8 Retire the component

1. Remove the `ordermaker-service` block from `hianshul100_Pacco/compose/services.yml:78-85`
   (a different repository).
2. Remove the `ordermaker-service` block from
   `Pacco.Services.Operations.Api/messages.json:75-83` (a different repository) — otherwise
   `operations-service` declares queues on an exchange nobody feeds.
3. Archive this repository.

*Consider first:* nothing depends on this service (§1.3), so retirement is clean. But it is the
platform's **only** worked example of orchestration and of the `Saga` header mechanism (§3.17) that
two other services implement handling for. Deleting it leaves that mechanism unexplained.

---

## 8. Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Entries are tagged **[ACTION NOW]** (a human decision is required before dependent work proceeds)
> or **[handled later by \<stage\>]** (a named later stage owns it). Nothing is smoothed over: each
> entry states what could not be determined from source and what would settle it.

### 8.1 Assumptions

| # | Assumption | Rationale | Impact if wrong | Validation path |
| --- | --- | --- | --- | --- |
| A1 | Convey `0.4.*` behaves as the call sites imply: `[Message("x")]` selects the exchange, `conventionsCasing: snakeCase` derives routing keys, `Post<T>` returns 202, `AddSecurity` alone does not enforce authentication | Consistent usage across all thirteen Pacco repositories; identical patterns in services whose behaviour is documented elsewhere in this inventory | Routing, response codes and the §3.34 finding would all need revisiting | Read the Convey `0.4.*` sources, or observe declared queues in the RabbitMQ management UI |
| A2 | Chronicle discovers `Saga<TData>` implementations by assembly scan; `AddChronicle()` alone is sufficient to register `AIOrderMakingSaga` | The saga is never registered explicitly (`Extensions.cs:43-46` registers only three services), yet the system is presented as working | If not, the service throws on the first `MakeOrder` and has never worked | Run the service and post to `/orders` |
| A3 | Queue names follow `ordermaker-service/{{exchange}}.{{message}}` with snake_case message names | `appsettings.json:93,123` plus Convey's documented templating | The queue-name table in §3.4 is wrong; nothing else changes | RabbitMQ management UI after startup |
| A4 | Chronicle silently ignores a message for which no `ISagaAction<T>` exists, and a non-start action for an unknown saga id | Required for §3.25 and §4.4 to be non-fatal; no evidence of errors in this shape | If it throws instead, `ResourceReserved` would be a visible error, and post-restart redelivery would produce a loud failure — **which would be an improvement** | Chronicle 3.2.1 source, or send a `ResourceReserved` and watch the logs |
| A5 | Chronicle does not serialise concurrent `ProcessAsync` calls for one saga id | Not verifiable from this repository | With multiple parcels, `Data.AddedParcelIds` (`List<Guid>`) could corrupt under concurrent adds (§3.9) | Chronicle source; or a load test once §7.2 lands |
| A6 | Convey's RabbitMQ subscriber logs and drops (rather than requeues) a message whose handler throws, since no dead-letter exchange is configured | No `deadLetter` key exists in any `appsettings*` file | If it requeues indefinitely, a poison message becomes a hot loop rather than a silent loss — a different but equally serious failure | Convey source; or force an exception and watch queue depth |
| A7 | Re-processing `ISagaStartAction<MakeOrder>` for an existing saga id does not create a second saga | Assumed for §3.5's duplicate-post discussion | A duplicate `POST /orders` with an explicit id could fork the workflow | Chronicle source |
| A8 | Chronicle's default saga state repository and log are in-process, non-durable | No persistence package is referenced (`.csproj:9-27`); no store is configured | If Chronicle defaulted to a durable store it would still need configuration, which is absent — so the conclusion holds either way | Chronicle source |
| A9 | Whether `OrderApproved` carries the `Saga` header depends on `orders-service`'s event-publishing path, which forwards headers only for messages it handles directly | `GetHeadersToForward` exists (`Orders/.../Extensions.cs:118-132`) but `ResourceReservedHandler` publishes via `IEventMapper`/`IMessageBroker` | Affects only §3.17's chain analysis, not this component's behaviour | Read `Orders/.../Infrastructure/MessageBroker.cs` |
| A10 | `AddRedis()` opens a connection at startup and may fail if Redis is unavailable | Convey's registration shape | §3.35's "spurious hard dependency" claim would soften to "spurious lazy dependency" | Start the service with Redis stopped |
| A11 | `dotnet test` exits 0 in a solution with no test project | Standard .NET CLI behaviour; Travis is green | If it exited non-zero, CI would be red and this would have been noticed | `./scripts/test.sh` |
| A12 | `netcoreapp3.1` is out of support | External fact; no lifecycle or migration plan exists in the repository | A framework migration is more or less urgent than implied | Platform owner |

### 8.2 Blockers

| # | Blocker | Blocks | Owner | Resolution path |
| --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** **Unauthenticated saga trigger composed with a bypassed identity guard.** `POST /orders` requires no authentication (§3.34, no `UseAuthentication`), and the blank correlation context (§3.18) makes `availability-service`'s `identity.IsAuthenticated` check fall through (`Availability/.../ReserveResourceHandler.cs:29-33`). An anonymous caller can reserve platform capacity in any customer's name. The container publishes port 5015 to the host (`compose/services.yml:82`) and is not behind the API gateway. | Any deployment of this service outside a sealed demo environment | Platform security owner, with the service owner | §7.5 then §7.4, in that order. Interim mitigation: remove the host port mapping from `compose/services.yml:81-82`. |
| B2 | **[ACTION NOW]** **In-memory saga state combined with durable queues and `restart: unless-stopped`.** A restart silently abandons every in-flight workflow, leaving orders half-built upstream with no compensation and no alert (§3.14, §4.4). | Treating this service as anything other than a demonstration; any scaling; any deploy during business hours | Owner of `hianshul100_Pacco.Services.OrderMaker` | Decide: add Chronicle persistence (§7.3) — Redis is already connected and unused (§3.35) — or formally mark the service demo-only and document that in its README. |
| B3 | **[handled later by architecture evolution]** **Credentials committed to source**: `rabbitMq.username/password` = `guest`/`guest` (`appsettings.json:97-98`, inherited by the docker profile) and `logger.seq.apiKey` = `"secret"` (`:67`). Unlike `availability-service`, this service does not call `.UseVault()`. | Any non-local deployment | Platform security owner | Adopt Vault as the sibling services do, or externalise to environment variables. Platform-wide, not specific to this component. |

### 8.3 Open questions

| # | Question | Why it matters | Proposed answer | Decision owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Is `ordermaker-service` a demonstration or an intended production path? | It gates B1, B2 and every other item here. A demo needs a README warning; a product needs §7.3–§7.7. | **Demonstration.** The evidence is overwhelming: no tests (§3.40), no persistence (§3.14), `// typical AI in a startup` (§3.28), a hard-coded `"Because I'm saga"` cancellation reason (§3.16), and no consumer anywhere in the platform (§1.3). Recommend documenting it as such in the repository README rather than hardening it. | Owner of `hianshul100_Pacco.Services.OrderMaker`. Extends baseline gap **G2**. |
| Q2 | **[ACTION NOW]** Should the inverted `AllPackagesAddedToOrder` predicate be fixed now, before multi-parcel support exists? | It is currently latent, but it is also the guard that would prevent a **redelivered** `ParcelAddedToOrder` from re-running vehicle selection today (§3.13). | **Yes** — fix the predicate *and* add the idempotency guard (§7.2 steps 3–4). Both are one-line changes and the second is reachable now. | Service owner |
| Q3 | **[handled later by high-level spec]** Should orchestration and choreography continue to coexist for order creation? | Both paths publish the same commands to the same exchange; the only distinguishing signal is the `Saga` header (§3.17), and it does not survive the availability hop. | Pick one. If orchestration wins, `orders-service`'s autonomous `ResourceReservedHandler` approval must be removed (§3.23) — which would break the non-saga flow, so the two decisions are the same decision. | Platform architecture owner. Relates to baseline question **Q11**. |
| Q4 | **[handled later by high-level spec]** Should `ApproveOrder` be published by the saga, or deleted? | An unused contract implies a capability that does not exist (§3.23), and its absence explains why `ResourceReserved` reaches no action (§3.25). | Delete `Commands/External/ApproveOrder.cs` and the inert `ResourceReserved` subscription + handler (§3.25 option 1) unless Q3 resolves toward full orchestration. Publishing it without removing Orders' autonomous approval would **double-approve every order**. | Service owner, after Q3 |
| Q5 | **[handled later by architecture evolution]** What is the `DateTime.Kind` contract for reservation dates across Pacco? | `ResourceReservationsService` mixes a `Utc` branch with a deserialised branch of unknown kind (§3.29), and the value is used as a reservation key in `availability-service`. A container in a non-UTC timezone could reserve the wrong day. | Normalise to UTC at every boundary; add `DateTimeZoneHandling.Utc` to the shared serializer settings. Platform-wide. | Platform architecture owner |
| Q6 | **[ACTION NOW]** Reconcile `patterns/orchestration/saga-process-manager.md`'s claim that the saga propagates correlation context. | That document is used as a reference for how the pattern works here. The call shape propagates a context; the constructor makes it empty (§3.18). A reader could reasonably conclude tracing works. | Amend the pattern document to describe the *intended* mechanism and cite §3.18 for the actual behaviour. **Deliberately not edited in this batch** — the maintenance contract assigns pattern documents to their own artifact, and changing it here would be an unreviewed edit to another team's baseline. | Owner of the architecture-inventory patterns set |
| Q7 | **[handled later by architecture evolution]** Should message contracts be shared packages instead of hand-copied types? | Two of six outbound commands have already drifted (§3.21, §3.22); the positional drift in `ReserveResource` is a live trap. `orders-service` publishes `/_contracts` and nobody consumes it. | Minimum: a CI check that fetches each peer's `/_contracts` and diffs against the local types. Better: a shared `Pacco.Contracts` package. Platform-wide. | Platform architecture owner |
| Q8 | **[handled later by architecture evolution]** What is the deploy procedure for a service holding in-flight workflow state? | A saga can be `Pending` indefinitely (§3.15) with no timeout, so "drain before deploy" is unachievable (§5.2). | Add a saga timeout that publishes `MakeOrderRejected` and compensates, then drain on that bound. Prerequisite for §7.3. | Service owner |

### 8.4 Explicitly unverifiable

| Claim | Status |
| --- | --- |
| Whether `ordermaker-service` has ever been exercised against the full platform | **`Unverifiable — Missing Source Evidence`** — no logs, no runbook, no integration test, no README mention beyond the generic template |
| Chronicle 3.2.1's exact behaviour for unknown saga ids, unbound message types, concurrency and duplicate start actions (A4, A5, A7) | **`Unverifiable — Missing Source Evidence`** in this workspace; the package source is not present. Every conclusion resting on these is labelled `[chronicle]` in §3 and repeated as an assumption above. |
| Whether Convey's RabbitMQ subscriber requeues or drops on handler exception (A6) | **`Unverifiable — Missing Source Evidence`** — package source absent; no dead-letter configuration exists to disambiguate |
| Whether `OrderApproved` carries the `Saga` header back to this service (A9) | **`Unverifiable — Missing Source Evidence`** without reading Orders' `IMessageBroker` implementation; it does not affect this component's behaviour, since the header is never read here (§3.17) |
| Whether CAKE (tenant `Q5SCXYFS`) holds governance for this component | **No.** A graph query scoped to `data_scope IN [$tenant_code, 'global']` returned **0 nodes for the entire tenant**, and a loosened retry (node count, no filters) also returned 0. The `cake_search` fallback returned content from an unrelated domain. There is **no ADR, decision, constraint or catalog record** for `ordermaker-saga-service` — consistent with every prior batch in this repository. Recorded in `cake_influence_report.json` as Case B. |

---

*End of `ordermaker-saga-service` component-internals model. Maintenance contract: any later phase
that changes this component's internals — the saga, its data, its handlers, its contracts, its
wiring or its configuration — must update this document in the same change. The companion artifact
for this batch is `operations-grpc-client.md`; the related pattern artifact is
`../patterns/orchestration/saga-process-manager.md`, which Q6 asks to be reconciled with §3.18.*
