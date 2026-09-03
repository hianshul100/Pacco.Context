---
title: "Repository Summary — Pacco.Services.Vehicles"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Vehicles"
status: "evidence-based inventory"
---

# Pacco.Services.Vehicles

**Primary name:** `Pacco.Services.Vehicles` (aliases used in this file: `vehicles-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `vehicles` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Vehicles`, path: `hianshul100_Pacco.Services.Vehicles/`

---

## 1. Primary purpose

Owns the vehicle catalogue: description, variant, capacity and price per service. Vehicles are the scarce resource that `Pacco.Services.Availability` reserves and that `Pacco.Services.Orders` assigns to an order.

Evidence: `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs`, `Variants.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service using Convey dispatcher endpoints, plus a RabbitMQ subscriber. Listens on `5009`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` | `src/Pacco.Services.Vehicles.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

Four source projects: `Pacco.Services.Vehicles.Api`, `.Application`, `.Core`, `.Infrastructure`. **No test project exists in this repository.**

- **Core:** `Entities/Vehicle.cs`, `Entities/Variants.cs`; `Repositories/IVehiclesRepository.cs`; exceptions `InvalidVehicleCapacity`, `InvalidVehicleDescriptionException`, `InvalidVehiclePricePerServiceException`.
- **Application:** commands `AddVehicle`, `DeleteVehicle`, `UpdateVehicle` with handlers; integration events `VehicleAdded`, `VehicleDeleted`, `VehicleUpdated`; three rejected events; queries `GetVehicle`, `SearchVehicles`.
- **Infrastructure:** MongoDB documents, repositories and query handlers, plus the shared outbox decorators, event mapper, message broker and exception mappers.

**Package anomaly:** `Pacco.Services.Vehicles.Api.csproj` **does not reference `Convey.Logging`**, although `Program.cs` calls `.UseLogging()`. Every other service's API project references it explicitly. The call presumably resolves through a transitive reference. **Needs validation.**

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus. No outbound HTTP service clients are defined.

## 6. Data stores / state

- **Store:** MongoDB, database `vehicles-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with explicit document classes and hand-written repositories.
- **Collections:** a vehicles collection, plus `inbox` and `outbox`.
- **Migration tool:** none anywhere in the repository.
- **Cross-domain coupling:** none stored. This service holds no copy of customer, order or reservation data — it is the cleanest bounded context in the platform. Coupling runs outward instead: the vehicle identifier is copied into the orders service (assigned vehicle) and into the availability service (the reserved resource).
- **Cache:** Redis.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `vehicles`, snake-case naming, queue template `vehicles-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`. Transactional outbox and inbox on MongoDB (`inbox`, `outbox`).

**Commands consumed:** `add_vehicle`, `delete_vehicle`, `update_vehicle`.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `vehicle_added` | `Application/Events/VehicleAdded.cs` | vehicle identifier |
| `vehicle_updated` | `Application/Events/VehicleUpdated.cs` | vehicle identifier |
| `vehicle_deleted` | `Application/Events/VehicleDeleted.cs` | vehicle identifier |

**Rejected events published:** `add_vehicle_rejected`, `delete_vehicle_rejected`, `update_vehicle_rejected`.

**External events consumed: none.** No `Events/External/` folder exists. `vehicle_deleted` is consumed by `Pacco.Services.Availability`, so the flow is one-way: this service tells others, and listens to nobody.

Wire names are confirmed against `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `vehicles/{vehicleId}` | `GetVehicle` |
| `GET` | `vehicles` | `SearchVehicles` → `PagedResult<VehicleDto>` from `Convey.CQRS.Queries` |
| `POST` | `vehicles` | `AddVehicle`, responds `Created` |
| `PUT` | `vehicles/{vehicleId}` | `UpdateVehicle` |
| `DELETE` | `vehicles/{vehicleId}` | `DeleteVehicle` |

**This is the only service that returns a paged result.** The gateway unwraps it with `onSuccess.data: response.data.items` on the `vehicles` list route, so clients receive a plain array and lose the paging metadata. **Needs validation.**

Called by: `Pacco.APIGateway` (module `vehicles`), `Pacco.Services.Orders` through its `VehiclesServiceClient`, and `Pacco.Services.OrderMaker` through its `VehiclesServiceClient` (`GetBestAsync()`).

This service consumes no other service's HTTP API.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.vehicles`, `5009:80` per the platform port map, network `pacco`. Consul registration on port `5009`.

CI: `.travis.yml` runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` on success.

**Repository completeness:** this repository has **no `LICENSE` file**, matching `Pacco.Services.Pricing` and unlike most of its siblings.

## 10. Security / auth clues

- JWT validation following the platform pattern, `validIssuer: pacco`.
- Vault: KV path `vehicles-service/settings`, PKI role for `vehicles-service`.
- No caller access-control list is defined, even though two other services call it directly.
- The write routes (`POST`, `PUT`, `DELETE vehicles`) are exposed through the gateway **without a role requirement**, unlike the customer administration routes which require the `admin` role. Any signed-in user can therefore add, change or delete a vehicle in the catalogue. **Needs validation.**

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: vehicles`, including RabbitMQ span propagation; Prometheus metrics via `Convey.Metrics.AppMetrics`; structured logging is invoked in `Program.cs` through `.UseLogging()` despite the missing package reference noted above.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs` — the validation rules behind the three invalid-vehicle exceptions.
- `src/Pacco.Services.Vehicles.Application/Queries/SearchVehicles.cs` — the paging decision, unique in the platform.
- `src/Pacco.Services.Vehicles.Infrastructure/Decorators/` — the outbox decorators.
- `src/Pacco.Services.Vehicles.Api/appsettings.json` — the infrastructure contract.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json`, which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` contains only the four C# projects. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes a vehicles service on .NET Core 3.1 built with Convey, following clean architecture | Confirmed: four layered projects, Convey `0.4.*`, `netcoreapp3.1` | Confirmed |
| The platform README treats vehicles as the reserved resource | Confirmed indirectly: the availability service consumes this service's deletion event, and the order-making saga picks a vehicle before reserving a slot | Confirmed |
| The build script chain includes `./scripts/test.sh` | There is no test project in this repository, so the step has nothing to execute | Needs validation |
| Administrative operations are role-protected at the gateway | True for customer routes, but the vehicle write routes carry no role requirement | Stale doc — the protection is not applied consistently |
| Every service project file declares the platform logging package | This service's API project does not, yet the code calls the logging setup | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the paged vehicle search and the vehicle validation rules — present in code, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The vehicle catalogue is managed by staff rather than by customers. | Vehicles are shared resources that customers reserve, not items customers own. |
| A2 | This service is the source of truth for a vehicle, and every other service holds only its identifier. | No other service stores vehicle details, and both callers fetch them live. |
| A3 | The logging setup works through an indirect package reference. | The code calls it, the service is expected to run, and no other explanation appears in the project files. |

### Blockers

_None._

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Should any signed-in customer be able to add or delete a vehicle from the shared catalogue? Today the gateway does not require an administrator role on those routes. | **[ACTION NOW]** Confirm with the requesting team; this is the widest permission gap found in the routing configuration. |
| Q2 | Clients never see the paging information for the vehicle list because the gateway unwraps it. Is that intended? | **[handled later by the ADR authoring stage]** Record whether the vehicle list is expected to be paged for clients. |
| Q3 | Why does this service have no automated tests when it is called by two other services and by the order-making flow? | **[handled later by the ADR authoring stage]** Record the testing approach for the platform as a whole. |
| Q4 | Why does this repository have no licence file when most of its siblings do? | **[handled later by the ADR authoring stage]** Confirm the intended licence for the platform and note the gap. |
