---
title: "Pacco — Architecture Repository Inventory"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
status: "evidence-based inventory"
repositories_in_scope: 13
---

# Pacco — Architecture Repository Inventory

Evidence-based inventory of the Pacco platform, built from the thirteen repositories cloned into this workspace. Every statement below is grounded in a file on disk. Where the documentation and the code disagree, **the code is treated as authoritative** and the disagreement is stated rather than smoothed over.

Per-repository detail lives in [`repo-summary/`](./repo-summary/), one file per repository.

## Contents

1. [Scope](#1-scope)
2. [Summary table — dimensions 1 to 7](#2-summary-table--dimensions-1-to-7)
3. [Summary table — dimensions 8 to 14](#3-summary-table--dimensions-8-to-14)
4. [Cross-repository relationships](#4-cross-repository-relationships)
5. [Suspected platform subsystems](#5-suspected-platform-subsystems)
6. [Gaps and unknowns](#6-gaps-and-unknowns)
7. [Documentation versus tree](#7-documentation-versus-tree)
8. [Coverage](#8-coverage)
9. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Scope

**Platform:** Pacco — an exclusive parcels delivery platform built around the idea of limited resource availability. It is a .NET Core 3.1 microservices system, event-driven, built on the **Convey** framework (`0.4.*`) with a **Ntrada** (`0.4.*`) API gateway.

**Repositories in scope (13).** Each is a separate git repository cloned into the workspace root under the prefix `hianshul100_`; the primary names below are the repository names as they appear in the platform's own solution and configuration files.

| # | Primary name | Workspace path | Kind |
|---|---|---|---|
| 1 | `Pacco` | `hianshul100_Pacco/` | Umbrella / orchestration |
| 2 | `Pacco.APIGateway` | `hianshul100_Pacco.APIGateway/` | Gateway |
| 3 | `Pacco.Services.Availability` | `hianshul100_Pacco.Services.Availability/` | Service |
| 4 | `Pacco.Services.Customers` | `hianshul100_Pacco.Services.Customers/` | Service |
| 5 | `Pacco.Services.Deliveries` | `hianshul100_Pacco.Services.Deliveries/` | Service |
| 6 | `Pacco.Services.Identity` | `hianshul100_Pacco.Services.Identity/` | Service |
| 7 | `Pacco.Services.Operations` | `hianshul100_Pacco.Services.Operations/` | Service |
| 8 | `Pacco.Services.OrderMaker` | `hianshul100_Pacco.Services.OrderMaker/` | Service (saga orchestrator) |
| 9 | `Pacco.Services.Orders` | `hianshul100_Pacco.Services.Orders/` | Service |
| 10 | `Pacco.Services.Parcels` | `hianshul100_Pacco.Services.Parcels/` | Service |
| 11 | `Pacco.Services.Pricing` | `hianshul100_Pacco.Services.Pricing/` | Service |
| 12 | `Pacco.Services.Vehicles` | `hianshul100_Pacco.Services.Vehicles/` | Service |
| 13 | `Pacco.Web` | `hianshul100_Pacco.Web/` | Empty — Unverifiable, Missing Source Evidence |

**Out of scope for the inventory table:** `Pacco.Context` (`hianshul100_Pacco.Context/`) is the artifact repository that holds this document. It is excluded by instruction.

**Platform port map**, from `hianshul100_Pacco/prod-services.yml`: `api` `5000`, `availability` `5001`, `customers` `5002`, `deliveries` `5003`, `identity` `5004`, `operations` `5005`, `orders` `5006`, `parcels` `5007`, `pricing` `5008`, `vehicles` `5009`. `Pacco.Services.OrderMaker` configures port `5015` but appears in no run list.

---

## 2. Summary table — dimensions 1 to 7

| Repository | 1 Purpose | 2 Runtime type | 3 Entrypoints | 4 Modules | 5 External integrations | 6 Data store, access, migrations | 7 Messaging |
|---|---|---|---|---|---|---|---|
| `Pacco` | Umbrella: solution, compose stacks, run lists, docs | Not a runtime | `compose/*.yml`, `services.yml`, `prod-services.yml`, `scripts/git-*.sh` | `compose/`, `scripts/`, `assets/` | Provisions Consul, Fabio, Vault, MongoDB, RabbitMQ, Redis, Jaeger, Seq, Prometheus, Grafana | None owned. No ORM, no migration tool | None of its own; provisions RabbitMQ |
| `Pacco.APIGateway` | Single public entry point; routes or publishes every client call | ASP.NET Core 3.1 hosting Ntrada `0.4.*`, port 5000 | `src/Pacco.APIGateway/Program.cs`, `Dockerfile`, `scripts/start.sh`, `scripts/start-async.sh` | 1 project; `Infrastructure/` correlation and span builders | RabbitMQ (async mode), Jaeger, Prometheus, all 10 services | **Stateless.** No store, no ORM, no migrations | RabbitMQ topic exchanges `availability`, `customers`, `deliveries`, `identity`, `orders`, `parcels`, `vehicles` — publish only, async config |
| `Pacco.Services.Availability` | Resources and their time-slot reservations — the platform's scarce-resource core | ASP.NET Core 3.1, dispatcher endpoints + RabbitMQ subscriber, port 5001 | `src/…Api/Program.cs`, `Dockerfile`, `scripts/` | 4 source + 5 test projects | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus, `customers-service` over HTTP | MongoDB `availability-service`; driver + hand-written repositories, no ORM; **no migration tool**; collections from `ResourceDocument`, `ReservationDocument`, plus `inbox` / `outbox` | Exchange `availability`; 5 events, 4 commands, 4 rejected; outbox and inbox enabled |
| `Pacco.Services.Customers` | Customer profile, state and VIP policy | ASP.NET Core 3.1, dispatcher endpoints + RabbitMQ subscriber, port 5002 | `src/…Api/Program.cs`, `Dockerfile`, `scripts/` | 4 source projects, **no tests** | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus | MongoDB `customers-service`; driver + repositories, no ORM; **no migration tool**; customers collection plus `inbox` / `outbox` | Exchange `customers`; 3 events, 2 commands, 2 rejected; consumes `signed_up`, `order_completed` |
| `Pacco.Services.Deliveries` | Delivery lifecycle: start, register, complete or fail | ASP.NET Core 3.1, dispatcher endpoints + RabbitMQ subscriber, port 5003 | `src/…Api/Program.cs`, `Dockerfile`, `scripts/` | 4 source projects, **no tests** | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus | MongoDB `deliveries-service`; driver + repositories, no ORM; **no migration tool**; deliveries collection plus `inbox` / `outbox` | Exchange `deliveries`; 4 events, 4 commands, 3 rejected in the catalogue (a fourth rejected class exists in code); consumes nothing |
| `Pacco.Services.Identity` | Users, passwords, roles, refresh tokens; issues the platform JWT | ASP.NET Core 3.1, **hand-written endpoints** + RabbitMQ, port 5004 | `src/…Api/Program.cs`, `Dockerfile`, `scripts/` | 4 source projects, **no tests** | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus | MongoDB `identity-service`; driver + repositories, no ORM; **no migration tool**; users and refresh-tokens collections plus `inbox` / `outbox`; revoked tokens held in Redis | Exchange `identity`; 2 events, 2 commands, 2 rejected; consumes nothing |
| `Pacco.Services.Operations` | Tracks every platform message as an operation; pushes updates to browsers | ASP.NET Core 3.1 serving **HTTP + SignalR + gRPC**, port 5005 | `src/…Api/Program.cs`, gRPC client `Program.cs`, `Dockerfile`, `scripts/proto/*-compile.sh` | 2 projects, **no layering** | MongoDB, RabbitMQ, Redis (cache **and** SignalR backplane), Consul, Fabio, Vault, Jaeger, Prometheus, every service exchange | MongoDB `operations-service`; driver, no ORM; **no migration tool**; operations collection; **no `inbox` / `outbox`** | Own exchange `operations`; subscribes to **all** exchanges via runtime-generated types driven by `messages.json` |
| `Pacco.Services.OrderMaker` | Saga that places a whole order in one request | ASP.NET Core 3.1 + Chronicle_ saga host, configured port 5015 | `src/Pacco.Services.OrderMaker/Program.cs`, `Dockerfile`, `scripts/` | 1 flat project, **no layering** | RabbitMQ, Redis, Consul, `availability-service` and `vehicles-service` over HTTP. **No Vault, no Jaeger** | **No database at all** — no `mongo` block, no persistence package; saga state not durably stored from anything visible | Exchange `ordermaker`; publishes `make_order_completed`, `make_order_rejected`; drives 4 orders commands and 1 availability command; consumes 5 external events |
| `Pacco.Services.Orders` | The order aggregate and its lifecycle — the platform's busiest node | ASP.NET Core 3.1, dispatcher endpoints + RabbitMQ subscriber, port 5006 | `src/…Api/Program.cs`, `Dockerfile`, `scripts/` | 4 source + 1 contract-test project | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus, plus `parcels-service`, `pricing-service`, `vehicles-service` over HTTP | MongoDB `orders-service`; driver + repositories, no ORM; **no migration tool**; collections from `OrderDocument` and **`CustomerDocument`** plus `inbox` / `outbox` | Exchange `orders`; 9 events, 7 commands, 10 rejected; consumes 7 external events |
| `Pacco.Services.Parcels` | Parcels: size, variant, volume; provider of the platform's only contract | ASP.NET Core 3.1, dispatcher endpoints + RabbitMQ subscriber, port 5007 | `src/…Api/Program.cs` (public host builder), `Dockerfile`, `scripts/start-test.sh` | 4 source + 1 contract-test project | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus | MongoDB `parcels-service` (tests use `test-parcels-service`); driver + repositories, no ORM; **no migration tool**; parcels and **customers** collections plus `inbox` / `outbox` | Exchange `parcels`; 2 events, 2 commands, 2 rejected; consumes 5 external events |
| `Pacco.Services.Pricing` | Order pricing with customer discounts | ASP.NET Core 3.1, single read endpoint, port 5008 | `src/Pacco.Services.Pricing.Api/Program.cs`, `Dockerfile`, `scripts/` | 1 flat project, **no layering** | Consul, Fabio, Vault, Jaeger, Prometheus, `customers-service` over HTTP | **Stateless.** No `mongo`, `redis` or `rabbitMq` block; no ORM, no migrations, no collections | **None.** No broker connection; the only service absent from `messages.json` |
| `Pacco.Services.Vehicles` | Vehicle catalogue: variant, capacity, price per service | ASP.NET Core 3.1, dispatcher endpoints + RabbitMQ subscriber, port 5009 | `src/…Api/Program.cs`, `Dockerfile`, `scripts/` | 4 source projects, **no tests** | MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus | MongoDB `vehicles-service`; driver + repositories, no ORM; **no migration tool**; vehicles collection plus `inbox` / `outbox`; **no replicated foreign data** | Exchange `vehicles`; 3 events, 3 commands, 3 rejected; consumes nothing |
| `Pacco.Web` | **Unknown — the repository is empty** | None | None | None | None | None | None |

---

## 3. Summary table — dimensions 8 to 14

| Repository | 8 APIs exposed / consumed | 9 Deployment | 10 Security | 11 Observability | 12 Decision files; feature flags | 13 Open questions | 14 Frontend |
|---|---|---|---|---|---|---|---|
| `Pacco` | None. Owns the port map | Compose network `pacco`; images `devmentors/pacco.*`; no Kubernetes, Helm or Terraform anywhere | Vault dev mode, root token in compose; **example unseal keys and a root token committed in `docker-images.txt`**; RabbitMQ `guest`/`guest` | Provisions Jaeger, Seq, Prometheus, Grafana; scrape path `/metrics-text` | `Pacco.sln`, `compose/*.yml`, `services.yml`, `prod-services.yml`, `docker-images.txt`. **No feature-flag system; no flag keys** | Branch mismatch; missing production target | No frontend assets detected — checked `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, view templates. `assets/` holds only PNG diagrams |
| `Pacco.APIGateway` | Exposes 10 route modules (`home`, `availability`, `customers`, `deliveries`, `identity`, `operations`, `orders`, `parcels`, `pricing`, `vehicles`); Swagger at `docs`. Consumes all 10 services | `Dockerfile` sets `NTRADA_CONFIG ntrada.docker`; compose overrides to `ntrada-async.docker.yml`; Travis build → test → dockerize | JWT `validIssuer: pacco`; role claim checks; **signing key committed in the routing YAML**; CORS `'*'` with credentials; `operations/{operationId}` is `auth: false` | Jaeger `api-gateway`; request and trace identifiers generated; Prometheus metrics; retries 2, exponential | `ntrada.yml`, `ntrada-async.yml`, `Infrastructure/SpanContextBuilder.cs`. **No feature-flag system; no flag keys** | Sync or async mode; unauthenticated operations route | No frontend assets detected — checked the same directory list. Only generated Swagger UI |
| `Pacco.Services.Availability` | `GET`/`POST`/`DELETE resources…`, reservations by `{dateTime}`. Consumes `customers-service` | Image `devmentors/pacco.services.availability`, `5001:80`; **Travis omits the test step** | JWT; `security.certificate.header: Certificate`; Vault KV, PKI `availability-service.pacco.io`, dynamic MongoDB credentials | Jaeger `availability` + RabbitMQ spans; log template mapper; **bespoke metrics middleware and job** | `Decorators/Outbox*.cs`, `Services/EventMapper.cs`, `Core/Entities/Resource.cs`. **No feature-flag system; no flag keys** | Tests never run in CI; who supplies the client certificate | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Customers` | `GET customers`, `GET customers/{id}`, `GET customers/{id}/state`, `POST customers`, `PUT customers/{id}/state/{state}`. Consumes nothing | Image `devmentors/pacco.services.customers`, `5002:80`; Travis build → test → dockerize (no tests exist) | **The only caller access list in the platform**: `availability-service` granted `customers:read`, `allowedDomains: ["pacco.io"]`; Vault KV and PKI | Jaeger `customers` + RabbitMQ spans; log template mapper; Prometheus | `Core/Services/VipPolicy.cs`, `Events/External/Handlers/SignedUpHandler.cs`, `appsettings.json` access list. **No feature-flag system; no flag keys** | Pricing service calls it but is not on the access list | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Deliveries` | `GET deliveries/{id}`, `POST deliveries`, `.../fail`, `.../complete`, `.../registrations`. Consumes nothing | Image `devmentors/pacco.services.deliveries`, `5003:80`; Travis build → test → dockerize (no tests exist) | JWT; Vault KV. **The only service with no `security` block at all** | Jaeger `deliveries` + RabbitMQ spans; log template mapper; Prometheus | `Application/Commands/Handlers/`, `Exceptions/ExceptionToMessageMapper.cs`. **No feature-flag system; no flag keys** | No service-to-service security; who creates a delivery | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Identity` | `GET users/{id}`, `GET me`, `POST sign-in`, `POST sign-up`, `POST access-tokens/revoke`, `POST refresh-tokens/use`, `POST refresh-tokens/revoke` | Image `devmentors/pacco.services.identity`, `5004:80`; Travis build → test → dockerize (no tests exist) | Issues the JWT: certificate `certs/localhost.pfx`, `expiryMinutes: 60`, `issuer: pacco`, `validateIssuer: false`. **Private keys and the certificate password committed** | Jaeger `identity` + RabbitMQ spans; log template mapper; Prometheus | `Infrastructure/Auth/JwtProvider.cs`, `Application/Services/Identity/RefreshTokenService.cs`. **No feature-flag system; no flag keys** | Which token scheme is real; refresh endpoints not exposed | No frontend assets detected — checked the same directory list. `certs/` holds certificate files only |
| `Pacco.Services.Operations` | `GET operations/{operationId}`; SignalR hub `/pacco` with method `initializeAsync`; gRPC `GrpcOperationsService.GetOperation` and `SubscribeOperations` | Image `devmentors/pacco.services.operations`, `5005:80`, **the only `depends_on` in the stack**; gRPC client defaults to `localhost:50050` | JWT via the hub call; `allowAnonymousEndpoints` names endpoints that do not exist here; gRPC client disables certificate checking; status readable without a token | Jaeger `operations`; correlation context; Prometheus. Is itself the platform's operation tracker | `Infrastructure/Subscriptions.cs` (runtime type generation), **`messages.json` — the platform message contract**, `Operations.proto`, `Hubs/PaccoHub.cs`. **No feature-flag system; no flag keys** | No outbox; message list already out of step; who uses gRPC | **The only frontend in the platform**: `wwwroot/ui/index.html`, `js/app.js` (vanilla JS), `js/signalr.js` (vendored 180,968-byte webpack bundle). Bootstrap 4.0.0 via CDN. No framework, no `package.json`, no build step |
| `Pacco.Services.OrderMaker` | `GET ""`, `POST orders`. Consumes `vehicles-service` and `availability-service`. **Not exposed through the gateway** | **Absent from `services.yml`, `prod-services.yml` and `compose/services.yml`**; Consul `address: localhost`, port 5015; `httpClient.type: ""` so no load balancer | JWT via `certs/localhost.cer`. **No Vault block at all**; no `security` block | Logging only. **No `jaeger` block — the one flow that most needs tracing has none** | `Sagas/AIOrderMakingSaga.cs`, `Sagas/AIMakingOrderData.cs`, `Services/ResourceReservationsService.cs`. **No feature-flag system; no flag keys** | No durable saga state; only one compensation step implemented | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Orders` | `GET`/`POST`/`DELETE orders…`, parcel and vehicle sub-routes. Consumes `parcels-service`, `pricing-service`, `vehicles-service` | Image `devmentors/pacco.services.orders`, `5006:80`; Travis build → test → dockerize, and the tests exist | JWT; Vault KV and PKI. Ownership filtering applied by the gateway, not by the service | Jaeger `orders` + RabbitMQ spans; log template mapper; Prometheus | `Core/Entities/Order.cs`, `Infrastructure/Mongo/Documents/CustomerDocument.cs`, `PACT/ParcelsApiPactConsumerTests.cs`. **No feature-flag system; no flag keys** | Replicated customer data may drift; direct access bypasses ownership filter | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Parcels` | `GET parcels/volume`, `GET parcels`, `GET parcels/{id}`, `POST parcels`, `DELETE parcels/{id}`. Consumes nothing | Image `devmentors/pacco.services.parcels`, `5007:80`; `appsettings.test.json` disables all shared infrastructure; Travis runs the provider contract tests | JWT; Vault KV and PKI. No caller access list although the orders service calls it | Jaeger `parcels` + RabbitMQ spans; log template mapper; Prometheus — all disabled in the test profile | `PACT/ParcelsApiPactProviderTests.cs`, `Api/Program.cs` public host builder, `Core/Services/ParcelsService.cs`. **No feature-flag system; no flag keys** | Contract testing covers one boundary only | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Pricing` | `GET pricing` only. Consumes `customers-service` | Image `devmentors/pacco.services.pricing`, `5008:80`; **no `LICENSE` file**; `.idea/` IDE files committed | JWT; Vault KV and PKI, **no database lease** (correctly, there is no database). Not on the customers access list although it calls that service | Jaeger `pricing`; Prometheus. References the RabbitMQ tracing package with no broker to trace | `Core/Services/CustomerDiscountsService.cs`, `Services/Clients/CustomersServiceClient.cs`. **No feature-flag system; no flag keys** | Behaviour when the customers service is down | No frontend assets detected — checked the same directory list |
| `Pacco.Services.Vehicles` | `GET vehicles/{id}`, `GET vehicles` (**the only paged result in the platform**), `POST`, `PUT`, `DELETE vehicles`. Consumes nothing | Image `devmentors/pacco.services.vehicles`, `5009:80`; **no `LICENSE` file**; Travis build → test → dockerize (no tests exist) | JWT; Vault KV and PKI. **Write routes carry no role requirement at the gateway** | Jaeger `vehicles` + RabbitMQ spans; Prometheus. **`Convey.Logging` missing from the API project file although `Program.cs` calls it** | `Core/Entities/Vehicle.cs`, `Application/Queries/SearchVehicles.cs`. **No feature-flag system; no flag keys** | Any signed-in user can change the shared vehicle catalogue | No frontend assets detected — checked the same directory list |
| `Pacco.Web` | None | Absent from the solution, every run list and every compose file | None | None | None. **No feature-flag system; no flag keys** | Whether a web application exists at all | **No frontend assets detected — checked `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, view templates. None of these exist.** Despite the name, there is no web front end here |

---

## 4. Cross-repository relationships

### 4.1 Synchronous HTTP calls between services

| Caller | Callee | Client class | Configured as |
|---|---|---|---|
| `Pacco.APIGateway` | all ten services | Ntrada `downstream` routes | `localUrl: localhost:5001` … `localhost:5009` |
| `Pacco.Services.Availability` | `Pacco.Services.Customers` | `Infrastructure/Services/Clients/CustomersServiceClient.cs` | `httpClient.services.customers: customers-service`, `type: fabio` |
| `Pacco.Services.Orders` | `Pacco.Services.Parcels` | `Infrastructure/Services/Clients/ParcelsServiceClient.cs` | `httpClient.services.parcels: parcels-service` |
| `Pacco.Services.Orders` | `Pacco.Services.Pricing` | `Infrastructure/Services/Clients/PricingServiceClient.cs` | `httpClient.services.pricing: pricing-service` |
| `Pacco.Services.Orders` | `Pacco.Services.Vehicles` | `Infrastructure/Services/Clients/VehiclesServiceClient.cs` | `httpClient.services.vehicles: vehicles-service` |
| `Pacco.Services.Pricing` | `Pacco.Services.Customers` | `Services/Clients/CustomersServiceClient.cs` | `httpClient.services.customers: customers-service` |
| `Pacco.Services.OrderMaker` | `Pacco.Services.Availability` | `Services/Clients/AvailabilityServiceClient.cs` | `httpClient.services.availability`, **`type: ""` — no load balancer** |
| `Pacco.Services.OrderMaker` | `Pacco.Services.Vehicles` | `Services/Clients/VehiclesServiceClient.cs` | `httpClient.services.vehicles`, **`type: ""`** |

Every caller except `Pacco.Services.OrderMaker` resolves the callee through Consul and routes through Fabio.

### 4.2 Asynchronous event flow

Publisher → consumer, taken from each service's `Events/External/` folders and confirmed against `messages.json`:

| Event | Published by | Consumed by |
|---|---|---|
| `signed_up` | `Pacco.Services.Identity` | `Pacco.Services.Customers` |
| `customer_created` | `Pacco.Services.Customers` | `Pacco.Services.Availability`, `Pacco.Services.Orders`, `Pacco.Services.Parcels` |
| `order_completed` | `Pacco.Services.Orders` | `Pacco.Services.Customers` (feeds the VIP policy) |
| `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order` | `Pacco.Services.Orders` | `Pacco.Services.Parcels` |
| `order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved` | `Pacco.Services.Orders` | `Pacco.Services.OrderMaker` (saga steps) |
| `parcel_deleted` | `Pacco.Services.Parcels` | `Pacco.Services.Orders` |
| `delivery_started`, `delivery_completed`, `delivery_failed` | `Pacco.Services.Deliveries` | `Pacco.Services.Orders` |
| `resource_reserved`, `resource_reservation_canceled` | `Pacco.Services.Availability` | `Pacco.Services.Orders`, `Pacco.Services.OrderMaker` |
| `vehicle_deleted` | `Pacco.Services.Vehicles` | `Pacco.Services.Availability` |
| **every command, event and rejected event** | all services | `Pacco.Services.Operations` |

`Pacco.Services.Pricing` neither publishes nor consumes anything.

### 4.3 The message contract

`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` is the platform's only written message contract. It maps eight service keys to their exchange and to the exact wire names of their commands, events and rejected events. `Pacco.Services.Operations` reads it at start-up and generates .NET types from it at runtime (`Infrastructure/Subscriptions.cs`), so a service that adds a message without updating this file becomes invisible to operation tracking.

It is already out of step: `Pacco.Services.Deliveries` defines `AddDeliveryRegistrationRejected` in code, and the catalogue has no matching entry.

### 4.4 Shared data coupling

Customer identity is the one concept that crosses every boundary. It is created in `Pacco.Services.Identity` at sign-up, turned into a customer by `Pacco.Services.Customers`, and then **replicated into two further databases**: `orders-service` (`Infrastructure/Mongo/Documents/CustomerDocument.cs`) and `parcels-service` (`Core/Entities/Customer.cs` with its own repository). Only `Pacco.Services.Pricing` reads customer data live instead of copying it.

There are no foreign keys anywhere — every store is MongoDB — so these copies are kept in step only by events, with no reconciliation code in any repository.

### 4.5 Shared conventions, not shared code

There is **no shared library repository**. Consistency comes from three sources: the external `Convey` `0.4.*` packages, a repeated file layout (`Decorators/Outbox*.cs`, `Services/EventMapper.cs`, `EventProcessor.cs`, `MessageBroker.cs`, `Exceptions/ExceptionToMessageMapper.cs`, `ExceptionToResponseMapper.cs` appear near-identically in six services), and copied `appsettings.json` blocks. The copying is visible in its defects: `Pacco.Services.Operations` carries `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` for endpoints it does not have, and `Pacco.Services.Pricing` references a RabbitMQ tracing package with no broker.

---

## 5. Suspected platform subsystems

Grouping is inferred from ownership, message flow and configuration, not from any document.

**A. Edge and access** — `Pacco.APIGateway`, `Pacco.Services.Identity`.
The only two components a client touches directly. The gateway holds all routing, role checks and the sync-or-async decision; identity mints the tokens. A change to either affects every client call.

**B. Order fulfilment core** — `Pacco.Services.Orders`, `Pacco.Services.Parcels`, `Pacco.Services.Deliveries`.
The order aggregate plus the two domains it composes. `Pacco.Services.Orders` is the hub: nine published events, seven accepted commands, seven consumed external events, three outbound HTTP dependencies.

**C. Resource and capacity** — `Pacco.Services.Availability`, `Pacco.Services.Vehicles`.
The scarce resources the platform is built around. Vehicles is the catalogue; availability is the booking calendar over it. `Pacco.Services.Availability` is the most complete implementation in the workspace and reads as the reference example the others were copied from.

**D. Customer and commercial** — `Pacco.Services.Customers`, `Pacco.Services.Pricing`.
Who the customer is and what they pay. These two make opposite data choices — one is replicated everywhere, the other keeps no data at all — which makes them the clearest illustration of the platform's data trade-off.

**E. Automation and orchestration** — `Pacco.Services.OrderMaker`.
The only orchestrated flow in an otherwise choreographed platform, and the only user of `Chronicle_`. It is missing from every run list, has no database, no secrets management and no tracing, which is why it reads as a demonstration rather than a supported component.

**F. Operations and visibility** — `Pacco.Services.Operations`.
Cross-cutting. Subscribes to everything, stores an operation per message, and pushes state to browsers. It exists because every write in Pacco is asynchronous, so a client needs somewhere to learn the outcome.

**G. Platform infrastructure** — `Pacco`.
Compose stacks, run lists, port map, image names, and the infrastructure playbook.

**H. Unbuilt** — `Pacco.Web`.
Named as a web front end; contains nothing.

---

## 6. Gaps and unknowns

### 6.1 Unverifiable — Missing Source Evidence

| Item | Where it is referenced | What is missing |
|---|---|---|
| `Pacco.Web` | Jira work item `12915`; cloned into the workspace | The repository contains only a one-line README. No source, no manifest, no configuration |
| `Pacco.APIGateway.Ocelot` | `hianshul100_Pacco/Pacco.sln:152`, referencing `..\Pacco.APIGateway.Ocelot\src\Pacco.APIGateway.Ocelot\Pacco.APIGateway.Ocelot.csproj` | No such repository exists in the workspace. The solution references 41 projects; 40 exist on disk |

### 6.2 Requires runtime capture

Message **names** are known exactly, from `messages.json` and from each service's event classes. Message **payloads** are known only as the property lists on those classes; the serialised form on the wire — envelope, headers, casing, null handling — was not observed. Every per-repository summary marks this explicitly.

The same applies to: the actual `Saga` header values in flight, the contents of the `inbox` and `outbox` collections, and whether Vault dynamic credentials are genuinely issued at start-up.

### 6.3 Structural gaps found in the code

| Gap | Evidence |
|---|---|
| **No database migration tool anywhere.** No Entity Framework migrations, no Flyway, Liquibase or equivalent in any of the thirteen repositories. Schema exists only as C# document classes | absence across all `src/` trees |
| **No feature-flag system anywhere.** No LaunchDarkly, Unleash, Split, Flagsmith or bespoke flag store in any repository. Every toggle found is a per-integration `enabled` boolean in `appsettings.json` | absence across all `appsettings.json` and project files |
| **No shared library repository.** Cross-cutting code is duplicated by copying | six services contain near-identical `Decorators/`, `Services/EventMapper.cs`, `Exceptions/` files |
| **No production deployment definition.** Docker Compose only; no Kubernetes, Helm or Terraform | `hianshul100_Pacco/compose/` |
| **.NET Core 3.1 throughout**, which is past end of support | every `<TargetFramework>netcoreapp3.1</TargetFramework>` |
| **Uneven test coverage.** `Pacco.Services.Availability` has five test projects, `Pacco.Services.Orders` and `Pacco.Services.Parcels` have one each, and **six repositories have none**: `Pacco.Services.Customers`, `Pacco.Services.Deliveries`, `Pacco.Services.Identity`, `Pacco.Services.Operations`, `Pacco.Services.OrderMaker`, `Pacco.Services.Vehicles`. Several of those still run `./scripts/test.sh` in CI with nothing to execute | `tests/` directories and `.travis.yml` files |
| **The one repository with a real test suite does not run it in CI** | `hianshul100_Pacco.Services.Availability/.travis.yml` omits `./scripts/test.sh` |
| **The cross-cutting subscriber has no outbox or inbox** | `Pacco.Services.Operations.Api.csproj` references no `Convey.MessageBrokers.Outbox` package |
| **The saga service has no database and no tracing** | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json` has no `mongo` and no `jaeger` block |

### 6.4 Security findings

Reported as evidence, not remediated in this stage. Values are referenced by location and setting name only; no secret material is reproduced in this document.

| Finding | Location |
|---|---|
| Symmetric JWT signing key committed in clear text | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` and `ntrada-async.yml`, key `extensions.jwt.issuerSigningKey` |
| Private key material committed | `hianshul100_Pacco.Services.Identity/.../certs/localhost.key`, `localhost.pem`, `localhost.pfx` |
| Certificate password committed in clear text | `hianshul100_Pacco.Services.Identity/.../appsettings.json`, key `jwt.certificate.password` |
| Example secrets-manager unseal keys and root token committed | `hianshul100_Pacco/docker-images.txt` |
| Secrets manager runs in development mode with a fixed root token | `hianshul100_Pacco/compose/infrastructure.yml`, `VAULT_DEV_ROOT_TOKEN_ID` |
| Default broker credentials `guest` / `guest` | `hianshul100_Pacco.APIGateway/.../ntrada-async.yml`, `hianshul100_Pacco/compose/` |
| Server certificate validation disabled in the gRPC client | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.GrpcClient/`, `DangerousAcceptAnyServerCertificateValidator` |
| CORS allows any origin together with credentials | `hianshul100_Pacco.APIGateway/.../ntrada.yml`, `extensions.cors` |
| Operation status readable with no token | `ntrada.yml`, module `operations`, `auth: false` |
| Vehicle write routes carry no role requirement | `ntrada.yml`, module `vehicles` |
| Ownership filtering exists only at the gateway; services do not re-check | `ntrada.yml` rewrites for `orders`, `parcels`, `pricing` |
| Two different token schemes configured on the two sides | certificate signing in `Pacco.Services.Identity`; symmetric key validation in `Pacco.APIGateway` |
| Service-to-service authorisation applied in one place only | `Pacco.Services.Customers` defines an access list; `Pacco.Services.Deliveries` has no security block at all |

---

## 7. Documentation versus tree

A platform-level view. Per-repository detail is in the "README vs repository" subsection of each summary file.

**Where the documentation is correct.** The `Pacco` README's core claims hold up: `.NET Core 3.1` is the target framework everywhere; every service references `Convey` `0.4.*`; the `.Api` / `.Application` / `.Core` / `.Infrastructure` split is real in eight of the ten services; the platform genuinely is event-driven, with a topic exchange per service and a transactional outbox in most of them; and the infrastructure the README names is exactly what `compose/infrastructure.yml` starts.

**Where it is stale.**

| Documentation says | The tree shows |
|---|---|
| Twelve repositories to clone | Thirteen exist. `Pacco.Web` is cloned but not in the list |
| `Pacco.Services.OrderMaker` is part of the platform | It is in the clone list but in no run list, no compose file and no gateway route |
| Clean architecture, four projects per service | Three services do not follow it: `Pacco.Services.Operations` (2 projects, no layering), `Pacco.Services.OrderMaker` (1 flat project), `Pacco.Services.Pricing` (1 flat project) |
| Every service is event-driven | `Pacco.Services.Pricing` has no broker connection and is absent from the message catalogue |
| SQL Server, PostgreSQL, InfluxDB, Elasticsearch, Kibana and Logstash are documented in `docker-images.txt` | No service configuration references any of them. All persistence is MongoDB plus Redis |
| Vault-managed secrets and Jaeger tracing are platform-wide | `Pacco.Services.OrderMaker` has neither |
| Certificate-based service security is part of the design | Only `Pacco.Services.Customers` defines an access list; `Pacco.Services.Deliveries` has no security block |
| The build chain runs tests | Six repositories have no test project; the one with the fullest suite skips the test step |
| The work item names branch `master` | The clones in this workspace are on `feature/12915/aidlc` |

**Future or intended state (not implemented).** Two things are named but not built: `Pacco.Web`, a web front end that exists only as a repository name, and `Pacco.APIGateway.Ocelot`, an alternative gateway referenced by the solution file with no source anywhere. Both are recorded above as missing source evidence rather than as design.

**Disk-only, undocumented components.** The gRPC contract and console client, the runtime type generation driven by `messages.json`, the SignalR hub and its browser page, the Chronicle saga, the bespoke metrics middleware in `Pacco.Services.Availability`, the paged vehicle search, the contract test pair, and the `host-` variants of the compose files — none of these appear in the platform README, and several are among the most consequential design decisions in the system.

---

## 8. Coverage

The authoritative project list for each repository was taken from the manifests on disk — every `*.csproj` file, cross-checked against `hianshul100_Pacco/Pacco.sln`. **40 project files exist across the workspace; all 40 are accounted for below.**

| Repository | Projects enumerated | Documented | Excluded, with reason |
|---|---|---|---|
| `Pacco` | 0 | — (no projects of its own; `Pacco.sln` aggregates other repositories' projects by relative path) | none |
| `Pacco.APIGateway` | 1 | 1 — `Pacco.APIGateway` | none |
| `Pacco.Services.Availability` | 9 | 9 — `…Api`, `…Application`, `…Core`, `…Infrastructure` individually; `…Tests.Unit`, `…Tests.Integration`, `…Tests.EndToEnd`, `…Tests.Performance`, `…Tests.Shared` covered as the test suite | none |
| `Pacco.Services.Customers` | 4 | 4 — `…Api`, `…Application`, `…Core`, `…Infrastructure` | none |
| `Pacco.Services.Deliveries` | 4 | 4 — `…Api`, `…Application`, `…Core`, `…Infrastructure` | none |
| `Pacco.Services.Identity` | 4 | 4 — `…Api`, `…Application`, `…Core`, `…Infrastructure` | none |
| `Pacco.Services.Operations` | 2 | 2 — `Pacco.Services.Operations.Api`, `Pacco.Services.Operations.GrpcClient` | none |
| `Pacco.Services.OrderMaker` | 1 | 1 — `Pacco.Services.OrderMaker` | none |
| `Pacco.Services.Orders` | 5 | 5 — `…Api`, `…Application`, `…Core`, `…Infrastructure`, `Pacco.Services.Orders.PactConsumerTests` | none |
| `Pacco.Services.Parcels` | 5 | 5 — `…Api`, `…Application`, `…Core`, `…Infrastructure`, `Pacco.Services.Parcels.PactProviderTests` | none |
| `Pacco.Services.Pricing` | 1 | 1 — `Pacco.Services.Pricing.Api` | none |
| `Pacco.Services.Vehicles` | 4 | 4 — `…Api`, `…Application`, `…Core`, `…Infrastructure` | none |
| `Pacco.Web` | 0 | — (no manifest of any kind; the repository holds one README file) | none |
| **Total** | **40** | **40** | **0** |

**One project is enumerated by the solution but has no source in this workspace:** `Pacco.APIGateway.Ocelot`, referenced at `hianshul100_Pacco/Pacco.sln:152`. It is counted in the solution's 41 entries but not in the 40 on disk, and is recorded in section 6.1 as missing source evidence rather than excluded.

**Repository coverage:** 13 of 13 in-scope repositories have a summary file in [`repo-summary/`](./repo-summary/). `Pacco.Context` is excluded by instruction as the artifact repository holding this document.

**Dimension coverage:** all 14 dimensions are answered for all 13 repositories. Where a dimension does not apply, that is stated explicitly rather than left blank — for example, "No frontend assets detected" with the checked directory list appears in twelve of the thirteen summaries.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The code in these repositories is the truth, and the written documentation is only a guide. | Where the two disagree, the code is what actually runs. |
| A2 | The thirteen repositories cloned into this workspace are the whole platform. | They match the list in the work item, and no repository refers to another one outside this set, apart from the two missing items recorded above. |
| A3 | The port numbers in the umbrella repository's production run list are the platform's real port map. | It is the only place where every port is listed together, and each service's own settings agree with it. |
| A4 | The shared message list file in the operations service is the platform's message contract. | It is the only document that names every service's messages, and one service reads it directly to decide what to listen to. |
| A5 | Docker Compose is the only deployment method available today. | There are no cloud, cluster or infrastructure-as-code files anywhere in the workspace. |
| A6 | The service that reserves resources is the reference example the other services were copied from. | It is the most complete, and the same files appear in simplified form in the others. |
| A7 | The saga service is a demonstration rather than a supported part of the platform. | It cannot be started from any shared run file and has no database, secrets or tracing. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | The repository named as the web front end is empty, so there is no client application to describe. Any later work that assumes a user interface has nothing to build on. | **[ACTION NOW]** Tell the requesting team and ask whether the web application exists elsewhere, is planned, or should leave scope. |
| B2 | The solution file points at a gateway project whose source is not in this workspace, so the solution cannot be opened cleanly. | **[ACTION NOW]** Ask the requesting team whether that repository still exists or should be removed from the solution. |
| B3 | Sign-in keys, private certificate files and a certificate password are stored openly in the repositories, so anyone with repository access can create valid logins for the whole platform. | **[ACTION NOW]** Report to the security owner of the platform. This stage records the locations and changes nothing. |
| B4 | Message payloads can only be read from the code, not observed. Anything that needs the exact format sent between services cannot be settled from these repositories alone. | **[handled later by the ADR authoring stage]** Capture one real message per event from a running system if exact formats are needed. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Which branch is the real source of truth: the `master` branch named in the work item, or the `feature/12915/aidlc` branch that was actually cloned? | **[ACTION NOW]** Confirm with the requesting team before any later stage quotes file locations. |
| Q2 | Should the gateway run in its direct-call mode or its message-publishing mode? The container image and the shared start-up file choose differently, and the answer changes how every write reaches the services. | **[ACTION NOW]** Confirm with the requesting team. |
| Q3 | Which sign-in scheme is really in use: the certificate the identity service signs with, or the shared key the gateway checks against? | **[ACTION NOW]** Confirm with the requesting team; this decides how sign-in is described for every service. |
| Q4 | Should any signed-in customer be able to add or remove vehicles from the shared catalogue? Today nothing stops them. | **[ACTION NOW]** Confirm with the requesting team. |
| Q5 | Should the platform use a central coordinator, independent services reacting to each other, or both? Both exist today with no written rule. | **[handled later by the ADR authoring stage]** Record the intended way services should work together. |
| Q6 | What keeps the three separate copies of customer data in step if a message is lost? No repository contains repair code. | **[handled later by the ADR authoring stage]** Record how duplicated data is kept correct. |
| Q7 | How is this platform meant to run in production? Only local development setup exists here. | **[handled later by the ADR authoring stage]** Record the intended production target, or note that there is none. |
| Q8 | The platform runs on a version of .NET that is past end of support. Is an upgrade planned? | **[handled later by the ADR authoring stage]** Record the intended platform version. |
| Q9 | Six of the twelve code repositories have no automated tests, and the one with the fullest test suite never runs it during a build. Is that the intended level of testing? | **[handled later by the ADR authoring stage]** Record the intended testing approach for the platform. |
| Q10 | The shared message list is already missing one message that exists in code, which means that failure would never reach a waiting client. Which is authoritative, the list or the code? | **[ACTION NOW]** Confirm with the requesting team, since the list drives what clients can be told. |
| Q11 | Should the pricing service be added to the customers service's list of allowed callers? It reads customer data on every request but is not listed. | **[ACTION NOW]** Confirm with the requesting team. |
| Q12 | A caller inside the platform network can read any customer's orders or parcels, because ownership is only checked at the public entry point. Is that acceptable? | **[ACTION NOW]** Confirm with the requesting team. |
