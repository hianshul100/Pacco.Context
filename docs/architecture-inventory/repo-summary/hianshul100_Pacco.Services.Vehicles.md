# Repository summary — `hianshul100_Pacco.Services.Vehicles`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Vehicles` (also known as: Pacco.Services.Vehicles, the Vehicles service)
**Deployable:** `Pacco.Services.Vehicles.Api` (also known as: `vehicles-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.vehicles`, its published image). Repository: `hianshul100_Pacco.Services.Vehicles`, path: `src/Pacco.Services.Vehicles.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

Owns the delivery fleet catalogue: each vehicle's brand, model, capacity and price per service. It is a reference-data service — orders assign a vehicle, availability releases reservations when a vehicle disappears, and the OrderMaker saga asks it to pick the best vehicle for an order.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, Convey-based, serving HTTP dispatcher endpoints and consuming RabbitMQ commands in the same process.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Vehicles.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Vehicles.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Vehicles.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Vehicles.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Dispatches to |
|---|---|---|
| GET | `vehicles` | `SearchVehicles`, returning `PagedResult<VehicleDto>` |
| GET | `vehicles/{vehicleId}` | `GetVehicle` |
| POST | `vehicles` | `AddVehicle` |
| PUT | `vehicles/{vehicleId}` | `UpdateVehicle` |
| DELETE | `vehicles/{vehicleId}` | `DeleteVehicle` |

## 4. Important modules / packages

Four projects: `src/Pacco.Services.Vehicles.Api`, `.Application`, `.Core`, `.Infrastructure`.

- **Core** — `Entities/Vehicle.cs`, `Entities/Variants.cs`.
- **Application** — commands `AddVehicle`, `UpdateVehicle`, `DeleteVehicle` with handlers; events `VehicleAdded`, `VehicleUpdated`, `VehicleDeleted`; rejected events `AddVehicleRejected`, `UpdateVehicleRejected`, `DeleteVehicleRejected`; queries `GetVehicle`, `SearchVehicles`.
- **Infrastructure** — `Mongo/Documents/VehicleDocument.cs`, the Mongo repository, `Mongo/Queries/Handlers/GetVehicleHandler.cs` and `SearchVehiclesHandler.cs`, `Services/EventMapper.cs`, `Services/MessageBroker.cs`, outbox decorators, exception mappers, logging template mapper.

Packages are the platform's standard Convey `0.4.*` set.

## 5. External integrations

MongoDB (database `vehicles-service`), Redis (instance prefix `vehicles:`), RabbitMQ (exchange `vehicles`), Consul (service `vehicles-service`, port 5009, ping endpoint `ping`), Fabio (`http://localhost:9999`), Vault (kv v2 mount `kv`, path `vehicles-service/settings`; PKI common name `vehicles.pacco.io`; dynamic MongoDB credentials with auto-renewal), Jaeger (service name `vehicles`, UDP `localhost:6831`, `sampler: const`), Prometheus, Seq.

`httpClient.services` is **empty** — no outbound service-to-service HTTP. Two services call in: `orders-service` and `ordermaker-service`.

## 6. Data stores and state handling

**Store:** MongoDB, database `vehicles-service`. **Query mechanism:** Convey MongoDB repositories over the MongoDB .NET driver. **No ORM. No migration tool.**

| Collection | Fields |
|---|---|
| `vehicles` | `Id` (Guid), `Brand` (string), `Model` (string), `Description` (string), `PayloadCapacity` (double), `LoadingCapacity` (double), `PricePerService` (decimal), `Variants` (enum `Variants`) |
| `inbox` | Convey inbox, de-duplication |
| `outbox` | Convey outbox, pending publishes |

Outbox: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Cross-domain coupling.** This is the least coupled data model in the platform. `VehicleDocument` holds no identifier belonging to another service — no customer, no order, no reservation. The coupling runs the other way: `orders-service` stores a `VehicleId`, and `availability-service` holds reservations that are released when a vehicle is deleted. Because `vehicles` keeps no back-reference, deleting a vehicle that an in-flight order depends on is possible, and the `vehicle_deleted` event is the only thing that lets the rest of the platform react.

