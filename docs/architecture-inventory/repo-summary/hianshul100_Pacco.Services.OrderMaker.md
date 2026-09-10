# Repository summary — `hianshul100_Pacco.Services.OrderMaker`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.OrderMaker` (also known as: Pacco.Services.OrderMaker, the OrderMaker service, "the uber AI order maker service" in its own root endpoint text)
**Deployable:** `Pacco.Services.OrderMaker` (also known as: `ordermaker-service` — its Consul service name, container name and compose service name — and `devmentors/pacco.services.ordermaker`, its published image). Repository: `hianshul100_Pacco.Services.OrderMaker`, path: `src/Pacco.Services.OrderMaker`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

A saga orchestrator. It takes a single "make me an order" request and drives the whole multi-service sequence to completion: create the order, attach the parcels, pick a vehicle, choose a reservation date, reserve the resource, and finish when the order is approved. It is the only orchestrated flow in an otherwise choreographed platform, and the only place where a business process is written down as a single readable unit.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1` (`WebHost.CreateDefaultBuilder`), Convey-based, with **`Chronicle_ 3.2.1`** supplying saga coordination. It is a single-project service — no Api/Application/Core/Infrastructure split, unlike the seven layered services.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.OrderMaker/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.OrderMaker`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll` |
| Message subscriptions | `src/Pacco.Services.OrderMaker/Extensions.cs` → `UseApp()` |
| Saga definition | `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` |
| Request collection | `Pacco.Services.OrderMaker.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Behaviour |
|---|---|---|
| GET | `` (root) | writes `Welcome to Pacco uber AI order maker Service!` |
| POST | `orders` | dispatches `MakeOrder` |

Note the composition differs from every other service: `Program.cs` calls `.UseApp()` rather than `UseInfrastructure()`, and it does **not** call `.UseVault()`.

## 4. Important modules / packages

One project: `src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj`.

- `Sagas/AIOrderMakingSaga.cs` — the process definition.
- `Sagas/AIMakingOrderData.cs` — the saga's state.
- `Handlers/AIOrderMakingHandler.cs` — a single class implementing `ICommandHandler<MakeOrder>` and `IEventHandler<>` for five events, each forwarding to `ISagaCoordinator.ProcessAsync`.
- `Commands/MakeOrder.cs` — `{ Guid OrderId, Guid CustomerId, Guid ParcelId }`; an empty `OrderId` is replaced with a new one in the constructor.
- `Commands/External/` — locally declared copies of other services' commands: `CreateOrder`, `AddParcelToOrder`, `AssignVehicleToOrder`, `ApproveOrder`, `CancelOrder` (all `[Message("orders")]`), `ReserveResource` (`[Message("availability")]`).
- `Events/External/` — locally declared copies of other services' events: `OrderCreated`, `OrderApproved`, `ParcelAddedToOrder`, `VehicleAssignedToOrder` (all `[Message("orders")]`), `ResourceReserved` (`[Message("availability")]`).
- `Events/MakeOrderCompleted.cs`, `Events/Rejected/MakeOrderRejected.cs`.
- `Services/Clients/{AvailabilityServiceClient,VehiclesServiceClient}.cs`, `Services/ResourceReservationsService.cs`.
- `DTO/{VehicleDto,ResourceDto,ReservationDto}.cs`, `CorrelationContext.cs`, `ExceptionToResponseMapper.cs`, `Extensions.cs`.

Packages: `Chronicle_ 3.2.1` plus Convey `0.4.*` for CQRS, message brokers, RabbitMQ, Redis, Consul, Fabio, HTTP, WebApi, Swagger, Security, Metrics. **No Mongo package, no Vault package, no Jaeger package** — the only service in the platform missing all three.

Convey composition from `Extensions.cs` → `AddInfrastructure()`: `.AddErrorHandler<ExceptionToResponseMapper>().AddHttpClient().AddConsul().AddFabio().AddCommandHandlers().AddEventHandlers().AddInMemoryCommandDispatcher().AddInMemoryEventDispatcher().AddRedis().AddMetrics().AddRabbitMq().AddWebApiSwaggerDocs().AddSecurity()`, then `services.AddChronicle()` and the three client registrations.

## 5. External integrations

RabbitMQ (exchange `ordermaker`), Redis (instance prefix `ordermaker:`), Consul (service `ordermaker-service`, port 5015, ping endpoint `ping`), Fabio, Prometheus, Seq.

