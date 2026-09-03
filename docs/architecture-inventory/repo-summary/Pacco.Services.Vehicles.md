# Repository: `Pacco.Services.Vehicles`

`vehicles-service` (also known as: Vehicles Service, `Pacco.Services.Vehicles`, Docker image
`devmentors/pacco.services.vehicles`) owns the vehicle catalogue — the fleet whose capacity the
rest of the platform reserves and assigns.

- **Repository:** `Pacco.Services.Vehicles`, path: `src/Pacco.Services.Vehicles.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5009`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- The four-project clean-architecture split, and `Core/Entities/Vehicle.cs`,
  `Core/Entities/Variants.cs`.
- All five HTTP endpoints, including the **paged search** on `GET /vehicles` returning
  `PagedResult<VehicleDto>` — the only paged endpoint on the platform.
- The RabbitMQ `vehicles` exchange with its nine messages.
- MongoDB database `vehicles-service`, collection `vehicles`.
- That this service consumes **no** external event — it is a pure source.
- That the write routes are admin-gated at the gateway.

**Present in most sibling repositories, absent here:** a `LICENSE` file. Eight of the ten service
repositories have one; this repository and `Pacco.Services.Pricing` are the two that do not.
**Recorded as an observation.**

**Stale doc:** none identified.

**Unknown:** what `Core/Entities/Variants.cs` represents commercially — variants influence vehicle
pricing attributes, but the rule is only visible in code.

---

## 1. Primary purpose

Maintain the catalogue of vehicles available to the platform, each with its variants and pricing
attributes, and announce catalogue changes so that dependent services can react — in particular so
`availability-service` can withdraw a resource when its vehicle is deleted.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process, using Convey CQRS dispatcher
endpoints. It consumes only its own commands; it subscribes to no other service's events.

## 3. Key entrypoints

- `src/Pacco.Services.Vehicles.Api/Program.cs` — composition root and route table.
- `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` — DI composition root.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Vehicles.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Project | Role |
|---|---|
| `Pacco.Services.Vehicles.Api` | Host, route table, configuration, `certs/` |
| `Pacco.Services.Vehicles.Application` | Commands (`AddVehicle`, `UpdateVehicle`, `DeleteVehicle`), queries (including `SearchVehicles`), events, DTOs, handlers |
| `Pacco.Services.Vehicles.Core` | `Entities/Vehicle.cs`, `Entities/Variants.cs`, `Repositories/IVehicleRepository.cs`, domain exceptions |
| `Pacco.Services.Vehicles.Infrastructure` | Mongo documents and repository, RabbitMQ broker, decorators, contexts, logging |

