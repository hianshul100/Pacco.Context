# Repository summary — `Pacco.Services.Vehicles`

**Repository:** `Pacco.Services.Vehicles` (workspace clone path: `hianshul100_Pacco.Services.Vehicles`)
**Deployable:** `vehicles-service` (also known as: Vehicles Service, `Pacco.Services.Vehicles.Api`, image `devmentors/pacco.services.vehicles`). **Repository: `Pacco.Services.Vehicles`, path: `src/Pacco.Services.Vehicles.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Vehicles
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Owns the **vehicle catalogue** — the fleet available to carry orders. A vehicle has a brand, model, description, and a base price; the service is a straightforward CRUD-style master-data store. Its records are the thing `ordermaker-service` picks from when choosing the "best" vehicle, and its `vehicle_deleted` event is what makes `availability-service` retire the corresponding resource.

**Evidence:** `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs`, `src/Pacco.Services.Vehicles.Api/Program.cs`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice plus RabbitMQ consumer, in one process, using the canonical four-project clean-architecture layering (`.Api`, `.Application`, `.Core`, `.Infrastructure`) on Convey.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Vehicles.Api/Program.cs` — `AddConvey().AddWebApi().AddApplication().AddInfrastructure()`, then `UseInfrastructure()` + `UseDispatcherEndpoints(...)` |
| RabbitMQ subscriptions | `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` → `UseInfrastructure` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Vehicles.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Vehicles.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Vehicles.Api` | Host, endpoint map, `appsettings.json` |
| `src/Pacco.Services.Vehicles.Application` | `Commands/` (`AddVehicle`, `UpdateVehicle`, `DeleteVehicle`) + handlers; `Queries/` (`GetVehicle`, `BrowseVehicles`) + handlers; `Events/`, `Events/Rejected/`; `DTO/` |
| `src/Pacco.Services.Vehicles.Core` | `Entities/Vehicle.cs`, `Entities/AggregateRoot.cs`, `Events/`, `Exceptions/`, `Repositories/IVehicleRepository` |
| `src/Pacco.Services.Vehicles.Infrastructure` | `Mongo/Documents/VehicleDocument.cs`, `Mongo/Repositories/VehicleMongoRepository.cs`, `Decorators/` (outbox), `Contexts/`, `Exceptions/ExceptionToResponseMapper.cs` + `ExceptionToMessageMapper.cs`, `Logging/`, `Extensions.cs` |

**No test projects exist in this repository.**

Convey package set matches the platform standard, and includes `Convey.CQRS.Queries` with **paged-result support** — `BrowseVehicles` returns `PagedResult<VehicleDto>`, the only paged endpoint in the platform.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| RabbitMQ | in + out | exchange `vehicles`, topic |
| MongoDB | out | database `vehicles-service` |
| Redis | out | instance prefix `vehicles:` |
| Consul | out | registers `vehicles-service` on port `5009` |
| Fabio | out | `http://localhost:9999` |
| Vault | out | KV v2 `kv/vehicles-service/settings`; PKI role `vehicles-service`, CN `vehicles-service.pacco.io`; dynamic Mongo credentials |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

`httpClient.services` is **empty** — no outbound HTTP calls to other services.

**Called by two services:** `orders-service` → `GET /vehicles/{id}` (single vehicle), and `ordermaker-service` → `GET /vehicles` (the whole paged list, from which it selects the "best"). Neither call presents a caller credential.

**No fleet-management, telematics, GPS, or maintenance integration.** Vehicles are static master data: there is no location, no capacity constraint enforcement, no availability state on the vehicle itself (availability lives in `availability-service` as a separate resource), and no link to any real fleet system.

## 6. Data stores / state handling

- **Store:** MongoDB, database `vehicles-service`.
- **Collections:** `vehicles` (`AddMongoRepository<VehicleDocument, Guid>("vehicles")`), plus `inbox` and `outbox`.
- **Access mechanism:** Convey `IMongoRepository<>` over `MongoDB.Driver`. **No ORM.**
- **Migration tool: none.** No Flyway, Liquibase, Alembic, or EF Core migrations.
- **Document shape** (`Infrastructure/Mongo/Documents/VehicleDocument.cs`): vehicle id, brand, model, description, base price.
- **Cross-domain coupling:** none stored. Unlike `orders-service` and `parcels-service`, this service keeps **no local `customers` replica** and subscribes to no external events at all. Coupling runs outward only: the vehicle id it mints appears in `orders-service` order documents and in `availability-service` resources, and its `vehicle_deleted` event is what triggers resource removal there.
- **Outbox:** enabled, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: inbox`, `outboxCollection: outbox`, `disableTransactions: true`.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `vehicles`; `conventionsCasing: snakeCase`; queue template `vehicles-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — commands:**

| Message | Wire name | Key payload fields |
|---|---|---|
| `AddVehicle` | `add_vehicle` | `VehicleId`, `Brand`, `Model`, `Description`, `BasePrice` |
| `UpdateVehicle` | `update_vehicle` | `VehicleId`, `Brand`, `Model`, `Description`, `BasePrice` |
| `DeleteVehicle` | `delete_vehicle` | `VehicleId` |

