# Component internals — `operations-service`

| | |
| --- | --- |
| **Component** | `operations-service` |
| **Source repository** | `hianshul100_Pacco.Services.Operations` (read-only clone; inspected, never modified) |
| **Scoped path** | `src/Pacco.Services.Operations.Api` |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 3 of 7 |
| **Status** | New artifact — no prior `component-internals/operations-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` and `baselines/api-inventory.md` remain valid and are **complemented**, not replaced. This document **resolves baseline gap G4 / open question Q4** ("where does operations-service keep operation state?" — answer: Redis, §3.4), **refines G5/Q14** (field-less emitted types, §3.11), and **corrects a baseline count** (`messages.json` totals, §3.8, §8.4). |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** The scoped path contains the service in full: 27 C# files, four
> `appsettings*.json` profiles, `messages.json`, `Operations.proto` and a three-file `wwwroot/ui`.
> A sibling project, `src/Pacco.Services.Operations.GrpcClient`, is a console sample outside the
> scoped path; it is read here only as evidence of the gRPC contract's intended use (§3.31) and is
> not part of the deployed component. There is **no test project of any kind** — no `tests/`
> directory, no `*.Tests.csproj` — yet `scripts/test.sh` and `.travis.yml:12-14` still invoke
> `dotnet test` (§3.45). `Convey 0.4.*` supplies the RabbitMQ subscriber, `MessageAttribute`,
> `ICorrelationContextAccessor`, `IMessagePropertiesAccessor`, `IJwtHandler`, the Redis
> `IDistributedCache` registration, the WebApi endpoint builder, Consul, Fabio, Jaeger, AppMetrics
> and Vault; it is a NuGet reference with **no source in this workspace**, so its mechanisms are
> marked `[convey]` and, where their exact semantics change a conclusion, flagged
> `Unverifiable — Missing Source Evidence`. ASP.NET Core and gRPC primitives are marked
> `[framework]` on the same basis; the Ntrada gateway's correlation-context payload is marked
> `[ntrada]`. The producers of every message this service consumes live in the other twelve
> service clones and are modelled in their own `component-internals/*.md` documents; the gateway
> half of the contract is modelled in `component-internals/api-gateway.md`.

---

## Contents

