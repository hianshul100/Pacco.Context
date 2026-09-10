# Repository summary — `hianshul100_Pacco.Services.Orders`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Orders` (also known as: Pacco.Services.Orders, the Orders service)
**Deployable:** `Pacco.Services.Orders.Api` (also known as: `orders-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.orders`, its published image). Repository: `hianshul100_Pacco.Services.Orders`, path: `src/Pacco.Services.Orders.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

The centre of the domain. It owns the order aggregate — which customer placed it, which parcels it contains, which vehicle is assigned, what it costs, and where it is in its lifecycle. It is the busiest integration point in the platform: it consumes seven commands and seven external events, publishes nine events and ten rejected events, and calls three other services synchronously.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, Convey-based, serving HTTP dispatcher endpoints and consuming RabbitMQ messages in the same process. It is both a consumer and a producer in every direction.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Orders.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Orders.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Orders.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Orders.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Dispatches to |
|---|---|---|
| GET | `orders/{orderId}` | `GetOrder` |
| GET | `orders` | `GetOrders` |
| POST | `orders` | `CreateOrder` |
| DELETE | `orders/{orderId}` | `DeleteOrder` |
| POST | `orders/{orderId}/parcels/{parcelId}` | `AddParcelToOrder` |
| DELETE | `orders/{orderId}/parcels/{parcelId}` | `DeleteParcelFromOrder` |
| POST | `orders/{orderId}/vehicles/{vehicleId}` | `AssignVehicleToOrder` |

## 4. Important modules / packages

Four source projects plus one test project, listed in `Pacco.Services.Orders.sln`:

| Project | Path | Role |
|---|---|---|
| `Pacco.Services.Orders.Api` | `src/Pacco.Services.Orders.Api` | host, endpoint registration, configuration |
| `Pacco.Services.Orders.Application` | `src/Pacco.Services.Orders.Application` | commands, events, external events, rejected events, queries, DTOs, service client interfaces |
| `Pacco.Services.Orders.Core` | `src/Pacco.Services.Orders.Core` | `Order`, `Parcel`, `Customer`, `OrderStatus`, `AggregateRoot`, `AggregateId`; domain events `OrderStateChanged`, `ParcelAdded`, `ParcelDeleted` |
| `Pacco.Services.Orders.Infrastructure` | `src/Pacco.Services.Orders.Infrastructure` | Mongo documents and repositories, RabbitMQ broker, HTTP service clients, decorators, logging, exception mappers |
| `Pacco.Services.Orders.PactConsumerTests` | `tests/Pacco.Services.Orders.PactConsumerTests` | **`Pactify 1.1.0`** — `PACT/ParcelsApiPactConsumerTests.cs`, the consumer side of a contract test against `parcels-service` |

Application commands: `CreateOrder`, `ApproveOrder`, `CancelOrder`, `DeleteOrder`, `AddParcelToOrder`, `DeleteParcelFromOrder`, `AssignVehicleToOrder`.
Application events: `OrderCreated`, `OrderApproved`, `OrderCanceled`, `OrderCompleted`, `OrderDeleted`, `OrderDelivering`, `ParcelAddedToOrder`, `ParcelDeletedFromOrder`, `VehicleAssignedToOrder`.
Rejected events: `AddParcelToOrderRejected`, `ApproveOrderRejected`, `AssignVehicleToOrderRejected`, `CancelOrderRejected`, `CompleteOrderRejected`, `CreateOrderRejected`, `DeleteOrderRejected`, `DeleteParcelFromOrderRejected`, `DeliveringOrderRejected`, `OrderForDeliveryNotFound`, `OrderForReservedVehicleNotFound`.
External events consumed: `CustomerCreated`, `DeliveryStarted`, `DeliveryCompleted`, `DeliveryFailed`, `ParcelDeleted`, `ResourceReserved`, `ResourceReservationCanceled`, each with a handler under `Application/Events/External/Handlers/`.

Packages are the platform's standard Convey `0.4.*` set.

## 5. External integrations

MongoDB (database `orders-service`), Redis (instance prefix `orders:`), RabbitMQ (exchange `orders`), Consul (service `orders-service`, port 5006, ping endpoint `ping`), Fabio (`http://localhost:9999`), Vault (kv v2 mount `kv`, path `orders-service/settings`; PKI common name `orders.pacco.io`; dynamic MongoDB credentials with auto-renewal), Jaeger (service name `orders`, UDP `localhost:6831`, `sampler: const`), Prometheus, Seq.