**Absent, unlike every other service:** MongoDB, Vault (there is no `vault` block in `appsettings.json` at all), and Jaeger (no `jaeger` name is configured).

`httpClient.services` declares `availability: availability-service` and `vehicles: vehicles-service`.

## 6. Data stores and state handling

**There is no database.** No MongoDB, no collection, no repository, no ORM, no migration tool, no table.

The only state is the saga's own: `AIMakingOrderData { Guid OrderId, Guid CustomerId, Guid VehicleId; DateTime ReservationDate; int ReservationPriority; List<Guid> ParcelIds; List<Guid> AddedParcelIds; bool AllPackagesAddedToOrder => AddedParcelIds.Any() && AddedParcelIds.All(ParcelIds.Contains) }`.

`Extensions.cs` calls `services.AddChronicle()` with no persistence provider configured, so **saga state is held in the process**. Two consequences follow directly and neither is documented anywhere:

- A restart loses every in-flight saga. The orders involved are left half-built — created, perhaps with parcels attached, never approved, and with no compensation triggered.
- More than one instance cannot be run. A second instance would receive some of the events for a saga it has no state for.

Redis is registered in the composition but no cache call appears in this service's code, and it is not wired to Chronicle as a saga store. What it holds is **unknown — requires runtime capture**.

**Cross-domain coupling.** The saga holds identifiers owned by four other services — order, customer, vehicle, parcels — and drives them purely through messages. There is no foreign key and no local copy of any of those aggregates. One coupling deserves attention: `ResourceReservationsService.GetBestAsync(vehicleId)` passes the **vehicle** identifier to `availability-service` as a **resource** identifier, so the platform treats vehicles and availability resources as sharing one identifier space. Nothing in either service states that contract.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with `Chronicle_` on top for saga coordination. **This service has no outbox** — `AddMessageOutbox` is not called and `appsettings.json` has no `outbox` block, so every publish here is a direct, unguaranteed send.

**Broker settings:** exchange `ordermaker`, `conventionsCasing: snakeCase`, queue name template `ordermaker-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`.

**Published:**

| Message | Exchange | Payload key fields |
|---|---|---|
| `CreateOrder` | `orders` | `OrderId`, `CustomerId` |
| `AddParcelToOrder` | `orders` | `OrderId`, `ParcelId`, `CustomerId` |
| `AssignVehicleToOrder` | `orders` | `OrderId`, `VehicleId`, `DeliveryDate` |
| `CancelOrder` | `orders` | `OrderId`, `Reason` (compensation only; the literal reason sent is `"Because I'm saga"`) |
| `ReserveResource` | `availability` | `ResourceId`, `CustomerId`, `DateTime`, `Priority` |
| `MakeOrderCompleted` (`make_order_completed`) | `ordermaker` | `OrderId` |

Every publish carries `messageContext: _accessor.CorrelationContext` and a header `["Saga"] = SagaStates.<state>.ToString()` — `Pending` while the process runs, `Completed` at the end, `Rejected` on compensation. `availability-service` and `orders-service` both forward that `Saga` header, which is what keeps the correlation intact across hops.

`Events/Rejected/MakeOrderRejected.cs` declares `{ Guid OrderId, string Reason, string Code }` and matches the catalogue's `make_order_rejected`, but **no code publishes it**. The rejection path is declared and never used.

**Consumed** (from `UseApp()`):

| Kind | Message | Origin exchange |
|---|---|---|
| Event | `OrderCreated` | `orders` |
| Event | `OrderApproved` | `orders` |
| Event | `ParcelAddedToOrder` | `orders` |
| Event | `VehicleAssignedToOrder` | `orders` |
| Event | `ResourceReserved` | `availability` |

There are **no command subscriptions** — `MakeOrder` arrives only over HTTP.

**The saga flow**, from `AIOrderMakingSaga.cs`. `ResolveId` maps every message to its `OrderId`, so the order identifier is the saga key.

1. `MakeOrder` (HTTP) — record the order, customer and parcel; publish `CreateOrder`.
2. `OrderCreated` — publish one `AddParcelToOrder` per parcel identifier.
3. `ParcelAddedToOrder` — record the parcel; once `AllPackagesAddedToOrder` is true, call `IVehiclesServiceClient.GetBestAsync()` over HTTP, then `IResourceReservationsService.GetBestAsync(vehicleId)` over HTTP, then publish `AssignVehicleToOrder`.
4. `VehicleAssignedToOrder` — publish `ReserveResource`.
5. `OrderApproved` — publish `MakeOrderCompleted` and complete the saga.

