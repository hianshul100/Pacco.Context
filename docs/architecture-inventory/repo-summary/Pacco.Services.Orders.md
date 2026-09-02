# Repository summary — `Pacco.Services.Orders`

**Repository:** `Pacco.Services.Orders` (workspace clone path: `hianshul100_Pacco.Services.Orders`)
**Deployable:** `orders-service` (also known as: Orders Service, `Pacco.Services.Orders.Api`, image `devmentors/pacco.services.orders`). **Repository: `Pacco.Services.Orders`, path: `src/Pacco.Services.Orders.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Orders
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Owns the **order aggregate and the order lifecycle** — the hub of the platform. An order collects parcels, has a vehicle assigned, is priced, gets approved when its resource reservation succeeds, and then follows the delivery through to completion or cancellation. It is the most connected service in Pacco: three outbound HTTP clients, seven event subscriptions, seven command subscriptions, and nine published events.

**Evidence:** `src/Pacco.Services.Orders.Core/Entities/Order.cs`, `src/Pacco.Services.Orders.Api/Program.cs`, `src/Pacco.Services.Orders.Infrastructure/Extensions.cs`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice plus RabbitMQ consumer, in one process, using the canonical four-project clean-architecture layering (`.Api`, `.Application`, `.Core`, `.Infrastructure`) on Convey.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Orders.Api/Program.cs` — `AddConvey().AddWebApi().AddApplication().AddInfrastructure()`, then `UseInfrastructure()` + `UseDispatcherEndpoints(...)` |
| RabbitMQ subscriptions | `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` → `UseInfrastructure` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Orders.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Orders.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Orders.Api` | Host, endpoint map, `appsettings.json` |
| `src/Pacco.Services.Orders.Application` | `Commands/` (`CreateOrder`, `ApproveOrder`, `CancelOrder`, `DeleteOrder`, `AddParcelToOrder`, `DeleteParcelFromOrder`, `AssignVehicleToOrder`) + handlers; `Queries/` (`GetOrder`, `GetOrders`) + handlers; `Events/`, `Events/External/`, `Events/Rejected/`; `DTO/`; `Services/Clients/` (`IParcelsServiceClient`, `IPricingServiceClient`, `IVehiclesServiceClient`) |
| `src/Pacco.Services.Orders.Core` | `Entities/Order.cs`, `Entities/Parcel.cs`, `Entities/Customer.cs`, `Entities/AggregateRoot.cs`, `Events/`, `Exceptions/`, `Repositories/IOrderRepository`, `ICustomerRepository` |
| `src/Pacco.Services.Orders.Infrastructure` | `Mongo/Documents/OrderDocument.cs`, `CustomerDocument.cs`, `Mongo/Repositories/`, `Services/ParcelsServiceClient.cs`, `PricingServiceClient.cs`, `VehiclesServiceClient.cs`, `Decorators/` (outbox), `Contexts/`, `Exceptions/ExceptionToResponseMapper.cs` + `ExceptionToMessageMapper.cs`, `Logging/`, `Extensions.cs` |
| `tests/Pacco.Services.Orders.PactConsumerTests` | **Pactify 1.1.0** consumer-side contract tests — `PACT/ParcelsApiPactConsumerTests.cs` |

**Contract testing:** this repository is the **consumer** side of the platform's single Pact pair; `Pacco.Services.Parcels` is the provider. The contract covers `GET /parcels/{parcelId}` only. The other five cross-service HTTP calls in the platform have no contract tests.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| `parcels-service` | outbound HTTP | `GET {parcels-service}/parcels/{id}` → `ParcelDto` (`Infrastructure/Services/ParcelsServiceClient.cs`) |
| `pricing-service` | outbound HTTP | `GET {pricing-service}/pricing?customerId={customerId}&orderPrice={orderPrice}` → `OrderPricingDto` (`Infrastructure/Services/PricingServiceClient.cs`) |
| `vehicles-service` | outbound HTTP | `GET {vehicles-service}/vehicles/{id}` → `VehicleDto` (`Infrastructure/Services/VehiclesServiceClient.cs`) |
| RabbitMQ | in + out | exchange `orders`, topic |
| MongoDB | out | database `orders-service` |
| Redis | out | instance prefix `orders:` |
| Consul | out | registers `orders-service` on port `5006` |
| Fabio | out | `http://localhost:9999`, `httpClient.type: "fabio"` |
| Vault | out | KV v2 `kv/orders-service/settings`; PKI role `orders-service`, CN `orders-service.pacco.io`; dynamic Mongo credentials |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