`httpClient.services` declares three downstream services: `parcels: parcels-service`, `pricing: pricing-service`, `vehicles: vehicles-service`. This is the widest synchronous fan-out in the platform.

## 6. Data stores and state handling

**Store:** MongoDB, database `orders-service`. **Query mechanism:** Convey MongoDB repositories over the MongoDB .NET driver. **No ORM. No migration tool.**

| Collection | Registered at | Fields |
|---|---|---|
| `orders` | `AddMongoRepository<OrderDocument, Guid>("orders")` | `Id` (Guid), `CustomerId` (Guid), `VehicleId` (nullable Guid), `Status` (enum `OrderStatus`), `CreatedAt` (DateTime), `DeliveryDate` (nullable DateTime), `TotalPrice` (decimal), `Parcels` (embedded collection) |
| embedded parcel | inside `OrderDocument.cs` | `Id` (Guid), `Name` (string), `Variant` (string), `Size` (string) |
| `customers` | `AddMongoRepository<CustomerDocument, Guid>("customers")` | `Id` (Guid) only |
| `inbox` | outbox settings | Convey inbox, de-duplication |
| `outbox` | outbox settings | Convey outbox, pending publishes |

Outbox: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Cross-domain coupling — this is the most coupled data model in the platform.**

- `orders-service` keeps its own `customers` collection: a replicated read model of a domain owned by `customers-service`, populated by the `CustomerCreated` handler. It stores nothing but the identifier, so its only purpose is to answer "does this customer exist" without a network call. `parcels-service` has the identical pattern.
- `OrderDocument.CustomerId` points at that replicated customer.
- `OrderDocument.VehicleId` points at a vehicle owned by `vehicles-service`, with no local copy at all.
- The embedded parcel entries are a **denormalised snapshot** of parcels owned by `parcels-service` — name, variant and size are copied in at the time the parcel is added. If a parcel is renamed afterwards, the order keeps the stale values, and nothing reconciles them.
- `TotalPrice` is a value computed by `pricing-service` at a moment in time and then frozen into the order document.

None of these are database foreign keys. MongoDB enforces nothing; the consistency is entirely in event handlers, and there is no reconciliation job anywhere in the platform.

Redis is registered but no cache call appears in this service's own code, so what it holds is **unknown — requires runtime capture**.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Mongo-backed transactional outbox and the Jaeger RabbitMQ plugin.