Compensation is defined for all five steps but four of them return `Task.CompletedTask`; only `CompensateAsync(ParcelAddedToOrder)` does anything, publishing `CancelOrder`.

**Dead subscription.** `ResourceReserved` is subscribed and forwarded to the saga coordinator by `AIOrderMakingHandler`, but `AIOrderMakingSaga` implements no `ISagaAction<ResourceReserved>`. Step 4 publishes `ReserveResource` and the saga then waits for `OrderApproved` — which, on the evidence in this repository, nothing publishes in response to a reservation. See the open questions.

## 8. APIs exposed and consumed

**Exposed:** two HTTP routes (section 3), Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5015`; port 80 in the container.

**Not exposed through the gateway.** Neither `ntrada.yml` nor `ntrada-async.yml` has an `ordermaker` module. The saga entry point is unreachable from outside the platform — it can only be called from inside the Docker network.

**Consumed** — two synchronous HTTP clients, both through Fabio:

| Client | Call |
|---|---|
| `VehiclesServiceClient.GetBestAsync()` | `GET {vehicles-service}/vehicles`, then takes `Items.FirstOrDefault()`; the source comments this `// typical AI in a startup` |
| `AvailabilityServiceClient.GetResourceReservationsAsync(resourceId)` | `GET {availability-service}/resources/{resourceId}` |

`ResourceReservationsService.GetBestAsync` then picks a date: with no existing reservations it proposes tomorrow at priority 0; otherwise it takes the latest reservation and proposes the day after it, at priority plus one. Both clients throw `InvalidOperationException` on a miss, and — since the saga defines no compensation for those steps — a throw here leaves the saga stuck with an order already created and parcels already attached.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.OrderMaker`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `ordermaker-service` on host port 5015 in `hianshul100_Pacco/compose/services.yml` and `services-local.yml`.
- **Absent from `hianshul100_Pacco/services.yml` and `prod-services.yml`**, the two process lists used for local and release runs. Anyone starting the platform that way runs it without the orchestrator.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.ordermaker`.

## 10. Security and auth clues

- `Convey.Security` is composed and `certs/localhost.cer` is present, but **there is no `vault` block in `appsettings.json`** — the only service without one.
- **The `POST /orders` route performs no authorisation.** `Program.cs` registers it as a plain dispatcher endpoint, and `AIOrderMakingSaga` constructs its correlation context with an empty `UserContext` rather than reading the caller's identity. The `CustomerId` in the resulting order comes from the request body, not from a token. Anything that can reach port 5015 can create an order on behalf of any customer.
- The service is not routed through the gateway, which limits the exposure to the internal network — but that is containment by omission, not by design.
- `logger.excludeProperties` keeps secret-like values out of logs.

## 11. Observability, logging and tracing clues

Prometheus metrics via `Convey.Metrics.AppMetrics`; structured logs to console, file and Seq. The saga logs each step at information level, including the chosen vehicle and its price, which makes the flow followable in Seq.