`appsettings.json` → `httpClient.services`: `parcels: parcels-service`, `pricing: pricing-service`, `vehicles: vehicles-service`. **Three outbound service dependencies — the most of any service in the platform.** None of the three calls carries a caller credential (contrast `availability-service` → `customers-service`, which presents a certificate).

**No payment integration.** Orders are priced by `pricing-service` but nothing charges anyone: there is no payment gateway, no invoicing, no settlement, and no `payments` service. For an order aggregate that computes a total price, this is a significant functional absence.

## 6. Data stores / state handling

- **Store:** MongoDB, database `orders-service`.
- **Collections — two domain collections:**
  - `orders` — `AddMongoRepository<OrderDocument, Guid>("orders")`
  - **`customers`** — `AddMongoRepository<CustomerDocument, Guid>("customers")`
  - plus `inbox` and `outbox`.
- **Access mechanism:** Convey `IMongoRepository<>` over `MongoDB.Driver`. **No ORM.**
- **Migration tool: none.** No Flyway, Liquibase, Alembic, or EF Core migrations.
- **Document shapes:**
  - `OrderDocument` — order id, `CustomerId`, status, the embedded collection of parcels, the assigned `VehicleId`, the total price, delivery date, timestamps.
  - `CustomerDocument` — **`public Guid Id { get; set; }` and nothing else.** An id-only replica.
- **Cross-domain coupling — the platform's clearest example.** This service maintains a **local `customers` collection replicating customer identity owned by `customers-service`**, populated by the `customer_created` event. `parcels-service` does exactly the same, with an identically empty `CustomerDocument`. There is no database foreign key (MongoDB, separate logical databases, no relational constraints), but there is a hard operational dependency: if `customer_created` is missed, this service does not recognise the customer and rejects their orders. The replica stores no customer attributes — only the fact that the id is known — so the coupling is an existence check, not a data copy.
  The order document also holds a `VehicleId` and parcel identifiers owned by other services, again as logical references with no enforcement.
- **Outbox:** enabled, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: inbox`, `outboxCollection: outbox`, `disableTransactions: true`.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `orders`; `conventionsCasing: snakeCase`; queue template `orders-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — commands (seven, the most in the platform):**

| Message | Wire name | Key payload fields |
|---|---|---|
| `CreateOrder` | `create_order` | `OrderId`, `CustomerId` |
| `ApproveOrder` | `approve_order` | `OrderId` |
| `CancelOrder` | `cancel_order` | `OrderId`, `Reason` |
| `DeleteOrder` | `delete_order` | `OrderId` |
| `AddParcelToOrder` | `add_parcel_to_order` | `OrderId`, `ParcelId`, `CustomerId` |
| `DeleteParcelFromOrder` | `delete_parcel_from_order` | `OrderId`, `ParcelId` |
| `AssignVehicleToOrder` | `assign_vehicle_to_order` | `OrderId`, `VehicleId`, `DeliveryDate` |

**Consumed — external events (seven, the most in the platform):**

