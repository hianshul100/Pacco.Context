# Repository summary — `Pacco.Services.OrderMaker`

**Repository:** `Pacco.Services.OrderMaker` (workspace clone path: `hianshul100_Pacco.Services.OrderMaker`)
**Deployable:** `ordermaker-service` (also known as: OrderMaker Service, `Pacco.Services.OrderMaker`, "Pacco uber AI order maker Service", image `devmentors/pacco.services.ordermaker`). **Repository: `Pacco.Services.OrderMaker`, path: `src/Pacco.Services.OrderMaker`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.OrderMaker
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Orchestrates the **end-to-end order-making process** as a long-running saga. A single `POST /orders` call here fans out into a coordinated sequence across four other services: create the order, attach every parcel, choose the best vehicle, find the best reservation slot for it, assign the vehicle, reserve the resource, and finally report completion. It is the only **process manager / orchestration** component in a platform that is otherwise purely choreographed.

**Evidence:** `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`, `Program.cs`, `Extensions.cs`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP service plus RabbitMQ consumer, hosting a **Chronicle saga** (`Chronicle 3.2.1`). It is a **single-project service** — `src/Pacco.Services.OrderMaker` only, with no `.Api` / `.Application` / `.Core` / `.Infrastructure` split. It has no aggregate of its own; its state is saga state, held by Chronicle.

Despite the name, there is **no artificial intelligence, machine learning, or model inference anywhere in the repository**. "AI" in `AIOrderMakingSaga` and "uber AI order maker" in the welcome banner describe an aspiration, not an implementation: vehicle and slot selection are ordinary deterministic "pick the best from a list" code. This is recorded here as **Future/Intended State (Not Implemented)** so no downstream reader infers an ML capability that does not exist.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.OrderMaker/Program.cs` — `AddConvey().AddWebApi().AddInfrastructure()`, then **`UseApp()`** (not `UseInfrastructure()`) and `UseDispatcherEndpoints(...)` |
| `GET ""` | `Program.cs` — returns "Welcome to Pacco uber AI order maker Service!" |
| `POST orders` | `Program.cs` — dispatches the `MakeOrder` command, which **starts the saga** |
| Saga | `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` |
| RabbitMQ subscriptions | `src/Pacco.Services.OrderMaker/Extensions.cs` → `UseApp` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.OrderMaker.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.OrderMaker` | The entire service: `Program.cs`, `Extensions.cs`, `Sagas/AIOrderMakingSaga.cs`, `Commands/`, `Events/`, `Events/External/`, `Events/Rejected/`, `Services/` and `Services/Clients/`, `DTO/`, `Exceptions/`, `appsettings.json` |

**No test projects exist in this repository** — notable for the only component that coordinates a distributed transaction.

**Distinctive package: `Chronicle 3.2.1`** (registered via `services.AddChronicle()`), the saga/process-manager library. It appears in no other repository in the workspace.

**Internal services:** `IAvailabilityServiceClient`, `IVehiclesServiceClient`, `IResourceReservationsService` — the last is the "pick the best slot" logic.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| `availability-service` | outbound HTTP | `GET {availability-service}/resources/{resourceId}` → `ResourceDto` (`Services/Clients/AvailabilityServiceClient.cs`, `httpClient.services.availability`) |
| `vehicles-service` | outbound HTTP | `GET {vehicles-service}/vehicles` → `PagedResult<VehicleDto>`, then selects the "best" (`Services/Clients/VehiclesServiceClient.cs`, `httpClient.services.vehicles`) |
| RabbitMQ | in + out | exchange `ordermaker`; publishes commands onto **four other services' exchanges** |
| Redis | out | instance prefix `ordermaker:` |
| Consul | out | registers `ordermaker-service` on port **`5015`**, `address: localhost` |
| Fabio | out | `http://localhost:9999` — but see the deviation below |
| Jaeger / Seq / Prometheus | out | logs / metrics; **Jaeger is not registered in code** (see §11) |

