# Repository: `Pacco.Services.Orders`

`orders-service` (also known as: Orders Service, `Pacco.Services.Orders`, Docker image
`devmentors/pacco.services.orders`) owns the order aggregate: its parcels, its assigned vehicle,
its price, its approval and its delivery state. It has the largest contract surface on the
platform.

- **Repository:** `Pacco.Services.Orders`, path: `src/Pacco.Services.Orders.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5006`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- The four-project clean-architecture split and the `Order` aggregate.
- All seven HTTP endpoints and the RabbitMQ `orders` exchange with its **26 messages** — 7
  commands, 9 events and 10 rejected events, more than any other service.
- The three synchronous dependencies: `parcels-service`, `pricing-service`, `vehicles-service`.
- The seven external events this service subscribes to.
- MongoDB database `orders-service` with **two** collections, `orders` and a replicated
  `customers`.
- `tests/Pacco.Services.Orders.PactConsumerTests` — **Pact consumer contract tests** against
  `parcels-service`, the only consumer-driven contract testing on the platform.

**Stale doc:** none identified.

**Unknown:** how the Pact contract file reaches `parcels-service`; no Pact Broker configuration
exists in either repository or either Travis pipeline.

---

## 1. Primary purpose

Hold the order through its whole life: created, parcels added and removed, vehicle assigned, priced,
approved, delivering, completed or cancelled — reacting to events from Customers, Parcels,
Availability and Deliveries, and pulling price and validation data synchronously from three
services.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process, using Convey CQRS dispatcher
endpoints.

## 3. Key entrypoints

- `src/Pacco.Services.Orders.Api/Program.cs` — composition root and route table.
- `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` — DI composition root.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Orders.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Project | Role |
|---|---|
| `Pacco.Services.Orders.Api` | Host, route table, configuration, `certs/` |
| `Pacco.Services.Orders.Application` | 7 commands, queries, 9 events, 10 rejected events, DTOs, handlers, service client interfaces |
| `Pacco.Services.Orders.Core` | `Entities/Order.cs` (aggregate root), `Entities/Parcel.cs`, `Entities/Customer.cs`, `Entities/OrderStatus.cs`, repository interfaces, domain exceptions |
| `Pacco.Services.Orders.Infrastructure` | Mongo documents and repositories, RabbitMQ broker, the three HTTP service clients, decorators, contexts, logging |
| `tests/Pacco.Services.Orders.PactConsumerTests` | Pact **consumer** tests (`Pactify` 1.1.0) with a `PACT/` directory |