**Key packages:** `Convey`, `Convey.CQRS.Commands/.Events/.Queries`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Swagger`.

## 5. External integrations

Consul (registration, `pingEndpoint: ping`), Fabio, RabbitMQ (exchange `vehicles`), MongoDB
(database `vehicles-service`), Redis (prefix `vehicles:`), Vault (kv v2 path
`vehicles-service/settings`, PKI role `vehicles-service`, common name
`vehicles-service.pacco.io`, MongoDB dynamic credentials), Jaeger, Seq, Prometheus.

**It calls no other service.** `httpClient.services` is empty — a leaf in the synchronous call
graph, and the target of two inbound synchronous callers.

## 6. Data stores / state

- **Store:** MongoDB, database `vehicles-service`.
- **Query mechanism:** Convey `IMongoRepository<VehicleDocument, Guid>` over the MongoDB .NET
  driver. **Not a relational ORM.** The paged search on `GET /vehicles` is served through the
  driver's query support, returning `PagedResult<VehicleDto>`.
- **Registration:** `AddMongoRepository<VehicleDocument, Guid>("vehicles")` in
  `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs`.
- **Collection for the primary domain:** **`vehicles`**, mapped by
  `Infrastructure/Mongo/Documents/VehicleDocument.cs`, with variants held on the vehicle document.
- **Framework collections:** `inbox`, `outbox` (`type: sequential`, `disableTransactions: true`).
- **Migration tool:** **none.** No migration files or tooling in the repository.
- **Cross-domain coupling:** **none inbound.** Unlike `orders-service` and `parcels-service`, this
  service replicates no other domain's data — it keeps no `customers` collection and stores no
  foreign identifiers. It is a clean upstream source. The coupling runs the other way: `OrderId`
  and vehicle references live in *other* services' documents (`OrderDocument.VehicleId` in
  `orders-service`, resource records in `availability-service`), with no referential enforcement
  anywhere. Deleting a vehicle here therefore invalidates data held elsewhere, which is exactly
  what the `vehicle_deleted` event exists to signal.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `vehicles`, type `topic`, durable.
- **Conventions:** `snakeCase`; queue template `vehicles-service/{{exchange}}.{{message}}`; headers
  `message_context` and `span_context`.
- **Outbox:** enabled (`AddMessageOutbox(o => o.AddMongo())`) with outbox decorators on handlers.

**Commands consumed:** `add_vehicle`, `delete_vehicle`, `update_vehicle`.

**Events published:**

| Event | Observable payload fields |
|---|---|
| `vehicle_added` | `VehicleId`, `Name` |
| `vehicle_updated` | `VehicleId` |
| `vehicle_deleted` | `VehicleId` |

**Rejected events published:** `add_vehicle_rejected`, `delete_vehicle_rejected`,
`update_vehicle_rejected` — each with `Reason` and `Code`.

**External events consumed: none.** No `Events/External/Handlers/` content exists in this
repository. Together with `deliveries-service`, this is one of only two broker-connected services
that subscribe to nothing — but unlike Deliveries, its events do drive another service.

**Consumers of this service's events:** `vehicle_deleted` → `availability-service`, which removes
the corresponding resource. `vehicle_added` and `vehicle_updated` have **no domain consumer** —
only `operations-service` observes them. Notably `orders-service` stores a `VehicleId` on every
order and consumes **none** of these three events, so an order can retain a reference to a vehicle
that has been deleted or materially changed. **Needs validation.**

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Vehicles.Api/Program.cs`, verbatim):

| Method | Route | Dispatches | Returns |
|---|---|---|---|
| `GET` | `vehicles/{vehicleId}` | query — single vehicle | `VehicleDto` |
| `GET` | `vehicles` | `SearchVehicles` | `PagedResult<VehicleDto>` |
| `POST` | `vehicles` | `AddVehicle` | — |
| `PUT` | `vehicles/{vehicleId}` | `UpdateVehicle` | — |
| `DELETE` | `vehicles/{vehicleId}` | `DeleteVehicle` | — |

Swagger UI at route prefix `docs`.

**Consumed:** none.

**Inbound synchronous callers:**

- `orders-service` — `GET {vehicles}/vehicles/{id}`, validating a vehicle before assignment.
- `ordermaker-service` — `GET {vehicles}/vehicles`, then `.Items.FirstOrDefault()` to pick a
  vehicle for the saga. That call consumes the paged result but takes only the first item.

**Upstream:** the gateway module `vehicles` fronts all five routes; the three write routes carry a
`role: admin` claim gate. In async mode the writes arrive as RabbitMQ commands instead.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Vehicles.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.vehicles`.
- Port `5009` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` and the gateway
  `localUrl`. **Note:** in `Pacco/prod-services.yml` the vehicles entry is the last of the ten
  apps and takes the final port in the 5000–5009 block.
- Consul service name `vehicles-service`; `httpClient.type: fabio` (configured but unused, since
  the service calls nothing).
- **Runtime dependency note:** this service sits on the order-creation critical path for both
  `orders-service` (vehicle validation) and `ordermaker-service` (vehicle selection).

## 10. Security / auth clues

- **JWT bearer** with `certs/localhost.cer`, `validIssuer: pacco`.
- **Admin-gated writes at the edge.** The gateway applies `claims: {role: admin}` to `POST`,
  `PUT` and `DELETE` on `vehicles` — this is fleet-management data, and it is one of only three
  admin-gated areas on the platform (the others being customer administration and identity user
  lookup). Whether this service re-checks the role itself is **Unknown — needs validation**.
- **Reads are open to any authenticated caller** — both `GET` routes have no claim gate, so the
  full fleet catalogue is readable by any signed-in user.