**Deviations from every other service, all visible in `appsettings.json` and `Extensions.cs`:**
- **No MongoDB.** No `mongo` block, no `AddMongo()`, no repository. Saga state lives wherever Chronicle puts it — by default, in memory.
- **No Vault.** `Program.cs` does not call `.UseVault()`, and there is no `vault` configuration block. It is the only service without Vault integration.
- **No outbox.** No `outbox` block and no `AddMessageOutbox(...)`, so no inbox deduplication and no transactional publication.
- **No Jaeger.** `AddJaeger()`, `AddJaegerDecorators()`, `UseJaeger()`, and the RabbitMQ Jaeger plugin are all absent, though a `jaeger` configuration block exists.
- **`httpClient.type: ""`** rather than `"fabio"`, so its two outbound calls resolve differently from every other service's.
- **`consul.address: localhost`** rather than `docker.for.win.localhost`.
- Port **`5015`**, outside the contiguous `5000`–`5009` block used by everything else.

## 6. Data stores / state handling

- **No database.** There is no MongoDB configuration, no repository, no document class, no collection name.
- **Access mechanism / ORM: none.** **Migration tool: none.**
- **State:** the saga's `AIMakingOrderData` — which tracks the order id, the customer id, the parcel identifiers still to be added, the chosen vehicle, and the reservation date — is held by **Chronicle**. `Extensions.cs` calls `services.AddChronicle()` with no persistence provider configured, so state is kept **in process memory**. Consequences, stated plainly because they are load-bearing:
  - A restart loses every in-flight saga. Orders mid-process are stranded: already created, parcels possibly attached, no completion and no compensation.
  - Two instances cannot share saga state, so the service cannot be scaled horizontally as configured.
- **Redis is configured** (`redis.instance: "ordermaker:"`, `.AddRedis()`) but no cache or saga-persistence usage was found in the source. Whether Chronicle uses it is **Needs validation**.
- **Cross-domain coupling:** the saga holds identifiers owned by four other services (`OrderId`, `CustomerId`, parcel ids, `VehicleId`, reservation date) in its transient state. There is no foreign key and no persistent replica, but the saga's correctness depends on those four services' aggregates existing and remaining consistent.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `ordermaker`; `conventionsCasing: snakeCase`; queue template `ordermaker-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — external events** (`Extensions.cs` → `UseApp`), all published by other services:

| Event | Wire name | Origin |
|---|---|---|
| `OrderCreated` | `order_created` | `orders-service` |
| `ParcelAddedToOrder` | `parcel_added_to_order` | `orders-service` |
| `VehicleAssignedToOrder` | `vehicle_assigned_to_order` | `orders-service` |
| `OrderApproved` | `order_approved` | `orders-service` |
| `ResourceReserved` | `resource_reserved` | `availability-service` |

**Published — commands onto other services' exchanges:** `CreateOrder`, `AddParcelToOrder`, `AssignVehicleToOrder`, `CancelOrder` (all → `orders`), and `ReserveResource` (→ `availability`).

**Published — events on its own exchange:** `MakeOrderCompleted` (`make_order_completed`, key field `OrderId`) and the rejection `make_order_rejected`.

**The saga** (`Sagas/AIOrderMakingSaga.cs`):

```csharp
public class AIOrderMakingSaga : Saga<AIMakingOrderData>,
    ISagaStartAction<MakeOrder>, ISagaAction<OrderCreated>, ISagaAction<ParcelAddedToOrder>,
    ISagaAction<VehicleAssignedToOrder>, ISagaAction<OrderApproved>
