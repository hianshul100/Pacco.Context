---
title: "Repository Summary — Pacco.Services.Orders"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Orders"
status: "evidence-based inventory"
---

# Pacco.Services.Orders

**Primary name:** `Pacco.Services.Orders` (aliases used in this file: `orders-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `orders` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Orders`, path: `hianshul100_Pacco.Services.Orders/`

---

## 1. Primary purpose

The centre of the platform's domain. It owns the order aggregate and its lifecycle, links parcels and a vehicle to an order, prices the order, and reacts to reservation and delivery outcomes. It has by far the largest message surface of any service.

Evidence: `src/Pacco.Services.Orders.Core/Entities/Order.cs`, `OrderStatus.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service using Convey dispatcher endpoints, plus a RabbitMQ subscriber. Listens on `5006`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` | `src/Pacco.Services.Orders.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

Four source projects plus one test project: `Pacco.Services.Orders.Api`, `.Application`, `.Core`, `.Infrastructure`, and `tests/Pacco.Services.Orders.PactConsumerTests`.

- **Core:** `Entities/Order.cs`, `OrderStatus.cs`, `Parcel.cs`, `Customer.cs`, `AggregateId.cs`, `AggregateRoot.cs`; domain events `OrderStateChanged`, `ParcelAdded`, `ParcelDeleted`; `Repositories/ICustomerRepository.cs`, `Repositories/IParcelRepository.cs`; exceptions `CannotChangeOrderPriceException`, `CannotChangeOrderStateException`, `CannotDeleteOrderException`, `InvalidOrderPriceException`, `OrderParcelNotFoundException`, `ParcelAlreadyAddedToOrderException`.
- **Application:** commands `AddParcelToOrder`, `ApproveOrder`, `AssignVehicleToOrder`, `CancelOrder`, `CreateOrder`, `DeleteOrder`, `DeleteParcelFromOrder` with handlers; nine integration events; seven external events with handlers; ten rejected events; service client contracts `IParcelsServiceClient`, `IPricingServiceClient`, `IVehiclesServiceClient`.
- **Infrastructure:** `Mongo/Documents/OrderDocument.cs`, `Mongo/Documents/CustomerDocument.cs`, `Mongo/Repositories/OrderMongoRepository.cs`, `CustomerMongoRepository.cs`, `Mongo/Queries/Handlers/GetOrderHandler.cs`, `GetOrdersHandler.cs`, `Services/Clients/ParcelsServiceClient.cs`, `PricingServiceClient.cs`, `VehiclesServiceClient.cs`, plus the shared outbox decorators, event mapper, message broker and exception mappers.
- **Tests:** `PACT/ParcelsApiPactConsumerTests.cs` using `Pactify 1.1.0` — the **consumer** side of a consumer-driven contract with `Pacco.Services.Parcels`.

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus, and three services over HTTP: `httpClient.services: {parcels: parcels-service, pricing: pricing-service, vehicles: vehicles-service}` with `httpClient.type: fabio`.

## 6. Data stores / state

- **Store:** MongoDB, database `orders-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with explicit document classes and hand-written repositories.
- **Collections:** an orders collection from `OrderDocument` and a customers collection from `CustomerDocument`, plus `inbox` and `outbox`.
- **Migration tool:** none anywhere in the repository.
- **Cross-domain coupling:** this service keeps its **own copy of the customer**, `Mongo/Documents/CustomerDocument.cs`, populated from the `customer_created` event published by `Pacco.Services.Customers`. `Pacco.Services.Parcels` keeps a second, independent copy. There are no foreign keys — a MongoDB database has none — so the same customer exists as three separate documents in three databases, kept in step only by events. This is the platform's main data-consistency risk and its main reason for service independence.
- The order document also embeds parcel entries and holds a vehicle identifier owned by `Pacco.Services.Vehicles`.
- **Cache:** Redis.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `orders`, snake-case naming, queue template `orders-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`. Transactional outbox and inbox on MongoDB (`inbox`, `outbox`).

**Commands consumed:** `add_parcel_to_order`, `approve_order`, `assign_vehicle_to_order`, `cancel_order`, `create_order`, `delete_order`, `delete_parcel_from_order`.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `order_created` | `Application/Events/OrderCreated.cs` | `OrderId` (Guid) — read directly from the class |
| `order_approved` | `Application/Events/OrderApproved.cs` | order identifier |
| `order_canceled` | `Application/Events/OrderCanceled.cs` | order identifier, reason |
| `order_completed` | `Application/Events/OrderCompleted.cs` | order identifier, customer identifier |
| `order_deleted` | `Application/Events/OrderDeleted.cs` | order identifier |
| `order_delivering` | `Application/Events/OrderDelivering.cs` | order identifier |
| `parcel_added_to_order` | `Application/Events/ParcelAddedToOrder.cs` | order identifier, parcel identifier |
| `parcel_deleted_from_order` | `Application/Events/ParcelDeletedFromOrder.cs` | order identifier, parcel identifier |
| `vehicle_assigned_to_order` | `Application/Events/VehicleAssignedToOrder.cs` | order identifier, vehicle identifier |

**Rejected events published** (ten): `add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`, `cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`, `delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found`. The last two are named as findings rather than as rejections, which makes them read as domain notifications.

**External events consumed** (seven, each with a handler in `Application/Events/External/Handlers/`): `customer_created`, `delivery_completed`, `delivery_failed`, `delivery_started`, `parcel_deleted`, `resource_reservation_canceled`, `resource_reserved`.

This is the platform's busiest node: it consumes from `customers`, `deliveries`, `parcels` and `availability`, and the `ordermaker` saga drives it by publishing four of its commands.

Wire names are confirmed against `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `orders/{orderId}` | `GetOrder` |
| `GET` | `orders` | `GetOrders` |
| `POST` | `orders` | `CreateOrder`, responds `Created` |
| `DELETE` | `orders/{orderId}` | `DeleteOrder` |
| `POST` | `orders/{orderId}/parcels/{parcelId}` | `AddParcelToOrder` |
| `DELETE` | `orders/{orderId}/parcels/{parcelId}` | `DeleteParcelFromOrder` |
| `POST` | `orders/{orderId}/vehicles/{vehicleId}` | `AssignVehicleToOrder` |

Consumed: `parcels-service`, `pricing-service` and `vehicles-service` through the three clients in `Infrastructure/Services/Clients/`.

Called by: `Pacco.APIGateway` (module `orders`; the list route is rewritten to `orders-service/orders?customerId=@user_id`, so a signed-in user only ever sees their own orders).

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.orders`, published `5006:80` per the platform port map, `restart: unless-stopped`, network `pacco`. Consul registration on port `5006`.

CI: `.travis.yml` runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` on success — and here the test step has something to run, namely the contract tests.

## 10. Security / auth clues

- JWT validation following the platform pattern, `validIssuer: pacco`.
- Vault: KV path `orders-service/settings`, PKI role for `orders-service`.
- The gateway rewrites the order list route to filter by the signed-in user's identifier, so ownership is enforced at the edge rather than inside the service. A caller reaching this service directly on the platform network could list any customer's orders. **Needs validation.**
- This service is **not** listed in the caller access-control list defined by `Pacco.Services.Customers`, yet it consumes customer data by event rather than by call, so no entry is required.

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: orders`, including RabbitMQ span propagation; structured logging via `Convey.Logging` and `Convey.Logging.CQRS` with a message-to-log-template mapper; Prometheus metrics via `Convey.Metrics.AppMetrics`.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Orders.Core/Entities/Order.cs` and `OrderStatus.cs` — the order state machine, guarded by `CannotChangeOrderStateException` and `CannotChangeOrderPriceException`.
- `src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs` — the decision to replicate customer data instead of calling for it.
- `src/Pacco.Services.Orders.Application/Events/External/Handlers/` — the seven reactions that make this service the hub of the platform.
- `src/Pacco.Services.Orders.Infrastructure/Services/Clients/PricingServiceClient.cs` — the decision to price an order by a synchronous call rather than by an event.
- `tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs` — the consumer-driven contract testing decision, applied to exactly one of the three service dependencies.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json`, which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` and `tests/` contain only C# projects. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes an orders service on .NET Core 3.1 built with Convey, following clean architecture | Confirmed: four layered projects, Convey `0.4.*`, `netcoreapp3.1` | Confirmed |
| The platform README presents services as independent with their own data | Confirmed, and the cost is visible: the same customer is stored separately here, in the parcels service and in the customers service | Confirmed |
| The platform README describes an event-driven design | Mostly true here, but pricing is fetched by a direct synchronous call, and parcels and vehicles are queried directly as well | Needs validation — this service has both event and call dependencies on the same neighbours |
| Contract testing protects service boundaries | Only the boundary with the parcels service is covered; the pricing and vehicles boundaries have no contract tests | Stale doc |
| The message catalogue lists ten rejection messages for this service | Two of them are named as findings (`order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found`) rather than as rejections | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the contract test project, the order state machine rules, and the two finding-style messages — present in code, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The copy of customer data held here is kept up to date only by messages from the customers service. | The only place it is written is the handler for the customer-created message; there is no direct call to fetch a customer. |
| A2 | Restricting a customer to their own orders is done by the gateway, not by this service. | The filter is written into the gateway routing file, and no equivalent check appears in the service code inspected. |
| A3 | This service is the platform's central domain, so changes here affect the most other services. | It publishes nine events, accepts seven commands, reacts to seven other services' events and calls three services directly. |

### Blockers

_None._

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | What happens if the customer copy stored here falls behind the customers service? Nothing in the code repairs it. | **[handled later by the ADR authoring stage]** Record how duplicated customer data is kept correct over time. |
| Q2 | Can a caller inside the platform network list another customer's orders by calling this service directly? | **[ACTION NOW]** Confirm with the requesting team, since ownership checking currently sits only at the edge. |
| Q3 | Should the pricing, parcels and vehicles boundaries also have contract tests, or is the parcels one a sample? | **[handled later by the ADR authoring stage]** Record the intended testing scope for service boundaries. |
| Q4 | Two rejection messages are named as findings rather than failures. Are they errors or ordinary outcomes? | **[handled later by the ADR authoring stage]** Record what a client should do when it receives them. |