| Message | Wire name | Origin | Effect |
|---|---|---|---|
| `CustomerCreated` | `customer_created` | `customers-service` | records the customer in the local `customers` replica |
| `ParcelDeleted` | `parcel_deleted` | `parcels-service` | removes the parcel from any order holding it |
| `ResourceReserved` | `resource_reserved` | `availability-service` | **approves the order** — the reservation succeeding is what makes an order valid |
| `ResourceReservationCanceled` | `resource_reservation_canceled` | `availability-service` | cancels the order whose reservation was lost |
| `DeliveryStarted` | `delivery_started` | `deliveries-service` | moves the order to delivering |
| `DeliveryCompleted` | `delivery_completed` | `deliveries-service` | completes the order |
| `DeliveryFailed` | `delivery_failed` | `deliveries-service` | fails/cancels the order |

**Published — events (nine, the most in the platform):**

| Event | Wire name | Key payload fields |
|---|---|---|
| `OrderCreated` | `order_created` | `OrderId` |
| `OrderApproved` | `order_approved` | `OrderId` |
| `OrderCanceled` | `order_canceled` | `OrderId`, `Reason` |
| `OrderCompleted` | `order_completed` | `OrderId`, `CustomerId` |
| `OrderDeleted` | `order_deleted` | `OrderId` |
| `OrderDelivering` | `order_delivering` | `OrderId` |
| `ParcelAddedToOrder` | `parcel_added_to_order` | `OrderId`, `ParcelId` |
| `ParcelDeletedFromOrder` | `parcel_deleted_from_order` | `OrderId`, `ParcelId` |
| `VehicleAssignedToOrder` | `vehicle_assigned_to_order` | `OrderId`, `VehicleId` |

**Published — rejection events (ten, the most in the platform):** `create_order_rejected`, `approve_order_rejected`, `cancel_order_rejected`, `delete_order_rejected`, `add_parcel_to_order_rejected`, `delete_parcel_from_order_rejected`, `assign_vehicle_to_order_rejected`, `delivering_order_rejected`, plus two that are not `*_rejected`-suffixed but serve the same purpose: **`order_for_delivery_not_found`** and **`order_for_reserved_vehicle_not_found`** (key field `VehicleId`). The last two are emitted when an inbound event references an order this service does not have — the platform's only explicit signal that two services' views have diverged.

**Downstream consumers:** `customers-service` (`order_completed` → VIP evaluation), `parcels-service` (`order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order`), `ordermaker-service` (`order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved`), `operations-service` (everything).

**Reliability:** outbox/inbox decorators wrap every command and event handler. The `Saga` header is forwarded, so the `AIOrderMakingSaga` in `ordermaker-service` stays correlated.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5006`, container port `80`):

| Method | Path | Maps to | Gateway exposure |
|---|---|---|---|
| GET | `orders` | `GetOrders` | `/orders` → `orders-service/orders?customerId=@user_id` — scoped to the caller |
| GET | `orders/{orderId}` | `GetOrder` | `/orders/{orderId}` |
| POST | `orders` | `CreateOrder` | `/orders` — gateway generates `orderId`, binds `customerId: @user_id` |
| DELETE | `orders/{orderId}` | `DeleteOrder` | `/orders/{orderId}` |
| POST | `orders/{orderId}/parcels/{parcelId}` | `AddParcelToOrder` | same |
| DELETE | `orders/{orderId}/parcels/{parcelId}` | `DeleteParcelFromOrder` | same |
| POST | `orders/{orderId}/vehicles/{vehicleId}` | `AssignVehicleToOrder` | same |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**Note:** `ApproveOrder` has **no HTTP endpoint** — approval is reachable only as a broker command, in practice triggered by the `resource_reserved` event. Approval is deliberately not a user action.

**Consumed:** `GET /parcels/{id}` (Parcels), `GET /pricing?customerId=&orderPrice=` (Pricing), `GET /vehicles/{id}` (Vehicles).

**Called by:** nothing over HTTP — `ordermaker-service` interacts with this service purely through the broker.

**Ownership scoping:** `GET /orders` is scoped to `@user_id` at the gateway and `POST /orders` binds `customerId` from the token, so a caller cannot create an order for someone else. However `GET /orders/{orderId}`, `DELETE /orders/{orderId}`, and the three parcel/vehicle routes are addressed **by order id with no ownership binding at the gateway**. Whether the service checks that the order belongs to the caller is **Needs validation**.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Orders.Api.dll`.
- Composed as `orders-service` on `5006:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5006`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- Whether `./scripts/build.sh` runs the Pact consumer tests, and whether the resulting pact is published anywhere for the provider to verify, is **Unknown** — no broker URL or publish step appears in the repository.