- **No certificate ACL**, even though two services call this one synchronously. Neither
  `orders-service`'s nor `ordermaker-service`'s client attaches a client certificate.
  `customers-service` protects its equivalent inbound call with an ACL; this service does not.
  **Needs validation.**
- **Vault:** kv v2 settings, PKI role `vehicles-service`, MongoDB dynamic credentials with lease
  auto-renewal.
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: vehicles`, UDP `6831`, `const` sampler rate 1, with the
  `Convey.Tracing.Jaeger.RabbitMQ` plugin propagating `span_context` across AMQP.
- **Logging:** console, file and Seq sinks enabled; ELK sink present but `enabled: false`.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Vehicles.sln` | Four-project clean-architecture split |
| `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` | Capability chain, Mongo repository registration, outbox decorators |
| `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs`, `Entities/Variants.cs` | That a vehicle carries variants and pricing attributes as part of the aggregate, rather than pricing living only in `pricing-service` |
| `src/Pacco.Services.Vehicles.Application/Queries/` (`SearchVehicles`) | That vehicle listing is **paged** — the only paged query on the platform, and a statement that the fleet is expected to be large enough to need it |
| `src/Pacco.Services.Vehicles.Api/Program.cs` | That this service is a pure catalogue: five CRUD-shaped routes, no reaction to any external event |
| `src/Pacco.Services.Vehicles.Api/appsettings.json` | Exchange, outbox with `disableTransactions: true`, Vault PKI |
| `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` | That vehicle writes are administrative operations, gated on the `admin` role |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. Why `orders-service` stores a `VehicleId` but consumes none of the three vehicle events, so an
   order can point at a deleted vehicle.
2. Why `vehicle_added` and `vehicle_updated` have no domain consumer.
3. Whether this service re-checks the `admin` role or relies entirely on the gateway.
4. Why the two inbound synchronous calls carry no client certificate, when the equivalent
   Availability → Customers call does.
5. Whether the full fleet catalogue should be readable by any authenticated user.
6. Whether `ordermaker-service` taking `.Items.FirstOrDefault()` from a paged result is intended
   behaviour or a placeholder.