```

- **Correlation:** `ResolveId` keys every message on `OrderId`, so all five message types join the same saga instance.
- **Step 1 — `MakeOrder`:** publishes `CreateOrder(OrderId, CustomerId)` with the message header `["Saga"] = SagaStates.Pending`.
- **Step 2 — `OrderCreated`:** publishes one `AddParcelToOrder(OrderId, parcelId, CustomerId)` per parcel.
- **Step 3 — `ParcelAddedToOrder`:** once `Data.AllPackagesAddedToOrder` is true, calls `_vehiclesServiceClient.GetBestAsync()` then `_resourceReservationsService.GetBestAsync(vehicleId)`, then publishes `AssignVehicleToOrder(OrderId, VehicleId, ReservationDate)`.
- **Step 4 — `VehicleAssignedToOrder`:** publishes `ReserveResource(VehicleId, CustomerId, ReservationDate, ReservationPriority)`.
- **Step 5 — `OrderApproved`:** publishes `MakeOrderCompleted(OrderId)` with `["Saga"] = SagaStates.Completed`, then calls `CompleteAsync()`.

**The `Saga` header** (`SagaStates.Pending` / `Completed` / `Rejected`) is the platform's saga-correlation mechanism. Every other service forwards it — `Infrastructure/Extensions.cs` in each service has a `GetHeadersToForward` that explicitly preserves `"Saga"` — so `operations-service` can relate the whole chain back to one process.

**Compensation — incomplete, and the most important finding in this repository:**

| Step | Compensation implemented |
|---|---|
| `ParcelAddedToOrder` | **Yes** — publishes `CancelOrder(OrderId, "Because I'm saga")` with `["Saga"] = SagaStates.Rejected` |
| `MakeOrder` | No — `Task.CompletedTask` |
| `OrderCreated` | No — `Task.CompletedTask` |
| `VehicleAssignedToOrder` | No — `Task.CompletedTask` |
| `OrderApproved` | No — `Task.CompletedTask` |

So a failure after the vehicle has been assigned or the resource reserved leaves the **vehicle assignment and the resource reservation in place with no release** — the scarce resource the whole product is built around stays locked. `availability-service` exposes `ReleaseResourceReservation` (`release_resource`), so the capability exists; the saga simply never invokes it. The one compensation that does exist carries the placeholder reason string `"Because I'm saga"`, which reaches the customer-visible `order_canceled` event as its `Reason` field.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5015`, container port `80`):

| Method | Path | Maps to |
|---|---|---|
| GET | `` (root) | "Welcome to Pacco uber AI order maker Service!" |
| POST | `orders` | `MakeOrder` command → starts `AIOrderMakingSaga` |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus |

**Not routed at the gateway.** There is **no `ordermaker` module in `ntrada.yml` or `ntrada-async.yml`**, so `POST /orders` at the gateway reaches `orders-service` directly and the saga is bypassed entirely. The service is reachable only by direct network access to port `5015`, or by a `MakeOrder` message published straight to the `ordermaker` exchange.

**Consumed:** `GET {availability-service}/resources/{resourceId}` and `GET {vehicles-service}/vehicles`.

