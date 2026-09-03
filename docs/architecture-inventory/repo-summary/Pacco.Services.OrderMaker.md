---
title: "Repository Summary — Pacco.Services.OrderMaker"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.OrderMaker"
status: "evidence-based inventory"
---

# Pacco.Services.OrderMaker

**Primary name:** `Pacco.Services.OrderMaker` (aliases used in this file: `ordermaker-service` — the value of `app.service` and the key in the platform message catalogue; `ordermaker` — the RabbitMQ exchange).
Repository: `Pacco.Services.OrderMaker`, path: `hianshul100_Pacco.Services.OrderMaker/`
Deployable project: `Pacco.Services.OrderMaker`, path: `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj`

---

## 1. Primary purpose

Places a complete order on the customer's behalf in one request. It runs a **saga** that creates the order, adds parcels, picks the best vehicle, reserves an availability slot for it, waits for approval, and compensates by cancelling the order if any step fails.

Evidence: `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`, `src/Pacco.Services.OrderMaker/Sagas/AIMakingOrderData.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service with a Convey dispatcher endpoint, a RabbitMQ subscriber, and a saga host. Configured port `5015`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` — `.UseApp()`, `GET ""` returning `Welcome to Pacco uber AI order maker Service!`, `POST orders` dispatching `MakeOrder` | `src/Pacco.Services.OrderMaker/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

**One flat project.** This repository does not use the `.Api` / `.Application` / `.Core` / `.Infrastructure` split that the other services use.

Folders: `Commands/MakeOrder.cs`; `Commands/External/` (`AddParcelToOrder.cs`, `ApproveOrder.cs`, `AssignVehicleToOrder.cs`, `CancelOrder.cs`, `CreateOrder.cs`, `ReserveResource.cs`); `Events/MakeOrderCompleted.cs`; `Events/External/` (`OrderApproved.cs`, `OrderCreated.cs`, `ParcelAddedToOrder.cs`, `ResourceReserved.cs`, `VehicleAssignedToOrder.cs`); `Events/Rejected/MakeOrderRejected.cs`; `Handlers/AIOrderMakingHandler.cs`; `Sagas/AIMakingOrderData.cs`, `Sagas/AIOrderMakingSaga.cs`; `Services/IResourceReservationsService.cs`, `ResourceReservationsService.cs`; `Services/Clients/AvailabilityServiceClient.cs`, `IAvailabilityServiceClient.cs`, `VehiclesServiceClient.cs`, `IVehiclesServiceClient.cs`; `DTO/ReservationDto.cs`, `ResourceDto.cs`, `VehicleDto.cs`; `CorrelationContext.cs`; `ExceptionToResponseMapper.cs`; `Extensions.cs`.

Key package: **`Chronicle_ 3.2.1`** — the saga library. This is the only repository in the platform that uses it. Also `Convey.Persistence.Redis`; there is **no MongoDB package and no Vault package**.

## 5. External integrations

RabbitMQ, Redis, Consul, and two services over HTTP: `Pacco.Services.Availability` and `Pacco.Services.Vehicles` (`httpClient.services: {availability: availability-service, vehicles: vehicles-service}`).

## 6. Data stores / state

- **Store:** none. There is **no `mongo` block in `appsettings.json`** and no MongoDB package in the project file. This is the only stateful-looking service in the platform with no database.
- **Access mechanism:** not applicable — no ORM, no repositories, no document classes.
- **Collections / tables:** none. There are consequently no `inbox` or `outbox` collections either.
- **Migration tool:** none.
- **Cross-domain coupling:** the saga holds order, customer, parcel, vehicle and reservation identifiers together in `Sagas/AIMakingOrderData.cs` — `OrderId`, `CustomerId`, `ParcelIds`, `AddedParcelIds`, `VehicleId`, `ReservationDate`, `ReservationPriority`, and the derived `AllPackagesAddedToOrder`. This in-flight state spans four domains at once and is the tightest cross-domain coupling anywhere in the platform.
- **Saga state:** held by Chronicle. With no database configured, saga state is not durably persisted from anything visible in this repository, so an in-flight order would be lost if the process restarts. **Needs validation.**
- **Cache:** Redis, instance prefix `ordermaker:`.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `ordermaker`.

**The saga, step by step** (`Sagas/AIOrderMakingSaga.cs`, `Saga<AIMakingOrderData>`):

| Trigger | Handler action | Message published |
|---|---|---|
| `MakeOrder` (saga start) | stores `OrderId`, `CustomerId`, first parcel identifier | `CreateOrder(OrderId, CustomerId)` |
| `OrderCreated` | for each stored parcel identifier | `AddParcelToOrder(OrderId, ParcelId, CustomerId)` |
| `ParcelAddedToOrder` | records the parcel; once `AllPackagesAddedToOrder` is true, calls the vehicles service for the best vehicle and the availability service for the best reservation slot | `AssignVehicleToOrder(OrderId, VehicleId, ReservationDate)` |
| `VehicleAssignedToOrder` | — | `ReserveResource(VehicleId, CustomerId, ReservationDate, ReservationPriority)` |
| `OrderApproved` | completes the saga via `CompleteAsync()` | `MakeOrderCompleted(OrderId)` |

Correlation: `ResolveId` maps every one of those messages to the saga instance by `OrderId`.

Compensation: `CompensateAsync(ParcelAddedToOrder, …)` publishes `CancelOrder(OrderId, "Because I'm saga")`. The compensations for `MakeOrder`, `OrderCreated`, `VehicleAssignedToOrder` and `OrderApproved` are empty (`Task.CompletedTask`), so **only the parcel step is actually compensated**.

Every publish carries a header named `Saga` whose value is one of `SagaStates.Pending`, `SagaStates.Rejected` or `SagaStates.Completed`, alongside the correlation context.

**Events published** (per the platform catalogue): `make_order_completed` (payload: order identifier) and the rejected event `make_order_rejected`. This service publishes **no commands under its own exchange**; the commands it emits are the other services' commands, routed to their exchanges.

**External events consumed:** `order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved` from `orders`, and `resource_reserved` from `availability`. `Events/External/ResourceReserved.cs` exists in the repository but is **not one of the saga's declared `ISagaAction` steps** — the saga moves from `VehicleAssignedToOrder` straight to publishing the reservation and then waits for `OrderApproved`. **Needs validation.**

Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed:

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `` (root) | returns `Welcome to Pacco uber AI order maker Service!` |
| `POST` | `orders` | `MakeOrder` |

Consumed over HTTP: `Services/Clients/VehiclesServiceClient.cs` (`GetBestAsync()` → `VehicleDto`) and `Services/Clients/AvailabilityServiceClient.cs` via `Services/ResourceReservationsService.cs` (`GetBestAsync(vehicleId)` → `ReservationDto`).

**This service is not exposed through `Pacco.APIGateway`** — there is no `ordermaker` module in `ntrada.yml` or `ntrada-async.yml`.

## 9. Deployment / runtime clues

- Configured Consul port `5015`, and Consul `address: localhost` — every other service uses `docker.for.win.localhost`. **Anomaly.**
- `httpClient.type: ""` — empty, where every other calling service sets `fabio`. Outbound calls therefore bypass the load balancer. **Anomaly.**
- **Absent from `hianshul100_Pacco/services.yml`, from `hianshul100_Pacco/prod-services.yml` and from `hianshul100_Pacco/compose/services.yml`.** There is no container definition and no port mapping for it anywhere in the umbrella repository.
- CI: `.travis.yml` present with the standard build and dockerize chain.

## 10. Security / auth clues

- JWT validation using the certificate at `certs/localhost.cer`, following the platform pattern.
- **No Vault configuration at all** — no KV path, no PKI role, no dynamic credentials. Every other service has a `vault` block.
- No `security` certificate block, so no service-to-service certificate checking.

## 11. Observability / logging / tracing

- Logging through `ILogger<AIOrderMakingSaga>`, with informational messages at each saga step (`Started a saga for order: …`, `Searching for a vehicle...`, `Found a vehicle with id: …`, `Reserving a date for vehicle: …`, `Completed a saga for order: …`).
- `CorrelationContext.cs` propagates correlation across published messages.
- **No `jaeger` block in `appsettings.json`** — this is the only service in the platform with no distributed tracing configuration, which means the saga, the single most distributed flow in the system, is the hardest one to trace.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` — the orchestration decision: this is the one place in Pacco where a central coordinator drives other services, in contrast to the choreography used everywhere else.
- `src/Pacco.Services.OrderMaker/Sagas/AIMakingOrderData.cs` — the shape of in-flight cross-domain state.
- `src/Pacco.Services.OrderMaker/Handlers/AIOrderMakingHandler.cs` — the saga entry point.
- `src/Pacco.Services.OrderMaker/Services/ResourceReservationsService.cs` — the "best slot" selection rule.
- `src/Pacco.Services.OrderMaker/appsettings.json` — the absence of a database, of secrets management and of tracing.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json`. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` contains only the single C# project. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| This repository's README states the service runs on port `5015` | Confirmed by the Consul port in `appsettings.json`, but no run list or container definition anywhere allocates that port | Needs validation |
| The platform README lists this repository among the twelve to clone | It is absent from every run list and from the container stack, so cloning it does not make it run | Stale doc |
| The platform README describes clean architecture with four layers per service | This service is a single flat project | Stale doc |
| The platform README describes Vault-managed secrets and Jaeger tracing as platform-wide | Neither is configured in this service | Stale doc |
| The platform README describes an event-driven, choreographed design | This service is a central orchestrator, the opposite pattern; both styles coexist in the platform | Needs validation — the two integration styles are not reconciled in any document |
| The compensation logic cancels a failed order | Only the parcel step compensates; four of the five compensation methods are empty | Needs validation |