**Tracing gap.** This is the only service with no Jaeger integration: `Convey.Tracing.Jaeger` is not referenced, `AddJaeger()` is not called, no `jaeger` service name is configured, and `AddRabbitMq()` is called without the Jaeger plugin that every other service passes. A distributed trace therefore breaks at the orchestrator — precisely the component whose job is to span five services. The `span_context` header is still configured on the broker, so the identifier flows through even though nothing here records a span.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` is the single most decision-dense file in the platform's business logic: it is the only written form of the end-to-end ordering process. `src/Pacco.Services.OrderMaker/Extensions.cs` records the technology choices, including the three that are missing relative to every other service. `src/Pacco.Services.OrderMaker/Services/ResourceReservationsService.cs` holds the date-and-priority rule. `src/Pacco.Services.OrderMaker/Sagas/AIMakingOrderData.cs` defines what "finished" means through `AllPackagesAddedToOrder`.

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.OrderMaker/`, and its `Sagas/`, `Handlers/`, `Commands/`, `Events/`, `Services/`, `DTO/` subdirectories, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5015`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.ordermaker`; `Pacco.Services.OrderMaker.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the saga itself. The README is the same template as the other nine services and never mentions Chronicle, orchestration, the five-step flow, the two synchronous dependencies, or that this service behaves differently from every other one in the platform. The single most distinctive component in Pacco has the least distinctive documentation.

**Conflicts to surface:**

- **Correct doc.** Unusually, the README's `dotnet run` path — `/src/Pacco.Services.OrderMaker` — is right here, because this is a single-project service with no `.Api` suffix.
- **Platform conflict.** `hianshul100_Pacco/services.yml` and `prod-services.yml` omit `ordermaker`, while `compose/services.yml` and `compose/services-local.yml` include it. Two of the four documented ways to run Pacco silently exclude the orchestrator.
- **Reachability conflict.** The gateway has no `ordermaker` module in either profile, so `POST /orders` on this service cannot be called from outside. Either the gateway is missing a module or this service is not meant to be client-facing; nothing states which.
- **Composition conflict.** Every other service composes `.UseVault()`, Jaeger and an outbox. This one composes none of them, and no file explains the difference.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`, but this repository contains **no test project at all**. The platform's only orchestrated process has no automated test.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next. The saga's state handling (B1) is the most consequential item in this file.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | `AddChronicle()` without a persistence provider means in-memory saga state. | No store is registered, the service references no database package, and Redis is not wired to Chronicle. |
| A2 | Vehicles and availability resources share one identifier space. | `ResourceReservationsService.GetBestAsync` passes a vehicle identifier straight to `availability-service` as a resource identifier. |
| A3 | The service is an experiment or demonstration rather than a production path. | It has no gateway route, no persistence, no tracing, no secrets management and no tests, while its siblings have all five. |
| A4 | `messages.json` in the Operations repository is meant to be the complete catalogue of this service's messages. | Operations builds live subscriptions from it at start-up. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** Saga state is held in the process, with no persistence configured. | A restart abandons every in-flight order mid-flight with no compensation, and the service cannot be run with more than one instance. | OrderMaker service maintainer: configure a Chronicle persistence provider, or state in writing that the service is single-instance and non-durable. |
| B2 | **[ACTION NOW]** `POST /orders` accepts a `CustomerId` from the request body and performs no authorisation. | Anything on the internal network can create orders for any customer. | OrderMaker service maintainer, with the security-architecture owner. |
| B3 | **[ACTION NOW]** The service publishes without an outbox, unlike every other service. | A crash between handling a message and publishing the next one drops a step of the saga silently. | OrderMaker service maintainer. |
| B4 | **[ACTION NOW]** No Jaeger integration in the one component that spans five services. | Distributed traces break exactly where they are most needed. | OrderMaker service maintainer. |
| B5 | **[ACTION NOW]** `ResourceReserved` is subscribed and routed to the saga coordinator, but the saga defines no action for it. | A subscription exists that can never do anything; if the flow was meant to advance on reservation, it does not. | OrderMaker service maintainer. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** What publishes `order_approved`, and does anything approve an order once a resource is reserved? | The saga's final step waits for it. If nothing publishes it after a reservation, no saga ever completes. | OrderMaker service maintainer with the Orders service maintainer. |
| Q2 | **[ACTION NOW]** Should this service be reachable through the API gateway? | Today the flow it orchestrates cannot be started by any external client. | Gateway maintainer with the OrderMaker service maintainer. |
| Q3 | **[ACTION NOW]** Should `MakeOrder` accept more than one parcel? | The command carries a single `ParcelId` while the saga's state keeps a list, so the multi-parcel design is present but unreachable. | OrderMaker service maintainer. |
| Q4 | **[ACTION NOW]** Is `make_order_rejected` meant to be published? | The event class and the catalogue entry both exist, and no code publishes it, so a failed order-making attempt tells the caller nothing. | OrderMaker service maintainer. |
| Q5 | **[handled later by the domain-model stage]** Should compensation do more than cancel the order when a parcel step fails? | Four of the five compensation methods are empty, so most failure paths leave state behind in other services. | Domain-model stage. |
| Q6 | **[handled later by the deployment-architecture stage]** Should `ordermaker` be added to `services.yml` and `prod-services.yml`? | Two of the four ways to run the platform omit the orchestrator entirely. | Deployment-architecture stage. |