## 10. Security/auth clues

- **JWT bearer** validation via `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- `.AddSecurity()` is registered, but there is **no `security.certificate` block** — this service neither presents nor verifies client certificates, so its three outbound calls are unauthenticated.
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- Log redaction via `logger.excludeProperties`.
- **Authorisation is enforced only at the gateway**, and only partially: `@user_id` binding protects order *creation* and *listing*, but per-order routes are not ownership-bound at the gateway (§8). Direct access to port `5006` bypasses all of it.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: orders-service`, UDP `localhost:6831`, `sampler: const`) with the RabbitMQ Jaeger plugin, so broker hops stay in the trace. Given this service's three synchronous dependencies and fourteen inbound message types, it is the most valuable trace source in the platform.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`. Handler logging via `.AddHandlersLogging()`.
- **Correlation:** `Correlation-Context` header; `Saga` header forwarded.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics — no counters for orders created, approved, cancelled, or for the two `*_not_found` divergence events, which are exactly the signals worth alerting on.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Orders.Core/Entities/Order.cs` | The order state machine — the platform's central lifecycle — and its invariants |
| `src/Pacco.Services.Orders.Infrastructure/Extensions.cs` | Composition, and the subscription set: seven commands and seven external events, making this the platform's integration hub |
| `src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs` | The decision to replicate customer identity locally as an id-only document rather than call `customers-service` |
| `src/Pacco.Services.Orders.Infrastructure/Services/PricingServiceClient.cs` | Pricing is fetched synchronously at order time rather than subscribed to |
| `src/Pacco.Services.Orders.Application/Events/Rejected/` | Including `OrderForDeliveryNotFound` and `OrderForReservedVehicleNotFound` — the platform's only explicit divergence signals |
| `tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs` | The decision to contract-test one of six cross-service calls |
| `src/Pacco.Services.Orders.Api/appsettings.json` | Three HTTP service dependencies; outbox with `disableTransactions: true` |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`). No order-lifecycle behaviour is gated.

## 13. Open questions / ambiguities