Redis is registered but no cache call appears in this service's own code, so what it holds is **unknown — requires runtime capture**.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Mongo-backed transactional outbox and the Jaeger RabbitMQ plugin.

**Broker settings:** exchange `vehicles`, `conventionsCasing: snakeCase`, queue name template `vehicles-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `vehicles`:**

- Events: `vehicle_added`, `vehicle_updated`, `vehicle_deleted`
- Rejected events: `add_vehicle_rejected`, `update_vehicle_rejected`, `delete_vehicle_rejected`
- Commands accepted on the same exchange: `add_vehicle`, `update_vehicle`, `delete_vehicle`

Payload key fields: the vehicle events carry the vehicle identifier; exact field lists are **unknown — requires runtime capture**, since no `[Contract]`-annotated payload was read for these classes.

**Consumed:**

| Kind | Message | Origin exchange |
|---|---|---|
| Command | `AddVehicle` | `vehicles` |
| Command | `UpdateVehicle` | `vehicles` |
| Command | `DeleteVehicle` | `vehicles` |

**This service subscribes to no events from any other service.** Like `deliveries-service`, it is purely command-driven.

`vehicle_deleted` is consumed by `availability-service` (`Events/External/VehicleDeleted.cs`, `[Message("vehicles")]`), which releases the reservations tied to that vehicle. That is the only cross-service consumer of this service's events. `vehicle_added` and `vehicle_updated` are published but, as far as the source shows, **nothing in the platform subscribes to them**.

## 8. APIs exposed and consumed

**Exposed:** the five HTTP routes in section 3, Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5009`; port 80 in the container.

`GET /vehicles` returns a `PagedResult<VehicleDto>`; the gateway unwraps it with `onSuccess: data: response.data.items`, so external callers see a plain array and lose the paging metadata.

Two internal consumers call in through Fabio:

- `orders-service` → `GET {vehicles-service}/vehicles/{vehicleId}` (`Infrastructure/Services/Clients/VehiclesServiceClient.cs` in that repository).
- `ordermaker-service` → `GET {vehicles-service}/vehicles`, in `IVehiclesServiceClient.GetBestAsync()`.

**Worth surfacing.** `GetBestAsync()` in `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs` fetches the whole first page of `GET /vehicles` and returns `vehicles?.Items?.FirstOrDefault()` — the source comments it as `// typical AI in a startup`. There is no best-fit selection endpoint here and none is used: the saga's "best vehicle" is simply whichever vehicle the unfiltered, unsorted listing returns first, with no regard for `PayloadCapacity`, `LoadingCapacity` or `PricePerService`. As the catalogue grows, the client pulls a page of vehicles on every order and discards all but one.