**Docs-only claims:** the port allocation.
**Disk-only components:** the entire saga implementation, the Chronicle dependency, and the "best vehicle / best slot" selection services — present in code, not described in the platform README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | This service is a demonstration of the saga pattern rather than a supported part of the platform. | It is missing from every run list and container definition, and it has no database, no secrets management and no tracing. |
| A2 | It is meant to be reached directly, not through the public gateway. | The gateway routing files contain no module for it. |
| A3 | The word used in the class names to mean "automatic order placing" refers to the automatic selection of a vehicle and a time slot, not to any machine-learning component. | The selection logic is two ordinary calls to other services followed by a choice; there is no model, training data or inference library anywhere in the repository. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | The service has no database and no visible durable storage for in-flight orders, so a restart in the middle of placing an order would lose it with no record. | **[ACTION NOW]** Recorded here for the requesting team; this stage does not change any source repository. |
| B2 | There is no way to run this service from the shared setup, so its behaviour cannot be confirmed from this workspace. | **[ACTION NOW]** Ask the requesting team whether it should be added to the run configuration or treated as out of scope. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | If a step after adding parcels fails, nothing undoes the earlier work. Is that intended? | **[handled later by the ADR authoring stage]** Record the intended clean-up behaviour for a failed order. |
| Q2 | The service defines a reaction to the resource-reserved message but never uses it in the sequence. Is a step missing? | **[handled later by the ADR authoring stage]** Trace the intended sequence and record it. |
| Q3 | Should the platform use central orchestration, event choreography, or both? Both exist today with no written rule. | **[handled later by the ADR authoring stage]** Record the intended integration style. |
| Q4 | Why does this service call other services directly instead of through the platform load balancer, unlike every other caller? | **[ACTION NOW]** Confirm with the requesting team whether the empty setting is deliberate. |