**Key packages:** `Convey`, `Convey.CQRS.Commands/.Events/.Queries`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Swagger`; plus `Pactify` 1.1.0 in the test project.

## 5. External integrations

| Integration | How |
|---|---|
| `parcels-service` | HTTP `GET {parcels}/parcels/{id}` — validates a parcel before adding it to an order |
| `pricing-service` | HTTP `GET {pricing}/pricing?customerId={customerId}&orderPrice={orderPrice}` — obtains the discounted price |
| `vehicles-service` | HTTP `GET {vehicles}/vehicles/{id}` — validates a vehicle before assignment |
| RabbitMQ | Exchange `orders` |
| MongoDB | Database `orders-service` |
| Redis | Instance prefix `orders:` |
| Vault | kv v2 path `orders-service/settings`, PKI role `orders-service`, common name `orders-service.pacco.io`, MongoDB dynamic credentials |
| Consul, Fabio | Registration and load-balanced outbound calls (`httpClient.type: fabio`) |
| Jaeger, Seq, Prometheus | Tracing, logs, metrics |

This is the platform's most connected service: three synchronous dependencies and seven inbound
event subscriptions.

## 6. Data stores / state

- **Store:** MongoDB, database `orders-service`.
- **Query mechanism:** Convey `IMongoRepository<OrderDocument, Guid>` and
  `IMongoRepository<CustomerDocument, Guid>` over the MongoDB .NET driver. **Not a relational ORM.**
- **Registrations** (`src/Pacco.Services.Orders.Infrastructure/Extensions.cs`):
  `AddMongoRepository<CustomerDocument, Guid>("customers")` and
  `AddMongoRepository<OrderDocument, Guid>("orders")`.
- **Collections for the primary domain:**
  - **`orders`** — `Infrastructure/Mongo/Documents/OrderDocument.cs`, with parcels embedded as a
    nested collection, so the order document is the consistency boundary for its parcels.
  - **`customers`** — `Infrastructure/Mongo/Documents/CustomerDocument.cs`, a **replica of another
    service's domain**.
- **Framework collections:** `inbox`, `outbox` (`type: sequential`, `disableTransactions: true`).
- **Migration tool:** **none.** No migration files or tooling in the repository.
- **Cross-domain coupling — flagged.** MongoDB has no foreign keys, so the relational-FK test does
  not apply; the equivalent here is a locally stored copy of another domain's data plus stored
  identifiers with no enforcement:
  1. **`customers` collection** — owned by `customers-service`, replicated here and populated
     **only** by the `customer_created` event. `customer_state_changed` and `customer_became_vip`
     are published by the owner but **not consumed here**, so this copy is a creation-time snapshot
     that never updates. `parcels-service` holds the same coupling.
  2. **`OrderDocument` stores `VehicleId`** (owned by `vehicles-service`) and parcel identifiers
     (owned by `parcels-service`), validated once at write time by a synchronous call and never
     revalidated. The service does subscribe to `parcel_deleted`, so parcel removal is handled;
     there is **no** corresponding handling of `vehicle_deleted` here, though
     `availability-service` does consume it. **Needs validation.**

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `orders`, type `topic`, durable.
- **Conventions:** `snakeCase`; queue template `orders-service/{{exchange}}.{{message}}`; headers
  `message_context` and `span_context`.
- **Outbox:** enabled (`AddMessageOutbox(o => o.AddMongo())`) with outbox decorators on every
  command and event handler.

**Commands consumed** (verbatim): `add_parcel_to_order`, `approve_order`,
`assign_vehicle_to_order`, `cancel_order`, `create_order`, `delete_order`,
`delete_parcel_from_order`.

**Events published:**

| Event | Observable payload fields |
|---|---|
| `order_created` | `OrderId`, `CustomerId` |
| `order_approved` | `OrderId`, `CustomerId` |
| `order_canceled` | `OrderId`, `CustomerId`, `Reason` |
| `order_completed` | `OrderId`, `CustomerId` |
| `order_deleted` | `OrderId` |
| `order_delivering` | `OrderId` |
| `parcel_added_to_order` | `OrderId`, `ParcelId` |
| `parcel_deleted_from_order` | `OrderId`, `ParcelId` |
| `vehicle_assigned_to_order` | `OrderId`, `VehicleId` |

**Rejected events published** (10 — the largest rejection set on the platform):
`add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`,
`cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`,
`delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`,
`order_for_reserved_vehicle_not_found`. Note the last two break the `*_rejected` naming pattern —
they are stated as findings rather than rejections. **Recorded as an observation.**

**External events consumed** (`Application/Events/External/Handlers/`):

| Event | Source exchange | Purpose |
|---|---|---|
| `customer_created` | `customers` | Populates the local `customers` replica |
| `parcel_deleted` | `parcels` | Removes the parcel from any order holding it |
| `resource_reserved` | `availability` | Ties a reservation to the order |
| `resource_reservation_canceled` | `availability` | Reverses that tie |
| `delivery_started` | `deliveries` | Moves the order to a delivering state |
| `delivery_completed` | `deliveries` | Completes the order |
| `delivery_failed` | `deliveries` | Handles delivery failure |

**Consumers of this service's events:** `order_completed` → `customers-service`;
`order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order` →
`parcels-service`; `order_created`, `order_approved`, `parcel_added_to_order`,
`vehicle_assigned_to_order` → `ordermaker-service`; all of them → `operations-service`.
`order_delivering` has **no domain consumer**. **Needs validation.**

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Orders.Api/Program.cs`, verbatim):

| Method | Route | Dispatches |
|---|---|---|
| `GET` | `orders/{orderId}` | query — single order |
| `GET` | `orders` | query — order list, scoped by `IAppContext` (caller identity) |
| `POST` | `orders` | `CreateOrder` |
| `DELETE` | `orders/{orderId}` | `DeleteOrder` |
| `POST` | `orders/{orderId}/parcels/{parcelId}` | `AddParcelToOrder` |
| `DELETE` | `orders/{orderId}/parcels/{parcelId}` | `DeleteParcelFromOrder` |
| `POST` | `orders/{orderId}/vehicles/{vehicleId}` | `AssignVehicleToOrder` |

Swagger UI at route prefix `docs`.

**Consumed:** `GET {parcels}/parcels/{id}`, `GET {pricing}/pricing?customerId=&orderPrice=`,
`GET {vehicles}/vehicles/{id}`.

**Inbound synchronous callers:** none.

**Upstream:** the gateway module `orders` fronts all seven routes and rewrites `GET /` to
`orders-service/orders?customerId=@user_id`. In async mode the five write routes arrive as RabbitMQ
commands instead.