7. Why this repository has no `LICENSE` file.
8. Whether `outbox.disableTransactions: true` is deliberate.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Vehicles.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Vehicles.Application/`,
`src/Pacco.Services.Vehicles.Core/`, `src/Pacco.Services.Vehicles.Infrastructure/`, and the
repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor) — notably, there is no fleet-administration UI despite the
admin-gated write routes. The only browser-facing surface is the Convey Swagger UI at `/docs`,
generated by `Convey.WebApi.Swagger`.

---

## Evidence

| Fact | File |
|---|---|
| Route table, paged search registration, host composition | `src/Pacco.Services.Vehicles.Api/Program.cs` |
| DI composition, Mongo collection registration, outbox decorators | `src/Pacco.Services.Vehicles.Infrastructure/Extensions.cs` |
| Vehicle aggregate and variants | `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs`, `Entities/Variants.cs` |
| Persistence document | `src/Pacco.Services.Vehicles.Infrastructure/Mongo/Documents/VehicleDocument.cs` |
| Commands | `src/Pacco.Services.Vehicles.Application/Commands/AddVehicle.cs`, `UpdateVehicle.cs`, `DeleteVehicle.cs` |
| Paged search query and result type | `src/Pacco.Services.Vehicles.Application/Queries/SearchVehicles.cs`, `Queries/Handlers/` |
| Published events and payloads | `src/Pacco.Services.Vehicles.Application/Events/*.cs` |
| Rejected events | `src/Pacco.Services.Vehicles.Application/Events/Rejected/*.cs` |
| Absence of external event subscriptions | `src/Pacco.Services.Vehicles.Application/Events/` (no `External/Handlers` content) |
| Exchange, outbox, Vault, JWT, logging, metrics, tracing configuration | `src/Pacco.Services.Vehicles.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Package set | `src/Pacco.Services.Vehicles.Infrastructure/Pacco.Services.Vehicles.Infrastructure.csproj`, `src/Pacco.Services.Vehicles.Api/Pacco.Services.Vehicles.Api.csproj` |
| Project list | `Pacco.Services.Vehicles.sln` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Absence of a `LICENSE` file | repository root |
| The one consumer of `vehicle_deleted` | `../hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Application/Events/External/Handlers/` |
| Inbound synchronous callers | `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Services/Clients/VehiclesServiceClient.cs`, `../hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs` |
| Admin role gate on write routes | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | This service is the single source of truth for vehicle data | It is the only service with vehicle write endpoints and the only publisher of vehicle events; no other service holds a vehicle collection | If vehicle data is maintained anywhere else, two catalogues could disagree about what the fleet contains | Search the platform for any other vehicle write path |
| A2 | Variants are held on the vehicle document rather than in their own collection | Only `vehicles` is registered as a repository, and `Core/Entities/Variants.cs` appears as part of the vehicle aggregate | Statements about the vehicle consistency boundary would be wrong | Inspect a vehicle document in a running MongoDB instance |
| A3 | The admin role gate at the gateway is the only thing restricting vehicle writes | No role check is visible in this repository's route registration or handlers | Anything reaching this service without passing the gateway could add, change or delete fleet records | Read the command handlers for a role check, and confirm whether direct service access is possible in the deployed network |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should `orders-service` react to `vehicle_deleted` and `vehicle_updated`? | Every order stores a `VehicleId`, validated once at assignment. `availability-service` withdraws its resource when a vehicle is deleted, but `orders-service` does nothing — so an active order can keep pointing at a vehicle that no longer exists, and nothing detects it | Subscribe `orders-service` to the vehicle events, or state why a stale vehicle reference on an order is acceptable | Domain owners for Vehicles and Orders |
| Q2 | **[ACTION NOW]** Should the two inbound synchronous calls be certificate-authenticated? | `customers-service` requires a Vault-issued client certificate from its one caller and holds a matching ACL. This service is called by two services with no such protection, on the order-creation critical path | Either add the ACL and certificates as Availability/Customers do, or record why these calls are treated differently | Whoever owns Pacco authentication |
| Q3 | **[ACTION NOW]** Should the full fleet catalogue be readable by any authenticated user? | Writes are admin-gated but both read routes are open to anyone signed in, so the entire vehicle list and its pricing attributes are visible to every customer | Confirm whether fleet visibility is intended to be public to signed-in users or restricted | Domain owner for Vehicles |
| Q4 | **[ACTION NOW]** Does this service check the `admin` role itself? | The gate exists only at the gateway as far as the code shows. Anything that reaches this service directly could modify or delete fleet records without any role | Confirm whether direct service-to-service access is possible in the deployed network, and whether the check should also live here | Whoever owns Pacco authentication |
| Q5 | **[handled later by HLD]** Should `vehicle_added` and `vehicle_updated` have consumers? | Both are published and nobody listens, while `vehicle_deleted` does drive `availability-service`. A new vehicle therefore never becomes a bookable resource automatically — something or someone must create the resource separately | Either add the consumer in `availability-service`, or record how resources are created for new vehicles | Domain owners for Vehicles and Availability |
| Q6 | **[handled later by HLD]** Is `ordermaker-service` taking the first item from a paged vehicle list intended? | The saga calls `GET /vehicles` and takes `.Items.FirstOrDefault()`, so vehicle choice depends on whatever the first page happens to return. The code itself flags this as placeholder logic | Define what selecting the right vehicle should mean, or confirm the placeholder is acceptable for now | Domain owner for Orders |
| Q7 | **[ACTION NOW]** Why does this repository have no `LICENSE` file? | Eight sibling service repositories carry one and this one does not (`Pacco.Services.Pricing` is the only other omission), which leaves the terms for this code unstated | Add the platform licence, or state deliberately that it differs | Platform owner |
| Q8 | **[handled later by HLD]** Is `outbox.disableTransactions: true` the intended setting? | Without transactions the vehicle write and the outbox write are not atomic, so a crash between them can delete a vehicle that `availability-service` never hears about — leaving a bookable resource for a vehicle that no longer exists | Likely a single-node MongoDB constraint in development; confirm the production topology | Platform architect |