1. [Purpose & boundary](#1-purpose--boundary)
2. [Core concepts (exhaustive)](#2-core-concepts-exhaustive)
3. [Per concept](#3-per-concept)
4. [Primary control flows](#4-primary-control-flows)
5. [Persistence & schema evolution](#5-persistence--schema-evolution)
6. [Surface → internals map](#6-surface--internals-map)
7. [Change/extension guide](#7-changeextension-guide)
8. [Assumptions, Blockers & Open Questions](#8-assumptions-blockers--open-questions)

---

## 1. Purpose & boundary

### 1.1 What this component is responsible for

`operations-service` is the platform's **asynchronous-request status board**. In the async gateway
profile, a client that issues a write does not get a result — it gets `202 Accepted` and a
correlation id (`api-gateway.md` §3.7). `operations-service` exists to answer the only question
that follows: *what happened to that request?*

It answers it in three ways simultaneously, from one piece of state:

1. **Pull** — `GET /operations/{operationId}` returns the current status
   (`Program.cs:32-43`).
2. **Push** — a SignalR hub streams `operation_pending` / `operation_completed` /
   `operation_rejected` to the requesting user's browser (`Services/HubService.cs:15-45`).
3. **Stream** — a gRPC server-streaming RPC pushes the same updates to non-browser clients
   (`Infrastructure/GrpcServiceHost.cs:32-41`).

Four properties make it structurally unlike every other service in the workspace:

1. **It has no domain and no aggregates.** There is no `Core` project, no entity, no repository,
   no value object and no domain exception. The single data shape is
   `DTO/OperationDto.cs:5-13`, a six-property mutable bag of strings.
2. **It subscribes to the entire platform, generically.** It does not declare a single message
   contract in C#. It reads a declarative manifest, `messages.json`, and **generates the CLR types
   at runtime with `System.Reflection.Emit`** (`Infrastructure/Subscriptions.cs:39-83`), then binds
   all 80 of them to three generic handlers. Adding a message to the status board is a JSON edit,
   not a code change (§3.8, §3.9).
3. **It is a pure consumer.** It declares a RabbitMQ exchange named `operations`
   (`appsettings.json:125-131`) and **never publishes anything to it** — there is no
   `IBusPublisher`, no `IMessageBroker`, no outbox and no event type in the repository (§3.43).
4. **It is the only component in the platform that ships a user interface.**
   `wwwroot/ui/{index.html, js/app.js, js/signalr.js}` is served by `UseStaticFiles()`
   (`Infrastructure/Extensions.cs:88`) and is the only front-end code in all fourteen repositories
   (§3.39).

### 1.2 What it explicitly is not

| It is not | Evidence |
| --- | --- |
| A saga orchestrator | It never sends a command, never compensates, never decides anything. It records what other services announce. There is no `IBusPublisher` reference in the scoped path. |
| A durable audit log | State lives in Redis with a **300-second sliding expiry** (`appsettings.json:149-151`, `Services/OperationsService.cs:50-55`). Anything idle for five minutes is gone (§3.5). |
| A MongoDB-backed service | `AddMongo()` is called (`Infrastructure/Extensions.cs:71`) and `mongo.database: "operations-service"` is configured (`appsettings.json:98-102`), but **no `AddMongoRepository<,>` call exists anywhere** and no code injects a Mongo type. The whole block is inert (§3.37). |
| An authenticated API | `UseInfrastructure` (`Infrastructure/Extensions.cs:81-93`) calls **neither `UseAuthentication()` nor `UseAccessTokenValidator()`**, and the gateway route is `auth: false` (`ntrada.yml:281-284`). The only token check in the whole service is inside the SignalR hub (§3.23, §8.2/B1). |
| A CQRS service | `AddCommandHandlers()`, `AddEventHandlers()` and `AddQueryHandlers()` are all registered (`Infrastructure/Extensions.cs:64-66`) but no dispatcher is registered and none is ever resolved. `Queries/GetOperation.cs` has **no handler** (§3.32). |
| Highly available | The gRPC stream and the `OperationUpdated` event are **per-process, in-memory** (`Infrastructure/GrpcServiceHost.cs:15,21`). Only the SignalR path is backplane-aware (§3.25, §3.29). |

### 1.3 The three delivery channels, side by side

| | HTTP pull | SignalR push | gRPC stream |
| --- | --- | --- | --- |
| Entry point | `Program.cs:32-43` | `Hubs/PaccoHub.cs:18-42` | `Infrastructure/GrpcServiceHost.cs:24-41` |
| Route / address | `GET operations/{operationId}` | `/pacco` (`Program.cs:46`) | `MapGrpcService<GrpcServiceHost>()` (`Program.cs:47`) |
| Authentication | **none** | JWT passed as a hub-method argument (§3.23) | **none** |
| Scoping | any id, by anyone | per-user group (§3.22) | **all operations, to every subscriber** (§3.29) |
| Fan-out across replicas | N/A (reads shared Redis) | yes, via the Redis backplane (§3.25) | **no** — in-process only |
| Reachable in Docker | yes (`5005:80`) | yes | **no** (§3.44) |
| Consumed by | the gateway `operations` module (`ntrada.yml:277-289`) | `wwwroot/ui/js/app.js:6-44` | `src/Pacco.Services.Operations.GrpcClient` (sample only) |

### 1.4 Position in the platform

| Relationship | Detail | Evidence |
| --- | --- | --- |
| Consumes from | **all eight** message-producing services: availability, customers, deliveries, identity, ordermaker, orders, parcels, vehicles | `messages.json:1-152` |
| Produces to | nothing | no publisher anywhere in the scoped path |
| Depends on (runtime) | RabbitMQ (consume), Redis (state **and** SignalR backplane) | `Infrastructure/Extensions.cs:70,72,75`, `:106` |
| Depends on (config only, inert) | MongoDB | `Infrastructure/Extensions.cs:71`, `appsettings.json:98-102` |
| Depended on by | the API gateway (`operations` module), the browser UI, any gRPC client | `ntrada.yml:277-289`; `wwwroot/ui/js/app.js:7` |
| Startup ordering | the **only** service in `compose/services.yml` with a `depends_on`, listing **eight** services | `hianshul100_Pacco/compose/services.yml:51-67` |
| Cross-repo coupling | the JWT subject format emitted by `identity-service` (`…Identity.Infrastructure/Auth/JwtProvider.cs:21`, `ToString("N")`) determines the SignalR group name | §3.22, `component-internals/identity-service.md` §6.4 |

That `depends_on` block is worth dwelling on: it declares startup ordering against every service
whose messages appear in `messages.json`, even though the service needs none of them to be up —
RabbitMQ queue bindings are declared by this service, not by the producers. The dependency is
**conceptual, not technical**, and it makes `operations-service` the last container to start.

---

## 2. Core concepts (exhaustive)

Every distinct idea a maintainer must hold to change this component safely. The list is complete
with respect to the scoped path: every `.cs`, `.json`, `.proto`, `.js` and build file has been read,
and each is accounted for by at least one row.

| § | Concept | Kind | Primary source |
| --- | --- | --- | --- |
| 3.1 | **Operation** — the only data shape in the service | data | `DTO/OperationDto.cs:5-13` |
| 3.2 | `OperationState` — the three-value lifecycle | data | `Types/OperationState.cs:3-8` |
| 3.3 | **Correlation id as operation identity** | identity | `Handlers/GenericEventHandler.cs:31,41` |
| 3.4 | `IOperationsService` / `OperationsService` — the Redis-backed store | service | `Services/OperationsService.cs:11-63` |
| 3.5 | `RequestsOptions` and the 300-second sliding expiry | config | `Infrastructure/RequestsOptions.cs:3-6`; `appsettings.json:149-151` |
| 3.6 | **The terminal-state latch** in `TrySetAsync` | invariant | `Services/OperationsService.cs:39-42` |
| 3.7 | `OperationUpdated` — the in-process fan-out event | seam | `Services/OperationsService.cs:22,57` |
| 3.8 | `messages.json` — the declarative subscription manifest | config | `messages.json:1-152` |
| 3.9 | `SubscribeMessages()` — the manifest reader and its three silent exits | wiring | `Infrastructure/Subscriptions.cs:19-59` |
| 3.10 | `BindMessages<T>` — runtime type emission via `Reflection.Emit` | wiring | `Infrastructure/Subscriptions.cs:61-83` |
| 3.11 | **Field-less emitted types** — payload discard, and the `RejectedEvent` exception | invariant | `Infrastructure/Subscriptions.cs:72`; `Types/RejectedEvent.cs:5-9` |
| 3.12 | Reflective `IBusSubscriber.Subscribe` and the three local `Handle` functions | wiring | `Infrastructure/Subscriptions.cs:85-152` |
| 3.13 | `IMessage` / `Command` / `Event` / `RejectedEvent` — the emitted types' base classes | data | `Types/{IMessage,Command,Event,RejectedEvent}.cs` |
| 3.14 | The three generic handlers, and their triplication | logic | `Handlers/Generic{Command,Event,RejectedEvent}Handler.cs` |
| 3.15 | **The blank-correlation-id early return** | invariant | `Handlers/GenericEventHandler.cs:32-35` |
| 3.16 | `GetSagaState()` — the `Saga` header decoder | logic | `Handlers/Extensions.cs:9-26` |
| 3.17 | The per-handler default state (Pending / Completed / Rejected) | invariant | `Handlers/*.cs:40` |
| 3.18 | `CorrelationContext` and `GetCorrelationContext` — the JSON round-trip | data | `Infrastructure/CorrelationContext.cs:6-24`; `Infrastructure/Extensions.cs:35-47` |
| 3.19 | Operation `Name` derivation — context name, else CLR type name | logic | `Handlers/*.cs:38` |
| 3.20 | `IHubService` / `HubService` — the three notification payload shapes | service | `Services/HubService.cs:6-46` |
| 3.21 | `IHubWrapper` / `HubWrapper` — the SignalR send seam | service | `Services/HubWrapper.cs:8-22` |
| 3.22 | `ToUserGroup` — two overloads, and the cross-repo `"N"` coupling | invariant | `Infrastructure/Extensions.cs:32-33` |
| 3.23 | `PaccoHub.InitializeAsync` — JWT authentication over the hub | security | `Hubs/PaccoHub.cs:18-42` |
| 3.24 | **The blank-token fall-through** in `InitializeAsync` | defect | `Hubs/PaccoHub.cs:20-23` |
| 3.25 | The SignalR Redis backplane and its silent degradation | wiring | `Infrastructure/Extensions.cs:95-109` |
| 3.26 | Legacy SignalR package versions on `netcoreapp3.1` | build | `Pacco.Services.Operations.Api.csproj:31-32` |
| 3.27 | `Operations.proto` — the gRPC contract | contract | `Operations.proto:1-24` |
| 3.28 | `GrpcServiceHost` — per-call instance, and the handler leak | service | `Infrastructure/GrpcServiceHost.cs:11-22` |
| 3.29 | `BlockingCollection` fan-out and the uncancellable `while (true)` | logic | `Infrastructure/GrpcServiceHost.cs:15,32-41` |
| 3.30 | `Map(null)` → an empty response instead of `NOT_FOUND` | defect | `Infrastructure/GrpcServiceHost.cs:43-54` |
| 3.31 | `Pacco.Services.Operations.GrpcClient` — the sample client | tooling | `src/Pacco.Services.Operations.GrpcClient/Program.cs` |
| 3.32 | `GetOperation` — a query type with no handler | dead code | `Queries/GetOperation.cs:6-9`; `Program.cs:32` |
| 3.33 | `Program.cs` — two `UseEndpoints` calls against two different builders | wiring | `Program.cs:28-48` |
| 3.34 | `ExceptionToResponseMapper` — the discard-all mapper | error handling | `Infrastructure/ExceptionToResponseMapper.cs:7-15` |
| 3.35 | `AddInfrastructure` — the composition root, and the duplicated `AddRedis()` | wiring | `Infrastructure/Extensions.cs:49-79` |
| 3.36 | `UseInfrastructure` — the middleware chain, and what is **absent** from it | wiring | `Infrastructure/Extensions.cs:81-93` |
| 3.37 | The inert MongoDB configuration | config | `Infrastructure/Extensions.cs:71`; `appsettings.json:98-102` |
| 3.38 | `AddJwt()`, the `.cer` certificate and the shared `issuerSigningKey` | security | `Infrastructure/Extensions.cs:63`; `appsettings.json:32-43`; `certs/localhost.cer` |
| 3.39 | `UseStaticFiles()` and `wwwroot/ui` — the platform's only front-end | surface | `Infrastructure/Extensions.cs:88`; `wwwroot/ui/js/app.js` |
| 3.40 | Configuration layering and the four profiles | config | `appsettings{,.local,.docker,.development}.json` |
| 3.41 | Vault integration | config | `appsettings.json:164-193`; `Program.cs:50` |
| 3.42 | Consul, Fabio, Jaeger and metrics | ops | `Infrastructure/Extensions.cs:68-69,73-74`; `appsettings.json:7-31,80-97` |
| 3.43 | The `operations` exchange — declared, never published to | wiring | `appsettings.json:125-131` |
| 3.44 | Deployment topology, and the unreachable gRPC port | ops | `Dockerfile:9`; `Properties/launchSettings.json:20`; `compose/services.yml:51-67` |
| 3.45 | The absent test suite | process | `.travis.yml:12-14`; `scripts/test.sh` |

**Concept map — how they compose.**

```
RabbitMQ ──▶ [3.9 manifest] ──▶ [3.10 Reflection.Emit] ──▶ 80 runtime types (3.11, 3.13)
                                                                  │
                                                    Convey subscriber (3.12)
                                                                  ▼
                                    ┌──────── 3.14 three generic handlers ────────┐
                                    │  3.15 blank-correlation-id guard            │
                                    │  3.16 Saga header  →  3.17 default state    │
                                    │  3.18 correlation context → 3.19 name       │
                                    └──────────────────┬──────────────────────────┘
                                                       ▼
                                     3.4 OperationsService.TrySetAsync
                                     (3.3 id · 3.5 expiry · 3.6 latch)
                                          │                     │
                        Redis `requests:{id}` (5.1)      3.7 OperationUpdated
                                          │                     │
                     ┌────────────────────┼─────────────────────┴────────────┐
                     ▼                    ▼                                  ▼
        3.32 GET /operations/{id}   3.20/3.21 SignalR → 3.22 group    3.28/3.29 gRPC stream
                     │                    │  (3.23 hub auth, 3.25 backplane)  │
                gateway (auth:false)   browser UI (3.39)             sample client (3.31)
```

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement** (with an explicit note on whether a violation fails
loudly or silently), **Extension procedure**, **Failure modes**.

### 3.1 `OperationDto` — the only data shape in the service

**Definition.** A six-property mutable class
(`DTO/OperationDto.cs:5-13`) that is simultaneously the domain model, the persistence model, the
HTTP response body and the source of the gRPC and SignalR payloads. There is no other data type in
the service.

```csharp
public class OperationDto
{
    public string Id { get; set; }        // the correlation id — §3.3
    public string UserId { get; set; }    // "" when unknown, never null — §3.4
    public string Name { get; set; }      // message type name or context name — §3.19
    public OperationState State { get; set; }
    public string Code { get; set; }      // "" unless rejected — §3.14
    public string Reason { get; set; }    // "" unless rejected
}
```

**Representation & storage.** Serialised with `JsonConvert.SerializeObject` (Newtonsoft defaults —
PascalCase property names, `State` as its **integer** ordinal) into a Redis string
(`Services/OperationsService.cs:50-51`). See §5.2 for why the integer matters.

**Lifecycle.** Created on the first message for a correlation id
(`Services/OperationsService.cs:37`), mutated in place on each subsequent message (`:44-49`),
re-serialised, and destroyed by Redis when the sliding expiry lapses (§3.5).

**Invariants & enforcement.**

- **`Id` is always overwritten** with the method argument (`:44`), so it can never diverge from the
  Redis key.
- **`UserId`, `Code` and `Reason` are coalesced to `string.Empty`** (`:45`, `:48`, `:49`). They are
  never `null` in a persisted document. `Name` is **not** coalesced (`:46`) — but §3.19 shows it
  cannot be `null` on any reachable path.
- **Nothing validates anything.** There are no attributes, no constructor, no guards. Every
  property is publicly settable. A violation is not detected at all — it is **silent** by
  construction.
- The type has no equality, no `ToString`, and no versioning field.

**Extension procedure.** Add the property here, then follow it through five places, in this order:
`Services/IOperationsService.TrySetAsync` (the parameter list, `Services/IOperationsService.cs:13-14`),
the three generic handlers' call sites (`Handlers/*.cs:41`), `Services/OperationsService.TrySetAsync`
(`:44-49`), `Operations.proto`'s `GetOperationResponse` **plus** `GrpcServiceHost.Map` (`:46-53`),
and the relevant `HubService` payload (`Services/HubService.cs:18-22`, `:28-32`, `:38-44`). Missing
any of the last three is a **silent** omission — the field simply never reaches that channel.

**Failure modes.** Because the same class serves persistence and three wire formats, a change made
for one channel silently changes the others. In particular, adding a property changes the Redis
payload, which existing cached entries do not have (§5.4).

### 3.2 `OperationState` — the three-value lifecycle

**Definition.** `Types/OperationState.cs:3-8`:

```csharp
public enum OperationState { Pending, Completed, Rejected }
```

**Representation & storage.** As the **integer ordinal** in Redis (Newtonsoft's default for enums
without `StringEnumConverter`), as the integer in the HTTP JSON response
(`Program.cs:42` writes the DTO directly), and as a **lower-cased string** over gRPC
(`Infrastructure/GrpcServiceHost.cs:53` — `operation.State.ToString().ToLowerInvariant()`). Three
representations of one value; only the gRPC one is human-readable.

**Lifecycle.** Set on every `TrySetAsync` call; `Completed` and `Rejected` are terminal (§3.6).

**Invariants & enforcement.**

- The state machine is **not** enforced as a machine. `TrySetAsync` accepts any state at any time;
  the only constraint is the terminal latch (§3.6). `Pending → Pending` is legal and common (a
  workflow that emits several commands under one correlation id).
- The `switch` in each handler (`Handlers/*.cs:47-60`) has a `default` arm that throws
  `ArgumentException` — the one **loud** failure in the whole service. It is unreachable today
  (the enum has exactly three values and all three are handled), but it would fire the moment a
  fourth value is added without updating all three handlers.

**Extension procedure — adding a state.** Add the value here, then: all three handlers' `switch`
statements (`Handlers/*.cs:47-60`), a corresponding `IHubService` method
(`Services/IHubService.cs:8-10`) and its `HubService` implementation, the `app.js` listener list
(`wwwroot/ui/js/app.js:34-44`), and — critically — **append the value at the end of the enum**, or
every cached Redis entry silently reinterprets its integer as a different state (§5.4).

**Failure modes.** The ordinal serialisation is the trap: reordering or inserting an enum member is
a **silent** data-corruption change for every live operation.

### 3.3 Correlation id as operation identity

**Definition.** An operation is not created by anyone — it *is* the correlation id that the API
gateway generated for an inbound request, seen from the message bus. Every handler reads
`_messagePropertiesAccessor.MessageProperties?.CorrelationId` (`Handlers/*.cs:30-31`) and passes it
straight to `TrySetAsync` as the id (`Handlers/*.cs:41`).

**Representation & storage.** A string. The Redis key is `$"requests:{id}"`
(`Services/OperationsService.cs:62`), so with the configured instance prefix
`operations:` (`appsettings.json:145-148`) the physical key is `operations:requests:{id}`
`[convey]`.

**Where the id comes from.** The gateway sets `generateRequestId: true`
(`ntrada-async.yml:16`) and forwards a `message_context` header
(`ntrada-async.yml:76-78`); the correlation id it mints travels as the AMQP `CorrelationId`
property and is echoed to the client as the `Request-ID` response header
(`ntrada-async.yml:37-41` exposes it through CORS). The client then polls
`GET /operations/{that id}`. The precise field mapping inside Ntrada is `[ntrada]` and
`Unverifiable — Missing Source Evidence` from this workspace; what *is* verifiable here is that the
service reads `IMessageProperties.CorrelationId` and nothing else.

**Lifecycle.** Comes into existence when the first message bearing that correlation id is consumed;
ceases to exist 300 seconds after the last one (§3.5).

**Invariants & enforcement.**

- **A blank correlation id means the message is ignored entirely** (`Handlers/*.cs:32-35`) — a
  **silent** drop, with no log line. §3.15.
- Nothing namespaces the id. Two unrelated workflows that somehow shared a correlation id would
  share an operation, and the terminal latch (§3.6) would make the second one invisible.
- Nothing validates the id's shape — it is used verbatim in a Redis key. A correlation id
  containing whitespace or control characters is accepted.

**Extension procedure.** Do not change the key format lightly: `GetKey` is `private static`
(`Services/OperationsService.cs:62`) and is the only definition, but the `requests:` prefix is
undocumented anywhere else, so any operator tooling that inspects Redis by hand depends on it.

**Failure modes.** Two: the silent drop above, and the fact that an operation's identity is owned
by a component in a different repository (the gateway) with no shared contract type.

### 3.4 `IOperationsService` / `OperationsService` — the Redis-backed store

**This subsection resolves baseline gap G4 / open question Q4.**

**Definition.** The one stateful service (`Services/IOperationsService.cs:8-15`,
`Services/OperationsService.cs:11-63`), registered as a **singleton**
(`Infrastructure/Extensions.cs:58`). Three members: the `OperationUpdated` event (§3.7),
`GetAsync(string id)`, and `TrySetAsync(id, userId, name, state, code = null, reason = null)`
returning `(bool updated, OperationDto operation)`.

**Representation & storage.** `Microsoft.Extensions.Caching.Distributed.IDistributedCache`, backed
by Redis via Convey's `AddRedis()` (`Infrastructure/Extensions.cs:72`) `[convey]`. **There is no
other store.** `AddMongo()` is called at `:71` but no repository is ever registered or injected
(§3.37), so the `mongo` configuration block is dead weight.

**`GetAsync` (`:24-29`):**

```csharp
var operation = await _cache.GetStringAsync(GetKey(id));
return string.IsNullOrWhiteSpace(operation) ? null : JsonConvert.DeserializeObject<OperationDto>(operation);
```

A miss and a blank value are indistinguishable; both yield `null`, which the HTTP endpoint turns
into `404` (`Program.cs:36-40`) and the gRPC endpoint turns into an **empty response** (§3.30).

**`TrySetAsync` (`:31-60`), step by step:**

1. `GetAsync(id)` — read-modify-write begins (`:34`).
2. If absent, start from `new OperationDto()` (`:35-38`).
3. **Else if the existing state is `Completed` or `Rejected`, return `(false, operation)`
   immediately** — the terminal latch (`:39-42`, §3.6).
4. Overwrite all six fields, coalescing `UserId`, `Code` and `Reason` to `""` (`:44-49`).
5. `SetStringAsync` with `SlidingExpiration = TimeSpan.FromSeconds(_options.ExpirySeconds)`
   (`:50-55`).
6. `OperationUpdated?.Invoke(this, new OperationUpdatedEventArgs(operation))` (`:57`) — §3.7.
7. `return (true, operation)`.

**Lifecycle.** Singleton for the process lifetime. Note the consequence: `IDistributedCache` and
`RequestsOptions` are captured once, and the `OperationUpdated` event's invocation list is
process-wide and never trimmed (§3.28).

**Invariants & enforcement.**

- **The read-modify-write at steps 1–5 is not atomic.** There is no Redis `WATCH`, no Lua script,
  no optimistic version and no lock. Two messages for the same correlation id arriving on two
  consumer threads (or two replicas) can both read `Pending`, both pass the latch, and both write —
  **last writer wins, silently**. Because RabbitMQ delivers to a single consumer per queue but this
  service binds ~80 distinct queues, concurrent messages for one workflow are the normal case, not
  an edge case. §8.2/B4.
- Step 5 uses `SlidingExpiration`, so **every write extends the lifetime by another 300 seconds**;
  a long-running workflow that keeps emitting messages never expires, and a stalled one disappears.
- The `(bool updated, …)` tuple is the *only* signal that a write was suppressed, and all three
  handlers respond to `false` by returning without notifying (`Handlers/*.cs:42-45`) — again
  **silently**.

**Extension procedure.**

- To add a store operation (e.g. "list a user's operations"), note that `IDistributedCache` has no
  enumeration — you would need `IConnectionMultiplexer` directly, or a secondary index key.
- To make the update atomic, the smallest correct change is to replace `IDistributedCache` with
  StackExchange.Redis and perform step 3–5 inside a Lua script, keeping `GetKey` as the key.
- To make operations durable, wire the already-configured Mongo (§3.37) as a write-behind — but see
  §5.4, because nothing on this platform has migration tooling.

**Failure modes.** Lost updates under concurrency; total loss of state on Redis restart (no
persistence is configured — `hianshul100_Pacco/compose/mongo-rabbit-redis.yml` runs a stock
`redis` image); and the sliding-expiry semantics described in §3.5.

### 3.5 `RequestsOptions` and the 300-second sliding expiry

**Definition.** A one-property options class
(`Infrastructure/RequestsOptions.cs:3-6`: `public int ExpirySeconds { get; set; }`), bound from the
`requests` configuration section and registered as a singleton
(`Infrastructure/Extensions.cs:51-52`):

```csharp
var requestsOptions = builder.GetOptions<RequestsOptions>("requests");
builder.Services.AddSingleton(requestsOptions);
```

**Representation & storage.** `"requests": { "expirySeconds": 300 }` — present in
`appsettings.json:149-151` **and** `appsettings.docker.json:87-89`, absent from
`appsettings.local.json` and `appsettings.development.json` (which inherit the base value).

**Lifecycle.** Bound once at startup; the value is a plain `int` captured by the singleton
`OperationsService`. **Changing it requires a restart**; there is no `IOptionsMonitor`.

**Invariants & enforcement.**

- **If the `requests` section were missing or misspelled, `GetOptions<T>` returns a
  default-constructed object** `[convey]`, giving `ExpirySeconds == 0` and therefore
  `SlidingExpiration = TimeSpan.Zero`. What Redis does with a zero sliding expiration is
  `Unverifiable — Missing Source Evidence` — either immediate expiry or no expiry, both of which
  are catastrophic and **neither of which logs anything**. This is the single most dangerous
  configuration typo in the service.
- Nothing validates that the value is positive.
- Five minutes is a **sliding** window measured from the last read *or* write `[framework]` —
  `IDistributedCache.GetStringAsync` refreshes it too. So a client polling
  `GET /operations/{id}` every few seconds keeps its own operation alive indefinitely, while a
  client that stops polling loses the record five minutes later even if the workflow is still
  running.

**Extension procedure.** Change the value in `appsettings.json:150` **and**
`appsettings.docker.json:88` — the two are independent copies and will drift. If you need
per-message-type expiry, `TrySetAsync` would have to take the expiry as a parameter; the options
object is currently consulted at exactly one site (`Services/OperationsService.cs:54`).

**Failure modes.** The zero-value trap above; and the general fact that **operation history is
undiscoverable after five idle minutes**, which makes post-hoc debugging of a failed workflow
impossible. Nothing is written to Mongo, to a log sink in structured form, or anywhere else.

### 3.6 The terminal-state latch

**Definition.** Three lines in `TrySetAsync` (`Services/OperationsService.cs:39-42`):

```csharp
else if (operation.State == OperationState.Completed || operation.State == OperationState.Rejected)
{
    return (false, operation);
}
```

**Representation & storage.** No storage of its own; it is a read of the persisted `State`.

**Lifecycle.** Evaluated on every `TrySetAsync` after the first.

**Invariants & enforcement.** This is **the service's only business rule**, and it encodes: *the
first terminal outcome wins.* Concretely:

| Sequence for one correlation id | Final stored state | Notifications sent |
| --- | --- | --- |
| `create_order` (cmd, Pending) → `order_created` (evt, Completed) | Completed | pending, completed |
| `create_order` (Pending) → `create_order_rejected` (Rejected) | Rejected | pending, rejected |
| `order_created` (Completed) → `order_delivering` (Completed) | **Completed, from the first** | completed **once** — the second is silently dropped |
| `create_order_rejected` (Rejected) → `order_created` (Completed) | **Rejected** | rejected only |

Row 3 is the important one: **a multi-step workflow that reports `Completed` at an intermediate
step becomes permanently frozen at that step.** Since events default to `Completed` (§3.17), any
workflow that emits more than one event under one correlation id shows only its first event. The
`Saga` header (§3.16) exists precisely to let a producer say "this event is still `pending`" and
avoid latching — but that requires the producer to set the header, and nothing in this repository
can check whether it did.

Enforcement is **silent**: the suppressed update returns `false`, the handler returns
(`Handlers/*.cs:42-45`), and nothing is logged.

**Extension procedure.** To allow post-terminal updates (e.g. an order completed then cancelled),
either delete the `else if` — accepting that late-arriving out-of-order messages could reopen a
finished operation — or replace it with an explicit transition table keyed on
`(currentState, newState)`. There is no test to update, because there are none (§3.45).

**Failure modes.** Frozen operations (row 3); and the invisibility of the freeze — from the outside
it looks exactly like a workflow that stopped.

### 3.7 `OperationUpdated` — the in-process fan-out event

**Definition.** A CLR event on the singleton service
(`Services/OperationsService.cs:22`: `public event EventHandler<OperationUpdatedEventArgs> OperationUpdated;`),
raised at `:57` after every successful write, carrying the DTO in
`Services/OperationUpdatedEventArgs.cs:6-14`.

**Representation & storage.** An in-memory multicast delegate on a singleton. **Process-local.**

**Lifecycle.** Subscribed by `GrpcServiceHost`'s constructor (`Infrastructure/GrpcServiceHost.cs:21`)
and **never unsubscribed** (§3.28). Raised synchronously inside `TrySetAsync`, on the message
consumer's thread.

**Invariants & enforcement.**

- The null-conditional `?.Invoke` (`:57`) means "no subscribers" is a legal, silent no-op — which
  is the normal state when no gRPC client has ever connected.
- **Invocation is synchronous and unguarded.** If any subscriber throws, the exception propagates
  out of `TrySetAsync`, out of the handler, and into Convey's subscriber — after the Redis write
  has already succeeded. The current subscriber is `(s, e) => _operations.TryAdd(e.Operation)`,
  which does not throw in practice, so this is latent.
- **It is not the SignalR path.** SignalR notifications are pushed by the *handlers*
  (`Handlers/*.cs:47-60` → `IHubService`), not by this event. The event feeds gRPC only. A
  maintainer who adds a subscriber here will not affect the browser UI, and vice versa.

**Extension procedure.** To add a third push channel (webhooks, Kafka, …), subscribing here is the
natural seam — but do it from a singleton, not from a transient or scoped service, or you will
reproduce the leak in §3.28. Better: change `TrySetAsync` to publish to a bounded
`Channel<OperationDto>` that a hosted service drains.

**Failure modes.** The handler leak (§3.28); synchronous propagation of subscriber exceptions; and
process-locality — a gRPC client connected to replica A never sees updates processed by replica B.

### 3.8 `messages.json` — the declarative subscription manifest

**Definition.** A 152-line JSON file at the project root (`messages.json`) mapping each producing
service to its exchange and three lists of message names. It is the **entire** definition of what
this service listens to; there is not one message contract in C# anywhere in the scoped path.

**Shape** (`messages.json:2-23`, the `availability-service` block):

```json
"availability-service": {
  "exchange": "availability",
  "commands":       [ "add_resource", "delete_resource", "release_resource", "reserve_resource" ],
  "events":         [ "resource_added", "resource_deleted", "resource_reservation_released",
                      "resource_reservation_canceled", "resource_reserved" ],
  "rejectedEvents": [ "add_resource_rejected", "delete_resource_rejected",
                      "release_resource_rejected", "reserve_resource_rejected" ]
}
```

Bound to `Subscriptions.ServiceMessages` (`Infrastructure/Subscriptions.cs:154-160`), whose four
properties are `Exchange`, `Commands`, `Events`, `RejectedEvents` — matched case-insensitively by
Newtonsoft `[framework]`. The top-level dictionary key (`"availability-service"`) is **discarded**
(`Subscriptions.cs:45` destructures it as `_`); only the `exchange` value is used.

**The full manifest — verified counts.**

| Block | Lines | Exchange | Commands | Events | Rejected | Total |
| --- | --- | --- | --- | --- | --- | --- |
| `availability-service` | `:2-23` | `availability` | 4 | 5 | 4 | 13 |
| `customers-service` | `:24-39` | `customers` | 2 | 3 | 2 | 7 |
| `deliveries-service` | `:40-59` | `deliveries` | 4 | 4 | 3 | 11 |
| `identity-service` | `:60-74` | `identity` | 2 | 2 | 2 | 6 |
| `ordermaker-service` | `:75-83` | `ordermaker` | **0 (key absent)** | 1 | 1 | 2 |
| `orders-service` | `:84-118` | `orders` | 7 | 9 | 10 | 26 |
| `parcels-service` | `:119-133` | `parcels` | 2 | 2 | 2 | 6 |
| `vehicles-service` | `:134-151` | `vehicles` | 3 | 3 | 3 | 9 |
| **Total** | | **8 exchanges** | **24** | **29** | **27** | **80** |

> **Baseline correction.** `baselines/service-summaries.md` records 26 commands, 30 events and
> 31 rejected events for this manifest. The file as committed contains **24 / 29 / 27 = 80**. The
> counts above were derived by enumerating every array element in `messages.json:1-152`. See §8.4.

**Representation & storage.** A file on disk, read from the **process working directory** at
startup (`Subscriptions.cs:21`: `const string path = "messages.json";`). Under `scripts/start.sh`
the working directory is the project directory, so the file is found. In the container, the
`Microsoft.NET.Sdk.Web` default globs include `**/*.json` as `Content` with
`CopyToPublishDirectory` set `[framework]`, and `Dockerfile:7-8` sets `WORKDIR /app` with the
publish output copied there — so it is found. Neither fact is asserted by an explicit `.csproj`
entry: `Pacco.Services.Operations.Api.csproj` contains **no `messages.json` item**, only
`certs\**` (`:44-46`), a `wwwroot\ui\js` folder marker (`:36-38`) and the `Protobuf` item
(`:40-42`). The file's presence at runtime therefore rests on SDK defaults, not on intent.

**Lifecycle.** Read exactly once, during `UseInfrastructure` → `SubscribeMessages`
(`Infrastructure/Extensions.cs:90`). **There is no reload**; adding a message requires a restart.

**Invariants & enforcement.**

- **Nothing validates the manifest against reality.** A message name that no service publishes
  produces a queue that stays empty forever. Two verified instances in the identity block alone:
  `sign_in` (`:63`) is declared as a command, but `identity-service` subscribes only
  `SubscribeCommand<SignUp>()` and no one publishes `sign_in`; and `sign_in_rejected` /
  `sign_up_rejected` (`:70-73`) are unreachable in the producer
  (`component-internals/identity-service.md` §3.17, §3.29). Roughly a quarter of the declared
  identity messages can never arrive.
- Conversely, a message that *is* published but is **missing** from the manifest is silently
  invisible on the status board. Nothing detects this either.
- `ordermaker-service` (`:75-83`) omits the `commands` key entirely. `BindMessages` handles this
  with `if (messages is null) yield break;` (`Subscriptions.cs:64-67`) — a deliberate, safe null
  guard.
- The name must match the routing key the producer uses. Producers derive routing keys from CLR
  type names via `rabbitMq.conventionsCasing: "snakeCase"` `[convey]`, so `OrderCreated` becomes
  `order_created`. **A typo here is a silent no-op**, because the queue is still declared and bound
  — it simply never receives anything.

**Extension procedure — adding a message to the status board.**

1. Confirm the producer's exchange name and the snake_case form of its CLR type name.
2. Add the string to the correct array in the correct block. **Choose the array carefully** — it
   determines the default state (§3.17) and whether `Code`/`Reason` are captured (§3.11).
3. Restart the service. No code change, no rebuild of any other component.
4. Nothing else is needed: the queue, the binding and the CLR type are all created at startup.

**Failure modes.** Every failure in this concept is silent: a missing file (§3.9), a typo, a
message that no one publishes, or a published message that is not listed.

### 3.9 `SubscribeMessages()` — the manifest reader and its three silent exits

**Definition.** `Infrastructure/Subscriptions.cs:19-59`, an extension method on Convey's
`IBusSubscriber`, invoked as the last link of the middleware chain
(`Infrastructure/Extensions.cs:89-90`: `.UseRabbitMq().SubscribeMessages()`).

**The four guard clauses, verbatim:**

```csharp
const string path = "messages.json";
if (!File.Exists(path))            { return subscriber; }   // :22-25  — silent

var messages = File.ReadAllText(path);
if (string.IsNullOrWhiteSpace(messages)) { return subscriber; }  // :28-31 — silent

var servicesMessages = JsonConvert.DeserializeObject<IDictionary<string, ServiceMessages>>(messages);
if (!servicesMessages.Any())       { return subscriber; }   // :34-37  — silent
```

**Each of the first three exits leaves the service running, healthy, registered in Consul, serving
HTTP, and subscribed to absolutely nothing.** No log line, no warning, no failed health check. The
status board would simply never update, and every operation lookup would return `404`. This is the
highest-impact silent failure in the component (§8.2/B3).

Note also that the third guard would be preceded by a `NullReferenceException` if the JSON
deserialised to `null` (`servicesMessages.Any()` on a null reference), and by a
`JsonSerializationException` if the JSON were malformed — those two *would* fail loudly, crashing
startup. So the failure mode is asymmetric: **malformed JSON crashes; missing or empty JSON is
ignored.**

**The main loop (`:39-56`):**

```csharp
var commands = new List<Command>();
var events = new List<Event>();
var rejectedEvents = new List<Types.RejectedEvent>();
var assemblyName = new AssemblyName("Pacco.Services.Operations.Api.Messages");
var assemblyBuilder = AssemblyBuilder.DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.Run);
var moduleBuilder = assemblyBuilder.DefineDynamicModule(assemblyName.Name);
foreach (var (_, serviceMessages) in servicesMessages)
{
    var exchange = serviceMessages.Exchange;
    commands.AddRange(BindMessages<Command>(moduleBuilder, exchange, serviceMessages.Commands));
    events.AddRange(BindMessages<Event>(moduleBuilder, exchange, serviceMessages.Events));
    rejectedEvents.AddRange(BindMessages<Types.RejectedEvent>(moduleBuilder, exchange,
        serviceMessages.RejectedEvents));
}

SubscribeCommands(subscriber, commands);
SubscribeEvents(subscriber, events);
SubscribeRejectedEvents(subscriber, rejectedEvents);
```

One dynamic assembly, one dynamic module, `AssemblyBuilderAccess.Run` (never persisted to disk, so
the generated types are invisible to any decompiler or profiler that reads the file system).

**Representation & storage.** The emitted assembly lives in the process's default
`AssemblyLoadContext` for the lifetime of the process; it is collectible only if built with
`RunAndCollect`, which it is not.

**Lifecycle.** Once, at startup, synchronously, before the host begins serving.

**Invariants & enforcement.**

- **Type names must be unique across the whole manifest**, because all 80 types go into one module
  with `DefineType(message, …)` (§3.10). Two services declaring the same message name — say both
  `orders` and `deliveries` declaring `order_completed` — would make the second
  `DefineType` throw `ArgumentException: Duplicate type name within an assembly`, **crashing
  startup loudly**. The current manifest has no duplicates; this was verified by comparing all 80
  names.
- Insertion order of the outer dictionary determines nothing functionally, but it does determine
  which duplicate would win the race to be defined.

**Extension procedure.** If you ever need per-service namespacing (to allow duplicate message
names), change `DefineType(message, …)` at `:72` to
`DefineType($"{serviceKey}.{message}", …)` and stop discarding the dictionary key at `:45` — but
note that Convey derives the **queue name and routing key from the type name** `[convey]`
(`queue.template: "operations-service/{{exchange}}.{{message}}"`, `appsettings.json:132-138`), so
this would change every queue name and every binding.

**Failure modes.** The three silent exits; the duplicate-name crash; and the fact that all of this
happens inside a middleware-registration chain where a reader would not think to look for I/O.

### 3.10 `BindMessages<T>` — runtime type emission

**Definition.** `Infrastructure/Subscriptions.cs:61-83`. For each name in a list, it emits a public
CLR type deriving from `T` (`Command`, `Event` or `RejectedEvent`), decorates it with Convey's
`[Message]` attribute, instantiates it, and yields the instance.

**The code, verbatim (`:69-82`):**

```csharp
foreach (var message in messages)
{
    var type = typeof(T);
    var typeBuilder = moduleBuilder.DefineType(message, TypeAttributes.Public, type);
    var attributeConstructorParams = new[] {typeof(string), typeof(string), typeof(string), typeof(bool)};
    var constructorInfo = typeof(MessageAttribute).GetConstructor(attributeConstructorParams);
    var customAttributeBuilder = new CustomAttributeBuilder(constructorInfo,
        new object[] {exchange, null, null, true});
    typeBuilder.SetCustomAttribute(customAttributeBuilder);
    var newType = typeBuilder.CreateType();
    var instance = Activator.CreateInstance(newType);

    yield return instance as T;
}
```

**Representation & storage.** In-memory types in the dynamic module from §3.9.

**Lifecycle.** Lazily — it is an iterator, so nothing is emitted until `AddRange` enumerates it
(`Subscriptions.cs:48-51`). All three calls enumerate immediately, so the laziness is
inconsequential.

**Invariants & enforcement.**

- **The type name is the raw manifest string.** `order_created` becomes a CLR type literally named
  `order_created`. This is what makes Convey's snake_case convention resolve to the right routing
  key: the type name already *is* the routing key, so `conventionsCasing: "snakeCase"`
  (`appsettings.json:107`) is effectively a no-op on these types `[convey]`. A maintainer who
  "tidies" the manifest to PascalCase would silently rebind every queue.
- **The `[Message]` constructor arguments are `(exchange, null, null, true)`** (`:76`). The
  four-parameter overload is `(string exchange, string routingKey, string queue, bool external)`
  `[convey]` — so the routing key and queue name are left to convention (derived from the type
  name) and `external: true` marks the message as coming from another service. If Convey's
  parameter order or arity ever changes, `GetConstructor` returns `null` at `:74` and the next
  line throws `ArgumentNullException` — a **loud** failure, but one that would only appear on a
  package upgrade.
- **`Activator.CreateInstance` requires a public parameterless constructor**, which the
  `where T : class, IMessage, new()` constraint (`:62`) guarantees for the base and which
  `DefineType` inherits by emitting a default constructor `[framework]`.
- The emitted instance is used **only** to carry its `Type` to `MakeGenericMethod`
  (§3.12) — the object itself is never deserialised into or read from. Emitting an instance per
  type where a `Type` would do is wasteful but harmless.

**Extension procedure.** To give emitted types real fields, add `typeBuilder.DefineField` /
`DefineProperty` calls here — but the manifest currently carries no schema, so you would first need
to extend `ServiceMessages` (`:154-160`) with a per-message property list. That is the concrete fix
for §3.11 and for baseline gap G5/Q14.

**Failure modes.** A package upgrade breaking `GetConstructor`; manifest names that are not valid
CLR identifiers (a name containing a space or a `+` would throw at `DefineType`, loudly); and the
payload discard in §3.11.

### 3.11 Field-less emitted types — the payload discard

**This subsection refines baseline gap G5 / open question Q14.**

**Definition.** The types emitted by `BindMessages` (§3.10) have **no fields and no properties of
their own**. `DefineType(message, TypeAttributes.Public, type)` (`:72`) creates an empty subclass;
nothing in `Subscriptions.cs` ever calls `DefineField`, `DefineProperty` or `DefineMethod`.

**Consequently, the entire message body is discarded on deserialisation.** Convey's subscriber
deserialises the AMQP payload into the emitted type `[convey]`; with no matching members, every
JSON property is dropped. `order_created`'s `orderId`, `customerId` and so on never reach this
service.

**The one exception, and it is load-bearing.** The three base classes differ:

| Base | Source | Declared members |
| --- | --- | --- |
| `Types/Command.cs:5-7` | `class Command : ICommand, IMessage` | **none** |
| `Types/Event.cs:5-7` | `class Event : IEvent, IMessage` | **none** |
| `Types/RejectedEvent.cs:5-9` | `class RejectedEvent : IRejectedEvent, IMessage` | **`string Reason`, `string Code`** |

So every emitted rejected-event type inherits two settable properties, and Convey's deserialiser
populates them. `GenericRejectedEventHandler` reads them at
`Handlers/GenericRejectedEventHandler.cs:41-42` (`@event.Code, @event.Reason`) and stores them on
the DTO. **Rejection detail survives; command and event detail does not.** That is why the browser
UI can show a failure reason (`wwwroot/ui/js/app.js:42-44` renders the `operation` payload) but
cannot show an order id.

`Types/IMessage.cs:3-6` is a marker interface whose only purpose is the `where T : class, IMessage, new()`
constraint at `Subscriptions.cs:62`; its body carries the comment `// Marker`.

**Representation & storage.** N/A — the data does not exist after deserialisation.

**Lifecycle.** Per message.

**Invariants & enforcement.** None. The discard is **silent and total**: no log line records that
a payload was received and dropped. Nothing can distinguish a message with a rich body from an
empty one.

**Extension procedure — capturing a payload field.** Three options, in ascending order of effort:

1. **Narrow:** add the property to `Types/Event.cs` (or `Command.cs`), exactly as
   `RejectedEvent` does. Every emitted event type then gains it, and every producer that happens to
   send a JSON property of that name populates it. Crude but two lines, and it matches the existing
   precedent.
2. **Manifest-driven:** extend `ServiceMessages` (`Subscriptions.cs:154-160`) so each message can be
   an object with a property list, and emit `DefineProperty` calls in `BindMessages` (§3.10).
3. **Raw capture:** deserialise the body separately and stash it as a JSON string on the DTO. This
   requires access to the raw payload, which Convey's typed subscriber does not expose
   `[convey]` — `Unverifiable — Missing Source Evidence`.

**Failure modes.** The status board can only ever show *that* something happened, never *what* —
except for rejections. Any feature request of the form "show the order number on the status page"
is blocked on this concept.

### 3.12 Reflective `IBusSubscriber.Subscribe` and the three local `Handle` functions

**Definition.** Three near-identical private methods —
`SubscribeCommands` (`:85-106`), `SubscribeEvents` (`:108-129`), `SubscribeRejectedEvents`
(`:131-152`) — each of which reflects over the subscriber to find its generic `Subscribe` method
and invokes it once per emitted type.

**The pattern, from `SubscribeEvents` (`:115-128`):**

```csharp
var subscribeMethod = subscriber.GetType().GetMethod(nameof(IBusSubscriber.Subscribe));
if (subscribeMethod is null) { return; }        // :116-119 — silent

foreach (var message in messages)
{
    subscribeMethod.MakeGenericMethod(message.GetType()).Invoke(subscriber,
        new object[] {(Func<IServiceProvider, IEvent, object, Task>) Handle});
}

static Task Handle(IServiceProvider sp, IEvent @event, object ctx) =>
    sp.GetService<IEventHandler<IEvent>>().HandleAsync(@event);
```

**Why reflection is necessary.** `IBusSubscriber.Subscribe<T>` is generic in the message type
`[convey]`, and the message types do not exist until §3.10 has run. `MakeGenericMethod` is the only
way to call a generic method with a `Type` known only at runtime.

**The `Handle` local function is the crucial trick.** It resolves
`IEventHandler<IEvent>` — the **interface**, not the concrete emitted type — from the service
provider. That works because `AddInfrastructure` registers exactly three open-interface handlers
(`Infrastructure/Extensions.cs:53-55`):

```csharp
.AddTransient<ICommandHandler<ICommand>, GenericCommandHandler<ICommand>>()
.AddTransient<IEventHandler<IEvent>, GenericEventHandler<IEvent>>()
.AddTransient<IEventHandler<IRejectedEvent>, GenericRejectedEventHandler<IRejectedEvent>>()
```

So all 24 command types share one handler instance-type, all 29 events share another, and all 27
rejected events share a third. **This is why there are exactly three handlers for eighty
messages.** The registrations at `Infrastructure/Extensions.cs:64-66`
(`AddCommandHandlers()`, `AddEventHandlers()`, `AddQueryHandlers()`) are assembly scans `[convey]`
that find these same three classes plus nothing else.

Note the ordering constraint this creates: `IRejectedEvent` derives from `IEvent` `[convey]`, so a
rejected-event type satisfies both `IEventHandler<IEvent>` and `IEventHandler<IRejectedEvent>`.
Rejected events are routed correctly only because `SubscribeRejectedEvents` passes a
`Func<…, IRejectedEvent, …>` delegate whose body resolves the `IRejectedEvent` handler explicitly
(`:150-151`). The two registrations are distinct service keys, so there is no ambiguity — but a
maintainer who "simplifies" by deleting the third registration would silently route every rejection
through `GenericEventHandler`, losing `Code` and `Reason` and defaulting the state to `Completed`
instead of `Rejected` (§3.17).

**Representation & storage.** N/A.

**Lifecycle.** Once, at startup.

**Invariants & enforcement.**

- **`if (subscribeMethod is null) return;`** (`:93-96`, `:116-119`, `:139-142`) — repeated three
  times. If a Convey upgrade renamed or overloaded `Subscribe`, `GetMethod` would return `null` (or
  throw `AmbiguousMatchException` on an overload) and the service would **silently subscribe to
  nothing**. Same failure surface as §3.9.
- `Invoke` wraps any exception in `TargetInvocationException`, so a binding failure at startup
  produces a confusing stack trace — but it does fail loudly.

**Extension procedure.** These three methods are byte-identical apart from the interface type; a
single generic helper parameterised on `(TMessage, THandler)` would collapse them. If you do that,
preserve the `static` local functions — they avoid a closure allocation per subscription and, more
importantly, they are what pins the handler resolution to the interface rather than the concrete
type.

**Failure modes.** The silent `null` exit; `AmbiguousMatchException` on a Convey overload; and the
handler-registration coupling described above.

### 3.13 `IMessage`, `Command`, `Event`, `RejectedEvent` — the base types

**Definition.** Four tiny types in `Types/` that exist solely to give the emitted types
(§3.10) a base class and to satisfy Convey's marker interfaces.

| File | Declaration | Purpose |
| --- | --- | --- |
| `Types/IMessage.cs:3-6` | `public interface IMessage { }` | marker; satisfies `where T : class, IMessage, new()` (`Subscriptions.cs:62`) |
| `Types/Command.cs:5-7` | `public class Command : ICommand, IMessage { }` | base for the 24 command types |
| `Types/Event.cs:5-7` | `public class Event : IEvent, IMessage { }` | base for the 29 event types |
| `Types/RejectedEvent.cs:5-9` | `public class RejectedEvent : IRejectedEvent, IMessage { Reason; Code; }` | base for the 27 rejected types; **the only one with data** (§3.11) |

`ICommand`, `IEvent` and `IRejectedEvent` come from `Convey.CQRS.Commands` / `Convey.CQRS.Events`
`[convey]`.

**Representation & storage.** Compile-time types in the main assembly; each is the base of many
runtime types in the dynamic assembly.

**Lifecycle.** Static.

**Invariants & enforcement.** All three must be **public, non-sealed, non-abstract, and have a
public parameterless constructor** — `DefineType` with a base type requires an accessible
constructor `[framework]`, and `new()` requires it at compile time. Sealing any of them, or adding
a constructor parameter, breaks type emission at startup **loudly**.

**Extension procedure.** Adding a property here is the one-line way to capture a payload field for
an entire message category (§3.11 option 1). Note that `RejectedEvent` sets the precedent: its two
properties have public setters, which is what lets the deserialiser populate them.

**Failure modes.** Sealing or parameterising a base class; and the temptation to make `IMessage`
carry members, which would then have to be emitted on every generated type.

### 3.14 The three generic handlers

**Definition.** `Handlers/GenericCommandHandler.cs`, `Handlers/GenericEventHandler.cs` and
`Handlers/GenericRejectedEventHandler.cs` — three classes of 62–64 lines each that are **identical
except for three tokens**. They are the only business logic in the service.

**The differences, exhaustively:**

| | `GenericCommandHandler<T>` | `GenericEventHandler<T>` | `GenericRejectedEventHandler<T>` |
| --- | --- | --- | --- |
| Interface | `ICommandHandler<T>` | `IEventHandler<T>` | `IEventHandler<T>` |
| Constraint | `where T : class, ICommand` | `where T : class, IEvent` | `where T : class, IRejectedEvent` |
| Default state (`:40`) | `OperationState.Pending` | `OperationState.Completed` | `OperationState.Rejected` |
| `TrySetAsync` args (`:41`) | 4 | 4 | **6** — adds `@event.Code, @event.Reason` (`:41-42`) |

Everything else — the four constructor dependencies, the guard, the context read, the name
derivation, the `TrySetAsync` call, the `updated` check and the notification `switch` — is
character-for-character the same.

**Representation & storage.** Registered `AddTransient` (`Infrastructure/Extensions.cs:53-55`), so
one instance per message.

**Lifecycle — `HandleAsync`, common to all three:**

1. `var messageProperties = _messagePropertiesAccessor.MessageProperties;` (`:30`)
2. `var correlationId = messageProperties?.CorrelationId;` — blank ⇒ **`return`** (`:31-35`, §3.15)
3. `var context = _contextAccessor.GetCorrelationContext();` (`:37`, §3.18)
4. `var name = string.IsNullOrWhiteSpace(context?.Name) ? command.GetType().Name : context.Name;`
   (`:38`, §3.19)
5. `var userId = context?.User?.Id;` (`:39`) — **`null` when there is no context**
6. `var state = messageProperties.GetSagaState() ?? <default>;` (`:40`, §3.16, §3.17)
7. `var (updated, operation) = await _operationsService.TrySetAsync(...);` (`:41`)
8. `if (!updated) { return; }` (`:42-45`) — the terminal latch fired (§3.6); **silent**
9. `switch (state)` → one of three `IHubService` calls, `default` throws `ArgumentException`
   (`:47-60`)

**Invariants & enforcement.**

- **Step 2 is the only input validation in the service**, and it fails silently.
- Step 5 propagates `null` into `TrySetAsync`, which coalesces it to `""`
  (`Services/OperationsService.cs:45`). An operation with `UserId == ""` is stored and is
  **unreachable by SignalR**, because the group name becomes `users:` (§3.22) and no client is in
  that group. The HTTP and gRPC channels still see it. So a missing correlation context degrades
  push silently while pull keeps working.
- Step 9's `default` arm is the service's **only loud failure**; it is unreachable with the current
  three-value enum (§3.2).
- Note that step 6 reads `messageProperties.GetSagaState()` **without** a null-conditional, while
  step 2 read `messageProperties?.CorrelationId` **with** one. This is safe only because step 2
  already returned when `messageProperties` was `null` — a blank `correlationId` covers the null
  case. The asymmetry is fragile: reordering these lines would introduce a
  `NullReferenceException`.

**Extension procedure.** The three files should be one generic base with a virtual
`DefaultState`. Until then, **any behavioural change must be made three times**; the surest way to
introduce a divergence in this codebase is to edit one handler. If you add a message *category*
(e.g. "informational events" that never latch), you need: a fourth base type in `Types/`, a fourth
list in `ServiceMessages`, a fourth `BindMessages` call, a fourth `Subscribe*` method, a fourth
handler, and a fourth DI registration.

**Failure modes.** Triplication drift; the silent guards at steps 2 and 8; and the silent
degradation at step 5.

### 3.15 The blank-correlation-id early return

**Definition.** `Handlers/GenericEventHandler.cs:30-35` (and the identical code at `:30-35` of the
other two):

```csharp
var messageProperties = _messagePropertiesAccessor.MessageProperties;
var correlationId = messageProperties?.CorrelationId;
if (string.IsNullOrWhiteSpace(correlationId))
{
    return;
}
```

**Representation & storage.** N/A.

**Lifecycle.** Evaluated first, on every message.

**Invariants & enforcement.** The invariant is *an operation must have an id*, and it is enforced by
**dropping the message with no log line, no metric and no dead-letter**. From the outside this is
indistinguishable from the message never having been sent.

**When does it fire?** Whenever a producer publishes without an ambient inbound message and without
an HTTP correlation header. Traced concretely: `identity-service`'s `MessageBroker.PublishAsync`
resolves the correlation id from the ambient AMQP message first and the `Correlation-Context` HTTP
header second (`…Identity.Infrastructure/Services/MessageBroker.cs:55`, `:64-65`,
`component-internals/identity-service.md` §3.31). A `SignedUp` published from the **HTTP**
sign-up path has neither, so its correlation id is `null` — and this guard drops it. **Sign-up
therefore never appears on the status board in the synchronous gateway profile**, which is
consistent with `messages.json:66-69` declaring `signed_up` but the gateway routing sign-up as
`use: downstream` in all four manifests (`component-internals/identity-service.md` §3.29).

The same reasoning applies to any service-initiated event that is not a reaction to an inbound
message.

**Extension procedure.** The minimal improvement is a `_logger.LogWarning` here — but note the
handlers do **not** take an `ILogger` dependency (`Handlers/*.cs:18-26` list four constructor
parameters, none of them a logger). Adding one means editing all three constructors and all three
registrations are unaffected (they resolve by DI). A more useful change is a counter metric, since
AppMetrics is already registered (`Infrastructure/Extensions.cs:73`).

**Failure modes.** Silent, unbounded message loss with no observability whatsoever. This is the
mechanism behind "why doesn't my operation show up?" and it is unanswerable from logs.

### 3.16 `GetSagaState()` — the `Saga` header decoder

**Definition.** `Handlers/Extensions.cs:9-26`:

```csharp
public static OperationState? GetSagaState(this IMessageProperties messageProperties)
{
    const string sagaHeader = "Saga";
    if (messageProperties?.Headers is null || !messageProperties.Headers.TryGetValue(sagaHeader, out var saga))
    {
        return null;
    }

    return saga is byte[] sagaBytes
        ? Encoding.UTF8.GetString(sagaBytes).ToLowerInvariant() switch
        {
            "pending"   => OperationState.Pending,
            "completed" => OperationState.Completed,
            "rejected"  => OperationState.Rejected,
            _ => (OperationState?) null
        }
        : null;
}
```

**Representation & storage.** An AMQP header. RabbitMQ's .NET client surfaces string headers as
`byte[]` `[framework]`, which is why the `is byte[]` pattern is necessary — a header set as a plain
`string` by some other client would fall through to `null`.

**Lifecycle.** Read once per message, at `Handlers/*.cs:40`.

**Invariants & enforcement.** Four distinct ways to get `null`, all **silent**, all
indistinguishable from each other and from "no header":

1. No message properties or no headers at all.
2. No `Saga` key.
3. The value is not a `byte[]`.
4. The value decodes to something other than the three recognised words.

`null` means "use the handler's default" (§3.17), so an unrecognised or malformed header is
**exactly as good as no header** — a typo like `"complete"` silently yields the default rather than
an error.

**Who sets the header.** The producing services do, via their own `GetHeadersToForward`, which
forwards **only** the `Saga` header from an inbound message to every event published while handling
it (`…Identity.Infrastructure/Extensions.cs:120-134`,
`component-internals/identity-service.md` §3.33 — the same eight-times-duplicated method exists in
every service). The header therefore propagates along a causal chain: the gateway or the first
service in a saga sets it, and every downstream event inherits it. Nothing in *this* repository can
verify which producers actually set it; that is `Unverifiable — Missing Source Evidence` here and
is best answered from the producing services' documents.

**Extension procedure.** To add a state (§3.2), add a `switch` arm here **and** to all three
handlers. To make malformed headers loud, replace the `_ => (OperationState?) null` arm with a
throw — but note that a throw here escapes into Convey's subscriber and, with no error handling
around message consumption, would likely nack the message.

**Failure modes.** Four silent nulls; and the header's cross-service nature — the header name
`"Saga"` is a string constant repeated in nine repositories with no shared definition.

### 3.17 The per-handler default state

**Definition.** Line 40 of each handler, differing only in the coalesced value:

| Handler | Line | Default when no `Saga` header |
| --- | --- | --- |
| `GenericCommandHandler` | `Handlers/GenericCommandHandler.cs:40` | `OperationState.Pending` |
| `GenericEventHandler` | `Handlers/GenericEventHandler.cs:40` | `OperationState.Completed` |
| `GenericRejectedEventHandler` | `Handlers/GenericRejectedEventHandler.cs:40` | `OperationState.Rejected` |

**The model this encodes.** *A command means work has started; an event means work finished; a
rejected event means work failed.* For a single-step workflow this is exactly right and requires no
cooperation from producers. For a multi-step saga it is wrong by default, and the `Saga` header
exists to correct it (§3.16).

**Representation & storage.** N/A.

**Lifecycle.** Per message.

**Invariants & enforcement.** None — it is a coalesce. The consequence is the interaction with the
terminal latch (§3.6), which deserves stating plainly:

> **A multi-step workflow whose intermediate events do not carry `Saga: pending` will latch to
> `Completed` at its first event and show no further progress.**

This is not a hypothetical: `orders-service` declares nine events (`messages.json:95-105`) including
`order_created`, `order_approved`, `order_delivering` and `order_completed`. If all four fire under
one correlation id without a `Saga` header, the status board shows `Completed` after
`order_created` and ignores the rest. Whether the producers set the header is
`Unverifiable — Missing Source Evidence` from this repository.

**Extension procedure.** Changing a default is a one-token edit in one file — but do it knowing
that the *category* (command / event / rejected) is chosen in `messages.json` (§3.8), so an operator
can already re-categorise a message without touching code.

**Failure modes.** Premature latching, as above; and the asymmetry that a message listed in the
wrong `messages.json` array silently gets the wrong default.

### 3.18 `CorrelationContext` and the JSON round-trip

**Definition.** A local mirror of the platform's correlation-context shape
(`Infrastructure/CorrelationContext.cs:6-24`) plus an extension method that materialises it
(`Infrastructure/Extensions.cs:35-47`).

```csharp
public class CorrelationContext
{
    public string CorrelationId { get; set; }
    public string SpanContext { get; set; }
    public UserContext User { get; set; }
    public string ResourceId { get; set; }
    public string TraceId { get; set; }
    public string ConnectionId { get; set; }
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }

    public class UserContext
    {
        public string Id { get; set; }
        public bool IsAuthenticated { get; set; }
        public string Role { get; set; }
        public IDictionary<string, string> Claims { get; set; }
    }
}
```

**The round-trip (`Infrastructure/Extensions.cs:35-47`):**

```csharp
public static CorrelationContext GetCorrelationContext(this ICorrelationContextAccessor accessor)
{
    if (accessor.CorrelationContext is null) { return null; }
    var payload = JsonConvert.SerializeObject(accessor.CorrelationContext);
    return string.IsNullOrWhiteSpace(payload) ? null : JsonConvert.DeserializeObject<CorrelationContext>(payload);
}
```

Convey exposes the context as an untyped/`dynamic` object `[convey]`; serialising it and
deserialising into the local class is how the service re-types it. `identity-service` does the same
thing in `AppContextFactory` (`…Identity.Infrastructure/Contexts/AppContextFactory.cs:23-27`) —
the pattern is platform-wide.

**Representation & storage.** Transported as the AMQP `message_context` header
(`appsettings.json:139-142`), populated by the gateway (`ntrada-async.yml:76-78`,
`messageContext: {enabled: true, header: message_context}`) and forwarded by each producing service's
`MessageBroker`.

**Lifecycle.** Per message; resolved on demand at `Handlers/*.cs:37`.

**Invariants & enforcement.**

- Only **two** of the eight properties are ever read: `Name` (§3.19) and `User.Id` (§3.14 step 5).
  `Role`, `Claims`, `IsAuthenticated`, `ResourceId`, `TraceId`, `ConnectionId`, `SpanContext` and
  `CreatedAt` are dead in this service.
- A `null` context is handled by null-conditionals at both read sites, so it degrades rather than
  throws — **silently**, with the consequences in §3.14 step 5.
- **A property-name mismatch between the producer's context class and this one is silent**: the
  round-trip is by name, so a renamed field simply deserialises to `null`. The two classes are
  structurally identical today (compare `…Identity.Infrastructure/Contexts/CorrelationContext.cs:6-24`),
  but nothing enforces that, and they live in different repositories with no shared package.
- The `CreatedAt` round-trip goes through Newtonsoft's default `DateTime` handling, which applies
  local-time conversion unless configured `[framework]`. Since the value is never read, this is
  inert — but it would matter the moment someone displays it.

**Extension procedure.** To surface the role or claims on the status board, read them here and add
them to `OperationDto` (§3.1). Note that **this is the only place user identity enters the
service** — there is no authentication middleware (§3.36), so `context.User.Id` is trusted
transitively from whichever service built the context.

**Failure modes.** Silent cross-repo drift; a double serialisation cost per message; and the fact
that the service's notion of "who this operation belongs to" is an unauthenticated, unvalidated
string that arrived in a message header (§8.2/B2).

### 3.19 Operation `Name` derivation

**Definition.** One line, in each handler (`Handlers/GenericEventHandler.cs:38`):

```csharp
var name = string.IsNullOrWhiteSpace(context?.Name) ? @event.GetType().Name : context.Name;
```

**Representation & storage.** `OperationDto.Name`, surfaced on all three channels.

**Lifecycle.** Recomputed on every message and overwritten in the DTO (§3.1), so an operation's
displayed name is that of its **most recent** non-latched message.

**Invariants & enforcement.**

- The fallback `@event.GetType().Name` is the **emitted type's name**, which is the raw
  `messages.json` string (§3.10) — so an operation typically reads `order_created`, in snake_case,
  not a friendly label. The UI displays it verbatim (`wwwroot/ui/js/app.js:34-44` renders the
  whole payload as JSON).
- `context.Name` wins when present. What Ntrada puts there is `[ntrada]` and
  `Unverifiable — Missing Source Evidence`; based on the field's position in the shared context
  shape it is most likely the gateway route name.
- The value can never be `null`: `GetType().Name` is never null or blank, so the ternary always
  yields a non-blank string. This is why `TrySetAsync` does not coalesce `Name` the way it
  coalesces `UserId`, `Code` and `Reason` (`Services/OperationsService.cs:46`).

**Extension procedure.** For friendly names, the cheapest change is a display map in the UI
(`app.js`) rather than in the service, because the name is derived per message and the service has
no vocabulary beyond the manifest strings.

**Failure modes.** Name churn across a multi-message operation (each message renames it, until the
latch stops updates); and snake_case identifiers leaking to end users.

### 3.20 `IHubService` / `HubService` — the three notification payloads

**Definition.** A three-method interface (`Services/IHubService.cs:6-11`) and its implementation
(`Services/HubService.cs:6-46`), registered transient (`Infrastructure/Extensions.cs:56`). Each
method projects an `OperationDto` onto an anonymous type and hands it to `IHubWrapper`.

**The three payload shapes — note that they differ:**

| Method | SignalR message name | Payload |
| --- | --- | --- |
| `PublishOperationPendingAsync` (`:15-23`) | `"operation_pending"` | `{ id, name }` |
| `PublishOperationCompletedAsync` (`:25-33`) | `"operation_completed"` | `{ id, name }` |
| `PublishOperationRejectedAsync` (`:35-45`) | `"operation_rejected"` | `{ id, name, code, reason }` |

The message names are **lower_snake_case string literals**, written three times here and matched
three times in the browser (`wwwroot/ui/js/app.js:34`, `:38`, `:42`). There is no shared constant —
the contract between the service and its own UI is six string literals in two languages.

`UserId` is passed as the routing argument (`:16`, `:26`, `:36`) and is deliberately **not** in the
payload; the client already knows who it is.

**Representation & storage.** Anonymous types, serialised by SignalR's JSON protocol with
**camelCase by default** `[framework]`. The anonymous-type members are already lower-case
(`id`, `name`, `code`, `reason`), so the wire names are stable either way.

**Lifecycle.** Transient; one instance per handler invocation.

**Invariants & enforcement.**

- Nothing guards `operation.UserId` being `""` (which happens whenever there is no correlation
  context, §3.14 step 5). `HubWrapper` then sends to group `users:` (§3.22) — a group with no
  members — so the notification is **silently discarded**.
- The three methods are `async` one-liners that `await` the wrapper; an exception from SignalR
  propagates back through the handler into Convey's subscriber. There is no try/catch anywhere on
  this path, so **a SignalR failure fails the message**, potentially causing a redelivery that the
  terminal latch (§3.6) will then suppress — losing the notification permanently while the state is
  already stored.

**Extension procedure.** To add a field to a notification, edit the anonymous type here **and** the
matching `connection.on(...)` handler in `wwwroot/ui/js/app.js:34-44`. To add a fourth
notification, add the method to the interface, the implementation, the `switch` in **all three**
handlers (`Handlers/*.cs:47-60`), and a listener in `app.js`.

**Failure modes.** The `users:` black hole; the unprotected exception path; and the six duplicated
string literals.

### 3.21 `IHubWrapper` / `HubWrapper` — the SignalR send seam

**Definition.** A two-method abstraction over `IHubContext<PaccoHub>`
(`Services/IHubWrapper.cs:5-9`, `Services/HubWrapper.cs:8-22`), registered transient
(`Infrastructure/Extensions.cs:57`).

```csharp
public async Task PublishToUserAsync(string userId, string message, object data)
    => await _hubContext.Clients.Group(userId.ToUserGroup()).SendAsync(message, data);

public async Task PublishToAllAsync(string message, object data)
    => await _hubContext.Clients.All.SendAsync(message, data);
```

**Representation & storage.** None; it is a thin adapter whose purpose is to keep
`Microsoft.AspNetCore.SignalR` out of `HubService` — the only testability seam in the service, and
there are no tests to use it (§3.45).

**Lifecycle.** Transient. `IHubContext<T>` itself is a singleton `[framework]`, so the transient
wrapper is free.

**Invariants & enforcement.**

- **`PublishToAllAsync` is never called.** Grep across the scoped path finds the declaration, the
  implementation and no call site. It is a broadcast primitive with no consumer — and, given that
  the hub has no authorization beyond group membership, one that would leak every user's operations
  to every connected client if it were ever wired up.
- `PublishToUserAsync` calls the **`string`** overload of `ToUserGroup` (`:18`), which does not
  reformat the id. See §3.22 for why that matters.
- `SendAsync` on an empty group is a silent no-op `[framework]`.

**Extension procedure.** Add methods here rather than injecting `IHubContext` elsewhere; the
indirection is the only thing keeping SignalR from spreading through the codebase.

**Failure modes.** The dead broadcast method; and the group-name coupling below.

### 3.22 `ToUserGroup` — two overloads and a cross-repo coupling

**Definition.** Two extension methods, adjacent, in
`Infrastructure/Extensions.cs:32-33`:

```csharp
public static string ToUserGroup(this Guid userId) => userId.ToString("N").ToUserGroup();
public static string ToUserGroup(this string userId) => $"users:{userId}";
```

The `Guid` overload normalises to the **dash-less 32-character "N" format**; the `string` overload
does not normalise at all.

**Representation & storage.** A SignalR group name.

**Lifecycle.** Computed at two moments in an operation's life, in two different code paths:

| Site | Overload used | Input | Result |
| --- | --- | --- | --- |
| `Hubs/PaccoHub.cs:34` | **`Guid`** — `Guid.Parse(payload.Subject).ToUserGroup()` | JWT `sub` claim | `users:<32 hex, no dashes>` |
| `Services/HubWrapper.cs:18` | **`string`** — `userId.ToUserGroup()` | `CorrelationContext.User.Id` | `users:<whatever the header said>` |

**Invariants & enforcement — the invariant is real but it is enforced in another repository.**

For a notification to reach a browser, these two strings must be byte-identical. They are, because:

1. `identity-service` mints the JWT subject as `userId.ToString("N")`
   (`…Identity.Infrastructure/Auth/JwtProvider.cs:21`) — dash-less.
2. The gateway's correlation context carries `user_id` taken from that same claim `[ntrada]`, so
   `CorrelationContext.User.Id` is also dash-less.
3. Path 1 parses and re-formats with `"N"` — idempotent for a dash-less input.
4. Path 2 passes through unchanged — correct only because the input was already dash-less.

**Change `JwtProvider.cs:21` to `userId.ToString()` and identity-service still works perfectly,
while every real-time notification in this service silently stops arriving.** The break would
manifest as "the UI never updates" in a repository that was not touched. There is no test, no
assertion and no shared constant anywhere on the path. See `component-internals/identity-service.md`
§6.4 and §8.2/B5 below.

Enforcement is therefore: **none, and failure is silent** — `SendAsync` to a non-existent group
succeeds.

**Extension procedure.** Make `HubWrapper` use the `Guid` overload (parsing defensively) so both
paths normalise, or introduce a single `UserGroup.For(string)` helper used by both. Either change
is confined to this repository and removes the cross-repo coupling entirely. This is the
highest-value small fix in the component.

**Failure modes.** As above. Note also that the `string` overload accepts anything, including `""`
(§3.20) and including another user's id — group membership is the *only* authorization boundary on
the push channel.

### 3.23 `PaccoHub.InitializeAsync` — JWT authentication over the hub

**Definition.** `Hubs/PaccoHub.cs:9-53`, a `Microsoft.AspNetCore.SignalR.Hub` with one public
method and two private helpers, mapped at `/pacco` (`Program.cs:46`).

```csharp
public async Task InitializeAsync(string token)
{
    if (string.IsNullOrWhiteSpace(token))
    {
        await DisconnectAsync();
    }                                          // ← no return — §3.24
    try
    {
        var payload = _jwtHandler.GetTokenPayload(token);
        if (payload is null)
        {
            await DisconnectAsync();

            return;
        }

        var group = Guid.Parse(payload.Subject).ToUserGroup();
        await Groups.AddToGroupAsync(Context.ConnectionId, group);
        await ConnectAsync();
    }
    catch
    {
        await DisconnectAsync();
    }
}
```

`ConnectAsync` and `DisconnectAsync` (`:44-52`) send the bare messages `"connected"` and
`"disconnected"` to the calling connection; the browser listens for both
(`wwwroot/ui/js/app.js:26-32`).

**Representation & storage.** Group membership lives in SignalR's group store — in the Redis
backplane when configured (§3.25), in process memory otherwise.

**Lifecycle.** The client connects anonymously, then **calls `initializeAsync` as a hub method**
passing the JWT as a plain string argument (`wwwroot/ui/js/app.js:19-22`):

```javascript
connection.start().then(() => { connection.invoke('initializeAsync', $jwt.value); })
```

Note there is no `Hub.OnDisconnectedAsync` override and no group removal — SignalR removes a
connection from its groups automatically on disconnect `[framework]`.

**Invariants & enforcement.**

- **This is the only authentication check in the entire service.** There is no
  `[Authorize]` attribute on the hub, no `UseAuthentication()` in the pipeline (§3.36), and no
  `accessTokenFactory` on the client. **An unauthenticated connection to `/pacco` is accepted**;
  it simply never joins a group and therefore receives nothing. That is a defensible design for a
  notification channel, but it means the hub's connection count is unbounded by any credential.
- `_jwtHandler.GetTokenPayload(token)` is Convey's validator `[convey]`; it applies the `jwt`
  options (`appsettings.json:32-43`) — notably `validateLifetime: true` but
  `validateIssuer: false` and `validateAudience: false` (§3.38). It returns `null` for an invalid
  token, which is handled at `:28-32`.
- `Guid.Parse(payload.Subject)` (`:34`) throws on a non-Guid subject — caught by the bare `catch`
  at `:38-41`, which sends `"disconnected"`. So a malformed subject and an invalid token are
  indistinguishable to the client.
- **The `catch` is bare and swallows everything**, including a Redis backplane failure inside
  `AddToGroupAsync`. The client is told "disconnected, invalid token"
  (`wwwroot/ui/js/app.js:30-32` renders exactly that text) even when the token was fine.
- The connection is **not actually disconnected** by `DisconnectAsync` — it only *sends a message
  named* `"disconnected"`. The socket stays open, and the client may call `initializeAsync` again
  with a different token. Retry is therefore free and unthrottled.

**Extension procedure.** To require authentication properly: add `UseAuthentication()` to the
pipeline (§3.36), decorate the hub with `[Authorize]`, configure the client with
`accessTokenFactory`, and read `Context.User` instead of a method argument — at which point
`InitializeAsync` reduces to a group join in `OnConnectedAsync`. That is a four-file change and
removes §3.24 and half of §3.22 along the way.

**Failure modes.** §3.24; misleading `"disconnected"` on infrastructure errors; unthrottled token
guessing; and tokens travelling as a hub-method argument, which puts them in SignalR's message
logs when `configureLogging(signalR.LogLevel.Information)` is set — as it is, in
`wwwroot/ui/js/app.js:8`.

### 3.24 The blank-token fall-through

**Definition.** `Hubs/PaccoHub.cs:20-23`:

```csharp
if (string.IsNullOrWhiteSpace(token))
{
    await DisconnectAsync();
}
```

**There is no `return`.** Compare the structurally parallel guard eleven lines below (`:28-32`),
which *does* return. Execution therefore falls through into the `try` block with `token == null`
or `""`.

**Lifecycle / consequence.**

1. `DisconnectAsync()` sends `"disconnected"` to the client. The browser displays
   *"Disconnected, invalid token."* (`wwwroot/ui/js/app.js:30-32`).
2. Execution continues to `:26`: `_jwtHandler.GetTokenPayload(null)`.
3. That either returns `null` — in which case `:29` sends `"disconnected"` a **second time** — or
   throws, in which case `:40` sends it a second time.

So the observable defect is a **duplicate `disconnected` message**, and the browser appends two
identical error rows. Functionally the outcome is correct (no group join), which is why the bug is
harmless today and easy to miss.

**Why it still matters.** The correctness of this path depends entirely on
`GetTokenPayload(null)` not succeeding — a Convey behaviour this repository cannot verify
(`Unverifiable — Missing Source Evidence`). If a future version returned a non-null payload for a
blank token, the missing `return` would turn a cosmetic bug into an authentication bypass. The
guard is written as if it were load-bearing and is not.

Note the browser guards blank tokens client-side too (`wwwroot/ui/js/app.js:13-16` rejects empty or
whitespace-containing input with an `alert`), so the path is normally unreachable from the shipped
UI — but a hand-rolled client reaches it trivially.

**Invariants & enforcement.** The intended invariant — *a blank token must not proceed* — is not
enforced by this guard; it is enforced accidentally, downstream.

**Extension procedure.** Add `return;` after `await DisconnectAsync();` at `:22`. One line, no
behavioural change today, and it makes the guard mean what it says.

**Failure modes.** Duplicate client messages now; a latent bypass on any change to Convey's token
handling.

### 3.25 The SignalR Redis backplane and its silent degradation

**Definition.** A private extension method (`Infrastructure/Extensions.cs:95-109`) that shadows the
framework's `AddSignalR` on `IConveyBuilder`:

```csharp
private static IConveyBuilder AddSignalR(this IConveyBuilder builder)
{
    var options = builder.GetOptions<SignalrOptions>("signalR");
    builder.Services.AddSingleton(options);
    var signalR = builder.Services.AddSignalR();
    if (!options.Backplane.Equals("redis", StringComparison.InvariantCultureIgnoreCase))
    {
        return builder;
    }

    var redisOptions = builder.GetOptions<RedisOptions>("redis");
    signalR.AddRedis(redisOptions.ConnectionString);

    return builder;
}
```

`SignalrOptions` is a one-property class (`Types/SignalrOptions.cs:3-6`: `string Backplane`), bound
from `"signalR": { "backplane": "redis" }` — present in `appsettings.json:152-154` **and**
`appsettings.docker.json:90-92`.

**Representation & storage.** With the backplane on, SignalR group membership and message fan-out
go through Redis `[framework]` — the same Redis instance that stores operations (§3.4), addressed
by `redis.connectionString` (`appsettings.json:145-148`).

**Lifecycle.** Once, at service registration.

**Invariants & enforcement.**

- **`options.Backplane.Equals(...)` will throw `NullReferenceException` if the `signalR` section is
  missing**, because `GetOptions` returns a default-constructed object with a `null` `Backplane`
  `[convey]`. That is a **loud** startup crash — one of the few in this service, and arguably the
  right behaviour.
- **Any value other than `"redis"` silently yields an in-memory backplane.** A typo
  (`"Redis "` with a trailing space, `"REDIS"` is fine — the comparison is case-insensitive) means
  each replica keeps its own group table, and a notification is delivered only if the message
  happened to be consumed by the same replica the browser is connected to. With two replicas that
  is a ~50% silent notification-loss rate, with no error anywhere.
- The comparison is `InvariantCultureIgnoreCase`, so case is safe but whitespace is not.

**Extension procedure.** To support another backplane (Azure SignalR, for instance), extend the
`if` into a `switch` on `options.Backplane` with an explicit `default` that **throws** rather than
falling through to in-memory. That single change converts the most dangerous silent
misconfiguration in the service into a startup failure.

**Failure modes.** Silent partial notification loss under horizontal scaling — the classic
symptom being "it works locally with one instance and half the time in production".

### 3.26 Legacy SignalR package versions

**Definition.** Two package references (`Pacco.Services.Operations.Api.csproj:31-32`):

```xml
<PackageReference Include="Microsoft.AspNetCore.SignalR" Version="1.1.0" />
<PackageReference Include="Microsoft.AspNetCore.SignalR.Redis" Version="1.1.5" />
```

on a project targeting `netcoreapp3.1` (`:4`).

**Why this is notable.** From ASP.NET Core 3.0 onward, SignalR ships **inside the shared framework**
(`Microsoft.AspNetCore.App`), and the Redis backplane's 3.x-era package is
`Microsoft.AspNetCore.SignalR.StackExchangeRedis`. The `1.1.x` packages are the ASP.NET Core 2.x
line. Referencing them on a 3.1 web project is redundant at best; whether `signalR.AddRedis(...)`
at `Infrastructure/Extensions.cs:106` resolves to the 1.1.5 package's extension method or to the
shared framework's is `Unverifiable — Missing Source Evidence` without building.

The build evidently succeeds — the service is published by `Dockerfile:4` and CI runs
`dotnet build -c release` (`scripts/build.sh`) — so this is a latent-risk observation, not a
demonstrated defect. It is recorded because it is exactly the kind of thing that changes behaviour
on a framework upgrade with no source change.

**Extension procedure.** Remove both references and, if the Redis backplane is still wanted, add
`Microsoft.AspNetCore.SignalR.StackExchangeRedis` and change `AddRedis` to
`AddStackExchangeRedis` at `Infrastructure/Extensions.cs:106`. Verify `/pacco` end-to-end
afterwards; there is no test to catch a regression (§3.45).

**Failure modes.** Assembly-binding ambiguity; and a silent behaviour change on upgrade.

Compare `Grpc.AspNetCore 2.28.0` and `Grpc.Tools 2.28.1` (`:29-30`), which are contemporary with
`netcoreapp3.1` and are correctly paired, and `Google.Protobuf 3.11.4` (`:28`).

### 3.27 `Operations.proto` — the gRPC contract

**Definition.** `Operations.proto:1-24`, compiled by `Grpc.Tools` via the `Protobuf` item
(`Pacco.Services.Operations.Api.csproj:40-42`).

```protobuf
syntax = "proto3";
package Services.Operations;

service GrpcOperationsService {
    rpc GetOperation (GetOperationRequest) returns (GetOperationResponse) {}
    rpc SubscribeOperations (Empty) returns (stream GetOperationResponse) {}
}

message Empty { }
message GetOperationRequest { string id = 1; }
message GetOperationResponse {
    string id = 1;  string userId = 2;  string name = 3;
    string state = 4;  string code = 5;  string reason = 6;
}
```

**Representation & storage.** Generated C# in `obj/`; the server base class is
`GrpcOperationsService.GrpcOperationsServiceBase`, implemented by `GrpcServiceHost` (§3.28). An
identical copy of the file lives at `src/Pacco.Services.Operations.GrpcClient/Operations.proto`
for the sample client — **two copies, no shared package**, so they can drift.

**Lifecycle.** Compile-time.

**Invariants & enforcement.**

- `state` is a **string**, not a protobuf enum, and `GrpcServiceHost.Map` produces it as
  `operation.State.ToString().ToLowerInvariant()` (`:53`) — so the gRPC channel exposes
  `"pending"` / `"completed"` / `"rejected"` while HTTP exposes the integer ordinal (§3.2). Two
  channels, two encodings of the same field, and nothing reconciles them.
- All six response fields are `string`, so protobuf's default for an unset field is `""`, never
  `null` — which is what makes §3.30's "empty response means not found" convention *almost* work.
- Field numbers 1–6 are assigned in DTO order; adding a field must use 7 or higher, or every
  existing client misparses (§5.4).
- The `Empty` message is hand-declared rather than importing `google/protobuf/empty.proto` — fine,
  but it means the type is `Services.Operations.Empty`, not the well-known type, and a client that
  imports the well-known one will not interoperate.

**Extension procedure.** Add the field with the next free number, regenerate, update
`GrpcServiceHost.Map` (`:46-53`) **and** copy the change into
`src/Pacco.Services.Operations.GrpcClient/Operations.proto`. `scripts/proto/{lin,mac,win}-compile.sh`
exist for manual regeneration; the build does it automatically via `Grpc.Tools`.

**Failure modes.** The duplicated `.proto`; the string/int state divergence; and the empty-response
convention.

### 3.28 `GrpcServiceHost` — per-call instance and the handler leak

**Definition.** `Infrastructure/GrpcServiceHost.cs:11-55`, the gRPC service implementation, mapped
by `endpoints.MapGrpcService<GrpcServiceHost>()` (`Program.cs:47`).

```csharp
private readonly BlockingCollection<OperationDto> _operations = new BlockingCollection<OperationDto>();

public GrpcServiceHost(IOperationsService operationsService, ILogger<GrpcServiceHost> logger)
{
    _operationsService = operationsService;
    _logger = logger;
    _operationsService.OperationUpdated += (s, e) => _operations.TryAdd(e.Operation);
}
```

**Representation & storage.** An unbounded in-memory `BlockingCollection<OperationDto>` per
instance, plus a subscription to the singleton's event (§3.7).

**Lifecycle — this is the defect.** gRPC services registered with `MapGrpcService<T>` are resolved
**per call** by default `[framework]`. So:

1. Every gRPC call — including every plain `GetOperation` unary call — constructs a new
   `GrpcServiceHost`, which constructs a new `BlockingCollection` and **adds a new handler to the
   singleton's `OperationUpdated` invocation list**.
2. **Nothing ever removes it.** There is no `-=`, no `IDisposable` implementation, no
   `OnCompleted`. The instance is unreachable after the call returns but is kept alive by the
   event, along with its `BlockingCollection`.
3. Every subsequent operation update is pushed into every one of those orphaned collections
   (`TryAdd` on an unbounded collection always succeeds), which **grow without bound**.

The result is a compounding leak whose rate is (gRPC calls made) × (operations processed). Ten
`GetOperation` calls leave ten orphaned collections, each accumulating a copy of every future
operation, forever. See §8.2/B6.

**Invariants & enforcement.** None. Nothing measures the invocation-list length, and the leak is
invisible until the process runs out of memory.

**Extension procedure — the fix.** Two viable shapes:

1. **Register the service as a singleton** and move the fan-out to per-stream state: give
   `SubscribeOperations` its own local `BlockingCollection` (or better, a bounded
   `Channel<OperationDto>`), subscribe on entry, and unsubscribe in a `finally`. This also fixes
   §3.29 and confines subscriptions to actual stream subscribers rather than every unary call.
2. Keep per-call instances but implement `IDisposable`/`IAsyncDisposable` and unsubscribe there —
   gRPC disposes per-call service instances `[framework]`. Less good, because a unary call would
   still churn subscriptions.

Option 1 is the smaller change in practice and is what the code clearly meant to do.

**Failure modes.** Unbounded memory growth; and — subtler — a unary `GetOperation` caller
inadvertently causes work on every future message for the life of the process.

### 3.29 `BlockingCollection` fan-out and the uncancellable stream

**Definition.** `Infrastructure/GrpcServiceHost.cs:32-41`:

```csharp
public override async Task SubscribeOperations(Empty request,
    IServerStreamWriter<GetOperationResponse> responseStream, ServerCallContext context)
{
    _logger.LogInformation($"Received 'Subscribe operations' request from: {context.Peer}");
    while (true)
    {
        var operation = _operations.Take();
        await responseStream.WriteAsync(Map(operation));
    }
}
```

**Representation & storage.** `BlockingCollection<OperationDto>` (`:15`) with the default unbounded
`ConcurrentQueue` backing.

**Lifecycle.** For the duration of the stream — nominally forever.

**Invariants & enforcement — four distinct problems in six lines:**

1. **`while (true)` with no exit condition.** `context.CancellationToken` is available and unused.
   The loop ends only when `WriteAsync` throws because the client vanished — and only *after* the
   next operation arrives to unblock `Take()`. A disconnected client's loop therefore lingers
   indefinitely on an idle system.
2. **`Take()` is a synchronous blocking call on a thread-pool thread.** There is no
   `TakeAsync`; `BlockingCollection` predates `Channel<T>`. Each concurrent stream permanently
   occupies one thread-pool thread. With the default thread pool this is survivable for a handful
   of subscribers and pathological beyond that.
3. **No cancellation token is passed to `Take()`**, though the overload
   `Take(CancellationToken)` exists — passing `context.CancellationToken` would fix (1) and (2)'s
   linger.
4. **The stream is unfiltered and unauthenticated.** Every subscriber receives **every** operation
   for **every** user, including `userId`. There is no credential on the gRPC channel at all
   (§3.36) — compare the SignalR path, which at least scopes by group (§3.22). See §8.2/B2.

Additionally, because `_operations` is per-instance (§3.28) and the event is process-local (§3.7),
a subscriber connected to replica A never sees operations processed by replica B. The gRPC channel
is **not** backplane-aware, unlike SignalR.

**Extension procedure.** Replace the whole method body with a bounded
`Channel<OperationDto>` created per call, subscribed in a `try` and unsubscribed in a `finally`,
and loop on `await channel.Reader.WaitToReadAsync(context.CancellationToken)`. To scope the stream
per user, add a `userId` field to `SubscribeOperations`' request message (§3.27) — which currently
takes `Empty` — and authenticate it.

**Failure modes.** Thread-pool starvation; lingering loops; cross-tenant data exposure; and the
memory growth from §3.28.

### 3.30 `Map(null)` — an empty response instead of `NOT_FOUND`

**Definition.** `Infrastructure/GrpcServiceHost.cs:43-54`:

```csharp
private static GetOperationResponse Map(OperationDto operation)
    => operation is null
        ? new GetOperationResponse()
        : new GetOperationResponse { Id = …, UserId = …, Name = …, Code = …, Reason = …,
                                     State = operation.State.ToString().ToLowerInvariant() };
```

**Lifecycle.** Called from `GetOperation` (`:29`) and from the stream (`:39`).

**Invariants & enforcement.** A missing operation yields a **successful RPC with an all-default
response** rather than `StatusCode.NotFound`. gRPC has a first-class error model
(`throw new RpcException(new Status(StatusCode.NotFound, …))`) and it is not used. Compare the HTTP
endpoint, which does return `404` (`Program.cs:36-40`) — the two channels disagree about the same
condition.

The convention is not even documented in the `.proto`; it is inferred by the sample client, which
sniffs a blank id (`src/Pacco.Services.Operations.GrpcClient/Program.cs:125-129`):

```csharp
if (string.IsNullOrWhiteSpace(response.Id))
{
    Console.WriteLine($"* Operation was not found for id: {id} *");
    return;
}
```

So the "not found" contract is: *check whether `id` came back empty*. Every client must know this,
and nothing tells them.

**Note the interaction with `State`.** A default `GetOperationResponse` has `state == ""`, whereas a
real `Pending` operation has `state == "pending"`. So `state` would be an equally valid sentinel —
two undocumented sentinels for one condition.

**Extension procedure.** Throw `RpcException` with `StatusCode.NotFound` from `GetOperation` when
the DTO is `null`, and keep `Map` total for the streaming path (where `null` cannot occur, since
the stream only carries successfully-written operations). Update the sample client's check to catch
`RpcException`.

**Failure modes.** Silent misinterpretation by any client that does not implement the undocumented
sentinel check.

### 3.31 `Pacco.Services.Operations.GrpcClient` — the sample client

**Definition.** A `netcoreapp3.1` console project outside the scoped path
(`src/Pacco.Services.Operations.GrpcClient/`), containing its own copy of `Operations.proto` and a
150-line interactive `Program.cs`. It is **not deployed** — `Dockerfile:4` publishes only
`src/Pacco.Services.Operations.Api`.

**Why it is documented here.** It is the only executable specification of how the gRPC surface is
meant to be used, and it encodes three facts the service itself does not state:

| Fact | Evidence |
| --- | --- |
| The default address is **`https://localhost:50050`** | `GrpcClient/Program.cs:70-75` |
| TLS validation is **bypassed** (`DangerousAcceptAnyServerCertificateValidator`) | `GrpcClient/Program.cs:35-39`, with the comment `// Only for the local development purposes.` |
| "Not found" is a blank `Id` | `GrpcClient/Program.cs:125-129` (§3.30) |

Port **50050** is the HTTPS port from `Properties/launchSettings.json:20`
(`http://localhost:5005;https://localhost:50050`). gRPC over HTTP/2 requires TLS for the .NET
client unless explicitly opted out `[framework]`, which is why the sample targets the HTTPS
endpoint — and why §3.44's Docker analysis matters.

**Lifecycle.** Developer tooling only.

**Invariants & enforcement.** None; it is a sample. Its `.proto` copy can silently drift from the
service's (§3.27).

**Extension procedure.** Keep the two `.proto` files in sync, or make the client project reference
the API project's file with a `<Protobuf Include="..\Pacco.Services.Operations.Api\Operations.proto" />`
link.

**Failure modes.** Drift; and the possibility that someone treats
`DangerousAcceptAnyServerCertificateValidator` as production guidance — the comment at `:34` is the
only thing preventing that.

### 3.32 `GetOperation` — a query type with no handler

**Definition.** `Queries/GetOperation.cs:6-9`:

```csharp
public class GetOperation : IQuery<OperationDto>
{
    public string OperationId { get; set; }
}
```

**Representation & storage.** Used purely as a **model-binding target**. `Program.cs:32` registers
`.Get<GetOperation>("operations/{operationId}", async (query, ctx) => { … })` — the lambda overload,
which binds the route value into the query object and then **replaces dispatch entirely**
`[convey]`. The lambda calls `IOperationsService.GetAsync` directly.

**There is no `IQueryHandler<GetOperation, OperationDto>` anywhere in the repository.**
`AddQueryHandlers()` (`Infrastructure/Extensions.cs:66`) scans and finds none; no query dispatcher
is registered (there is no `AddInMemoryQueryDispatcher()` call) and `IQueryDispatcher` is never
injected. The CQRS surface is decorative.

Note the difference from the sibling service: `identity-service` *does* have a `GetUserHandler`, and
it too is dead (`component-internals/identity-service.md` §3.25). Here the handler was never
written at all.

**Lifecycle.** One instance per HTTP request, populated by Convey's binder.

**Invariants & enforcement.** The property must be named `OperationId` to match the route template
`operations/{operationId}` (case-insensitive binding `[convey]`). A rename on either side silently
yields `null`, and `GetAsync(null)` produces the Redis key `requests:` — which reliably misses and
returns `404`. **Silent.**

**Extension procedure.** If a second query is ever needed, either follow this pattern (lambda +
direct service call) or introduce a real dispatcher — but not both, or two endpoints will behave
differently for the same failure. Deleting `AddCommandHandlers()`/`AddQueryHandlers()` is *not*
safe: `AddCommandHandlers()` and `AddEventHandlers()` are what register the three generic handlers
that §3.12 resolves.

**Failure modes.** The silent binding mismatch; and the misleading presence of a CQRS vocabulary
that is never exercised.

### 3.33 `Program.cs` — two `UseEndpoints` calls against two different builders

**Definition.** `Program.cs:21-52`. The composition is short enough to read whole:

```csharp
public static async Task Main(string[] args)
    => await WebHost.CreateDefaultBuilder(args)
        .ConfigureServices(services => services
            .AddConvey()
            .AddWebApi()
            .AddInfrastructure()
            .Build())
        .Configure(app => app
            .UseInfrastructure()
            .UseEndpoints(endpoints => endpoints                    // ← Convey's builder
                .Get("", ctx => ctx.Response.WriteAsync(ctx.RequestServices.GetService<AppOptions>().Name))
                .Get<GetOperation>("operations/{operationId}", async (query, ctx) => { … }))
            .UseEndpoints(endpoints =>                              // ← ASP.NET Core's builder
            {
                endpoints.MapHub<PaccoHub>("/pacco");
                endpoints.MapGrpcService<GrpcServiceHost>();
            }))
        .UseLogging()
        .UseVault()
        .Build()
        .RunAsync();
```

**The two `UseEndpoints` calls are different methods.** The first (`:30-43`) is Convey's
`IApplicationBuilder.UseEndpoints(Action<IEndpointsBuilder>)` from `Convey.WebApi` `[convey]`,
whose builder offers `Get`/`Post`/`Put`/`Delete` with query- and command-binding overloads. The
second (`:44-48`) is ASP.NET Core's
`UseEndpoints(Action<IEndpointRouteBuilder>)` `[framework]`, needed because `MapHub` and
`MapGrpcService` are extensions on the framework's builder. They are not interchangeable, and the
resemblance is a genuine reading hazard: a maintainer adding an HTTP route to the second block will
find `Get` does not exist there, and one adding a hub to the first block will find `MapHub` does
not.

Also note **`UseRouting()` is never called explicitly** — both `UseEndpoints` overloads arrange it
internally `[convey]` `[framework]`. Calling `UseEndpoints` twice on one pipeline works because each
call adds its own endpoint middleware; the ordering means Convey's routes are matched first.

**Lifecycle.** Once, at startup.

**Invariants & enforcement.** `AddWebApi()` (`:25`) must precede `AddInfrastructure()` (`:26`) for
the WebApi services to exist; `UseInfrastructure()` (`:29`) must precede both `UseEndpoints` calls,
because it installs the error handler, static files and — crucially — `SubscribeMessages()`
(§3.9). None of this is asserted; a wrong order fails at startup, loudly, with a DI resolution
error.

`.UseLogging()` and `.UseVault()` (`:49-50`) are on the **`IWebHostBuilder`**, not the app —
they configure the host before it is built.

**Extension procedure.** New HTTP routes go in the first block, new hubs/gRPC services in the
second. For anything needing the framework's routing metadata (authorization policies, CORS per
endpoint), use the second block.

**Failure modes.** The two-builder confusion; and the fact that all routing lives in `Program.cs`
with no controllers, so there is no route table to inspect and Swagger
(`AddWebApiSwaggerDocs()`, `Infrastructure/Extensions.cs:77`) documents only what Convey can infer.

### 3.34 `ExceptionToResponseMapper` — the discard-all mapper

**Definition.** `Infrastructure/ExceptionToResponseMapper.cs:7-15`, registered via
`AddErrorHandler<ExceptionToResponseMapper>()` (`Infrastructure/Extensions.cs:62`) and activated by
`UseErrorHandler()` (`:83`):

```csharp
internal sealed class ExceptionToResponseMapper : IExceptionToResponseMapper
{
    public ExceptionResponse Map(Exception exception)
        => exception switch
        {
            _ => new ExceptionResponse(new {code = "error", reason = "There was an error."},
                HttpStatusCode.BadRequest)
        };
}
```

A `switch` expression with exactly one arm: the discard pattern.

**Representation & storage.** N/A.

**Lifecycle.** Per unhandled exception on the HTTP pipeline.

**Invariants & enforcement.** **Every** exception becomes `400 Bad Request` with the body
`{"code":"error","reason":"There was an error."}`:

| Actual condition | Reported as |
| --- | --- |
| Redis unreachable | `400` `{code: "error"}` |
| Malformed JSON in the cache | `400` `{code: "error"}` |
| A `NullReferenceException` in the endpoint lambda | `400` `{code: "error"}` |
| Any genuine client error | `400` `{code: "error"}` |

There is no `500`, ever, on the HTTP surface. A monitoring system watching for 5xx sees a perfectly
healthy service while Redis is down. This mirrors `identity-service`'s mapper, which at least
distinguishes three domain exceptions before defaulting
(`component-internals/identity-service.md` §3.19); here there is nothing to distinguish, because
the service defines **no exception types of its own** — there is no `Exceptions/` directory.

Note the scope: this covers only the HTTP pipeline. It does **not** cover the message consumers
(§3.14), where an exception propagates into Convey's subscriber, nor gRPC, where it becomes an
`RpcException` with `StatusCode.Unknown` `[framework]`, nor SignalR.

**Extension procedure.** Add typed arms above the discard, exactly as the sibling services do. If
you want infrastructure faults to surface as `503`, add an arm for the Redis client's exception
type — `StackExchange.Redis.RedisConnectionException` — which requires a package reference the
project does not currently have (it comes in transitively through
`Convey.Persistence.Redis`, `.csproj:23`).

**Failure modes.** Infrastructure outages presenting as client errors; and total loss of diagnostic
information in the response (the exception message is never surfaced — which is correct for
security, but nothing logs it here either; that is left to Convey's error middleware `[convey]`).

### 3.35 `AddInfrastructure` — the composition root

**Definition.** `Infrastructure/Extensions.cs:49-79`, the single registration site for the entire
service.

```csharp
public static IConveyBuilder AddInfrastructure(this IConveyBuilder builder)
{
    var requestsOptions = builder.GetOptions<RequestsOptions>("requests");
    builder.Services.AddSingleton(requestsOptions);
    builder.Services.AddTransient<ICommandHandler<ICommand>, GenericCommandHandler<ICommand>>()
        .AddTransient<IEventHandler<IEvent>, GenericEventHandler<IEvent>>()
        .AddTransient<IEventHandler<IRejectedEvent>, GenericRejectedEventHandler<IRejectedEvent>>()
        .AddTransient<IHubService, HubService>()
        .AddTransient<IHubWrapper, HubWrapper>()
        .AddSingleton<IOperationsService, OperationsService>();
    builder.Services.AddGrpc();

    return builder
        .AddErrorHandler<ExceptionToResponseMapper>()
        .AddJwt()
        .AddCommandHandlers()
        .AddEventHandlers()
        .AddQueryHandlers()
        .AddHttpClient()
        .AddConsul()
        .AddFabio()
        .AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())
        .AddMongo()
        .AddRedis()
        .AddMetrics()
        .AddJaeger()
        .AddRedis()          // ← :75, duplicate of :72
        .AddSignalR()
        .AddWebApiSwaggerDocs()
        .AddSecurity();
}
```

**Lifetimes, and why each matters:**

| Registration | Lifetime | Why |
| --- | --- | --- |
| `RequestsOptions` | singleton | immutable config (§3.5) |
| the three generic handlers | transient | one per message; they hold no state |
| `IHubService`, `IHubWrapper` | transient | thin adapters over a singleton `IHubContext` |
| `IOperationsService` | **singleton** | **load-bearing** — the `OperationUpdated` event (§3.7) must be process-wide, or the gRPC stream would never receive anything |
| `GrpcServiceHost` | **per-call** (implicit, via `MapGrpcService`) | the cause of §3.28 |

**`AddRedis()` is called twice**, at `:72` and `:75`. Convey's registrations are idempotent-ish
(`TryAdd`-based) `[convey]`, so this is harmless in practice — but it is a genuine duplicate and
an accurate reading of the file must note it, because a maintainer removing "the redundant one"
needs to know that `AddSignalR()` at `:76` reads the `redis` section itself (`:105-106`) and does
not depend on either call.

**Invariants & enforcement.**

- `GetOptions<RequestsOptions>("requests")` at `:51` runs **before** anything else and is the
  service's first contact with configuration — see §3.5 for the zero-value trap.
- `builder.Services.AddGrpc()` (`:59`) is the framework call; the Convey-style `.AddSignalR()`
  (`:76`) is the private one from §3.25. Both are needed before their respective `Map*` calls.
- **`AddSecurity()` (`:78`) registers encryption/hashing primitives** `[convey]` bound to the
  `security` section — which **does not exist** in any of this service's four `appsettings` files
  (compare `identity-service`'s `appsettings.json:160-168`, which has one). Nothing in the scoped
  path injects an `IEncryptor`, `IHasher` or `ISigner`, so this is a registration with no
  configuration and no consumer.
- Everything here is registration-only; nothing runs.

**Extension procedure.** Add registrations to the `builder.Services` block for concrete types and
to the fluent chain for Convey modules. Keep `IOperationsService` a singleton.

**Failure modes.** The duplicate `AddRedis`; the unconfigured `AddSecurity`; and the implicit
per-call lifetime of `GrpcServiceHost`, which is invisible here because it is never registered
explicitly.

### 3.36 `UseInfrastructure` — the middleware chain, and what is absent from it

**Definition.** `Infrastructure/Extensions.cs:81-93`:

```csharp
public static IApplicationBuilder UseInfrastructure(this IApplicationBuilder app)
{
    app.UseErrorHandler()
        .UseSwaggerDocs()
        .UseJaeger()
        .UseConvey()
        .UseMetrics()
        .UseStaticFiles()
        .UseRabbitMq()
        .SubscribeMessages();

    return app;
}
```

Eight calls. The last one is not middleware at all — it is the startup-time manifest reader (§3.9),
tucked onto the end of a fluent chain.

**What is absent, compared with every other service in the workspace:**

| Missing call | Present in `identity-service`? | Consequence here |
| --- | --- | --- |
| `UseAuthentication()` | yes (`…Identity.Infrastructure/Extensions.cs:101`) | **no HTTP endpoint is ever authenticated**; `HttpContext.User` is always anonymous |
| `UseAccessTokenValidator()` | yes (`:97`) | a **revoked** access token is still accepted by the SignalR hub (§3.23), because the Redis deny-list is never consulted |
| `UsePublicContracts<…>()` | yes (`:99`) | no contract discovery endpoint — consistent, since the service declares no contracts |
| an outbox / `UseMessageOutbox` | yes (`:80` registration) | no inbox de-duplication: **a redelivered message is reprocessed** (mitigated in effect by the terminal latch, §3.6, but not by design) |

The first two are the significant ones. `AddJwt()` **is** registered (`:63`), so the JWT services
exist — they are simply only ever used by the hub, by direct `IJwtHandler` injection
(`Hubs/PaccoHub.cs:11-16`). See §8.2/B1.

**Ordering constraints that are real:**

- `UseErrorHandler()` must be first, or exceptions from later middleware escape unmapped.
- `UseStaticFiles()` (`:88`) must precede endpoint routing (which happens in `Program.cs:30`, after
  `UseInfrastructure` returns) for `wwwroot/ui` to be served (§3.39). It is.
- `UseRabbitMq()` must precede `SubscribeMessages()`, because the latter is an extension on the
  `IBusSubscriber` the former returns.

**Extension procedure.** To authenticate the HTTP surface: insert `UseAuthentication()` after
`UseConvey()`, optionally `UseAccessTokenValidator()` before it, and then add an explicit check in
the `operations/{operationId}` lambda comparing `HttpContext.User`'s subject to
`operation.UserId` — the DTO already carries the owner. Also change the gateway route to
`auth: true` in all four manifests (§6.1). That is the complete fix for §8.2/B2 on the HTTP channel.

**Failure modes.** Everything in §8.2/B1 and B2 traces to this thirteen-line method.

### 3.37 The inert MongoDB configuration

**Definition.** `AddMongo()` (`Infrastructure/Extensions.cs:71`) plus a full `mongo` configuration
section in three profiles:

| Profile | Value | Line |
| --- | --- | --- |
| base | `mongodb://localhost:27017`, database `operations-service`, `seed: false` | `appsettings.json:98-102` |
| docker | `mongodb://mongo:27017`, same database | `appsettings.docker.json:73-77` |
| vault lease | a `mongo` dynamic-credential lease with a connection-string template | `appsettings.json:182-192` |

**What is missing.** There is **no `AddMongoRepository<,>` call**, no `IMongoRepository` injection,
no document type, no `IMongoDbSeeder`, and no `UseMongo()`. Grep across the scoped path finds
`Mongo` in exactly three places: the `using Convey.Persistence.MongoDB;` at
`Infrastructure/Extensions.cs:14`, the `AddMongo()` call at `:71`, and the configuration sections.

**Consequence.** The service opens a MongoDB connection at startup `[convey]` and never uses it.
The Vault lease block (`appsettings.json:182-192`) would even rotate dynamic Mongo credentials for
a database that is never read or written.

**Why it matters to a maintainer.** Two reasons:

1. It is a **false affordance**. A reader looking for durable operation history sees a configured
   database named `operations-service` and reasonably concludes state is persisted there. It is
   not — state is in Redis with a five-minute sliding expiry (§3.4, §3.5). This is precisely the
   confusion recorded as baseline gap **G4/Q4**, now resolved.
2. It is the **obvious seam** for making operations durable: the registration already exists, so
   adding a document type, an `AddMongoRepository<OperationDocument, string>("operations")` and a
   write in `TrySetAsync` is a contained change (§7.5).

**Extension procedure.** Either delete `AddMongo()` and all three `mongo` sections, or use it. The
half-state is the worst option, and it is what is committed.

**Failure modes.** Misleading configuration; an idle connection pool; and a Vault lease with no
purpose.

### 3.38 `AddJwt()`, the `.cer` certificate and the shared signing key

**Definition.** `AddJwt()` (`Infrastructure/Extensions.cs:63`) bound to `appsettings.json:32-43`:

```json
"jwt": {
  "certificate": { "location": "certs/localhost.cer" },
  "issuerSigningKey": "eiquief5phee9pazo0Faegaez9gohThailiur5woy2befiech1oarai4aiLi6ahVecah3ie9Aiz6Peij",
  "expiryMinutes": 60,
  "issuer": "pacco",
  "validateAudience": false,
  "validateIssuer": false,
  "validateLifetime": true,
  "allowAnonymousEndpoints": ["/sign-in", "/sign-up"]
}
```

**Representation & storage.** `src/Pacco.Services.Operations.Api/certs/` contains exactly one file:
`localhost.cer` — a **public** certificate, 1115 bytes. It is published into the image by
`Pacco.Services.Operations.Api.csproj:44-46`
(`<Content Include="certs\**" CopyToPublishDirectory="Always" />`).

Contrast `identity-service`, whose `certs/` holds `localhost.pfx`, `localhost.key`,
`localhost.pem` **and** `localhost.cer` — i.e. the private key
(`component-internals/identity-service.md` §3.37). The split is coherent: the issuer holds the
private key, the validator holds the public certificate.

**But the asymmetry is not actually used.** Both services also carry the *same symmetric*
`issuerSigningKey`, byte-identical here (`appsettings.json:36`) and in
`…Identity.Api/appsettings.json:36`, and identity mints tokens with it. Which of the two Convey
prefers when both `certificate.location` and `issuerSigningKey` are present is
`Unverifiable — Missing Source Evidence`. What is certain is that:

- `appsettings.local.json:26-29` blanks `certificate.location`, so **local runs must** fall back to
  the symmetric key;
- `appsettings.docker.json:55-62` has a `jwt` section with **no `certificate` key**, so the base
  value `certs/localhost.cer` stands in Docker;
- and the symmetric key is present in every profile.

**Consequences.**

1. Because the signing key is shared and `validateIssuer`/`validateAudience` are both `false`,
   **any service on the platform can mint a token this service accepts** (§8.2/B7).
2. `expiryMinutes: 60` here is inert — this service never *issues* a token; only
   `validateLifetime: true` matters, and it does apply, in `GetTokenPayload` (§3.23).
3. `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` is **copy-paste from identity-service** —
   this service has no such endpoints, and no access-token middleware is installed anyway (§3.36).
   It is dead configuration.

**Lifecycle.** Bound at startup; used only by `IJwtHandler` inside the hub.

**Invariants & enforcement.** If `certs/localhost.cer` were absent at runtime, whether `AddJwt()`
throws or falls back to the symmetric key is `[convey]` and unverifiable here. The `csproj` content
rule makes its presence reliable in published output.

**Extension procedure.** For a real deployment, drop `issuerSigningKey` from every service except
the issuer, switch to asymmetric signing (the certificate pair already exists), and enable
`validateIssuer`/`validateAudience`. That is a platform-wide change, tracked in
`component-internals/identity-service.md` §8.2/B1.

**Failure modes.** The shared symmetric key; dead `allowAnonymousEndpoints`; and the ambiguity
about which credential is actually in force.

### 3.39 `UseStaticFiles()` and `wwwroot/ui` — the platform's only front-end

**Definition.** `UseStaticFiles()` (`Infrastructure/Extensions.cs:88`) serving three files:

| File | Size | Role |
| --- | --- | --- |
| `wwwroot/ui/index.html` | 26 lines | a Bootstrap-4 page (CDN stylesheet) with a JWT text input, a **Connect** button and an empty `<ul id="messages">` |
| `wwwroot/ui/js/signalr.js` | ~181 KB | the vendored SignalR JavaScript client (UMD bundle) |
| `wwwroot/ui/js/app.js` | 53 lines | the application logic |

Reachable at `/ui/index.html` (the default-document middleware is not enabled — `UseDefaultFiles()`
is absent — so `/ui/` alone would 404).

**`app.js` in full behavioural summary (`wwwroot/ui/js/app.js:1-53`):**

1. `:6-9` — builds a hub connection to a **hard-coded** `http://localhost:5005/pacco` with
   `LogLevel.Information`.
2. `:11-24` — on **Connect**: rejects a blank token or one containing whitespace with an `alert`
   (`:13-16`), starts the connection, then `connection.invoke('initializeAsync', $jwt.value)`
   (`:21`) — the hub-method authentication of §3.23.
3. `:26-32` — listens for `connected` / `disconnected`, rendering *"Connected."* and
   *"Disconnected, invalid token."*
4. `:34-44` — listens for `operation_pending` / `operation_completed` / `operation_rejected`,
   matching `HubService`'s three literals (§3.20), and renders the raw payload as
   `JSON.stringify(data)` (`:46-52`).

**Invariants & enforcement.**

- The hub URL at `:7` is **hard-coded to `localhost:5005`**. In the compose topology that is the
  published host port (`compose/services.yml:55-56`), so opening the page from the host machine
  works; opening it through the API gateway would not, and the page is not served by the gateway
  anyway. It is a developer tool, not a product surface.
- `appendMessage` interpolates into `innerHTML` (`:51`) including `JSON.stringify(data)` — an
  XSS sink if any operation field ever carried markup. Today `Name` comes from the manifest or a
  CLR type name and `Reason`/`Code` come from a rejected event's payload (§3.11), so the reachable
  values are service-controlled. It is still an unescaped sink fed by remote data.
- The six message-name literals are duplicated between `HubService.cs` and `app.js` with no shared
  definition (§3.20).

**Extension procedure.** For a new notification type, add the `connection.on(...)` listener at
`:44` and the matching `HubService` method. To make the page deployable, replace the hard-coded URL
with a relative path (`/pacco`), since the page is served by the same origin as the hub.

**Failure modes.** The hard-coded URL; the `innerHTML` sink; and the fact that this — the only UI
in fourteen repositories — is an unversioned, untested debug page.

### 3.40 Configuration layering and the four profiles

**Definition.** Four `appsettings*.json` files, selected by `ASPNETCORE_ENVIRONMENT`.

| File | Selected by | Character |
| --- | --- | --- |
| `appsettings.json` | always (base) | 194 lines; every section |
| `appsettings.local.json` | `local` — set by `scripts/start.sh:2` and both `launchSettings.json` profiles (`:14`, `:22`) | 49 lines; **disables** consul, fabio, jaeger, metrics, file/seq logging, vault; blanks the JWT certificate |
| `appsettings.docker.json` | `docker` — set by `Dockerfile:10` | 117 lines; container hostnames; vault off; **re-states `requests`, `signalR`, `mongo`, `jwt`, `swagger` in full** |
| `appsettings.development.json` | `Development` | `{}` — empty |

**The base file's load-bearing sections:**

| Section | Lines | Notable |
| --- | --- | --- |
| `app` | `:2-6` | `name: "Pacco Operations Service"`, `service: "operations-service"` |
| `consul` | `:7-17` | `pingInterval: 3`, `removeAfterInterval: 3`; `address: "docker.for.win.localhost"` (a Windows-Docker artefact in the *base* profile) |
| `jwt` | `:32-43` | §3.38 |
| `logger` | `:44-79` | `excludeProperties` includes `Email`, `Token`, `Password`, `ConnectionString`, `Secret`; `excludePaths: ["/", "/ping", "/metrics"]`; seq `apiKey: "secret"` |
| `jaeger` | `:80-88` | `serviceName: "operations"` — **not** `operations-service`, unlike `consul.service` and `fabio.service` |
| `mongo` | `:98-102` | inert (§3.37) |
| `rabbitMq` | `:103-144` | exchange `operations` (§3.43); `queue.template: "operations-service/{{exchange}}.{{message}}"`; `conventionsCasing: "snakeCase"`; `context.header: "message_context"`; `spanContextHeader: "span_context"` |
| `redis` | `:145-148` | `instance: "operations:"` — the key prefix in front of `requests:{id}` (§3.3) |
| `requests` | `:149-151` | §3.5 |
| `signalR` | `:152-154` | §3.25 |
| `vault` | `:164-193` | §3.41 |

**Note there is no `outbox` section and no `security` section** — the former because the service
never publishes (§3.43), the latter despite `AddSecurity()` being called (§3.35).

**Invariants & enforcement.** As everywhere in this platform, a missing or misspelled section binds
to a default-constructed options object `[convey]` — **silently**. Two instances in this service
are genuinely dangerous: `requests` (§3.5, yields `TimeSpan.Zero`) and `signalR` (§3.25, yields a
`NullReferenceException`, which is at least loud).

**The `docker` profile duplicates rather than overrides.** `requests`, `signalR`, `mongo`,
`swagger` and most of `jwt` are restated with the same values. That is redundant today and a drift
hazard tomorrow: changing `expirySeconds` in the base file alone has **no effect in Docker**.

**Extension procedure.** Add the section to `appsettings.json`, then decide explicitly for each of
the other three whether an override is needed — and remember that `docker` restates several
sections wholesale, so a base-only change may be invisible there.

**Failure modes.** Silent default binding; base/docker drift; and `jaeger.serviceName` differing
from every other service identifier, which fragments traces in the Jaeger UI.

### 3.41 Vault integration

**Definition.** `appsettings.json:164-193` plus `.UseVault()` on the host builder
(`Program.cs:50`, from `Convey.Secrets.Vault`).

```json
"vault": {
  "enabled": true, "url": "http://localhost:8200",
  "authType": "token", "token": "secret",
  "username": "user", "password": "secret",
  "kv":   { "enabled": true, "engineVersion": 2, "mountPoint": "kv", "path": "operations-service/settings" },
  "pki":  { "enabled": true, "roleName": "operations-service", "commonName": "operations-service.pacco.io" },
  "lease": { "mongo": { "type": "database", "roleName": "operations-service", "enabled": true,
                        "autoRenewal": true,
                        "templates": { "connectionString": "mongodb://{{username}}:{{password}}@localhost:27017" } } }
}
```

**Lifecycle.** Values fetched at host build time and merged into `IConfiguration` `[convey]`.

**Invariants & enforcement.** **Vault is disabled in every runnable profile**:
`appsettings.local.json:35-47` and `appsettings.docker.json:102-116` both set
`vault.enabled: false` and disable `kv`, `pki` and the lease. The compose stack
(`hianshul100_Pacco/compose/`) starts no Vault container. The block is aspirational, exactly as in
`identity-service` (`component-internals/identity-service.md` §3.37).

Note also that the `lease.mongo` block would rotate credentials for the database this service never
uses (§3.37), and that `token: "secret"` / `password: "secret"` are committed literals.

**Extension procedure.** If Vault is ever enabled, the `kv.path` (`operations-service/settings`) is
where `redis.connectionString`, `rabbitMq` credentials and `jwt.issuerSigningKey` should move.

**Failure modes.** Committed dev credentials (§8.2/B7); and unknown startup behaviour if
`enabled: true` were left on with no Vault reachable — `Unverifiable — Missing Source Evidence`.

### 3.42 Consul, Fabio, Jaeger and metrics

**Definition.** Four Convey modules registered at `Infrastructure/Extensions.cs:68-69` and `:73-74`
(`AddConsul()`, `AddFabio()`, `AddMetrics()`, `AddJaeger()`), activated at `:85` (`UseJaeger()`) and
`:87` (`UseMetrics()`).

| Facility | Config | Notable value |
| --- | --- | --- |
| Consul | `appsettings.json:7-17` | service `operations-service`, `pingEndpoint: "ping"`, `pingInterval: 3`, `removeAfterInterval: 3` — **the tightest health-check interval of any Pacco service** (identity uses 5/10) |
| Fabio | `:18-22` | service `operations-service` |
| Jaeger | `:80-88` | `serviceName: "operations"`, `sampler: "const"` (samples everything), `excludePaths: ["/", "/ping", "/metrics"]` |
| Metrics | `:89-97` | AppMetrics, `prometheusEnabled: true`, `interval: 5`; scraped by `hianshul100_Pacco/compose/prometheus/prometheus.yml:26-32` |
| Jaeger↔RabbitMQ | `Infrastructure/Extensions.cs:70` | `AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())` — joins consumer spans to the producer's trace via the `span_context` header (`appsettings.json:143`) |

**Lifecycle.** Registration/deregistration around the process lifetime `[convey]`.

**Invariants & enforcement.**

- `GET /ping` is supplied by Convey `[convey]`, not by `Program.cs` — a reader grepping
  `Program.cs` for a health endpoint finds only `GET ""`.
- **The health check is pure liveness.** It cannot detect the conditions that actually break this
  service: no `messages.json` (§3.9), an in-memory SignalR backplane (§3.25), or Redis being
  unreachable. All three present as a healthy, registered, scraped service that silently does
  nothing.
- `jaeger.serviceName: "operations"` (`:82`) is the odd one out (§3.40).

**Extension procedure.** Renaming the service touches `app.service`, `consul.service`,
`fabio.service`, `jaeger.serviceName`, `rabbitMq.connectionName`, `rabbitMq.queue.template`,
`redis.instance`, `mongo.database`, `compose/services.yml:51-67`, `hianshul100_Pacco/services.yml`
and `prometheus.yml:26-32`.

**Failure modes.** Liveness ≠ readiness, as above.

### 3.43 The `operations` exchange — declared, never published to

**Definition.** `appsettings.json:125-131`:

```json
"exchange": { "declare": true, "durable": true, "autoDelete": false,
              "type": "topic", "name": "operations" }
```

**Lifecycle.** Declared at broker connection `[convey]`; a durable, non-auto-delete topic exchange
that exists forever with **zero publishers and zero bindings**.

**Evidence that nothing publishes.** The scoped path contains no `IBusPublisher`, no
`IMessageBroker`, no `PublishAsync` call, no event class, no `[Contract]` attribute and no outbox
configuration. Grep for `Publish` finds only `IHubService`/`IHubWrapper`'s SignalR methods
(§3.20, §3.21). Nothing in `messages.json` references an `operations` exchange either — the eight
blocks are the eight *other* services.

**Why it is declared at all.** Convey's `AddRabbitMq` requires a default exchange for the
connection's conventions `[convey]`; the queue template
`"operations-service/{{exchange}}.{{message}}"` (`:137`) substitutes the *message's* exchange (from
the `[Message]` attribute, §3.10), not this one. So the setting is structural boilerplate, present
in all eight services.

**Invariants & enforcement.** None. It is inert.

**Extension procedure.** If this service ever needs to publish — say, an `operation_expired` event
— the exchange is already there; you would add `IBusPublisher` (already available through
`Convey.MessageBrokers.RabbitMQ`), an event type, and, for durability, the outbox packages that are
**not** currently referenced (`.csproj:10-33` has no `Convey.Persistence.MongoDB`-backed outbox
registration).

**Failure modes.** None operationally; it is a reading hazard only — the presence of an exchange
suggests an outbound contract that does not exist.

### 3.44 Deployment topology and the unreachable gRPC port

**Definition.** How the service actually runs.

| Facet | Evidence | Value |
| --- | --- | --- |
| Build | `Dockerfile:1-4` | SDK 3.1, `dotnet publish src/Pacco.Services.Operations.Api -c release -o out` |
| Runtime | `Dockerfile:6-11` | aspnet 3.1, `WORKDIR /app`, `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker` |
| Compose | `hianshul100_Pacco/compose/services.yml:51-67` | `5005:80`, `depends_on` **eight** services |
| PM2 | `hianshul100_Pacco/services.yml:18-25` | app name `operations` |
| Local | `scripts/start.sh` | `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Operations.Api`, `dotnet run` |
| Local URLs | `Properties/launchSettings.json:20` | `http://localhost:5005;https://localhost:50050` |
| CI | `.travis.yml:12-16` | `build.sh`, `test.sh`, then `dockerize.sh` on success |
| Image | `scripts/dockerize.sh:16` | `$DOCKER_USERNAME/pacco.services.operations`, tagged `latest` on `master`, `dev` on `develop` |
| Gateway | `ntrada.yml:277-289` | module `operations`, one route, `auth: false` |

**The gRPC problem.** `MapGrpcService<GrpcServiceHost>()` (`Program.cs:47`) is mapped onto the same
host as everything else. Locally, `launchSettings.json:20` binds **two** URLs — HTTP 5005 and HTTPS
50050 — and the sample client targets `https://localhost:50050`
(`GrpcClient/Program.cs:70-75`). In the container, `Dockerfile:9` binds **only** `http://*:80`, and
compose publishes only `5005:80`. So:

- There is **no HTTPS listener** in the deployed container, and therefore no TLS-protected HTTP/2
  endpoint.
- gRPC over cleartext HTTP/2 (h2c) requires the endpoint to be configured for
  `Http2` explicitly `[framework]`; nothing here does — there is no `Kestrel` section in any
  `appsettings` file and no `ConfigureKestrel` call.
- The .NET gRPC client refuses cleartext HTTP/2 unless the
  `System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport` switch is set `[framework]`; the
  sample client does not set it.

**Conclusion: the gRPC surface is reachable only in local development and is effectively dead in
the deployed topology.** Whether Kestrel's default `HttpProtocols` on a plain HTTP endpoint in
ASP.NET Core 3.1 is `Http1` or `Http1AndHttp2` is `Unverifiable — Missing Source Evidence` here,
but the absence of both an HTTPS listener and a published port makes the outcome the same. See
§8.3/Q3.

**Invariants & enforcement.** None; nothing checks that a mapped endpoint is reachable.

**Extension procedure.** To expose gRPC in Docker: add a `Kestrel` configuration section binding a
second endpoint with `Protocols: Http2`, extend `ASPNETCORE_URLS` in `Dockerfile:9`, and publish
the port in `compose/services.yml:55-56`. Then decide about TLS — which is the harder half.

**Failure modes.** A whole delivery channel (§3.27–§3.30) that cannot be used where it is deployed,
while still costing a per-call `GrpcServiceHost` construction and an event subscription leak
(§3.28) for anyone who does reach it locally.

### 3.45 The absent test suite

**Definition.** There is **no test project**. `find` over the repository shows four directories of
C# under `src/` belonging to two projects — `Pacco.Services.Operations.Api` and
`Pacco.Services.Operations.GrpcClient` — and no `tests/` directory, no `*.Tests.csproj`, and no
xUnit/NUnit/MSTest/Moq/Shouldly/FluentAssertions reference in either `.csproj`.

**And CI runs tests anyway** (`.travis.yml:12-16`):

```yaml
script:
  - ./scripts/build.sh      # dotnet build -c release
  - ./scripts/test.sh       # dotnet test
after_success:
  - ./scripts/dockerize.sh
```

`dotnet test` with no test project exits 0 `[framework]`, so `after_success` fires and the image is
pushed. **The pipeline is green by construction**, and it publishes to Docker Hub on every `master`
and `develop` build.

**Invariants & enforcement.** Every invariant in §3 is enforced only by the code implementing it.
That is why §3.24 (the missing `return`), §3.28 (the handler leak), §3.22 (the cross-repo group
format) and §3.14's triplication all survive: none is visible to the compiler, and nothing else is
looking.

**Extension procedure — the first tests, in order of value.**

1. `Services/OperationsService` — the terminal latch (§3.6) and the coalescing rules (§3.1) are
   pure logic over an `IDistributedCache` that is trivially faked with
   `MemoryDistributedCache` `[framework]`. Roughly eight cases cover the whole state machine.
2. `Handlers/Extensions.GetSagaState` (§3.16) — a pure function over `IMessageProperties`; four
   null paths and three success paths.
3. `Infrastructure/Extensions.ToUserGroup` (§3.22) — two overloads, one assertion each, and it
   pins the cross-repo coupling that is currently unpinned.
4. `Infrastructure/Subscriptions.SubscribeMessages` (§3.9) — an integration-shaped test with a fake
   `IBusSubscriber` would assert that all 80 manifest entries produce 80 `Subscribe` calls, which
   would have caught the baseline count discrepancy in §8.4.
5. The three generic handlers (§3.14) need four interface doubles each — all four dependencies are
   interfaces, so no framework is required.

**Failure modes.** A meaningless CI badge, and automatic publication of untested images.

---

## 4. Primary control flows

Nine flows. Each is traced entry point → function → datastore/side-effect, with file:line at every
hop. Flow 4.2 is the one the service exists for; the rest are supporting or degenerate.

### 4.1 Startup — composition order

```
docker run  →  ASPNETCORE_ENVIRONMENT=docker (Dockerfile:10)
  Program.Main (Program.cs:21)
    WebHost.CreateDefaultBuilder (:23)                  ← loads appsettings{,.docker}.json
      ConfigureServices (:25-27)
        services.AddConvey()                            ← core Convey container
                .AddWebApi()                            ← MVC-less endpoint routing
                .AddInfrastructure()  ────────────────► Infrastructure/Extensions.cs:49-79
                                                          :50  RequestsOptions   ← "requests"
                                                          :51  SignalrOptions    ← "signalR"
                                                          :53  IHubWrapper (singleton)
                                                          :54  IHubService (singleton)
                                                          :58  IOperationsService (singleton)
                                                          :59  AddGrpc()
                                                          :60  AddSignalR(...)   ← private, :95-109
                                                          :62-66 Convey CQRS + queries
                                                          :68-78 Consul/Fabio/Mongo/Rmq/Redis
                                                                 ×2/Metrics/Jaeger/Security
      Configure (:28-48)
        app.UseInfrastructure()  ──────────────────────► Infrastructure/Extensions.cs:81-93
                                                          :83 UseErrorHandler()
                                                          :84 UseSwaggerDocs()
                                                          :85 UseJaeger()
                                                          :86 UseConvey()
                                                          :87 UseMetrics()
                                                          :88 UseStaticFiles()
                                                          :89 UseRouting()
                                                          :90 UseCertificateAuthentication()
                                                          :91 UseRabbitMq()
                                                                .SubscribeMessages()  ─┐
           .UseEndpoints(Convey)   (:30-43)                                            │
                GET ""                       → "Operations Service"                    │
                GET operations/{operationId} → GetOperation query                      │
           .UseEndpoints(ASP.NET)  (:44-48)                                            │
                MapHub<PaccoHub>("/pacco")   (:46)                                     │
                MapGrpcService<GrpcServiceHost>() (:47)                                │
      .UseLogging() (:49)  .UseVault() (:50)  .Build().Run()                           │
                                                                                       ▼
                                                        Infrastructure/Subscriptions.cs:19-59
```

The `SubscribeMessages()` branch is flow 4.3. Note that `UseRouting()` at `:89` comes *before* the
`UseEndpoints` calls in `Program.cs` only because `UseInfrastructure()` is the first statement in
`Configure` — the ordering is correct by accident of call order, not by an explicit contract
(§3.36).

### 4.2 A message arrives → a browser gets a notification

This is the service's reason to exist. Entry point is RabbitMQ, not HTTP.

```
RabbitMQ delivers to queue  operations-service/{exchange}.{message}
  Convey RawRabbitMq consumer  [convey]
    ├─ reads header "message_context"   → ICorrelationContextAccessor        (appsettings.json:139-142)
    ├─ reads header "span_context"      → Jaeger child span                  (appsettings.json:143)
    ├─ populates IMessagePropertiesAccessor (MessageId, CorrelationId, Headers)
    └─ resolves ICommandHandler<T> / IEventHandler<T> / IEventHandler<TRejected>
         where T is a RUNTIME-EMITTED type (flow 4.3)
  GenericCommandHandler<T>.HandleAsync                    Handlers/GenericCommandHandler.cs:28
    :30  messageProperties = _messagePropertiesAccessor.MessageProperties
    :31  correlationId     = messageProperties?.CorrelationId
    :32-35 if blank → return           ◄── SILENT DROP (§3.15)
    :37  context = _contextAccessor.GetCorrelationContext()   ─► Infrastructure/Extensions.cs:35-47
                                                                  JSON round-trip of the raw context
    :38  name   = context?.Name ?? command.GetType().Name    ◄── falls back to the EMITTED type name
    :39  userId = context?.User?.Id                          ◄── may be null
    :40  state  = messageProperties.GetSagaState()           ─► Handlers/Extensions.cs:9-26
                     header "saga" byte[] → "pending"|"completed"|"rejected"
                  ?? OperationState.Pending
    :41  (updated, operation) = _operationsService.TrySetAsync(correlationId, userId, name, state)
           Services/OperationsService.cs:31
             :34  GetAsync(id)  → _cache.GetStringAsync("requests:{id}")   ── REDIS READ
                                   (physical key "operations:requests:{id}", appsettings.json:147)
             :35-38 miss → new OperationDto()
             :39-42 hit AND state is Completed|Rejected → return (false, operation)  ◄── LATCH (§3.6)
             :44-49 overwrite Id/UserId/Name/State/Code/Reason  ◄── null → string.Empty
             :50-55 SetStringAsync(..., SlidingExpiration = requests.expirySeconds)  ── REDIS WRITE
             :57  OperationUpdated?.Invoke(this, new OperationUpdatedEventArgs(operation))
                    └─► GrpcServiceHost instances' _operations.Add(...)  (flow 4.6)
             :59  return (true, operation)
    :42-45 if !updated → return        ◄── no notification for a latched operation
    :47-60 switch (state)
             Pending   → _hubService.PublishOperationPendingAsync(operation)    Services/HubService.cs:15
             Completed → ...CompletedAsync                                                        :25
             Rejected  → ...RejectedAsync                                                         :35
             default   → throw ArgumentException                 ◄── unreachable (3-value enum)
  HubService.PublishOperation*Async         Services/HubService.cs:15-45
    → _hubContextWrapper.PublishToUserAsync(operation.UserId, "operation_<state>", payload)
         payload = { id, name }                for pending/completed   (:18-22, :28-32)
         payload = { id, name, code, reason }  for rejected            (:38-44)
  HubWrapper.PublishToUserAsync             Services/HubWrapper.cs:18
    → _hubContext.Clients.Group(userId.ToUserGroup()).SendAsync(message, payload)
         ToUserGroup(string) → $"users:{userId}"        Infrastructure/Extensions.cs:33
  SignalR → Redis backplane (§3.25) → whichever instance holds the socket → browser
  app.js listener :34-44 → appendMessage → <ul id="messages">
```

**Three ways this flow produces nothing, all silent:**

| Condition | Where | Observable effect |
| --- | --- | --- |
| Publisher omitted `CorrelationId` | `GenericCommandHandler.cs:32-35` | message acked, no Redis write, no notification |
| Operation already `Completed`/`Rejected` | `OperationsService.cs:39-42` | later steps of a multi-step saga are invisible (§3.6) |
| `context?.User?.Id` is null | `:39` → `OperationsService.cs:45` | `UserId = ""`, group `"users:"`, which no connection ever joins (§3.22) |

**And one that is loud but wrong:** if `requests` is unbound, `SlidingExpiration = TimeSpan.Zero`
and the write is rejected by `IDistributedCache` `[framework]` — see §3.5 and §8.3/Q2.

The event and rejected-event variants are byte-identical except that
`GenericRejectedEventHandler.cs:41-42` passes `@event.Code, @event.Reason` into `TrySetAsync`'s two
optional parameters, which is the **only** reason a rejected operation carries a reason (§3.14).

### 4.3 `messages.json` → runtime-emitted types → live subscriptions

Runs once, inside `UseInfrastructure()` (`Infrastructure/Extensions.cs:91`), before the host starts
serving.

```
UseRabbitMq().SubscribeMessages()            Infrastructure/Subscriptions.cs:19
  :21  const string path = "messages.json"           ← relative to the process CWD
  :22-25 if (!File.Exists(path)) return;             ◄── SILENT EXIT #1  (§3.9)
  :27  json = File.ReadAllText(path)
  :28-31 if blank → return                           ◄── SILENT EXIT #2
  :33  messages = JsonConvert.DeserializeObject<Dictionary<string, ServiceMessages>>(json)
  :34-37 if (messages is null) return;               ◄── SILENT EXIT #3
  :42-44 AssemblyBuilder.DefineDynamicAssembly("Pacco.Messages", Run) → ModuleBuilder
  :45-52 foreach (service, definition) in messages
           BindMessages<Command>      (definition.Commands,       exchange)   ─┐
           BindMessages<Event>        (definition.Events,         exchange)    │
           BindMessages<RejectedEvent>(definition.RejectedEvents, exchange)    │
                                                                               ▼
  BindMessages<T>                                            Subscriptions.cs:61-83
    foreach message name:
      :72  moduleBuilder.DefineType(message, TypeAttributes.Public, typeof(T))
      :75-76 CustomAttributeBuilder(MessageAttribute ctor,
                                    new object[] { exchange, null, null, true })
                                      exchange, routingKey=null, queue=null, external=true
      :78  typeBuilder.CreateType()      → a field-less runtime type (except RejectedEvent's
                                            inherited Reason/Code, §3.11)
  SubscribeCommands / SubscribeEvents / SubscribeRejectedEvents   :85-152
    reflect IBusSubscriber.SubscribeCommand<T>() / SubscribeEvent<T>()
      :.. if (subscribeMethod is null) return;       ◄── SILENT EXIT #4 (API drift, §3.13)
      MakeGenericMethod(emittedType).Invoke(subscriber, null)
        → Convey declares queue "operations-service/{exchange}.{snake_case_message}"
          bound to {exchange} with routing key {snake_case_message}   (appsettings.json:132-138)
```

**Result: 80 queues** — 24 commands + 29 events + 27 rejected events across 8 exchanges
(`messages.json:1-152`, enumerated in §3.8). The baseline records 26/30/31; that is corrected in
§8.4.

**The `messages.json` deployment hazard.** `Pacco.Services.Operations.Api.csproj:1-47` contains
**no `<Content Include="messages.json">` item** — only `certs\**` (`:44-46`) and the `wwwroot`
folder marker (`:36-38`). The file is nonetheless copied because `Microsoft.NET.Sdk.Web` includes
`**/*.json` in `Content` by default `[framework]`. That default is load-bearing and undocumented:
if it changed, or if someone added an explicit `<Content Remove>`, the service would start
perfectly, register in Consul, answer `/ping`, and subscribe to **nothing** (silent exit #1).

### 4.4 `GET /operations/{operationId}` — the dead HTTP query

```
Client → Ntrada gateway, module "operations"       APIGateway/ntrada.yml:277-289
           GET /operations/{operationId}, use: downstream, auth: false
       → operations-service:80
  Convey endpoint             Program.cs:36-42
    :37  var operation = await service.GetAsync(operationId)   ← IOperationsService, NOT a dispatcher
    :38  if (operation is null) → ctx.Response.NotFound()
    :40  else → ctx.Response.Ok(operation)
```

The route is registered with `Get<GetOperation>(...)` (`Program.cs:36`), so Convey model-binds the
path segment into a `GetOperation` instance — but the lambda **ignores it** and reads
`operationId` from the route directly. `Queries/GetOperation.cs:6-9` declares
`IQuery<OperationDto>` and **no `IQueryHandler<GetOperation, OperationDto>` exists anywhere in the
repository** (§3.31), so `AddQueryHandlers()` (`Infrastructure/Extensions.cs:65`) registers none
and `IQueryDispatcher` would throw for this query. The query type survives only as a binding
vehicle.

`auth: false` in all four gateway manifests means **any caller who guesses or observes a
correlation id reads that operation's `UserId`, `Name`, `State`, `Code` and `Reason`** (§3.32,
§8.2/B4).

### 4.5 Browser connects → joins its user group

The only authentication path in the service, and it is not middleware.

```
browser → wwwroot/ui/index.html → app.js:6-9
            new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco')
  app.js:13-16  client-side guard: blank / whitespace token → alert, abort
  app.js:19     connection.start()
                  → WebSocket upgrade at /pacco   (Program.cs:46, MapHub<PaccoHub>)
                  → ANONYMOUS: no [Authorize], no UseAuthentication() (§3.36)
  app.js:21     connection.invoke('initializeAsync', $jwt.value)
                  SignalR resolves hub methods case-insensitively [framework]
                  → PaccoHub.InitializeAsync(token)          Hubs/PaccoHub.cs:18
    :20-23  if (string.IsNullOrWhiteSpace(token)) await DisconnectAsync();
              ◄── NO `return` — execution FALLS THROUGH (§3.24)
    :26     payload = _jwtHandler.GetTokenPayload(token)     ← Convey, validates signature,
                                                               issuer, audience, lifetime [convey]
    :27-32  if (payload is null) → DisconnectAsync(); return;
    :34     group = Guid.Parse(payload.Subject).ToUserGroup()
              Infrastructure/Extensions.cs:32 → Guid.ToString("N")
              Infrastructure/Extensions.cs:33 → $"users:{userId}"
              ◄── must equal the "N" format issued by
                  Pacco.Services.Identity …/Auth/JwtProvider.cs:21   (§3.22)
    :35     Groups.AddToGroupAsync(Context.ConnectionId, group)
              → Redis backplane records the membership (§3.25)
    :36     ConnectAsync() → Clients.Client(ConnectionId).SendAsync("connected")
    :38-41  catch { DisconnectAsync(); }   ◄── bare catch, no logging (§3.24)
```

**What "disconnect" means here.** `DisconnectAsync` (`:49-52`) **sends a message named
`"disconnected"`** and returns. It does not call `Context.Abort()`, does not remove the connection
from any group, and does not close the socket. The connection stays open and the client may invoke
`initializeAsync` again — with a different token, unlimited times, with no rate limiting.

**The blank-token fall-through in full.** With `token = ""`:

1. `:22` sends `"disconnected"` — the browser prints *"Disconnected, invalid token."* (`app.js:31`).
2. Execution continues into the `try` at `:24`.
3. `_jwtHandler.GetTokenPayload("")` either returns `null` or throws.
4. Either way the connection is sent a **second** `"disconnected"` (`:29` or `:40`).

So the observable defect is a duplicate message, not an auth bypass — the group join at `:35` is
still gated on a validated payload. It is nevertheless a missing `return` in an authentication
routine, which is exactly the class of code where fall-through must never be tolerated. See
§8.2/B2.

### 4.6 gRPC `SubscribeOperations` — the server-stream fan-out

```
gRPC client → SubscribeOperations(Empty)          Operations.proto:11
  ASP.NET Core gRPC constructs a NEW GrpcServiceHost per call  [framework]
                                                    (transient by convention, §3.28)
    ctor Infrastructure/GrpcServiceHost.cs:17-22
      :15  new BlockingCollection<OperationDto>()   ← PER-INSTANCE, unbounded
      :21  _operationsService.OperationUpdated += (s, e) => _operations.TryAdd(e.Operation)
             ◄── subscribes to the SINGLETON's event (Infrastructure/Extensions.cs:58)
             ◄── NEVER unsubscribed: no IDisposable, no -=
  SubscribeOperations                                              :32-41
    :35  log the peer
    :36  while (true)
    :38    var operation = _operations.Take();     ← blocks a thread-pool thread
    :39    await responseStream.WriteAsync(Map(operation))
```

Meanwhile, flow 4.2 line `OperationsService.cs:57` invokes `OperationUpdated`, which pushes into
**every** live `_operations` collection.

**Three defects, all in these ten lines:**

| Defect | Line | Consequence |
| --- | --- | --- |
| Handler never removed | `:21` | every gRPC call that ever ran leaves a closure rooted by the singleton's event; the `GrpcServiceHost`, its `BlockingCollection` and every `OperationDto` ever added stay reachable — an unbounded leak growing at the rate of *(calls) × (operations)* |
| No cancellation | `:36-40` | `context.CancellationToken` is ignored; `Take()` has no token overload here, so after the client disconnects the loop still blocks, and the next `Add` makes `WriteAsync` throw into a background thread |
| Unbounded collection | `:15` | a slow or gone consumer accumulates every operation forever |

**And a semantic one:** because *each* call gets its own collection, `SubscribeOperations` is a
broadcast of **all users' operations to every subscriber**, with no filtering and no authentication
(§3.29). Whether that matters in practice is limited by §3.44 — the endpoint is unreachable in the
deployed topology.

### 4.7 gRPC `GetOperation`

```
gRPC client → GetOperation(GetOperationRequest{ id })     Operations.proto:10
  new GrpcServiceHost (same ctor side-effect as 4.6 — a leaked subscription per unary call)
  GetOperation                                     GrpcServiceHost.cs:24-30
    :27  _logger.LogInformation($"... (id: {request.Id}) request from: {context.Peer}")
    :29  return Map(await _operationsService.GetAsync(request.Id))
           OperationsService.cs:24-29 → Redis GET "operations:requests:{id}"
  Map                                              GrpcServiceHost.cs:43-54
    :44-45  operation is null → new GetOperationResponse()   ← ALL FIELDS EMPTY STRINGS
    :48-53  else project; State = Enum.ToString().ToLowerInvariant()
```

**Not-found is indistinguishable from found-but-empty.** proto3 scalar `string` fields have no
presence `[framework]`, so an unknown id returns `OK` with six empty strings. The sample client
compensates by sniffing a blank `Id` (`GrpcClient/Program.cs:125-129`) instead of receiving
`StatusCode.NotFound` — a contract encoded in the client, not the server. The HTTP route for the
same lookup *does* return 404 (`Program.cs:38`), so the two read surfaces disagree on the same
question.

### 4.8 Any exception → HTTP 400

```
any unhandled exception in an endpoint / query / middleware below UseErrorHandler
  Convey ErrorHandlerMiddleware      [convey]   (registered Infrastructure/Extensions.cs:83)
    → IExceptionToResponseMapper.Map(exception)
  ExceptionToResponseMapper.Map      Infrastructure/ExceptionToResponseMapper.cs:7-15
    :9-14  switch (exception) { _ => new ExceptionResponse(
                                       new { code = "error", reason = "There was an error." },
                                       HttpStatusCode.BadRequest) }
```

There is exactly one arm — a discard. The mapping table:

| Actual condition | Correct status | Reported |
| --- | --- | --- |
| Redis unreachable | 503 | **400** `"There was an error."` |
| Malformed route parameter | 400 | 400 |
| Bug in the query path (e.g. resolving the non-existent `GetOperation` handler, §4.4) | 500 | **400** |
| `SlidingExpiration = Zero` rejection (§3.5) | 500 | **400** |

Every infrastructure failure is reported to the caller — and to the gateway, and to any dashboard
counting 5xx — as *the client's fault*. Combined with §3.42 (liveness-only health checks) and §3.45
(no tests), an outage of the single datastore this service depends on is close to invisible.

The corresponding message-side mapper (`IExceptionToMessageMapper`) **does not exist** here at all,
because the service never publishes (§3.43); a handler that throws simply nacks per Convey's
RabbitMQ retry configuration (`appsettings.json:106-112`).

### 4.9 Expiry — how an operation disappears

There is no expiry *flow*; there is an absence of one.

```
OperationsService.TrySetAsync :50-55
  SetStringAsync(..., new DistributedCacheEntryOptions { SlidingExpiration = 300s })
    → StackExchange.Redis SET with a sliding TTL managed by the cache provider [framework]
  … 300 s of no read and no write …
    → Redis evicts the key
```

Consequences, none of them signalled:

- `GetAsync` (`:24-29`) returns `null`; the HTTP route 404s (`Program.cs:38`); gRPC returns an empty
  response (§4.7).
- Because the expiry is **sliding**, a read keeps the entry alive — so `GET /operations/{id}` in a
  polling loop pins a completed operation indefinitely.
- A late message for an expired correlation id re-creates the entry from scratch
  (`:35-38`), losing the terminal state and therefore **defeating the latch** (§3.6): a `Completed`
  operation that expires and then receives a stale `Pending` message becomes `Pending` again and
  re-notifies the browser.
- Nothing is archived. There is no audit trail of any operation once its key expires — see §5.4.

---

## 5. Persistence & schema evolution

### 5.1 What actually stores state

| Store | Registered | Used by | Verdict |
| --- | --- | --- | --- |
| **Redis** (`IDistributedCache`) | `Infrastructure/Extensions.cs:72` and again `:75` (§3.35) | `Services/OperationsService.cs:26`, `:50` | **the only state store** |
| **Redis** (SignalR backplane) | `Infrastructure/Extensions.cs:106` (`signalR.AddRedis(...)`, the legacy `Microsoft.AspNetCore.SignalR.Redis` 1.1.5 package) | group membership, message fan-out | infrastructure state, not domain state |
| **MongoDB** | `Infrastructure/Extensions.cs:71` (`AddMongo()`) | *nothing* | **inert** — no `AddMongoRepository`, no `IMongoRepository<>` injection, no document type (§3.37) |

This resolves baseline gap **G4 / Q4** (`baselines/service-summaries.md`), which recorded the
operations-service datastore as unknown: **it is Redis, and the Mongo configuration is dead**.

### 5.2 The one record shape

**Logical key.** `GetKey(id) => $"requests:{id}"` (`Services/OperationsService.cs:62`), where `id`
is the inbound RabbitMQ **`CorrelationId`** (`GenericCommandHandler.cs:31`, `:41`).

**Physical key.** `redis.instance` is `"operations:"` (`appsettings.json:147`), and the Convey
Redis module passes it as the `InstanceName` `[convey]`, which `RedisCache` prefixes to every key
`[framework]`. The key in the database is therefore:

```
operations:requests:{correlationId}
```

**Value.** `JsonConvert.SerializeObject(operation)` (`:51`) of `DTO/OperationDto.cs:5-13`:

| Property | Type | Populated from | Null handling |
| --- | --- | --- | --- |
| `Id` | `string` | the correlation id | `:44`, never null |
| `UserId` | `string` | `context?.User?.Id` | `:45` `?? string.Empty` |
| `Name` | `string` | `context?.Name` or the emitted CLR type name | `:46`, never null |
| `State` | `OperationState` | the `saga` header, else `Pending` | enum |
| `Code` | `string` | rejected events only | `:48` `?? string.Empty` |
| `Reason` | `string` | rejected events only | `:49` `?? string.Empty` |

Serialized with **default Json.NET settings** — there is no `JsonSerializerSettings`, no
`StringEnumConverter`, and no camel-case resolver on this path. Therefore:

- Property names are **PascalCase** on the wire: `{"Id":"…","UserId":"…","State":1,…}`.
- `State` is persisted as the **integer** enum value — `0` Pending, `1` Completed, `2` Rejected
  (`Types/OperationState.cs:3-8`).

That integer encoding is the single most fragile thing in this store: **reordering the enum members
silently re-labels every live record**. A `Completed` operation persisted as `1` becomes whatever
member now occupies ordinal 1.

Note the contrast with the *outbound* representations, both of which are strings:
`GrpcServiceHost.cs:53` lower-cases the enum name, and the HTTP route serializes through Convey's
camelCase-configured MVC-less writer `[convey]`. Three encodings of one enum.

### 5.3 Schema evolution — what happens when `OperationDto` changes

There is **no migration tooling** in this repository: no EF Core, no Mongo migration package, no
`scripts/migrate*`, no versioned document field, and no startup schema step (contrast
`identity-service`, whose fire-and-forget unique index is the *only* schema action anywhere on the
platform — `component-internals/identity-service.md` §3.24). This confirms baseline **G11 / A6**.

Evolution is therefore governed entirely by Json.NET's deserialization defaults
(`OperationsService.cs:28`) applied to whatever bytes are already in Redis:

| Change | Effect on records written by the previous version | Loud or silent? |
| --- | --- | --- |
| **Add** a property | absent in old JSON → CLR default (`null` for `string`) | silent; watch for `NullReferenceException` downstream |
| **Remove** a property | extra key in old JSON → ignored by default | silent, safe |
| **Rename** a property | old value dropped, new one defaults | **silent data loss** |
| **Retype** `string` → `int` etc. | `JsonSerializationException` inside `GetAsync` | **loud**, but surfaces as HTTP 400 (§4.8) |
| **Append** an enum member | old integers keep meaning | silent, safe |
| **Reorder/insert** an enum member | every persisted `State` re-labels | **silent corruption** — the worst case |
| Change `redis.instance` | old keys unreachable; effectively a full flush | silent |
| Change `GetKey`'s prefix | same | silent |

**The mitigating fact:** every record self-destructs after 300 s of inactivity (§4.9). The migration
strategy — never stated anywhere in the repository — is *wait five minutes*. A rolling deployment
in which two versions run concurrently is the only real exposure window, and for a
`string`-dominated DTO the additive cases are benign. This is a legitimate design for ephemeral
state; it is dangerous only because nothing records that the design **depends** on the TTL.

### 5.4 What is not persisted

- **No durability guarantee.** `hianshul100_Pacco/compose/infrastructure.yml:91-100` runs a stock
  `redis` image with a named volume at `/data` (`:99-100`, declared `:144-145`) and **no custom
  `redis.conf` and no command override** — so whatever the image's default persistence is applies,
  unchosen. The service itself makes no durability assumption either way and would not detect a
  loss: after a Redis restart, `GetAsync` simply returns `null` and subsequent messages re-create
  entries from scratch, with the latch-defeat of §4.9.
- **No history.** `TrySetAsync` overwrites in place (`:44-49`). The transition sequence
  Pending → Completed is not recoverable; only the latest state exists.
- **No audit.** There is no append-only log, no `CreatedAt`/`UpdatedAt`, and no user-attributed
  record of who triggered what. `OperationDto` has no timestamp of any kind.
- **No outbox.** The service publishes nothing (§3.43), so there is no outbox collection — unlike
  `identity-service`, which configures one (`identity-service.md` §3.30).
- **No inbox / no idempotency.** Convey's message inbox is not enabled. A redelivered message
  re-runs `TrySetAsync` and re-notifies the browser; duplicate notifications are the designed
  behaviour, not a bug, but they are also not deduplicated anywhere.

### 5.5 Concurrency: the read-modify-write is not atomic

`TrySetAsync` (`:31-60`) is `GET` → mutate in memory → `SET`. Two messages for the same correlation
id landing on two instances — which the compose topology does not produce today (one replica,
`compose/services.yml:51-67`) but which any scale-out would — interleave as:

```
inst A: GET requests:X → {State: Pending}
inst B: GET requests:X → {State: Pending}
inst A: SET requests:X → {State: Completed}
inst B: SET requests:X → {State: Rejected}      ← A's terminal state silently overwritten
```

The terminal-state latch at `:39-42` is checked against a **stale read** and therefore does not
protect against this. Redis offers `WATCH`/`MULTI` and Lua scripting to make it atomic, but
`IDistributedCache` exposes neither `[framework]`; fixing it requires dropping to
`IConnectionMultiplexer`. Today the exposure is bounded by the single replica — a constraint written
nowhere. See §8.1/A4 and §8.3/Q5.

### 5.6 The gRPC contract is a second, independently versioned schema

`Operations.proto:1-24` defines `GetOperationResponse` with six `string` fields
(`id=1, userId=2, name=3, state=4, code=5, reason=6`). This is a genuine wire schema with real
compatibility rules `[framework]` — field numbers are the contract, names are not — and it is
**duplicated by hand**: `src/Pacco.Services.Operations.GrpcClient/` carries its own copy of the
`.proto`. There is no shared package, no `buf`-style lint, and no check that the two files agree.
Regenerating one and not the other produces a silently mismatched client. Note also that `Empty` is
hand-declared rather than imported from `google/protobuf/empty.proto`.

---

## 6. Surface → internals map

Read this table right-to-left when debugging: given a symptom on a surface, it names the code that
produces it.

### 6.1 HTTP

| Surface | Registered | Handler | Reads/writes | Auth | Notes |
| --- | --- | --- | --- | --- | --- |
| `GET /` | `Program.cs:31-35` | inline lambda → `"Operations Service"` | none | none | liveness banner |
| `GET /operations/{operationId}` | `Program.cs:36-42` | inline lambda → `IOperationsService.GetAsync` | Redis read | **none** (§4.4) | binds `GetOperation` and ignores it; `GetOperation` has **no handler** (§3.31) |
| `GET /ping` | Convey `[convey]` | Convey | none | none | Consul health target (`appsettings.json:12`) |
| `GET /metrics` | AppMetrics `[convey]` | AppMetrics | none | none | Prometheus scrape target |
| `GET /swagger*` | `UseSwaggerDocs()` `Infrastructure/Extensions.cs:84` | Swashbuckle | none | none | documents only the two Convey routes |
| `GET /ui/index.html` + `/ui/js/*` | `UseStaticFiles()` `:88` | static file middleware | disk | none | §3.39 |
| *(any error)* | `UseErrorHandler()` `:83` | `Infrastructure/ExceptionToResponseMapper.cs:7-15` | — | — | **everything → 400** (§4.8) |

**Absent from this list and worth noting:** no `POST`, `PUT`, `PATCH` or `DELETE` anywhere. The
service is read-only over HTTP; all mutation arrives over RabbitMQ.

### 6.2 SignalR

| Surface | Registered | Internals |
| --- | --- | --- |
| Hub endpoint `/pacco` | `Program.cs:46` (`MapHub<PaccoHub>`) | `Hubs/PaccoHub.cs:9` |
| Client→server `initializeAsync(token)` | `Hubs/PaccoHub.cs:18` | `IJwtHandler.GetTokenPayload` → `ToUserGroup` → `Groups.AddToGroupAsync` (§4.5) |
| Server→client `connected` | `PaccoHub.cs:46` | after a successful group join |
| Server→client `disconnected` | `PaccoHub.cs:51` | blank token, null payload, or **any** exception |
| Server→client `operation_pending` | `Services/HubService.cs:17` | payload `{ id, name }` |
| Server→client `operation_completed` | `Services/HubService.cs:27` | payload `{ id, name }` |
| Server→client `operation_rejected` | `Services/HubService.cs:37` | payload `{ id, name, code, reason }` |
| Group naming | `Infrastructure/Extensions.cs:32-33` | `users:{guid:N}` — **must** match `identity-service`'s JWT subject format (§3.22) |
| Backplane | `Infrastructure/Extensions.cs:95-109` | Redis if `signalR.backplane == "redis"`, **silently in-memory otherwise** (§3.25) |

`IHubWrapper.PublishToAllAsync` (`Services/HubWrapper.cs:20-21`) is implemented and **never
called** — dead code (§3.21).

### 6.3 gRPC

| RPC | Proto | Implementation | Behaviour |
| --- | --- | --- | --- |
| `GetOperation(GetOperationRequest)` | `Operations.proto:6` | `GrpcServiceHost.cs:24-30` | Redis read; missing id → `OK` + all-empty response, **not** `NotFound` (§4.7) |
| `SubscribeOperations(Empty) returns (stream …)` | `Operations.proto:7` | `GrpcServiceHost.cs:32-41` | per-call `BlockingCollection` fed by the singleton's event; **all users' operations, unfiltered, unauthenticated, uncancellable** (§4.6) |

Reachable **only in local development** (§3.44).

### 6.4 RabbitMQ

| Surface | Source | Internals |
| --- | --- | --- |
| 80 queues `operations-service/{exchange}.{message}` | `messages.json:1-152` → `Infrastructure/Subscriptions.cs:19-152` | runtime-emitted types (§3.10) → the three generic handlers |
| Commands (24) | `definition.Commands` | `Handlers/GenericCommandHandler.cs:28` |
| Events (29) | `definition.Events` | `Handlers/GenericEventHandler.cs` |
| Rejected events (27) | `definition.RejectedEvents` | `Handlers/GenericRejectedEventHandler.cs` — the only one that reads payload data (`Code`, `Reason`) |
| Header `message_context` | `appsettings.json:139-142` | `ICorrelationContextAccessor` → `Infrastructure/Extensions.cs:35-47` → `Name`, `User.Id` |
| Header `span_context` | `appsettings.json:143` | Jaeger parent span (§3.42) |
| Header `saga` | `Handlers/Extensions.cs:9-26` | `pending`/`completed`/`rejected` → `OperationState` |
| Property `CorrelationId` | AMQP basic property | the Redis key; **blank ⇒ the message is silently dropped** (§3.15) |
| Exchange `operations` | `appsettings.json:125-131` | declared, **never published to** (§3.43) |

### 6.5 Which surface answers which question

| Question | Surface | Caveat |
| --- | --- | --- |
| "Is the service up?" | `GET /ping` | liveness only; says nothing about subscriptions, Redis or the backplane (§3.42) |
| "What is operation X doing?" | `GET /operations/{id}` **or** gRPC `GetOperation` | the two disagree on not-found (§4.7); neither is authenticated |
| "Tell me when my operations change" | SignalR `/pacco` | requires a valid identity JWT and the matching group format |
| "Tell me when *any* operation changes" | gRPC `SubscribeOperations` | unauthenticated firehose; local-only |
| "What happened before?" | *none* | no history, no audit, 300 s TTL (§5.4) |

---

## 7. Change/extension guide

### 7.1 Track a new message from an existing service

**Edit one file: `messages.json`.** Add the message's snake_case name to the right array under the
right exchange. No C# change, no rebuild of any handler, no new type — `Subscriptions.cs:61-83`
emits the type and `:85-152` subscribes it at the next start.

Choose the array by the message's base semantics, not by its name:

| Array | Emitted base | What you get |
| --- | --- | --- |
| `commands` | `Types/Command.cs` | state from the `saga` header, default `Pending` |
| `events` | `Types/Event.cs` | same |
| `rejectedEvents` | `Types/RejectedEvent.cs` | same **plus** `Reason` and `Code` deserialized into the operation (§3.11) |

Verify afterwards that the queue appeared — there is no log line, no metric and no health signal
confirming a subscription (§3.9). Bind and publish once by hand.

### 7.2 Track a message from a **new** service

1. Add a top-level block to `messages.json` keyed by the service name, with `exchange` and the three
   arrays. `ordermaker-service` (`messages.json:75-83`) shows that omitting `commands` is legal —
   `BindMessages` iterates a null-safe collection.
2. Confirm the publishing service actually declares that exchange with that name; a typo produces a
   queue bound to an exchange nobody publishes to, and no error.
3. Confirm the publisher sets `CorrelationId` and the `message_context` header — without them the
   message is dropped (§4.2) or renders with a CLR type name instead of a human label.

Note the existing inconsistency: the identity block (`messages.json:60-74`) declares `sign_in`,
which `identity-service` never publishes (`identity-service.md` §3.29). Manifest entries are not
validated against reality.

### 7.3 Add a field to an operation

1. Add the property to `DTO/OperationDto.cs`.
2. Populate it in `Services/OperationsService.cs:44-49` — and add a parameter to `TrySetAsync`
   (`:31-32`) if it comes from the message, following `code`/`reason`'s optional-parameter pattern.
3. Pass it from **all three** handlers (`Handlers/Generic*Handler.cs:41`) — the triplication of
   §3.14 means a change made in one is a change missing from two, with no compiler help.
4. Decide whether it belongs in the SignalR payloads (`Services/HubService.cs:18-22`, `:28-32`,
   `:38-44`) and the gRPC projection (`GrpcServiceHost.cs:46-54`).
5. If gRPC: add the field to **both** `.proto` copies with a **new field number** (§5.6).
6. Re-read §5.3 — an added property is silently `null` on records written by the previous version.

### 7.4 Add a notification type

`HubService` + `app.js` + the `switch` in all three handlers. Because the state machine is the
three-member `OperationState` enum, a genuinely new *state* means editing
`Types/OperationState.cs` — and see §5.2: **append, never insert**, or every persisted record
re-labels.

### 7.5 Fix the terminal-state latch

`Services/OperationsService.cs:39-42` is the single point. Today any message arriving after a
terminal state is discarded, which freezes multi-step sagas (§3.6). The minimal correct change is
to key the store by `(correlationId, step)` or to keep a transition list rather than a single
state — both of which change the record shape (§5.3) and the SignalR payloads.

### 7.6 Make the read surfaces safe

Three independent changes, in increasing order of blast radius:

1. **Gateway** — flip `auth: false` to `true` on the `operations` module in **all four** manifests
   (`ntrada.yml:277-289`, `ntrada.docker.yml`, `ntrada-async.yml:321-333`,
   `ntrada-async.docker.yml`). Cheapest, and closes the public read (§4.4).
2. **Service** — add `UseAuthentication()` and Convey's `UseAccessTokenValidator()` to
   `UseInfrastructure` (`Infrastructure/Extensions.cs:81-93`) and an authorization check on the
   route. Note `jwt.allowAnonymousEndpoints` (`appsettings.json:42`) is configured but currently
   dead because the validator is not installed (§3.38).
3. **Ownership** — even authenticated, the route returns another user's operation. Compare
   `operation.UserId` against the caller's subject.

### 7.7 Fix the gRPC leak

`Infrastructure/GrpcServiceHost.cs:21`. Either implement `IDisposable`/`IAsyncDisposable` and
`-=` the handler, or — better — move the fan-out into a singleton that owns per-subscriber channels
keyed by `context.CancellationToken`. Replace `while (true) { _operations.Take(); }` (`:36-40`)
with `GetConsumingEnumerable(context.CancellationToken)` or a `System.Threading.Channels` reader.
See §4.6.

### 7.8 Change the SignalR group format

Do not, without coordinating. `Infrastructure/Extensions.cs:32-33` must agree with
`Pacco.Services.Identity/…/Auth/JwtProvider.cs:21`'s `ToString("N")`. There is no shared constant
and no test (§3.45); a divergence produces a service that connects, authenticates and then never
delivers a notification — with no error anywhere (§3.22).

### 7.9 Scale out

Two blockers, in order:

1. **`TrySetAsync` is not atomic** (§5.5). Fix before adding a replica.
2. **Every instance subscribes to all 80 queues.** Convey binds each instance's consumer to the
   *same* queue name, so RabbitMQ round-robins deliveries across instances `[convey]` — which is
   what you want, and which also means the terminal-latch race of §5.5 becomes reachable
   immediately.

The SignalR backplane is already Redis-backed (`appsettings.json:153`), so socket affinity is not a
blocker — provided `signalR.backplane` is spelled exactly `"redis"` (§3.25).

### 7.10 Add the first test

See §3.45 for a prioritised list. The container is plain `IServiceCollection`, every collaborator is
an interface, and `MemoryDistributedCache` substitutes for Redis with no ceremony — there is no
structural obstacle, only the absence of a project.

---

## 8. Assumptions, Blockers & Open Questions (ABQ)

### 8.1 Assumptions

| # | Assumption | Basis | If wrong |
| --- | --- | --- | --- |
| **A1** | Statements marked `[convey]` describe **Convey 0.4.\*** package behaviour. The packages are referenced (`.csproj:10-27`) but their sources are **not** in this workspace | inference from call sites and configuration keys | Convey-attributed behaviour — queue naming, header extraction, `/ping`, error middleware, `GetTokenPayload` — may differ; every such claim is marked |
| **A2** | Statements marked `[framework]` describe .NET Core 3.1, ASP.NET Core, `IDistributedCache`, SignalR, gRPC or Json.NET defaults | inference | same caveat |
| **A3** | `messages.json` reaches the publish output via the Web SDK's default `Content` glob | `.csproj:1-47` has **no** item for it, yet `Subscriptions.cs:21` reads it from the CWD and the service demonstrably works | if the glob ever excludes it, the service starts healthy and subscribes to nothing (§4.3) |
| **A4** | One replica is deployed | `compose/services.yml:51-67` and `hianshul100_Pacco/services.yml:18-25` declare a single instance each | the non-atomic read-modify-write of §5.5 becomes reachable |
| **A5** | Every repository was read at base ref `feature/12998/aidlc` | run configuration | line citations drift |
| **A6** | No migration tooling exists anywhere on the platform | exhaustive search across all 14 clones; the only schema action found is the fire-and-forget unique index in `Identity.Infrastructure/Mongo/Extensions.cs:13-28` | §5.3's conclusion changes |
| **A7** | SignalR resolves hub methods case-insensitively | `app.js:21` invokes `initializeAsync`; `PaccoHub.cs:18` declares `InitializeAsync`; the pair is the shipped demo | the only auth path in the service never fires |
| **A8** | `redis.instance` is applied as `RedisCache.InstanceName` | Convey's `AddRedis` binds a `RedisOptions` with an `Instance` property; §5.2's physical key follows | the physical key is `requests:{id}` without the prefix — harmless, but `redis-cli` lookups fail |

### 8.2 Blockers & risks

Ordered by severity. Every entry is a defect in the code as written, not a stylistic preference.

| # | Severity | Blocker | Evidence | Effect |
| --- | --- | --- | --- | --- |
| **B1** | high | **Unpinned cross-repo coupling.** The SignalR group name `users:{guid:N}` must match the `"N"`-formatted JWT subject issued by a *different repository* | `Infrastructure/Extensions.cs:32-33` vs `Pacco.Services.Identity/…/Auth/JwtProvider.cs:21` | a format change in identity silently stops every notification; no shared constant, no test, no error |
| **B2** | high | **Unauthenticated read of any operation.** The gateway route sets `auth: false` in all four manifests; the service installs no authentication middleware | `ntrada.yml:277-289`, `ntrada-async.yml:321-333`, `Infrastructure/Extensions.cs:81-93` | anyone with a correlation id reads another user's `UserId`, `Name`, `State`, `Code`, `Reason` (§4.4) |
| **B3** | high | **Committed secrets.** The JWT `issuerSigningKey` is shared verbatim across services and gateway manifests; `vault.token`/`vault.password` are `"secret"`; Seq `apiKey` is `"secret"` | `appsettings.json:36`, `:164-193`, `:47-60`; identical key in `ntrada*.yml` | a single leaked key forges tokens for the whole platform; confirms baseline **G13** |
| **B4** | high | **`GrpcServiceHost` never unsubscribes** from the singleton's `OperationUpdated` event, and its stream loop ignores cancellation | `GrpcServiceHost.cs:21`, `:36-40` | unbounded memory growth per gRPC call; background exceptions after client disconnect (§4.6) |
| **B5** | medium‑high | **The terminal-state latch freezes multi-step workflows.** Once an operation is `Completed` or `Rejected`, every later message for that correlation id is discarded | `Services/OperationsService.cs:39-42` | a saga whose first step completes shows nothing thereafter (§3.6) |
| **B6** | medium‑high | **Eight silent failure paths.** Three early `return`s in `SubscribeMessages`; three `if (subscribeMethod is null) return;` guards; the blank-`CorrelationId` drop; the non-`"redis"` backplane fall-through | `Subscriptions.cs:22-25`, `:28-31`, `:34-37`, `:93`, `:116`, `:139`; `GenericCommandHandler.cs:32-35`; `Infrastructure/Extensions.cs:100-102` | the service reports healthy in Consul, answers `/ping`, is scraped by Prometheus, and does nothing |
| **B7** | medium | **Missing `return` in the hub's blank-token path** | `Hubs/PaccoHub.cs:20-23` | duplicate `"disconnected"` message today; a fall-through in an authentication routine structurally (§4.5) |
| **B8** | medium | **Every exception becomes HTTP 400.** A single discard arm | `Infrastructure/ExceptionToResponseMapper.cs:9-14` | Redis outages are reported as client errors and are invisible to 5xx-based alerting (§4.8) |
| **B9** | medium | **No tests, and CI is green by construction.** `dotnet test` with no test project exits 0, so `after_success` publishes an image | `.travis.yml:12-16`; no `*.Tests.csproj` anywhere | every invariant in §3 is unenforced; untested images ship automatically (§3.45) |
| **B10** | medium | **`OperationState` is persisted as an integer** under default Json.NET settings | `OperationsService.cs:51` + `Types/OperationState.cs:3-8` | inserting or reordering an enum member silently re-labels every live record (§5.2) |
| **B11** | low‑medium | **gRPC is unreachable in the deployed topology.** No HTTPS listener, no `Kestrel` HTTP/2 configuration, no published port | `Dockerfile:9`, `compose/services.yml:55-56`, no `Kestrel` section in any profile | a whole delivery channel exists only on a developer's machine (§3.44) |
| **B12** | low | **Legacy SignalR packages.** `Microsoft.AspNetCore.SignalR` 1.1.0 and `…SignalR.Redis` 1.1.5 referenced from a `netcoreapp3.1` web project | `.csproj:31-32` | 2.x-era packages against a 3.1 shared framework; the upgrade to `…SignalR.StackExchangeRedis` is unmade (§3.25) |
| **B13** | low | **Duplicate `AddRedis()` and an unconfigured `AddSecurity()`** | `Infrastructure/Extensions.cs:72`, `:75`, `:78`; no `security` section in any profile | dead/redundant registration; a reader cannot tell which is intentional (§3.35) |
| **B14** | low | **`docker` profile duplicates rather than overrides** `requests`, `signalR`, `mongo`, `jwt`, `swagger` | `appsettings.docker.json` vs `appsettings.json` | a base-file change silently has no effect in Docker (§3.40) |
| **B15** | low | **Handler triplication.** Three byte-identical handlers differing only in the rejected variant's two extra arguments | `Handlers/Generic{Command,Event,RejectedEvent}Handler.cs` | any change must be made three times with no compiler help (§3.14) |

### 8.3 Open questions

Each requires evidence this repository does not contain. **None is answered by guessing.**

| # | Question | Why it cannot be settled statically | How to settle it | Owner |
| --- | --- | --- | --- | --- |
| **Q1** | What are the actual **wire payloads** of the 80 messages consumed? | `Subscriptions.cs:61-83` emits field-less types; only `RejectedEvent`'s inherited `Reason`/`Code` are ever read | capture off RabbitMQ, or read each publisher's `Events/`/`Commands/` folder in the eight owning repositories | platform architect |
| **Q2** | What happens if the `requests` section is missing? | `RequestsOptions.ExpirySeconds` would be `0` → `SlidingExpiration = TimeSpan.Zero`; whether `IDistributedCache` throws or writes a non-expiring entry is framework behaviour not present here | unit test with `MemoryDistributedCache` and a `RedisCache` | any engineer, 30 min |
| **Q3** | Is the gRPC endpoint reachable over cleartext HTTP/2 in the container? | depends on Kestrel's default `HttpProtocols` for a plain HTTP endpoint in 3.1 and on the client's `Http2UnencryptedSupport` switch — neither is configured here | attempt a call against the compose stack | platform architect |
| **Q4** | Is the 300-second sliding expiry **intentional** as the workflow lifetime bound? | the value is a bare config number with no comment, no doc and no test | product/architecture decision | platform architect |
| **Q5** | Is a single replica a deliberate constraint? | §5.5's race is unreachable at one replica and reachable at two; nothing records the dependency | deployment owner | platform architect |
| **Q6** | Are the committed Vault/JWT/Seq values demo placeholders or live? | cannot be determined from source; confirms baseline **G13** | security owner | security owner |
| **Q7** | Why does `messages.json` declare `sign_in` under the identity block when `identity-service` never publishes it? | manifest entries are not validated against publishers | compare each block against the owning repository | platform architect |
| **Q8** | Is `Queries/GetOperation` intended to grow a handler, or should it be deleted? | it is bound by the route and then ignored; no handler exists | component owner | platform architect |
| **Q9** | Is the `operations` exchange reserved for a planned outbound contract? | declared durable, zero publishers, zero bindings | component owner | platform architect |

Items marked **`Unverifiable — Missing Source Evidence`** in the body: the effect of
`vault.enabled: true` with no reachable Vault (§3.41), and Kestrel's default protocol negotiation
(§3.44 / Q3).

### 8.4 Reconciliation with the batch-1 baselines

This model supersedes four baseline entries in
`docs/architecture-inventory/baselines/service-summaries.md`.

| Baseline entry | Baseline said | This model finds | Status |
| --- | --- | --- | --- |
| **G4 / Q4** (`:523`, `:635`, `:343`) | "the effective store is **unknown — requires runtime validation**"; Mongo *or* Redis | **Redis.** `IDistributedCache` at `Services/OperationsService.cs:26`,`:50`; key `operations:requests:{id}`; 300 s **sliding** expiry. `AddMongo()` (`Infrastructure/Extensions.cs:71`) registers a driver that **nothing injects** — no `AddMongoRepository`, no `IMongoRepository<>`, no document type | **RESOLVED from source.** Runtime validation is not required; the baseline's caution was warranted but the code settles it. The baseline's *consequence* — "any saga running longer than five minutes loses its status" — is confirmed and sharpened: the expiry is sliding, so an idle gap of 300 s (not a total duration of 300 s) is what evicts (§4.9) |
| **G5 / Q14** (`:524`, `:645`, `:357`) | "the generated types are **field-less**" | **Almost.** Command- and event-derived types are field-less. **Rejected-event types are not**: they inherit `Reason` and `Code` from `Types/RejectedEvent.cs:5-9`, and `GenericRejectedEventHandler.cs:41-42` reads both into the stored operation | **REFINED.** Q14 remains open as **Q1** above for full payloads, but the two fields the service actually consumes *are* statically knowable |
| **G11 / A6** (`:530`) | "no migration tooling of any kind exists in the workspace" | **Confirmed** for this component: no EF Core, no Mongo migration package, no migrate script, no version field, no startup schema step. Evolution is governed by Json.NET defaults plus the TTL (§5.3) | **CONFIRMED** |
| **G13** (`:532`) | committed JWT key / Vault token / Seq apiKey; "whether these are demo values cannot be determined" | **Confirmed and located**: `appsettings.json:36` (`issuerSigningKey`, identical to `identity-service`'s and to all four `ntrada*.yml`), `:164-193` (`vault.token`/`password` = `"secret"`), `:47-60` (Seq `apiKey`) | **CONFIRMED**, still open as **Q6** |
| **C3** (`:455`) and `:345` | "8 exchanges, **26 commands, 30 events, 31 rejected events**" | **8 exchanges, 24 commands, 29 events, 27 rejected events = 80 subscriptions** — enumerated block by block from `messages.json:1-152` in §3.8 | **CORRECTED.** The baseline over-counts by 11. C3's *finding* — hand-maintained, no generation step, no validation step — stands unchanged |

Two further baseline observations are extended rather than corrected: **A5** (single-project,
domain-less architecture) is confirmed in §1.2, and **D1** (the `depends_on` fan-out) is grounded in
§3.44 against `compose/services.yml:51-67` — eight dependencies, the only such declaration in the
platform.

---

*End of `operations-service` component-internals model.*