**Broker settings:** exchange `orders`, `conventionsCasing: snakeCase`, queue name template `orders-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `orders`:**

- Events: `order_created`, `order_approved`, `order_canceled`, `order_completed`, `order_deleted`, `order_delivering`, `parcel_added_to_order`, `parcel_deleted_from_order`, `vehicle_assigned_to_order`
- Rejected events: `add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`, `cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`, `delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found`
- Commands accepted on the same exchange: `create_order`, `approve_order`, `cancel_order`, `delete_order`, `add_parcel_to_order`, `delete_parcel_from_order`, `assign_vehicle_to_order`

Observable payload key fields:

- `OrderCreated { Guid OrderId }`
- `OrderCompleted { Guid OrderId, Guid CustomerId }`
- `ParcelAddedToOrder { Guid OrderId, Guid ParcelId }`
- `CreateOrderRejected : IRejectedEvent { Guid CustomerId, string Reason, string Code }`

The remaining event payloads follow the same identifier-carrying shape; complete field lists are **unknown — requires runtime capture**.

**Consumed:**

| Kind | Message | Origin exchange | Declared in |
|---|---|---|---|
| Command | `CreateOrder`, `ApproveOrder`, `CancelOrder`, `DeleteOrder`, `AddParcelToOrder`, `DeleteParcelFromOrder`, `AssignVehicleToOrder` | `orders` | own commands |
| Event | `CustomerCreated` | `customers` | `Events/External/CustomerCreated.cs`, `[Message("customers")]` |
| Event | `DeliveryStarted`, `DeliveryCompleted`, `DeliveryFailed` | `deliveries` | `Events/External/*.cs`, `[Message("deliveries")]` |
| Event | `ParcelDeleted` | **declared as `deliveries`** | `Events/External/ParcelDeleted.cs`, `[Message("deliveries")]` |
| Event | `ResourceReserved`, `ResourceReservationCanceled` | `availability` | `Events/External/*.cs`, `[Message("availability")]` |

**Probable defect — needs validation.** `Application/Events/External/ParcelDeleted.cs` is annotated `[Message("deliveries")]`, but the publisher of `parcel_deleted` is `parcels-service`, which publishes on exchange `parcels` (per the platform catalogue and per that service's own broker settings). The subscription therefore binds a queue to the wrong exchange and the handler `ParcelDeletedHandler` never runs. The consequence is that deleting a parcel does not remove it from any order it belongs to. This is the highest-impact finding in this inventory and is recorded as a blocker below.

`GetHeadersToForward` forwards the `Saga` header, which is how the OrderMaker saga correlates the messages it sends here.

## 8. APIs exposed and consumed

**Exposed:** the seven HTTP routes in section 3, Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5006`; port 80 in the container.

Through the gateway, `GET /orders` is rewritten to `orders-service/orders?customerId=@user_id`, so a caller only ever sees their own orders. All seven gateway order routes bind `customerId:@user_id` where the operation needs it.

**Consumed** — three synchronous HTTP clients, all in `Infrastructure/Services/Clients/`, all resolved through Fabio:

| Client | Call |
|---|---|
| `ParcelsServiceClient` | `GET {parcels-service}/parcels/{parcelId}` |
| `PricingServiceClient` | `GET {pricing-service}/pricing?customerId={customerId}&orderPrice={orderPrice}` |
| `VehiclesServiceClient` | `GET {vehicles-service}/vehicles/{vehicleId}` |

`httpClient.retries: 3`. There is no circuit breaker and no timeout configured beyond the Convey defaults, so a slow downstream service stalls order creation.

The Pact consumer test in `tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs` pins the `parcels-service` contract from this side; `hianshul100_Pacco.Services.Parcels` holds the matching provider test. **Only one of the three synchronous dependencies is covered by a contract test** — pricing and vehicles are not.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Orders.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `orders-service` on host port 5006.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.orders`.

## 10. Security and auth clues

- JWT validated against the committed `certs/localhost.cer`, `validIssuer: pacco`.
- `Convey.Security` is composed; no certificate authentication here.
- Ownership is enforced by the gateway rewriting `customerId` from the token, not by the service. Anything that can reach `orders-service` on port 5006 can read or modify any order for any customer.
- `logger.excludeProperties` keeps secret-like values out of logs; `httpClient.requestMasking` is configured.
- Vault supplies settings and dynamic MongoDB credentials; RabbitMQ credentials remain the plain defaults in `appsettings.json`.

## 11. Observability, logging and tracing clues

Jaeger under service name `orders`, with the RabbitMQ plugin continuing traces across message hops — important here because a single order touches five other services. Prometheus metrics via `Convey.Metrics.AppMetrics` (`metrics.database: pacco`, interval 5). Structured logs to console, file and Seq with `/`, `/ping` and `/metrics` excluded. `Infrastructure/Logging/MessageToLogTemplateMapper.cs` is the largest in the platform — it maps every one of the roughly thirty message types to its own log template.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Orders.Infrastructure/Extensions.cs` (technology choices, all fourteen subscriptions, the exchange bindings), `src/Pacco.Services.Orders.Api/Program.cs` (HTTP surface), `src/Pacco.Services.Orders.Api/appsettings.json` (integration endpoints and the three downstream services), `src/Pacco.Services.Orders.Core/Entities/Order.cs` and `OrderStatus.cs` (the order lifecycle, which is the platform's central state machine), `src/Pacco.Services.Orders.Infrastructure/Services/Clients/` (the synchronous dependency set).

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Orders.Api/`, `src/Pacco.Services.Orders.Application/`, `src/Pacco.Services.Orders.Core/`, `src/Pacco.Services.Orders.Infrastructure/`, `tests/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5006`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.orders`; `Pacco.Services.Orders.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the fourteen message subscriptions and nineteen published messages; the three synchronous service dependencies; the Pact contract test; the replicated `customers` read model; the denormalised parcel snapshot; the Vault integration; the transactional outbox. The README for the platform's most connected service describes only how to start it.

**Conflicts to surface:**

- **Stale doc.** The README instructs running `dotnet run` in `/src/Pacco.Services.Orders`. That directory does not exist — the host project is `src/Pacco.Services.Orders.Api`. `scripts/start.sh` has the correct path.
- **Code conflict — probable defect.** `ParcelDeleted` is bound to exchange `deliveries` in this repository while `parcels-service` publishes it on exchange `parcels`. Recorded as B1 below.
- **Catalogue conflict.** `Application/Events/Rejected/CompleteOrderRejected.cs` exists on disk but is not listed for `orders-service` in `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. The Operations dashboard therefore cannot surface it to the user.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`. The only test project is the Pact consumer test, so the build's test step exercises one contract and nothing else — no unit, integration or end-to-end test exists for the platform's central aggregate.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next. B1 is a suspected live defect in the running platform, not a documentation issue.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The replicated `customers` collection exists only to check that a customer is known, not to hold customer data. | The document has an `Id` field and nothing else. |
| A2 | `TotalPrice` is captured once at order creation and not recalculated. | It is a plain decimal on the document, and `PricingServiceClient` is called with an order price rather than being consulted on read. |
| A3 | The embedded parcel snapshot is intentional denormalisation for read performance. | The order query handlers read it directly rather than calling `parcels-service`. |
| A4 | `messages.json` in the Operations repository is meant to be the complete catalogue of this service's messages. | Operations builds live subscriptions from it at start-up. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** `Application/Events/External/ParcelDeleted.cs` is annotated `[Message("deliveries")]`, but `parcels-service` publishes `parcel_deleted` on exchange `parcels`. | The handler almost certainly never runs. Deleting a parcel leaves it attached to its order, so orders reference parcels that no longer exist. Needs validation against a running broker before it is treated as confirmed. | Orders service maintainer: confirm against the broker, then correct the annotation to `parcels`. |
| B2 | **[ACTION NOW]** Two of the three synchronous dependencies (pricing, vehicles) have no contract test, and there is no circuit breaker or explicit timeout on any of them. | A slow or changed downstream service breaks order creation with no early warning. | Orders service maintainer. |
| B3 | **[ACTION NOW]** Order ownership is enforced only at the gateway. | Anything on the internal network can read or modify any customer's orders. | Orders service maintainer, with the security-architecture owner. |
| B4 | **[ACTION NOW]** The platform's central aggregate has no unit, integration or end-to-end tests. | Changes to the order state machine ship with no safety net. | Orders service maintainer. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Should `complete_order_rejected` be added to the platform message catalogue? | It is published but invisible to the user-facing Operations feed. | Orders service maintainer. |
| Q2 | **[handled later by the domain-model stage]** What are the states of `OrderStatus` and the allowed transitions between them? | This is the platform's central state machine and it is documented nowhere. | Domain-model stage. |
| Q3 | **[handled later by the data-architecture stage]** What reconciles the denormalised parcel snapshot when a parcel changes? | Nothing found in source; orders may show stale parcel details indefinitely. | Data-architecture stage. |
| Q4 | **[handled later by the API-contract stage]** What are the full payloads of the nine published events and ten rejected events? | Six other components depend on them and only four payloads are asserted in source. | API-contract stage. |
| Q5 | **[handled later by the data-architecture stage]** What is stored in Redis under the `orders:` prefix? | Not answerable from source; determines whether Redis is a hard dependency here. | Data-architecture stage, by inspecting a running instance. |