- **Ownership checks on per-order routes.** `GET`/`DELETE /orders/{orderId}` and the parcel/vehicle routes are not ownership-bound at the gateway. **Needs validation** that the handlers check `CustomerId`.
- **No payment.** An order carries a total price and nothing collects it. Whether payment is out of scope or future is **Unknown**.
- **Three synchronous dependencies at order time** (Parcels, Pricing, Vehicles) with no fallback, cache, or circuit breaker visible beyond Convey's two HTTP retries. Behaviour when any is down is **Needs validation**.
- **The two `*_not_found` events have no subscriber** in the workspace — only `operations-service` records them, and nothing alerts. **Needs validation.**
- **Pact publication is unproven.** A consumer contract exists in this repository and a provider verification exists in `Pacco.Services.Parcels`, but no pact broker or publish step was found, so it is **Unknown** whether the two are actually exchanged rather than kept in step by hand.
- **`disableTransactions: true`** on the outbox, as everywhere else — order state and its outgoing events are not written atomically.
- The order status vocabulary was read from `Order.cs` but not exhaustively verified against every handler. **Needs validation.**

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (four C# projects only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Orders service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5006`. — **Confirmed** (`appsettings.json` `consul.port: 5006`, `Pacco/compose/services.yml` `5006:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Orders` directory"**; the actual host project is **`/src/Pacco.Services.Orders.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **The order lifecycle itself** — the platform's central state machine, its seven commands, seven event subscriptions, and nine published events. Entirely undocumented.
- **The Pact consumer tests**, and the fact that this repository is one half of the platform's only contract-testing pair.
- **The three synchronous service dependencies** (Parcels, Pricing, Vehicles) and what happens to order creation if any is unavailable.
- **The local `customers` replica collection** and its dependence on the `customer_created` event.
- **The two divergence events** `order_for_delivery_not_found` and `order_for_reserved_vehicle_not_found`.
- That `ApproveOrder` has no HTTP endpoint and is driven only by `resource_reserved`.
- The transactional outbox/inbox and the handler decorators; `scripts/`.

**Unknown (neither pass yielded proof):**
- Whether the Pact contract is published and verified automatically or maintained by hand.
- Whether order handlers enforce customer ownership.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | An order becomes approved when `resource_reserved` arrives, and this is the only path to approval. | `ApproveOrder` has no HTTP route, and the service subscribes to `resource_reserved` from `availability-service`. | The order lifecycle diagram would be wrong at its most important transition, and downstream design work would model approval as a user action. | Read `ResourceReservedHandler` and confirm it dispatches `ApproveOrder`. |
| A2 | The local `customers` collection is an existence check only, populated solely by the `customer_created` event. | `CustomerDocument` contains one field, `Id`, and the service subscribes to `customer_created`. | Customer data could be duplicated and drift between two services with no reconciliation defined. | Confirm the only insert path in `CustomerMongoRepository` is the event handler. |
| A3 | The three outbound HTTP calls are made synchronously during order handling with no cached fallback. | Each client issues a direct `GET` with no cache lookup, and the only resilience configured is Convey's two HTTP retries. | Availability and latency conclusions for order creation would be wrong, in either direction. | Trace `CreateOrderHandler` and `AddParcelToOrderHandler` through their client calls. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** It is not established that a caller can only act on their own orders. Order creation and listing are bound to the token's user id at the gateway, but `GET`/`DELETE /orders/{orderId}` and the parcel and vehicle routes are addressed by id with no ownership binding. | Security sign-off on the public API; any exposure of the gateway to real customers. | Security owner / service owner | Read the query and command handlers for a `CustomerId` check; if absent, add one, or bind `customerId: @user_id` on those gateway routes as is already done for creation. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is payment out of scope for this platform? Orders carry a total price and nothing collects it. | It is a large functional gap for a commercial ordering system and would change the service map substantially if added. | Unknown — no payment code, service, or integration exists anywhere in the thirteen repositories. | Product owner |
| Q2 | **[ACTION NOW]** What should happen when `parcels-service`, `pricing-service`, or `vehicles-service` is unavailable during order handling? | All three are called synchronously with only two retries, so any of them being down likely fails order creation outright. | Undefined in the repositories. | Service owner |
| Q3 | **[handled later by architecture_evolution_generation]** Is the Pact contract between Orders and Parcels published and verified automatically, or kept in step by hand? | An unpublished contract gives the appearance of contract testing without the protection, and only one of six cross-service calls is covered either way. | Unknown — no broker URL or publish step appears in either repository. | Service owner |
| Q4 | **[ACTION NOW]** Who is meant to act on `order_for_delivery_not_found` and `order_for_reserved_vehicle_not_found`? | They are the platform's only explicit signals that two services' views have diverged, and nothing subscribes to them or alerts on them. | Only `operations-service` records them, as it records everything. | Platform owner |
| Q5 | **[handled later by architecture_evolution_generation]** What are the legal order states and transitions, and which are reachable from the public API? | This is the platform's central state machine, and downstream design work will encode it. | Read from `Order.cs`, but not verified handler by handler. | Product owner |