**Consumed:** no outbound HTTP calls.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Vehicles.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `vehicles-service` on host port 5009.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.vehicles`.
- This repository and `hianshul100_Pacco.Services.Pricing` are the only two service repositories with **no `LICENSE` file**, while the other nine have one.

## 10. Security and auth clues

- JWT validated against the committed `certs/localhost.cer`, `validIssuer: pacco`.
- `Convey.Security` is composed; no certificate authentication here.
- **The gateway requires only authentication, not the `admin` role, on `POST /vehicles`, `PUT /vehicles/{vehicleId}` and `DELETE /vehicles/{vehicleId}`.** Compare the customer routes, which do require `admin`. Any signed-in customer can add, modify or delete a vehicle in the delivery fleet. This is the clearest authorisation gap in the platform's public surface.
- `logger.excludeProperties` keeps secret-like values out of logs; `httpClient.requestMasking` is configured.
- Vault supplies settings and dynamic MongoDB credentials; RabbitMQ credentials remain the plain defaults in `appsettings.json`.

## 11. Observability, logging and tracing clues

Jaeger under service name `vehicles`; Prometheus metrics via `Convey.Metrics.AppMetrics` (`metrics.database: pacco`, interval 5); structured logs to console, file and Seq with `/`, `/ping` and `/metrics` excluded. `Infrastructure/Logging/MessageToLogTemplateMapper.cs` gives each command and event its own log template.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` (technology choices and the three subscriptions), `src/Pacco.Services.Vehicles.Api/Program.cs` (HTTP surface, including the paged search that the gateway unwraps), `src/Pacco.Services.Vehicles.Api/appsettings.json` (integration endpoints), `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs` and `Variants.cs` (the fleet vocabulary), `src/Pacco.Services.Vehicles.Infrastructure/Mongo/Queries/Handlers/SearchVehiclesHandler.cs` (the search and paging behaviour that two other services depend on).

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Vehicles.Api/`, `src/Pacco.Services.Vehicles.Application/`, `src/Pacco.Services.Vehicles.Core/`, `src/Pacco.Services.Vehicles.Infrastructure/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5009`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.vehicles`; `Pacco.Services.Vehicles.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the three consumed commands and six published messages; the fact that two other services call this one synchronously; the paged search contract; the Vault integration; the transactional outbox.

**Conflicts to surface:**

- **Stale doc.** The README instructs running `dotnet run` in `/src/Pacco.Services.Vehicles`. That directory does not exist — the host project is `src/Pacco.Services.Vehicles.Api`. `scripts/start.sh` has the correct path.
- **Cross-repository observation.** `ordermaker-service` treats "the best vehicle" as the first item of `GET /vehicles`. This service's capacity and price fields play no part in that choice, so the fields exist but nothing in the platform selects on them. Recorded as Q1 below.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`, but this repository contains **no test project at all**. The step passes vacuously.
- **Repository hygiene.** No `LICENSE` file, unlike nine of the eleven repositories. Since the platform is published as open source, that is likely an oversight rather than a deliberate choice.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | `PayloadCapacity`, `LoadingCapacity` and `PricePerService` were meant to drive vehicle selection. | They are the only capacity and price fields in the platform, and the OrderMaker saga asks for a "best" vehicle — but its implementation ignores all three. |
| A2 | The missing `LICENSE` file is an oversight. | Nine of the eleven repositories carry the same licence and this one is otherwise identical in structure. |
| A3 | `messages.json` in the Operations repository is meant to be the complete catalogue of this service's messages. | Operations builds live subscriptions from it at start-up. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** Any signed-in customer can add, update or delete a fleet vehicle through the gateway; these routes require no role. | A customer could delete the vehicle assigned to someone else's order. | Gateway maintainer with the Vehicles service maintainer: require the `admin` role on the three write routes. |
| B2 | **[ACTION NOW]** A vehicle can be deleted while orders reference it, and nothing checks for that. | `availability-service` releases the reservations, but `orders-service` is left holding a `VehicleId` that no longer resolves. | Vehicles service maintainer, with the Orders service maintainer. |
| B3 | **[ACTION NOW]** CI runs `dotnet test` against a repository with no test projects. | The build reports success while testing nothing. | Vehicles service maintainer. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Should vehicle selection consider capacity and price, and should the filtering happen here rather than in the caller? | Today the saga takes the first vehicle in an unsorted list, so an order can be assigned a vehicle that cannot carry it. | Vehicles service maintainer with the OrderMaker service maintainer. |
| Q2 | **[handled later by the domain-model stage]** Does anything need `vehicle_added` and `vehicle_updated`? | Both are published and, as far as the source shows, consumed by nobody. | Domain-model stage. |
| Q3 | **[handled later by the API-contract stage]** What are the paging parameters and defaults of `GET /vehicles`? | The gateway discards the paging metadata, so external clients cannot page at all. | API-contract stage. |
| Q4 | **[handled later by the data-architecture stage]** What is stored in Redis under the `vehicles:` prefix? | Not answerable from source; determines whether Redis is a hard dependency here. | Data-architecture stage, by inspecting a running instance. |