**Note:** `ordermaker-service` also exposes `POST /orders`, though it is not routed by the gateway.
Two services accept an order-creation request at the same path shape.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Orders.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.orders`.
  **`scripts/test.sh` is not invoked by CI, so the Pact consumer tests do not run there.**
- Port `5006` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5006:80`), and the
  gateway `localUrl`.
- Consul service name `orders-service`; `httpClient.type: fabio`.
- **Runtime dependency note:** creating or pricing an order requires `parcels-service`,
  `pricing-service` and `vehicles-service` to be reachable synchronously. This service has the
  platform's largest synchronous failure surface.

## 10. Security / auth clues

- **JWT bearer** with `certs/localhost.cer`, `validIssuer: pacco`.
- **Identity-scoped reads:** `GetOrdersHandler` filters by `IAppContext`, so the order list returns
  only the caller's orders. This is a real in-service authorisation check, not just gateway
  rewriting — one of the few on the platform.
- **No role gate at the edge** for the `orders` module — any authenticated caller can create,
  delete or modify an order by id. Whether write operations verify ownership is **Unknown — needs
  validation**.
- **Vault:** kv v2 settings, PKI role `orders-service`, MongoDB dynamic credentials with lease
  auto-renewal.
- **No certificate ACL** — no service calls this one synchronously, so none is currently needed.
  Its own outbound calls attach no client certificate, unlike `availability-service`'s call to
  `customers-service`. **Needs validation** of whether that is intentional.
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: orders`, UDP `6831`, `const` sampler rate 1, with the
  `Convey.Tracing.Jaeger.RabbitMQ` plugin propagating `span_context` across AMQP — important here,
  since an order's life spans three synchronous calls and seven inbound event types.
- **Logging:** console, file and Seq sinks enabled; ELK sink present but `enabled: false`.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Orders.sln` | Four-project clean-architecture split plus a contract-test project |
| `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` | Capability chain, two Mongo repository registrations (including the customer replica), outbox decorators |
| `src/Pacco.Services.Orders.Core/Entities/Order.cs`, `Entities/OrderStatus.cs` | That the order aggregate owns its parcels and moves through an explicit status machine |
| `src/Pacco.Services.Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs`, `PricingServiceClient.cs`, `VehiclesServiceClient.cs` | That order validation and pricing are **synchronous** calls rather than event-driven — a deliberate departure from the platform's messaging default |
| `src/Pacco.Services.Orders.Application/Events/External/Handlers/` | That order state advances in reaction to Deliveries, Availability, Parcels and Customers events |
| `tests/Pacco.Services.Orders.PactConsumerTests/` | That the Orders → Parcels dependency is governed by a consumer-driven contract (`Pactify` 1.1.0) — the only such arrangement on the platform |
| `src/Pacco.Services.Orders.Api/appsettings.json` | Exchange, outbox with `disableTransactions: true`, Vault PKI, the three-service HTTP map |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. Whether the local `customers` replica is meant to stay a creation-time snapshot.
2. How the Pact contract file travels to `parcels-service`, and whether the tests run anywhere.
3. Why `order_delivering` has no domain consumer.
4. Why `order_for_delivery_not_found` and `order_for_reserved_vehicle_not_found` break the
   `*_rejected` naming convention.
5. Whether order write operations verify caller ownership, given reads do.
6. What happens to an order holding a vehicle that is later deleted — this service does not consume
   `vehicle_deleted`.