**Consumed — external events: none.** Along with `deliveries-service`, this is one of only two services that subscribe to no events from anywhere. It is a pure master-data owner: commands in, events out.

**Published — events:**

| Event | Wire name | Key payload fields |
|---|---|---|
| `VehicleAdded` | `vehicle_added` | `VehicleId` |
| `VehicleUpdated` | `vehicle_updated` | `VehicleId` |
| `VehicleDeleted` | `vehicle_deleted` | `VehicleId` |

**Published — rejection events:** `add_vehicle_rejected`, `update_vehicle_rejected`, `delete_vehicle_rejected`, each with `Reason` and `Code`, produced by `Infrastructure/Exceptions/ExceptionToMessageMapper.cs`.

**Downstream effect — the one that matters:** `availability-service` subscribes to `vehicle_deleted` and removes the corresponding resource. So deleting a vehicle here silently destroys its availability calendar, including any reservations already made against it. Nothing in this service checks whether the vehicle is assigned to an open order or holds a future reservation, and no event flows back to say it was in use. `orders-service` keeps a `VehicleId` on its order documents and does **not** subscribe to `vehicle_deleted`, so an order can retain a reference to a vehicle that no longer exists.

**Reliability:** outbox/inbox decorators wrap every command and event handler. The `Saga` header is forwarded, which matters because `ordermaker-service` reads the vehicle list mid-saga.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5009`, container port `80`):

| Method | Path | Maps to | Gateway exposure |
|---|---|---|---|
| GET | `vehicles` | `BrowseVehicles` → **`PagedResult<VehicleDto>`** | `/vehicles` — gateway unwraps the envelope with `onSuccess.data: response.data.items` |
| GET | `vehicles/{vehicleId}` | `GetVehicle` | `/vehicles/{vehicleId}` |
| POST | `vehicles` | `AddVehicle` | `/vehicles` — gateway generates `vehicleId` |
| PUT | `vehicles/{vehicleId}` | `UpdateVehicle` | `/vehicles/{vehicleId}` |
| DELETE | `vehicles/{vehicleId}` | `DeleteVehicle` | `/vehicles/{vehicleId}` |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**Paged result:** this is the only endpoint in the platform returning a paged envelope, and the only place the gateway rewrites a response shape (`onSuccess.data: response.data.items`). Note the unwrapping **discards the paging metadata**, so a gateway client receives a plain array with no total count or page information — while `ordermaker-service`, calling the service directly, receives the full `PagedResult<VehicleDto>` and consumes only the first page when selecting the "best" vehicle.

**Consumed:** none over HTTP.

**Called by:** `orders-service` (`GET /vehicles/{id}`), `ordermaker-service` (`GET /vehicles`).

**Access control:** all `/vehicles/*` gateway routes are `auth: true` but carry **no role claim requirement**. Any authenticated user — an ordinary customer — can add, update, or delete a vehicle from the fleet. Compare `/customers`, where listing and state changes require `role: admin`. Fleet management is master data and has no such gate.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Vehicles.Api.dll`.
- Composed as `vehicles-service` on `5009:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5009`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- **`LICENSE` is missing** from this repository. Eleven of the thirteen repositories carry one; `Pacco.Services.Vehicles` and `Pacco.Services.Pricing` do not.

## 10. Security/auth clues

- **JWT bearer** validation via `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- `.AddSecurity()` is registered, but there is **no `security.certificate` block** — this service neither presents nor verifies client certificates, so both inbound service calls are unauthenticated at the application layer.
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- Log redaction via `logger.excludeProperties`.
- **No role gate on fleet mutation.** Any authenticated caller can create, modify, or delete a vehicle through the gateway, and deletion cascades into `availability-service` (§7). This is the second-widest write surface in the platform after `deliveries-service`.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: vehicles-service`, UDP `localhost:6831`, `sampler: const`) with the RabbitMQ Jaeger plugin.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`. Handler logging via `.AddHandlersLogging()`.
- **Correlation:** `Correlation-Context` header; `Saga` header forwarded.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics — in particular no counter for vehicle deletions, which is the action with the largest blast radius in this service.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs` | The vehicle model — brand, model, description, base price. Notably it carries **no capacity, no volume limit, and no location**, even though the platform's purpose is fitting parcels into vehicles |
| `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` | Composition, and the decision to subscribe to commands only — no external event inputs |
| `src/Pacco.Services.Vehicles.Application/Queries/BrowseVehicles.cs` | The platform's only paged query |
| `src/Pacco.Services.Vehicles.Application/Events/VehicleDeleted.cs` | The event whose consumption by `availability-service` makes vehicle deletion a cascading operation |
| `src/Pacco.Services.Vehicles.Api/appsettings.json` | Outbox with `disableTransactions: true` |
| `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` | (Cross-repository) the `onSuccess.data: response.data.items` rewrite that flattens this service's paged envelope for gateway clients |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`). No business behaviour is gated.

## 13. Open questions / ambiguities

- **A vehicle has no capacity attribute.** `parcels-service` computes parcel volume and the product is about fitting parcels into vehicles, yet nothing on the vehicle records how much it can hold, and no code compares the two. Where — or whether — capacity is enforced is **Unknown**. This is the platform's most conspicuous domain gap.
- **Vehicle deletion cascades** into `availability-service`, destroying the resource and any reservations on it, with no check that the vehicle is in use and no notification to `orders-service`, which retains `VehicleId` references. **Treated as a blocker below.**
- **No role gate on fleet mutation** — see §10.
- **"Best vehicle" selection lives in `ordermaker-service`, not here.** This service exposes no ranking, no filtering, and no availability view; the consumer fetches the list and decides. Whether selection logic belongs here is **Unknown**.
- **`ordermaker-service` reads only the first page** of `GET /vehicles` when choosing a vehicle, so its "best" is drawn from a page, not the fleet. **Needs validation.**
- **Gateway clients lose the paging metadata** because of the `onSuccess.data` rewrite. Whether that is deliberate is **Unknown**.
- **No tests** in this repository.
- **`LICENSE` is missing.** **Unknown** whether deliberate.
- **`disableTransactions: true`** on the outbox, as everywhere else.

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (four C# projects only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Vehicles service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5009`. — **Confirmed** (`appsettings.json` `consul.port: 5009`, `Pacco/compose/services.yml` `5009:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Vehicles` directory"**; the actual host project is **`/src/Pacco.Services.Vehicles.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **That deleting a vehicle cascades into `availability-service`** and destroys its resource and reservations. This is the single most consequential undocumented behaviour in the repository.
- The three commands, three events, and three rejection events — the whole message contract.
- **The paged `GET /vehicles` response** and the gateway's rewrite that flattens it, so the same endpoint returns different shapes depending on the caller.
- That the service consumes **no** external events, unlike most of its siblings.
- The transactional outbox/inbox and the handler decorators; `scripts/`.

**Also present on disk and worth noting:**
- **No `LICENSE` file**, unlike eleven of the thirteen repositories.

**Unknown (neither pass yielded proof):**
- Whether vehicle capacity was ever intended to live on the vehicle.
- Whether the missing licence is deliberate.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | A vehicle in `vehicles-service` corresponds one-to-one with a resource in `availability-service`, identified by the same id. | `availability-service` subscribes to `vehicle_deleted` and removes the matching resource, which only works if the identifiers line up. | Deleting a vehicle would remove the wrong resource, or none at all, and the two services' views of the fleet would diverge silently. | Trace a vehicle creation and confirm which component creates the corresponding resource, and with what id. |
| A2 | Vehicle capacity is not modelled anywhere, so nothing enforces that a set of parcels fits the vehicle assigned to their order. | The vehicle entity has no capacity field, and no code compares parcel volume against any vehicle attribute. | The platform would be silently over-allocating vehicles, and the parcel volume calculation would exist with no consumer. | Ask the product owner where capacity is meant to be enforced. |
| A3 | Fleet master data is expected to be managed by an operator rather than by customers, even though nothing enforces that. | The endpoints are CRUD over fleet data, which is not a customer concern, but no role claim is required on any of them. | Any customer could add or remove vehicles from the fleet in production. | Confirm the intended operator model with the product owner. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Deleting a vehicle cascades into `availability-service`, destroying its resource and every reservation on it, with no check that the vehicle is assigned to an open order and no signal back to `orders-service`, which keeps `VehicleId` references. Any authenticated user can trigger it. | Any deployment where the fleet endpoints are reachable by real users; also any reliability claim about reservations. | Service owner / product owner | Require an operator role on the fleet mutation routes, and decide what deleting an in-use vehicle should do — refuse, soft-delete, or cascade with notification — then implement that consistently across Vehicles, Availability, and Orders. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Where is vehicle capacity meant to be modelled and enforced? The vehicle entity has no capacity, yet `parcels-service` computes parcel volume and the product is about fitting parcels into vehicles. | It is the platform's most conspicuous domain gap: the two halves of the core problem exist and are never compared. | Unknown — no capacity attribute or check exists anywhere in the thirteen repositories. | Product owner |
| Q2 | **[ACTION NOW]** Should fleet management require an operator or administrator role? | Today any authenticated customer can add, update, or delete vehicles, and deletion has the cascading effect described in B1. | Yes — add a role claim on the `/vehicles` write routes at the gateway. | Security owner |
| Q3 | **[handled later by architecture_evolution_generation]** Should vehicle selection ("best vehicle") live in this service rather than in `ordermaker-service`? | Today the consumer fetches a page of the fleet and ranks it itself, so the selection rule sits outside the service that owns the data — and it sees only the first page. | Unknown — a filtered or ranked query here would be an alternative. | Architecture team |
| Q4 | **[handled later by architecture_evolution_generation]** Is it intended that gateway clients lose the paging metadata from `GET /vehicles`? | The same endpoint returns a paged envelope to internal callers and a bare array to gateway clients, so no external client can page through the fleet. | Probably a convenience choice in the gateway configuration rather than a decision. | API owner |
