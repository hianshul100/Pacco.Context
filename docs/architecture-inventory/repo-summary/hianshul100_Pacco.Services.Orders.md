# Repository Summary — `hianshul100_Pacco.Services.Orders`

**Primary name:** `orders-service` (aliases used in this file: `Pacco.Services.Orders.Api` — the .NET project and assembly name; `devmentors/pacco.services.orders` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Orders`, path: `src/Pacco.Services.Orders.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Owns the order aggregate and the order lifecycle. It creates orders, adds and removes parcels, assigns vehicles, approves, cancels, deletes and completes orders, and reacts to delivery and resource-reservation outcomes from other services. It is the hub of the platform's domain: seven of the ten services exchange messages with it.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`, with an in-process RabbitMQ consumer. Layered `.Api` / `.Application` / `.Core` / `.Infrastructure`.

By message count it is the busiest service in the platform: 14 subscriptions and 19 declared published messages.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Orders.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Orders.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Orders.sln`: `src/Pacco.Services.Orders.Api`, `.Application`, `.Core`, `.Infrastructure`. `.Core` has no NuGet references. `.Infrastructure` carries the standard Convey stack.

**One test project exists on disk but is not in this repository's own solution file:** `tests/Pacco.Services.Orders.PactConsumerTests`, containing `tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs`. It is referenced only from the aggregate `Pacco.sln` in `hianshul100_Pacco`. See "README vs repository" and the coverage note below.

## 5. External integrations

- **`parcels-service`**, **`pricing-service`** and **`vehicles-service`** over HTTP through Fabio. `httpClient.services` in `src/Pacco.Services.Orders.Api/appsettings.json` maps `parcels` → `parcels-service`, `pricing` → `pricing-service`, `vehicles` → `vehicles-service`. This is the widest synchronous dependency fan-out of any service in the platform.
- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — through Convey extensions.

## 6. Data stores & state

- **MongoDB.** Database `orders-service`, connection string `mongodb://localhost:27017`, `seed: false`.
- **Collections:** two —
  - `orders` — `.AddMongoRepository<OrderDocument, Guid>("orders")`
  - `customers` — `.AddMongoRepository<CustomerDocument, Guid>("customers")`
- **Outbox collections:** `outbox` and `inbox` (`outbox.enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `connectionString: localhost`, key prefix `orders:`.
- **Query mechanism:** Convey `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver. **No ORM.**
- **Migration tool:** none.
- **Cross-domain coupling — the significant one.** The `customers` collection in the `orders-service` database is a **local read-model replica of another service's data**. It is populated by subscribing to `CustomerCreated`, an event owned by `customers-service`. `parcels-service` does exactly the same thing in its own database. So customer records exist in three separate MongoDB databases with no foreign key, no referential integrity and no reconciliation process; they are kept in step only by the event stream. An order document also holds parcel and vehicle identifiers owned by `parcels-service` and `vehicles-service` respectively, again with no enforcement.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `orders`, queue template `orders-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Mongo-backed outbox; Jaeger plugin registered on the broker.

**Consumed** (`src/Pacco.Services.Orders.Infrastructure/Extensions.cs`, lines 95–108) — 14 subscriptions, the most of any service:

| Kind | Message | Wire name | Origin |
|---|---|---|---|
| Command | `CreateOrder` | `create_order` | `api-gateway`, `ordermaker-service` |
| Command | `ApproveOrder` | `approve_order` | internal / `ordermaker-service` flow |
| Command | `CancelOrder` | `cancel_order` | `ordermaker-service` compensation |
| Command | `DeleteOrder` | `delete_order` | `api-gateway` |
| Command | `AddParcelToOrder` | `add_parcel_to_order` | `api-gateway`, `ordermaker-service` |
| Command | `DeleteParcelFromOrder` | `delete_parcel_from_order` | `api-gateway` |
| Command | `AssignVehicleToOrder` | `assign_vehicle_to_order` | `api-gateway`, `ordermaker-service` |
| Event | `CustomerCreated` | `customer_created` | `customers-service` |
| Event | `DeliveryStarted` | `delivery_started` | `deliveries-service` |
| Event | `DeliveryCompleted` | `delivery_completed` | `deliveries-service` |
| Event | `DeliveryFailed` | `delivery_failed` | `deliveries-service` |
| Event | `ParcelDeleted` | `parcel_deleted` | `parcels-service` |
| Event | `ResourceReserved` | `resource_reserved` | `availability-service` |
| Event | `ResourceReservationCanceled` | `resource_reservation_canceled` | `availability-service` |

**Published**, per the `orders-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`:

- Events: `order_approved`, `order_canceled`, `order_completed`, `order_created`, `order_deleted`, `order_delivering`, `parcel_added_to_order`, `parcel_deleted_from_order`, `vehicle_assigned_to_order`.
- Rejection events: `add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`, `cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`, `delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found`.

The last two rejection names are notable: `order_for_delivery_not_found` and `order_for_reserved_vehicle_not_found` are the platform's explicit acknowledgement that an event can arrive for an order this service does not know about — a direct consequence of eventual consistency across databases.

**Consumers of this service's events:** `ordermaker-service` (`order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved`), `customers-service` (`order_completed`), `parcels-service` (`order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order`), and `operations-service` (everything).

**Payload key fields**, from the `ordermaker-service` saga which publishes into this exchange: `create_order` carries `OrderId` and `CustomerId`; `assign_vehicle_to_order` carries `OrderId`, `VehicleId` and `ReservationDate`; `cancel_order` carries an order identifier and a reason. `api-gateway` binds `customerId:@user_id` into commands on this exchange. The complete field set of each message is **unknown — requires runtime capture**.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.Orders.Api/Program.cs`:

| Method | Path | Dispatched to |
|---|---|---|
| GET | `orders` | `GetOrders` |
| GET | `orders/{orderId}` | `GetOrder` |
| POST | `orders` | `CreateOrder` |
| DELETE | `orders/{orderId}` | `DeleteOrder` |
| POST | `orders/{orderId}/parcels/{parcelId}` | `AddParcelToOrder` |
| DELETE | `orders/{orderId}/parcels/{parcelId}` | `DeleteParcelFromOrder` |
| POST | `orders/{orderId}/vehicles/{vehicleId}` | `AssignVehicleToOrder` |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Consumed by:** `api-gateway`, which exposes `GET orders?customerId=@user_id` publicly and routes the five mutations through RabbitMQ.

**Consumes over HTTP:** `parcels-service`, `pricing-service` and `vehicles-service`, through Fabio with 3 retries and request masking (`maskTemplate: "*****"`).

**Contract testing:** `tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs` defines this service as the PACT **consumer** of the `parcels-service` API. The matching provider verification lives in `hianshul100_Pacco.Services.Parcels/tests/Pacco.Services.Parcels.PactProviderTests`. This is the only formal inter-service contract in the workspace, and it covers one of the platform's nine synchronous call paths.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`. Because `PactConsumerTests` is absent from `Pacco.Services.Orders.sln`, whether `test.sh` runs it depends on how the script selects projects — see open questions.
- Local port `5006`.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.orders`.

## 10. Security & auth clues

- No `security` access-control list and no certificate authentication.
- JWT validated against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- At the gateway, `GET orders` is scoped by `customerId:@user_id` bound from the JWT, so a caller can only list their own orders. The service's own `GET orders` route has no such constraint — the scoping is enforced by `api-gateway`, not by this service.
- Vault: `enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, key-value path `orders-service/settings`, PKI `roleName: orders-service`, `commonName: orders-service.pacco.io`, `lease.mongo` dynamic MongoDB credentials with `autoRenewal: true`.
- **Checked-in credentials:** `vault.token: "secret"`, RabbitMQ `guest`/`guest`, Seq `apiKey: secret` in `src/Pacco.Services.Orders.Api/appsettings.json`.
- `.UsePublicContracts<ContractAttribute>()` exposes the message contract surface.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: orders`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker.
- **Correlation:** `Correlation-Context` header on inbound requests; the `Saga` header is forwarded, which is how `ordermaker-service` saga state travels through this service.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]` and the standard `excludeProperties` redaction list. `.AddHandlersLogging()` logs each of the 14 handlers.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` — the composition root and the 14-message subscription list, which is in effect the order lifecycle expressed as integration points.
- `src/Pacco.Services.Orders.Api/Program.cs` — the route table and the decision to model parcel and vehicle attachment as sub-resource routes.
- `src/Pacco.Services.Orders.Api/appsettings.json` — the three-way HTTP dependency declaration.
- `tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs` — the decision to verify the `orders-service` → `parcels-service` contract rather than rely on integration tests.
- `Pacco.Services.Orders.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- **`PactConsumerTests` is absent from `Pacco.Services.Orders.sln`.** It exists on disk and is referenced only from the aggregate `Pacco.sln` in `hianshul100_Pacco`. Whether `./scripts/test.sh` in Travis picks it up is **Unknown**. **Needs validation** — if it does not, the platform's only contract test never runs in continuous integration.
- The order state machine — the legal transitions between created, approved, delivering, completed, canceled and deleted — lives in `src/Pacco.Services.Orders.Core/` and was not enumerated. **Unknown.**
- Only the `parcels-service` call is contract-tested. The `pricing-service` and `vehicles-service` calls are not. Whether that is a deliberate scope choice is **Unknown**.
- `order_delivering` is published but nothing in the workspace subscribes to it apart from `operations-service`. **Unknown** whether a consumer is planned.
- How `customers` replica records are corrected if a `customer_created` event is missed is **Unknown**; no reconciliation job exists.
- Full payload field sets are **unknown — requires runtime capture**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Orders.Api/`, `src/Pacco.Services.Orders.Application/`, `src/Pacco.Services.Orders.Core/`, `src/Pacco.Services.Orders.Infrastructure/`, `tests/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Orders` directory". | No such directory. The runnable project is `src/Pacco.Services.Orders.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5006`." | `appsettings.json` sets the Consul port to `5006`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.orders`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Orders.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README is the shared platform template and says nothing service-specific. | This is the platform's central aggregate: 14 subscriptions, 19 published messages, three synchronous dependencies, a replica of another service's data, and the only PACT contract test. | **Docs gap.** |

**On disk but not mentioned in `README.md`:** the entire messaging surface, the `customers` read-model replica, the three HTTP dependencies, the outbox, the Vault integration, and the `tests/` directory.

**Disk-only component not in this repository's own solution:** `tests/Pacco.Services.Orders.PactConsumerTests`.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The `customers` collection in the `orders-service` database is a read-model replica, not the system of record. | `customers-service` owns the `customers` domain and publishes `customer_created`; this service subscribes to that event and registers a `customers` repository of its own. | Data ownership across the platform would be misdescribed, and any statement about the customer system of record would be wrong. | Read the `CustomerCreated` handler under `src/Pacco.Services.Orders.Application/Events/External/`. |
| A2 | The events published by this service are exactly those listed under `orders-service` in `messages.json`. | `operations-service` reads that file at runtime, so a mismatch would break client notifications. | An event could be missing from the platform catalogue. | Read the event classes under `src/Pacco.Services.Orders.Application/Events/`. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does continuous integration actually run `Pacco.Services.Orders.PactConsumerTests`, given it is missing from `Pacco.Services.Orders.sln`? | It is the platform's only contract test. If it is not running, the one guarantee about inter-service compatibility is not being enforced. | Add the project to the repository's own solution file so `test.sh` picks it up. | Service owner |
| Q2 | **[handled later by the platform inventory review]** What corrects a `customers` replica that missed a `customer_created` event? | Three databases hold customer records with no reconciliation. A dropped message leaves a permanent inconsistency. | No mechanism exists today; the outbox reduces but does not eliminate the risk. | Platform architect |
| Q3 | **[handled later by the platform inventory review]** Is anything meant to consume `order_delivering`? | It is published but has no subscriber other than the operations feed. | It may exist purely to drive the client notification. | Service owner |
| Q4 | **[handled later by the platform inventory review]** Should the `pricing-service` and `vehicles-service` calls also be contract-tested? | Two of the three synchronous dependencies have no compatibility guarantee. | Extend the PACT approach to both. | Service owner |