7. Whether `outbox.disableTransactions: true` is deliberate.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Orders.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Orders.Application/`,
`src/Pacco.Services.Orders.Core/`, `src/Pacco.Services.Orders.Infrastructure/`, `tests/`, and the
repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor). The only browser-facing surface is the Convey Swagger UI at
`/docs`, generated by `Convey.WebApi.Swagger`.

---

## Evidence

| Fact | File |
|---|---|
| Route table and host composition | `src/Pacco.Services.Orders.Api/Program.cs` |
| DI composition, both Mongo repository registrations, outbox decorators | `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` |
| Order aggregate and status machine | `src/Pacco.Services.Orders.Core/Entities/Order.cs`, `Entities/OrderStatus.cs`, `Entities/Parcel.cs`, `Entities/Customer.cs` |
| Persistence documents, including the customer replica | `src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/OrderDocument.cs`, `CustomerDocument.cs` |
| Synchronous dependencies | `src/Pacco.Services.Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs`, `PricingServiceClient.cs`, `VehiclesServiceClient.cs` |
| Commands | `src/Pacco.Services.Orders.Application/Commands/*.cs` |
| Published events and payloads | `src/Pacco.Services.Orders.Application/Events/*.cs` |
| Rejected events | `src/Pacco.Services.Orders.Application/Events/Rejected/*.cs` |
| Consumed external events | `src/Pacco.Services.Orders.Application/Events/External/Handlers/*.cs` |
| Identity-scoped order list | `src/Pacco.Services.Orders.Application/Queries/Handlers/GetOrdersHandler.cs` |
| Pact consumer contract tests | `tests/Pacco.Services.Orders.PactConsumerTests/`, `tests/Pacco.Services.Orders.PactConsumerTests/PACT/` |
| Exchange, outbox, Vault, JWT, HTTP service map, logging, metrics, tracing | `src/Pacco.Services.Orders.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Package set | `src/Pacco.Services.Orders.Infrastructure/Pacco.Services.Orders.Infrastructure.csproj`, `src/Pacco.Services.Orders.Api/Pacco.Services.Orders.Api.csproj`, `tests/Pacco.Services.Orders.PactConsumerTests/*.csproj` |
| Project list | `Pacco.Services.Orders.sln` |
| Container build and CI, and that tests are not run in CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Pact provider counterpart | `../hianshul100_Pacco.Services.Parcels/tests/Pacco.Services.Parcels.PactProviderTests/` |
| Customer replica counterpart | `../hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Gateway routes and `@user_id` rewriting | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The local `customers` collection is a read-only projection, written only by the `customer_created` handler | The handler is the only writer visible in the repository, and no command touches customer data | If anything else writes it, this service becomes a second source of truth for customer data with no reconciliation against `customers-service` | Read every write path against `CustomerDocument` in this repository |
| A2 | Parcels are embedded in the order document rather than stored separately | Only `orders` and `customers` are registered as repositories, and `Core/Entities/Parcel.cs` appears as part of the order aggregate | Statements about the order consistency boundary would be wrong | Inspect an order document in a running MongoDB instance |
| A3 | The three synchronous dependencies are required for order creation to succeed | Each client is called on the write path, for parcel validation, pricing and vehicle validation | If any call is optional or tolerant of failure, the availability coupling described here is overstated; if none is, an outage in any of the three stops order creation | Read the command handlers' error handling, and test with each dependency down |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should this service consume `customer_state_changed` and `customer_became_vip`? | Its `customers` copy is written once when the customer is created and never updated. Every later change — including VIP promotion — leaves it stale, and nothing detects the drift. If any order decision reads that copy, customers are treated on out-of-date status | Either subscribe to the state events, or state that only the creation snapshot matters here | Domain owners for Orders and Customers |
| Q2 | **[ACTION NOW]** How does the Pact contract reach `parcels-service`, and does either side's test run? | Both repositories have a `PACT/` directory and `Pactify`, but no broker is configured and neither Travis pipeline invokes `scripts/test.sh`. A contract test that never runs, against a file that may be hand-copied, proves nothing about the provider's current behaviour | Introduce a Pact Broker or a shared contract artefact, and wire the tests into CI | Whoever owns the Orders/Parcels contract tests |
| Q3 | **[ACTION NOW]** Do order write operations check that the caller owns the order? | Reads are scoped by `IAppContext`, but the write routes take an order id and have no role gate at the gateway. If writes do not perform the same check, any authenticated user who knows an order id could modify or delete someone else's order | Read each write handler for an ownership check and confirm against a running instance | Whoever owns Pacco authentication |
| Q4 | **[ACTION NOW]** What happens to an order whose assigned vehicle is later deleted? | This service validates the vehicle once at assignment and does not subscribe to `vehicle_deleted`, although `availability-service` does. An order can end up pointing at a vehicle that no longer exists, with nothing to detect it | Subscribe to `vehicle_deleted`, or state why a stale vehicle reference is acceptable | Domain owner for Orders |
| Q5 | **[handled later by HLD]** Should `order_delivering` have a consumer? | It is published on every transition into delivery and nobody listens. If customer notification or tracking were intended, that consumer does not exist | Either add the consumer or record the event as observability-only | Domain owner for Orders |
| Q6 | **[handled later by HLD]** Why do `order_for_delivery_not_found` and `order_for_reserved_vehicle_not_found` break the rejection naming convention? | Every other failure message on the platform ends in `_rejected`. A client filtering on that suffix to detect failures silently misses these two, so the failure looks like silence | Rename them, or document them as a distinct message category | Domain owner for Orders |
| Q7 | **[handled later by HLD]** Is `outbox.disableTransactions: true` the intended setting? | This service publishes 19 message types; without transactions, the order write and the outbox write are not atomic, so a crash between them can leave an order that no downstream service — including the saga driving it — ever hears about | Likely a single-node MongoDB constraint in development; confirm the production topology | Platform architect |
