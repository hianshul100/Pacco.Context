# Repository Summary — `hianshul100_Pacco.Services.Vehicles`

**Primary name:** `vehicles-service` (aliases used in this file: `Pacco.Services.Vehicles.Api` — the .NET project and assembly name; `devmentors/pacco.services.vehicles` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Vehicles`, path: `src/Pacco.Services.Vehicles.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Owns the vehicle catalogue — the fleet available to carry orders. It adds, updates, deletes and searches vehicles, and supplies the "best" vehicle for an order when the saga asks for one. It is a reference-data service: other services read from it, and nothing writes to it except the gateway.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`, with an in-process RabbitMQ consumer. Layered `.Api` / `.Application` / `.Core` / `.Infrastructure`.

It is the only service whose read route returns a paged result type rather than a plain collection.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Vehicles.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Vehicles.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Vehicles.sln`: `src/Pacco.Services.Vehicles.Api`, `.Application`, `.Core`, `.Infrastructure`. `.Core` has no NuGet references. `.Infrastructure` carries the standard Convey stack (Vault, Consul, Fabio, HTTP, message brokers with Mongo outbox, AppMetrics, MongoDB, Redis, Security, Jaeger, Swagger).

## 5. External integrations

- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — through Convey extensions.
- No outbound HTTP calls to peer services: `httpClient.services` in `src/Pacco.Services.Vehicles.Api/appsettings.json` is empty. This service is read by `orders-service` and `ordermaker-service`; it calls no one.

## 6. Data stores & state

- **MongoDB.** Database `vehicles-service`, connection string `mongodb://localhost:27017`, `seed: false`.
- **Collection:** `vehicles` — `.AddMongoRepository<VehicleDocument, Guid>("vehicles")`.
- **Outbox collections:** `outbox` and `inbox` (`outbox.enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `connectionString: localhost`, key prefix `vehicles:`.
- **Query mechanism:** Convey `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver, with paging support behind the `SearchVehicles` query. **No ORM.**
- **Migration tool:** none.
- **Cross-domain coupling:** none in this service's own data. It stores no copy of another domain's entities and holds no identifiers it does not own. The coupling runs the other way — `orders-service` stores vehicle identifiers this service owns, and `availability-service` subscribes to `vehicle_deleted` so it can release reservations tied to a removed vehicle. There are no foreign keys; consistency depends on that event arriving.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `vehicles`, queue template `vehicles-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Mongo-backed outbox; Jaeger plugin registered on the broker.

**Consumed** (`src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs`, lines 86–88) — three commands, all published by `api-gateway` in async mode:

| Kind | Message | Wire name |
|---|---|---|
| Command | `AddVehicle` | `add_vehicle` |
| Command | `UpdateVehicle` | `update_vehicle` |
| Command | `DeleteVehicle` | `delete_vehicle` |

This service subscribes to **no events from other services**. Together with `deliveries-service` it is one of only two services driven purely by commands.

**Published**, per the `vehicles-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: events `vehicle_added`, `vehicle_deleted`, `vehicle_updated`; rejection events `add_vehicle_rejected`, `delete_vehicle_rejected`, `update_vehicle_rejected`. This is the platform's most symmetric message set — every command has both a success event and a rejection event.

**Consumer of this service's events:** `availability-service` subscribes to `VehicleDeleted`. Nothing in the workspace subscribes to `vehicle_added` or `vehicle_updated` apart from `operations-service`.

**Payload key fields:** the API gateway routes for this exchange carry the vehicle body directly with no `bind` entries, so the payload is whatever the client posts. The complete field set of each message is **unknown — requires runtime capture**.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.Vehicles.Api/Program.cs`:

| Method | Path | Dispatched to | Response |
|---|---|---|---|
| GET | `vehicles` | `SearchVehicles` | `PagedResult<VehicleDto>` |
| GET | `vehicles/{vehicleId}` | `GetVehicle` | `VehicleDto` |
| POST | `vehicles` | `AddVehicle` | — |
| PUT | `vehicles/{vehicleId}` | `UpdateVehicle` | — |
| DELETE | `vehicles/{vehicleId}` | `DeleteVehicle` | — |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Consumed by:**

- `api-gateway` — exposes `GET vehicles` publicly and routes the three mutations through RabbitMQ.
- `orders-service` — over HTTP through Fabio.
- `ordermaker-service` — over HTTP through Fabio, calling `IVehiclesServiceClient.GetBestAsync()` during the order-making saga.

**Consumes:** nothing over HTTP.

**A gap worth flagging:** the saga calls `GetBestAsync()`, but the route table above contains no "best vehicle" endpoint. Either `GetBestAsync` maps onto `SearchVehicles` with filter and paging parameters, or it targets a route not visible in `Program.cs`. See open questions.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5009` — the last in the contiguous `5001`–`5009` block.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.vehicles`, present in `hianshul100_Pacco/compose/services.yml`.
- **This repository has no `LICENSE` file**, unlike the eight other service repositories and the gateway. `hianshul100_Pacco.Services.Pricing` is the only other repository missing one.

## 10. Security & auth clues

- No `security` access-control list and no certificate authentication.
- JWT validated against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- **The gateway applies no role restriction to the vehicle routes.** `ntrada.yml` guards the `customers` module reads with `claims: role: admin`, but the `vehicles` module — including `add_vehicle`, `update_vehicle` and `delete_vehicle` — carries no claims requirement. Any authenticated user can alter the fleet. See open questions.
- Vault: `enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, key-value path `vehicles-service/settings`, PKI `roleName: vehicles-service`, `commonName: vehicles-service.pacco.io`, `lease.mongo` dynamic MongoDB credentials with `autoRenewal: true`.
- **Checked-in credentials:** `vault.token: "secret"`, RabbitMQ `guest`/`guest`, Seq `apiKey: secret` in `src/Pacco.Services.Vehicles.Api/appsettings.json`.
- `.UsePublicContracts<ContractAttribute>()` exposes the message contract surface.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: vehicles`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker.
- **Correlation:** `Correlation-Context` header on inbound requests; the `Saga` header is forwarded, which matters here because `ordermaker-service` calls this service mid-saga.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` (daily) + Seq at `http://localhost:5341`. ELK configured at `http://localhost:9200` but disabled. `excludePaths: ["/", "/ping", "/metrics"]` and the standard `excludeProperties` redaction list. `.AddHandlersLogging()` logs each handler.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` — the composition root and the three-command subscription list.
- `src/Pacco.Services.Vehicles.Api/Program.cs` — the route table and the decision to return `PagedResult<VehicleDto>` from the search route, the platform's only paged endpoint.
- `src/Pacco.Services.Vehicles.Api/appsettings.json` — the configuration contract.
- `Pacco.Services.Vehicles.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- **How does `ordermaker-service` fetch the "best" vehicle?** `IVehiclesServiceClient.GetBestAsync()` exists in `ordermaker-service`, but no matching route appears in this service's `Program.cs`. Whether it calls `SearchVehicles` with parameters or a route declared elsewhere is **Unknown**. **Needs validation** — this is the sharpest open item in the repository.
- What "best" means — how vehicles are ranked — is **Unknown**. Nothing in this repository defines an ordering.
- Nothing consumes `vehicle_added` or `vehicle_updated`. Whether a consumer is planned is **Unknown**.
- The vehicle model — capacity, type, availability attributes — lives in `src/Pacco.Services.Vehicles.Core/` and was not enumerated. **Unknown.**
- Whether fleet mutation should be restricted to administrators is **Unknown**; the gateway currently applies no role check.
- The absence of a `LICENSE` file here and in `hianshul100_Pacco.Services.Pricing` may be deliberate or an oversight. **Unknown.**
- Full payload field sets are **unknown — requires runtime capture**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Vehicles.Api/`, `src/Pacco.Services.Vehicles.Application/`, `src/Pacco.Services.Vehicles.Core/`, `src/Pacco.Services.Vehicles.Infrastructure/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Vehicles` directory". | No such directory. The runnable project is `src/Pacco.Services.Vehicles.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5009`." | `appsettings.json` sets the Consul port to `5009`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.vehicles`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Vehicles.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README is the shared platform template and says nothing service-specific. | Three command subscriptions, six published messages, and a synchronous read path used mid-saga by `ordermaker-service`. | **Docs gap.** |

**On disk but not mentioned in `README.md`:** the messaging surface, the outbox, the Vault integration, the paged search result, and the fact that the order-making saga depends on this service being reachable.

**Docs-only claims:** none beyond the `dotnet run` path.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory, and no `LICENSE`** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `IVehiclesServiceClient.GetBestAsync()` in `ordermaker-service` resolves to the `GET vehicles` search route with parameters, not to a separate undocumented endpoint. | `GET vehicles` is the only read route capable of returning a selected vehicle, and it supports paging and filtering. | The order-making saga would depend on an endpoint missing from this inventory, leaving a hidden integration point. | Read `src/Pacco.Services.OrderMaker/.../VehiclesServiceClient.cs` and compare the path it requests. |
| A2 | `availability-service` relies on `vehicle_deleted` to release reservations tied to a removed vehicle. | `availability-service` subscribes to `VehicleDeleted`, and reservations in that service are keyed to resources that vehicles occupy. | The consequence of deleting a vehicle would be misdescribed. | Read the `VehicleDeleted` handler in `hianshul100_Pacco.Services.Availability`. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Which endpoint does the order-making saga call to choose a vehicle, and how are vehicles ranked? | The saga cannot complete without it, and nothing in this repository defines what "best" means. If the ranking is arbitrary, order quality is arbitrary. | It most likely calls `GET vehicles` and takes the first result, which would mean there is no real ranking. | Service owner |
| Q2 | **[ACTION NOW]** Should adding, updating and deleting vehicles require an administrator role at the gateway? | Today any authenticated user can change the fleet. The gateway applies a `role: admin` check to customer reads but not to fleet writes. | Add a `claims: role: admin` requirement to the `vehicles` mutation routes in `ntrada.yml`. | Security owner |
| Q3 | **[handled later by the platform inventory review]** Is anything meant to consume `vehicle_added` or `vehicle_updated`? | Both are published and reach only the operations feed. A fleet change that affects existing reservations may be going unnoticed. | They may exist purely to drive client notifications. | Service owner |