**Called by:** nothing in the workspace.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll`.
- Composed as `ordermaker-service` in `Pacco/compose/services.yml`, and named in `operations-service`'s `depends_on` list.
- **Absent from `Pacco/services.yml` and `Pacco/prod-services.yml`.** Those two process-manager manifests list ten entries — api `5000`, availability `5001`, customers `5002`, deliveries `5003`, identity `5004`, operations `5005`, orders `5006`, parcels `5007`, pricing `5008`, vehicles `5009` — and **omit `ordermaker` entirely**. Anyone starting the platform without Docker never starts this service. Combined with its absence from the gateway, that means **the saga does not run in either documented start-up path**.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- **Scaling:** in-memory saga state means one instance only.

## 10. Security/auth clues

- **`.AddSecurity()`** is registered, and a `jwt` block exists in `appsettings.json` (`certs/localhost.cer`, `validIssuer: pacco`).
- **No Vault integration** — the only service without it. `Program.cs` omits `.UseVault()` and there is no `vault` configuration block, so it has no PKI identity, no `<service>.pacco.io` certificate, and no dynamic credentials.
- **No certificate authentication** in either direction: its two outbound calls to `availability-service` and `vehicles-service` carry no caller credential.
- **`POST /orders` performs no authorisation.** The `MakeOrder` command carries a `CustomerId` supplied by the caller. There is no gateway route binding `customerId: @user_id` (as `/orders` and `/parcels` have), and no check inside the service, so anyone who can reach port `5015` can create an order, attach parcels, assign a vehicle, and reserve a scarce resource **on behalf of any customer id they choose**.
- Log redaction via `logger.excludeProperties`.

## 11. Observability/logging/tracing

- **Tracing: absent in code.** `appsettings.json` contains a `jaeger` block (`serviceName: ordermaker`, UDP `localhost:6831`, `sampler: const`), but `Extensions.cs` never calls `AddJaeger()`, `AddJaegerDecorators()`, `UseJaeger()`, or `AddJaegerRabbitMqPlugin()`. Every other service does. **The one component that coordinates work across four services is the one component that emits no traces.** A distributed trace of an order-making run therefore has a hole exactly where the orchestration happens. This is a configuration/code mismatch, not merely a missing feature.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`.
- **Handler logging:** `.AddHandlersLogging()` is **not** called here, unlike in the eight domain services.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No saga-specific metrics — no counter for sagas started, completed, compensated, or abandoned, which is precisely what would be needed to notice stranded orders.
- **Correlation:** the `Saga` header is set by this service and forwarded by all the others.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` | The platform's only orchestration: the five-step order-making process, the `OrderId` correlation key, the `Saga` header protocol, and the decision to compensate only one of five steps |
| `src/Pacco.Services.OrderMaker/Extensions.cs` | `AddChronicle()` with no persistence provider (in-memory saga state); the omission of Mongo, Vault, outbox, Jaeger, and handler logging |
| `src/Pacco.Services.OrderMaker/Program.cs` | `UseApp()` instead of the platform-standard `UseInfrastructure()`; no `.UseVault()` |
| `src/Pacco.Services.OrderMaker/Services/ResourceReservationsService.cs` | The "best reservation slot" selection rule |
| `src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs` | The "best vehicle" selection rule — fetches the full paged vehicle list and picks from it |
| `src/Pacco.Services.OrderMaker/appsettings.json` | Port `5015`; `httpClient.type: ""`; no Vault, Mongo, or outbox blocks |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `metrics.enabled`, `jaeger.enabled` — the last having no effect, since no Jaeger code runs — `swagger.enabled`, `logger.*.enabled`). The selection rules for "best vehicle" and "best reservation" are hard-coded with no configuration and no flag.

## 13. Open questions / ambiguities

- **Is this service live?** It is absent from the gateway, absent from both process-manager manifests, and called by nothing in the workspace. It is present only in `compose/services.yml` and in `operations-service`'s `depends_on`. Whether it is an experiment, a demonstration, or intended production capability is **Unknown**.
- **Saga state is in memory.** Whether Chronicle is meant to be given a persistence provider is **Needs validation**.
- **Four of five compensations are empty**, so a mid-saga failure can strand a vehicle assignment and a resource reservation. Whether this is a known gap is **Unknown**.
- **No Vault**, unlike every other service — **Unknown** whether deliberate.
- **No Jaeger despite Jaeger configuration** — appears to be an omission. **Needs validation.**
- **"AI" is aspirational.** No model, no inference, no training data. Recorded as **Future/Intended State (Not Implemented)**.
- **The `"Because I'm saga"` cancellation reason** reaches customers through `order_canceled.Reason`. Almost certainly placeholder text. **Needs validation.**
- **No tests** for the only distributed-transaction coordinator in the platform.
- **Port `5015`** breaks the `5000`–`5009` convention with no stated reason. **Unknown.**

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (one C# project only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS. The only browser-facing surfaces are the plain-text welcome banner at `GET /` and the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- OrderMaker service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker. — **Confirmed.**
- Run **"in the `/src/Pacco.Services.OrderMaker` directory"** — **Confirmed, and uniquely so.** This is the **only one of the ten service repositories whose documented source path is correct**, because it is the only single-project service without an `.Api` suffix. The other nine all document a directory that does not exist.
- Port `5015`. — **Confirmed** against `appsettings.json` (`consul.port: 5015`).

**README claims not reflected in the clone — Stale doc:**
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **The saga itself.** The five-step orchestration, the `Saga` header protocol, the messages it publishes onto four other services' exchanges, and the fact that four of five compensations are empty — none of it is documented anywhere in the repository or the platform.
- **Chronicle** as a dependency and the in-memory saga state that follows from it.
- The two outbound HTTP clients and the "best vehicle" / "best reservation" selection rules.
- The absence of Vault, MongoDB, the outbox, Jaeger, and handler logging — every one a deviation from the platform norm, none explained.
- That the service is not routed through `api-gateway` and not present in either process-manager manifest, so neither documented start-up path exercises it.
- `scripts/` (`build.sh`, `test.sh`, `dockerize.sh`).

**Unknown (neither pass yielded proof):**
- Whether "AI" was ever intended to become a real model-backed selection, or was always shorthand for automated selection.
- Whether the service is part of the intended production topology.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | "AI" in `AIOrderMakingSaga` and in the welcome banner is aspirational naming, not a claim about implemented capability. | The vehicle and reservation selection code is ordinary deterministic list selection; there is no model, inference call, training data, or ML package anywhere in the repository. | Downstream design work would assume an intelligent selection capability that does not exist, and could build on it. | Ask the product owner what "best vehicle" and "best reservation" are meant to mean. |
| A2 | Chronicle keeps saga state in process memory, because `AddChronicle()` is called with no persistence provider. | No persistence provider is registered, no MongoDB is configured, and no saga-state collection or table name appears anywhere in the repository. | If state is in fact persisted somewhere, the durability and scaling conclusions here are too pessimistic; if it is not, every in-flight order is lost on restart. | Read Chronicle 3.2.1's default registration, or restart the service mid-saga and observe whether it resumes. |
| A3 | The service is not currently exercised by either documented way of starting the platform. | It has no module in any `ntrada*.yml`, and it is absent from both `Pacco/services.yml` and `Pacco/prod-services.yml`. | The saga could be running in an environment nobody realises, with its unbounded resource reservations and missing compensations already in effect. | Check any running environment for an `ordermaker-service` registration in Consul. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Four of the saga's five compensation handlers do nothing. A failure after the vehicle is assigned or the resource reserved leaves both in place, permanently holding the scarce resource the product exists to manage. | Any decision to run this service in an environment with real customers or real inventory. | Service owner / product owner | Implement the missing compensations — `availability-service` already accepts `release_resource`, so the capability exists — or disable the saga until they are written. | TBD |
| B2 | **[ACTION NOW]** Saga state is held in process memory, so a restart strands every in-flight order: created, possibly with parcels attached and a resource reserved, with no completion and no compensation. The service also cannot run more than one instance. | Production readiness; any availability or scaling commitment. | Service owner | Configure a persistence provider for Chronicle, or accept single-instance operation with a documented recovery procedure for stranded orders. | TBD |
| B3 | **[ACTION NOW]** `POST /orders` on port `5015` accepts a caller-supplied `CustomerId` with no authentication and no ownership check, so anyone who can reach the service can place orders and reserve resources as any customer. | Security sign-off; any deployment where port `5015` is reachable beyond trusted infrastructure. | Security owner | Route the endpoint through the gateway with `customerId` bound to the token's user id, as `/orders` and `/parcels` already are, or restrict the port to internal callers only. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is `ordermaker-service` intended to be part of the platform, and if so how is it meant to be reached? | It is absent from the gateway and from both process-manager manifests, so nothing in the documented set-up ever invokes it. Its status changes whether its other gaps matter at all. | Unknown — it appears only in `compose/services.yml` and in `operations-service`'s `depends_on`. | Platform owner |
| Q2 | **[ACTION NOW]** Why does this service alone skip Vault, and how are its secrets and its service identity meant to be provisioned? | Every other service draws settings, a PKI certificate, and dynamic database credentials from Vault; this one has no identity at all. | Unknown — likely an omission, since the service also skips Mongo and therefore the dynamic-credential lease. | Platform owner |
| Q3 | **[ACTION NOW]** Should this service emit traces? It ships a full Jaeger configuration block but registers no Jaeger code. | It is the only component that coordinates work across four services, so its absence leaves a hole in every distributed trace exactly where orchestration happens. | Yes — this looks like an omission rather than a decision. | Service owner |
| Q4 | **[handled later by architecture_evolution_generation]** What should the customer-visible cancellation reason be when the saga compensates? | The current text is the placeholder `"Because I'm saga"`, and it reaches customers through the `Reason` field of `order_canceled`. | Replace it with a real reason describing which step failed. | Product owner |
| Q5 | **[handled later by architecture_evolution_generation]** What do "best vehicle" and "best reservation" actually mean as business rules? | Both are hard-coded selection rules with no configuration, no documentation, and no tests, yet they decide how scarce capacity is allocated. | Read from the client and reservation service code, but no stated rule exists. | Product owner |
