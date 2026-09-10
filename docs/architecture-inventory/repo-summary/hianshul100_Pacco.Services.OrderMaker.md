# Repository Summary — `hianshul100_Pacco.Services.OrderMaker`

**Primary name:** `ordermaker-service` (aliases used in this file: `Pacco.Services.OrderMaker` — the .NET project and assembly name; `devmentors/pacco.services.ordermaker` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.OrderMaker`, path: `src/Pacco.Services.OrderMaker`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

A saga orchestrator. It takes a single `MakeOrder` request and drives the whole multi-service order-creation sequence on the caller's behalf — create the order, add every parcel, pick the best vehicle, pick the best resource reservation, assign the vehicle, reserve the resource, and report completion. It is the only service in the platform that coordinates other services rather than owning a domain.

It is also the only service that composes the platform's building blocks into a single business outcome, which makes it the best single file to read to understand how Pacco's services fit together.

## 2. Main runtime / service type

ASP.NET Core 3.1 host (`netcoreapp3.1`) on **Convey** `0.4.*` plus **Chronicle** `3.2.1`, a saga and process-manager library. It runs an HTTP API and a RabbitMQ consumer in one process.

It uses **none** of the platform's layering conventions: one project, no `.Api` / `.Application` / `.Core` / `.Infrastructure` split, and no domain layer.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.OrderMaker/Program.cs` |
| Dependency wiring | `src/Pacco.Services.OrderMaker/Extensions.cs` |
| The saga | `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll` |
| Local run script | `scripts/start.sh` |

`Program.cs` declares two routes and calls `.UseApp()` — this service's own pipeline method — rather than the `.UseInfrastructure()` every other service uses. It does **not** call `.UseVault()`.

## 4. Important modules / packages

One source project, enumerated from `Pacco.Services.OrderMaker.sln`: `src/Pacco.Services.OrderMaker`.

Types registered in `src/Pacco.Services.OrderMaker/Extensions.cs`:

- `IAvailabilityServiceClient` → `AvailabilityServiceClient`
- `IVehiclesServiceClient` → `VehiclesServiceClient`
- `IResourceReservationsService` → `ResourceReservationsService`
- `services.AddChronicle()` — registers the saga infrastructure.

Notable NuGet reference: **`Chronicle_` `3.2.1`** — used by no other repository in the workspace.

## 5. External integrations

- **`availability-service`** and **`vehicles-service`** over HTTP through Fabio. `httpClient.services` in `src/Pacco.Services.OrderMaker/appsettings.json` maps `availability` → `availability-service` and `vehicles` → `vehicles-service`.
- **RabbitMQ**, **Redis**, **Consul**, **Fabio**, **Prometheus**, **Seq** — through Convey extensions.
- **No MongoDB, no Jaeger, no Vault, no outbox** — see the data and observability sections.

## 6. Data stores & state

- **No MongoDB.** `AddInfrastructure()` in `src/Pacco.Services.OrderMaker/Extensions.cs` never calls `.AddMongo()`, and `appsettings.json` has no `mongo` block. There are no collections, no documents, no ORM, no query mechanism and no migration tool.
- **Redis** — `.AddRedis()` is called and `redis.connectionString: localhost` is configured. Chronicle needs somewhere to keep saga state between steps, and Redis is the only store present, so this is the presumed saga store. The exact key layout is **Unknown**; see assumptions.
- **No outbox and no inbox.** Every other messaging service in the platform wraps its handlers in `OutboxCommandHandlerDecorator` and `OutboxEventHandlerDecorator`. This one does not. Its published commands are therefore **not** transactionally tied to its state changes.
- **Cross-domain coupling:** the saga's state object `AIMakingOrderData` holds identifiers from four other domains at once — order, parcel, vehicle and resource. It is the single place in the platform where those four are correlated. It is transient rather than persisted in a database of record, and there are no foreign keys.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `ordermaker`, queue template `ordermaker-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Registered with a plain `.AddRabbitMq()` — **no Jaeger plugin**, unlike every other service.

**Consumed** (`UseApp()` in `src/Pacco.Services.OrderMaker/Extensions.cs`, lines 58–62) — five events, all from other services:

| Event | Wire name | Published by |
|---|---|---|
| `OrderCreated` | `order_created` | `orders-service` |
| `ParcelAddedToOrder` | `parcel_added_to_order` | `orders-service` |
| `VehicleAssignedToOrder` | `vehicle_assigned_to_order` | `orders-service` |
| `OrderApproved` | `order_approved` | `orders-service` |
| `ResourceReserved` | `resource_reserved` | `availability-service` |

**Published**, per the `ordermaker-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: event `make_order_completed`, rejection event `make_order_rejected`. In addition the saga publishes commands onto *other* services' exchanges: `create_order`, `add_parcel_to_order`, `assign_vehicle_to_order`, `reserve_resource` and, on compensation, `cancel_order`.

**The saga, step by step** (`src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`, `AIOrderMakingSaga : Saga<AIMakingOrderData>`, `ISagaStartAction<MakeOrder>`, `ISagaAction<OrderCreated>`, `ISagaAction<ParcelAddedToOrder>`, `ISagaAction<VehicleAssignedToOrder>`, `ISagaAction<OrderApproved>`):

1. `MakeOrder` starts the saga → publishes `CreateOrder(OrderId, CustomerId)`.
2. `OrderCreated` → publishes one `AddParcelToOrder` per parcel.
3. `ParcelAddedToOrder`, once every parcel has been added → calls `IVehiclesServiceClient.GetBestAsync()`, then `IResourceReservationsService.GetBestAsync(vehicleId)`, then publishes `AssignVehicleToOrder(OrderId, VehicleId, ReservationDate)`.
4. `VehicleAssignedToOrder` → publishes `ReserveResource(VehicleId, CustomerId, ReservationDate, ReservationPriority)`.
5. `OrderApproved` → publishes `MakeOrderCompleted(OrderId)` and calls `CompleteAsync()`.

`ResolveId` keys the saga on `OrderId`, so the order identifier is the saga correlation key.

**Payload key fields** — these are the only message field lists in the workspace that could be read directly from code:

| Message | Fields observed in the saga |
|---|---|
| `create_order` | `OrderId`, `CustomerId` |
| `assign_vehicle_to_order` | `OrderId`, `VehicleId`, `ReservationDate` |
| `reserve_resource` | `VehicleId`, `CustomerId`, `ReservationDate`, `ReservationPriority` |
| `make_order_completed` | `OrderId` |
| `cancel_order` | order identifier, reason |

**Saga header.** `const string SagaHeader = "Saga"`. Every message the saga publishes carries that header set to a value from `SagaStates` — `Pending`, `Completed` or `Rejected`. The other services forward this header onward: `GetHeadersToForward` in each service's `Extensions.cs` explicitly forwards `"Saga"`. This is the platform's saga-context propagation mechanism.

**Compensation.** `CompensateAsync(ParcelAddedToOrder)` publishes `CancelOrder(message.OrderId, "Because I'm saga")` — the literal reason string in the code. Every other `CompensateAsync` overload is a no-op, so failures at steps 1, 4 and 5 trigger no compensating action.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.OrderMaker/Program.cs`:

| Method | Path | Behaviour |
|---|---|---|
| GET | `""` | returns the literal string `Welcome to Pacco uber AI order maker Service!` |
| POST | `orders` | dispatches `MakeOrder`, starting the saga |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Not exposed through `api-gateway`.** `ntrada.yml` has no `ordermaker` module. This service has no public route and can only be reached directly on its own port or over the broker.

**Consumes over HTTP:** `vehicles-service` (`GetBestAsync`) and `availability-service` (via `IResourceReservationsService`), both through Fabio with 3 retries and request masking.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5015`. Every other service sits in the contiguous block `5001`–`5009`; this one is set apart, which suggests it was added later.
- Consul at `http://localhost:8500`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.ordermaker`, present in `hianshul100_Pacco/compose/services.yml`.
- **Scaling caveat:** Chronicle sagas are stateful. Running more than one instance is only safe if saga state is genuinely shared, which depends on the Redis assumption below.

## 10. Security & auth clues

- **No Vault.** `AddInfrastructure()` never calls `.UseVault()` and `appsettings.json` has no `vault` block — the only messaging service without one.
- `.AddSecurity()` is called, but there is no `security` access-control list, no certificate authentication, and no `jwt` validation configured for inbound requests.
- The service is not exposed through `api-gateway`, so no JWT ever reaches it from a public caller. It publishes commands on other services' exchanges without presenting any credential; the broker's `guest`/`guest` login is the only control.
- **Checked-in credentials:** RabbitMQ `guest`/`guest` and Seq `apiKey: secret` in `src/Pacco.Services.OrderMaker/appsettings.json`.

## 11. Observability / logging / tracing

- **No distributed tracing.** `.AddJaeger()` is not called, there is no `jaeger` block in `appsettings.json`, and `.AddRabbitMq()` is registered without `AddJaegerRabbitMqPlugin()`. This is a real gap: the one service that spans five others end to end is the one service invisible to Jaeger. A trace of an order-making flow will break at this hop.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]` and the standard `excludeProperties` redaction list.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.
- **Correlation:** the `Saga` header carries saga state across every hop, which is a partial substitute for tracing but is not collected anywhere.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` — the single most decision-dense file in the workspace. It encodes the order-creation process, the choice of orchestration over choreography for this flow, the saga correlation key, the `Saga` header protocol, and the compensation policy.
- `src/Pacco.Services.OrderMaker/Extensions.cs` — the decision to adopt Chronicle, and the decisions *not* to use Mongo, Vault, Jaeger or the outbox.
- `src/Pacco.Services.OrderMaker/appsettings.json` — the configuration contract, notable for what it omits.
- `Pacco.Services.OrderMaker.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- **Where is saga state stored?** Chronicle needs a persistence mechanism. `.AddRedis()` is the only store registered. Whether Chronicle is configured to use it, or is defaulting to in-memory state, is **Unknown**. **Needs validation** — if it is in-memory, every restart loses in-flight orders. This is the single most important unknown in the repository.
- **Compensation is largely absent.** Only `ParcelAddedToOrder` compensates. A failure after `AssignVehicleToOrder` leaves a vehicle assigned and a resource possibly reserved with no rollback. Whether that is deliberate is **Unknown**.
- **No outbox** means a crash between changing saga state and publishing a command loses the command. Whether that risk was accepted is **Unknown**.
- The name `AIOrderMakingSaga` and the README's "uber AI order maker" phrasing suggest machine-learning involvement. **There is none.** `GetBestAsync` on the vehicles and reservations clients is an ordinary HTTP call; no model, inference library or scoring code exists in the repository. What "best" means is decided by the downstream services and is **Unknown**.
- The service is absent from `ntrada.yml` entirely, so how a real client triggers `MakeOrder` is **Unknown**. **Needs validation.**
- `messages.json` lists no commands for `ordermaker-service`, yet `MakeOrder` is dispatched over HTTP and the saga publishes commands on other exchanges. The catalogue therefore under-reports this service's message activity.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.OrderMaker/`, `src/Pacco.Services.OrderMaker/Sagas/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.OrderMaker` directory". | That directory does exist — this service has no `.Api` suffix. | **Confirmed.** This is the only service README whose `dotnet run` path is correct, and it is correct by accident of naming. |
| "By default, the service will be available under `http://localhost:5015`." | `appsettings.json` sets the Consul port to `5015`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.ordermaker`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| "Pacco.Services.OrderMaker is the microservice being part of Pacco solution" — the shared template wording. | The service is not a domain microservice at all. It is a cross-service orchestrator that publishes commands into five other services' exchanges. | **Docs conflict.** The template description actively misleads about what this service does. |
| The README's phrase "uber AI order maker Service", echoed by the `GET ""` route. | No artificial-intelligence or machine-learning code exists. Vehicle and resource selection are plain HTTP calls to `GetBestAsync` endpoints on other services. | **Conflict — docs-only claim.** Treat the "AI" framing as **Future/Intended State (Not Implemented)**. |
| The README says nothing about the saga. | `AIOrderMakingSaga` is the entire point of the service. | **Docs gap — severe.** |

**On disk but not documented anywhere:** the Chronicle saga, the `Saga` header protocol, the compensation behaviour, the HTTP dependencies on `vehicles-service` and `availability-service`, and the absence of Vault, Jaeger, Mongo and the outbox.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Chronicle keeps saga state in Redis. | `.AddRedis()` is the only persistence registered in `Extensions.cs`, and a saga must keep state between steps. | If Chronicle is actually using in-memory state, every restart or second instance loses in-flight orders silently. | Read the Chronicle `3.2.1` package defaults, or inspect Redis while a saga is running. |
| A2 | "AI" in the service and saga names is branding, not a description of an implemented capability. | No model, inference library, training data or scoring code exists in the repository; selection is delegated to `GetBestAsync` HTTP endpoints on other services. | The platform would be credited with a capability it does not have. | Read `VehiclesServiceClient` and `ResourceReservationsService`, and the downstream `GetBest` handlers. |
| A3 | This service is reached directly or over the broker, not through `api-gateway`. | `ntrada.yml` has no `ordermaker` module and no route to port `5015`. | The platform's public API surface would be understated. | Check `ntrada.docker.yml` and any deployment-time gateway configuration. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** It cannot be determined from the repository whether in-flight sagas survive a restart of `ordermaker-service`. | Any statement about the platform's resilience or about running more than one instance of this service. | Service owner | Confirm Chronicle's configured persistence; if it is in-memory, decide whether that is acceptable. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should the saga compensate for failures after the vehicle-assignment step? | Today only `ParcelAddedToOrder` compensates. A later failure can leave a vehicle assigned and a resource reserved with nothing to undo them. | Add compensating actions for `VehicleAssignedToOrder` and `OrderApproved`. | Service owner |
| Q2 | **[ACTION NOW]** Should `ordermaker-service` be brought under Jaeger tracing? | It is the only service that spans five others, and it is the only one invisible to the tracing system. Every end-to-end trace breaks at this hop. | Yes — add `.AddJaeger()` and the RabbitMQ Jaeger plugin to match the other services. | Platform architect |
| Q3 | **[handled later by the platform inventory review]** Should this service adopt the outbox pattern used by every other messaging service? | Without it, a crash between a saga state change and a command publish loses the command, stalling the order with no error. | Yes, for consistency with the rest of the platform. | Service owner |
| Q4 | **[handled later by the platform inventory review]** Why does this service run without Vault when the other eight domain services use it? | It is the only messaging service with no dynamic credential management. | It has no database, so there are no dynamic database credentials to issue — but it still has broker credentials. | Security owner |
