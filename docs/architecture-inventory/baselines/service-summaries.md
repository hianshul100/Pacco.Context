# Pacco — Service & Bounded-Context Summaries (Discovery Baseline)

**Project Key:** Common Architecture
**Stage:** `architecture_discovery` — rollup of the repository inventory and platform map into a
service / bounded-context baseline. **No ADRs, no recommendations, no KG JSON, no test inventory.**
**Date of analysis:** 2026-09-04
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`

**Inputs used (all read from this repository on disk):**

- `docs/architecture-inventory/repo-inventory.md` — repository inventory, 14 dimensions per repo
- `docs/architecture-inventory/architecture-views.md` — platform map (C1 contexts, dependency
  graph, runtime flows, deployment topology, ER views)
- `docs/architecture-inventory/repo-summary/*.md` — 13 per-repo summaries
- The thirteen cloned source repositories, which are the **source of truth** for every statement
  below. Where a summary and the code disagree, the code wins and the disagreement is stated
  explicitly in §5.
- `.attachments/01_product_backlog_20260903_170135_37cf143b.xlsx` — backlog issue **12998**
  "Pacco - Discovery - Attempt-2", which fixes the thirteen-repository scope.

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Service and bounded-context catalogue](#2-service-and-bounded-context-catalogue)
3. [Repository → service / component table](#3-repository--service--component-table)
4. [Bounded-context boundaries and coupling surfaces](#4-bounded-context-boundaries-and-coupling-surfaces)
5. [Documentation versus source code](#5-documentation-versus-source-code)
6. [Gaps — unknown or missing summary data](#6-gaps--unknown-or-missing-summary-data)
7. [Coverage](#7-coverage)
8. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Executive summary

Pacco is a .NET Core 3.1 microservices platform for exclusive parcel delivery, organised around the
idea of limited resource availability. Thirteen repositories are in scope. Between them they
produce **eleven deployed services** (one edge gateway plus ten domain and cross-cutting services),
**one non-containerised console client**, **one repository that owns no deployable but owns every
environment definition**, and **one empty repository**.

Ten bounded contexts are observable in code, and each maps one-to-one onto a repository, a
RabbitMQ topic exchange, and — for the eight stateful ones — a dedicated MongoDB logical database.
The boundary is drawn by *ownership of a write model*: `availability-service` owns `resources`,
`customers-service` owns `customers`, `deliveries-service` owns `deliveries`, `identity-service`
owns `users`/`refreshTokens`, `orders-service` owns `orders`, `parcels-service` owns `parcels`, and
`vehicles-service` owns `vehicles`. No service reads another service's database
(`Pacco/compose/infrastructure.yml` provisions one shared `mongo` container; each service's
`appsettings.json` names its own `mongo.database`).

Three components are deliberately *not* bounded contexts and are best read as platform roles:

- `api-gateway` (repo `Pacco.APIGateway`) is the single north-south edge and owns no domain state.
- `operations-service` (repo `Pacco.Services.Operations`) is a read-only projection of every
  message on the platform, pushed to browsers over SignalR and to subscribers over gRPC.
- `ordermaker-service` (repo `Pacco.Services.OrderMaker`) is a cross-context saga orchestrator; it
  holds no aggregate and issues commands onto four other services' exchanges.

Two services break the platform template in ways that matter for downstream design.
`pricing-service` (repo `Pacco.Services.Pricing`) is the only service with **no message broker, no
database, and no clean-architecture split** — it is a stateless HTTP calculator, verified by the
absence of any `rabbitMq` section in
`Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json` and of any
`Convey.MessageBrokers.*` or `Convey.Persistence.MongoDB` package reference.
`ordermaker-service` is the only broker-participating service with **no Jaeger tracing package**
and **no Mongo persistence**, and it is absent from both PM2 process manifests and from all four
gateway configurations, so its `POST /orders` entry point has no caller path visible anywhere in
the workspace.

Confidence in the service catalogue itself is **high**: every service name in this document is a
compose `container_name`, a Docker image name, and a `*.csproj` host project simultaneously.
Confidence in *responsibility attribution* is high for the eight aggregate-owning services and
medium for the two cross-cutting ones, whose runtime state stores could not be settled from static
reading (see §6, G4 and G3).

**No external catalog record of the Pacco services, their domains, or their ownership was
available for this baseline**; every entry below is grounded in the cloned source and in the two
prior discovery documents in this repository. Ownership by team remains unrecorded anywhere in the
thirteen clones — there is no `CODEOWNERS` and no `CONTRIBUTING.md` in any of them.

---

## 2. Service and bounded-context catalogue

Every service is named by its **deployable name** — the `container_name` in
`Pacco/compose/services.yml`, which is also the Consul/Fabio service key used in every
`httpClient.services` map. Aliases are bound on first mention and are never used as the primary
name. Ownership by team is **unknown** for all entries; the "Owning domain" column records the
*domain* the service owns, not the team, because no team metadata exists in any clone.

### 2.1 Edge and access

#### `api-gateway`

`api-gateway` (also known as: Pacco API Gateway, `Pacco.APIGateway`, Docker image
`devmentors/pacco.apigateway`, PM2 app `api`) is the single north-south entry point for the
platform. Repository: `Pacco.APIGateway`, path: `src/Pacco.APIGateway`.

- **Owning domain:** none — it is edge infrastructure and holds no domain state.
- **Responsibility:** terminate untrusted traffic, validate the JWT it did not issue, enforce
  `role: admin` claim gates on five routes, bind `@user_id` from the token into downstream URLs and
  message payloads, and then either proxy the call over HTTP (`downstream:`) or publish it to a
  RabbitMQ exchange (`publish:`), depending on which of the four `ntrada*.yml` files
  `NTRADA_CONFIG` selects.
- **Runtime:** ASP.NET Core 3.1 host embedding Ntrada 0.4.*; port `5000`.
- **Dependencies:** all nine HTTP-exposed services downstream; RabbitMQ for the async mode
  (6 exchanges, 20 routing keys); Fabio/Consul for resolution; Jaeger, Seq, Prometheus.
- **Consumed by:** external clients only. No in-platform service calls it.
- **Confidence: high.** The routing table is fully declarative and readable in
  `src/Pacco.APIGateway/ntrada.yml` and `ntrada-async.yml`.
- **Caveat:** the sync and async configurations are *architecturally* different systems, not
  environment variants — see Q1.

#### `identity-service`

`identity-service` (also known as: Identity Service, `Pacco.Services.Identity`, Docker image
`devmentors/pacco.services.identity`, PM2 app `identity`) is the platform's authentication origin.
Repository: `Pacco.Services.Identity`, path: `src/Pacco.Services.Identity.Api`.

- **Owning domain:** Identity & access — users, credentials, roles, refresh tokens.
- **Write model owned:** MongoDB `identity-service`, collections `users` and `refreshTokens`
  (`Core/Entities/User.cs`, `RefreshToken.cs`, `Role.cs`), plus Convey `inbox`/`outbox`.
- **Responsibility:** sign-up, sign-in, JWT issuance (`Infrastructure/Auth/JwtProvider.cs`),
  password hashing (`PasswordService.cs`), Redis-backed access-token revocation
  (`UseAccessTokenValidator()`), refresh-token use and revocation.
- **Contract surface:** exchange `identity`; subscribes the command `sign_up`; publishes
  `signed_up`, `signed_in`, `sign_in_rejected`, `sign_up_rejected`. HTTP: `POST /sign-in`,
  `POST /sign-up`, `GET /me`, `GET /users/{userId}`, `POST /access-tokens/revoke`,
  `POST /refresh-tokens/use`, `POST /refresh-tokens/revoke`.
- **Dependencies:** calls no other service (`httpClient.services` is empty). Its `signed_up` event
  is consumed by `customers-service`, which is the only in-platform consumer of this context.
- **Confidence: high.** Port `5004`.

### 2.2 Order fulfilment core

#### `orders-service`

`orders-service` (also known as: Orders Service, `Pacco.Services.Orders`, Docker image
`devmentors/pacco.services.orders`, PM2 app `orders`) owns the order aggregate. Repository:
`Pacco.Services.Orders`, path: `src/Pacco.Services.Orders.Api`.

- **Owning domain:** Order lifecycle — parcels on an order, vehicle assignment, price, approval,
  cancellation, delivery state.
- **Write model owned:** MongoDB `orders-service`, collections `orders` and `customers`
  (`Core/Entities/Order.cs`, `Parcel.cs`, `OrderStatus.cs`, `Customer.cs`), plus `inbox`/`outbox`.
  The local `customers` collection is a **replica of another context's data** — see §4.
- **Contract surface:** the largest on the platform — exchange `orders` with 7 commands, 9 events
  and 10 rejected events. Subscribes seven external events: `customer_created`, `parcel_deleted`,
  `resource_reserved`, `resource_reservation_canceled`, `delivery_started`, `delivery_completed`,
  `delivery_failed`.
- **Dependencies (synchronous):** `parcels-service`, `pricing-service`, `vehicles-service`, each
  via a `Convey.HTTP` `IHttpClient` in `Orders.Infrastructure/Services/Clients/`.
- **Consumed by:** `ordermaker-service` (4 events), `parcels-service` (4 events),
  `customers-service` (`order_completed`).
- **Notable:** the only Pact **consumer** on the platform
  (`tests/Pacco.Services.Orders.PactConsumerTests`, `Pactify` 1.1.0).
- **Confidence: high.** Port `5006`.

#### `parcels-service`

`parcels-service` (also known as: Parcels Service, `Pacco.Services.Parcels`, Docker image
`devmentors/pacco.services.parcels`, PM2 app `parcels`) owns the parcel catalogue. Repository:
`Pacco.Services.Parcels`, path: `src/Pacco.Services.Parcels.Api`.

- **Owning domain:** Parcels — parcel identity, size, variant, and volume aggregation.
- **Write model owned:** MongoDB `parcels-service`, collections `parcels` and `customers`
  (`Core/Entities/Parcel.cs`, `Size.cs`, `Variant.cs`, `Customer.cs`), plus `inbox`/`outbox`. The
  local `customers` collection is a replica — see §4.
- **Contract surface:** exchange `parcels`; publishes `parcel_added`, `parcel_deleted` and their
  two rejected events; subscribes `customer_created`, `order_canceled`, `order_deleted`,
  `parcel_added_to_order`, `parcel_deleted_from_order`. HTTP: `GET /parcels`,
  `GET /parcels/{parcelId}`, `GET /parcels/volume`, `POST /parcels`, `DELETE /parcels/{parcelId}`.
- **Dependencies:** calls no other service.
- **Notable:** the only Pact **provider** (`tests/Pacco.Services.Parcels.PactProviderTests`), the
  counterpart to `orders-service`. Unlike the other clean-architecture services, its
  `Core/Entities/` folder contains **no `AggregateRoot.cs` or `AggregateId.cs`** — the base
  aggregate abstraction present in Availability, Customers, Deliveries, Identity and Orders is
  absent here.
- **Confidence: high.** Port `5007`.

#### `deliveries-service`

`deliveries-service` (also known as: Deliveries Service, `Pacco.Services.Deliveries`, Docker image
`devmentors/pacco.services.deliveries`, PM2 app `deliveries`) owns the delivery lifecycle.
Repository: `Pacco.Services.Deliveries`, path: `src/Pacco.Services.Deliveries.Api`.

- **Owning domain:** Deliveries — delivery state and tracking registrations (scans).
- **Write model owned:** MongoDB `deliveries-service`, collection `deliveries`
  (`Core/Entities/Delivery.cs`, `DeliveryStatus.cs`), plus `inbox`/`outbox`.
- **Contract surface:** exchange `deliveries`; publishes `delivery_started`, `delivery_completed`,
  `delivery_failed`, `registration_added_to_delivery` and three rejected events. HTTP:
  `POST /deliveries`, `GET /deliveries/{deliveryId}`, `POST /deliveries/{deliveryId}/complete`,
  `POST /deliveries/{deliveryId}/fail`, `POST /deliveries/{deliveryId}/registrations`.
- **Dependencies:** calls no other service and **subscribes to no external event** — it is a pure
  sink at the async layer. Its three events are consumed only by `orders-service`.
- **Confidence: high** for the context boundary; **medium** for how it is driven, because nothing
  in the workspace publishes a `deliveries` command (see G7 / Q6).
- **Notable:** unlike `availability-service` and `customers-service` it carries **no
  `Convey.WebApi.Security` package**, so the certificate header it configures is not enforced by
  the same mechanism.
- **Port:** `5003`.

### 2.3 Resource and capacity

#### `availability-service`

`availability-service` (also known as: Availability Service, `Pacco.Services.Availability`, Docker
image `devmentors/pacco.services.availability`, PM2 app `availability`) owns the platform's
organising concept: limited resources reserved against dates. Repository:
`Pacco.Services.Availability`, path: `src/Pacco.Services.Availability.Api`.

- **Owning domain:** Resource availability and reservation.
- **Write model owned:** MongoDB `availability-service`, collection `resources`. The aggregate root
  is `Core/Entities/Resource.cs`; reservations are held inside it, not as a separate collection.
- **Contract surface:** exchange `availability`; publishes `resource_added`, `resource_deleted`,
  `resource_reserved`, `resource_reservation_released`, `resource_reservation_canceled` plus four
  rejected events; subscribes `customer_created` and `vehicle_deleted`. HTTP:
  `GET/POST /resources`, `GET/DELETE /resources/{resourceId}`,
  `POST/DELETE /resources/{resourceId}/reservations/{dateTime}`.
- **Dependencies (synchronous):** `customers-service` — `GET /customers/{id}/state`, via
  `Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs`.
- **Consumed by:** `orders-service` and `ordermaker-service` (`resource_reserved`),
  `orders-service` (`resource_reservation_canceled`).
- **Notable:** the strongest security posture on the platform — Vault-PKI client-certificate auth
  (`security.certificate.header: Certificate`, certificate attached in the `CustomersServiceClient`
  constructor) on top of JWT, plus a transactional outbox (`outbox.enabled: true`).
- **Confidence: high.** Port `5001`.

#### `vehicles-service`

`vehicles-service` (also known as: Vehicles Service, `Pacco.Services.Vehicles`, Docker image
`devmentors/pacco.services.vehicles`, PM2 app `vehicles`) owns the catalogue of vehicles.
Repository: `Pacco.Services.Vehicles`, path: `src/Pacco.Services.Vehicles.Api`.

- **Owning domain:** Vehicles — vehicle identity, description, per-service pricing attributes.
- **Write model owned:** MongoDB `vehicles-service`, collection `vehicles`
  (`Core/Entities/Vehicle.cs`, `Variants.cs`), plus `inbox`/`outbox`.
- **Contract surface:** exchange `vehicles`; publishes `vehicle_added`, `vehicle_updated`,
  `vehicle_deleted` plus three rejected events; **subscribes to nothing**. HTTP: `GET /vehicles`
  (paged search), `GET /vehicles/{vehicleId}`, `POST /vehicles`, `PUT /vehicles/{vehicleId}`,
  `DELETE /vehicles/{vehicleId}`.
- **Dependencies:** calls no other service.
- **Consumed by:** `orders-service` and `ordermaker-service` synchronously;
  `availability-service` asynchronously, but only for `vehicle_deleted`. `vehicle_added` and
  `vehicle_updated` have **no consumer anywhere in the workspace**.
- **Confidence: high.** Port `5009`.

### 2.4 Customer and commercial

#### `customers-service`

`customers-service` (also known as: Customers Service, `Pacco.Services.Customers`, Docker image
`devmentors/pacco.services.customers`, PM2 app `customers`) owns the customer profile. Repository:
`Pacco.Services.Customers`, path: `src/Pacco.Services.Customers.Api`.

- **Owning domain:** Customers — profile, registration completion, VIP status, account state.
- **Write model owned:** MongoDB `customers-service`, collection `customers`
  (`Core/Entities/Customer.cs`, `State.cs`), plus `inbox`/`outbox`.
- **Contract surface:** exchange `customers`; publishes `customer_created`, `customer_became_vip`,
  `customer_state_changed` plus two rejected events; subscribes `signed_up` (from
  `identity-service`) and `order_completed` (from `orders-service`). HTTP: `GET /customers`,
  `GET /customers/{customerId}`, `GET /customers/{customerId}/state`, `POST /customers`,
  `PUT /customers/{customerId}/state/{state}`.
- **Dependencies:** calls no other service — a leaf in the synchronous graph.
- **Consumed by:** the most-depended-on context on the platform. Synchronously by
  `availability-service` and `pricing-service`; asynchronously, its `customer_created` event is
  consumed by `availability-service`, `orders-service` and `parcels-service`.
- **Notable:** enforces a certificate ACL — `security.certificate.acl` grants
  `availability-service` the `customers:read` permission, with `allowedDomains: ['pacco.io']`.
- **Confidence: high** for the context; **medium** for the ACL's runtime effect (see Q9).
- **Port:** `5002`.

#### `pricing-service`

`pricing-service` (also known as: Pricing Service, `Pacco.Services.Pricing`, Docker image
`devmentors/pacco.services.pricing`, PM2 app `pricing`) computes order prices and customer
discounts. Repository: `Pacco.Services.Pricing`, path: `src/Pacco.Services.Pricing.Api`.

- **Owning domain:** Pricing rules. It is a **stateless calculator, not a bounded context with a
  write model** — it owns no data and emits no events.
- **Write model owned:** none. `appsettings.json` has no `mongo` and no `redis` section, and the
  single `*.csproj` references no persistence package.
- **Contract surface:** exchange — **none**. Verified: `appsettings.json` contains zero occurrences
  of `rabbitMq` and the project references no `Convey.MessageBrokers.*` package. This is the only
  service on the platform with no broker participation. HTTP:
  `GET /pricing?customerId=&orderPrice=`.
- **Dependencies (synchronous):** `customers-service` — `GET /customers/{id}`, via
  `Pricing.Api/Services/Clients/CustomersServiceClient.cs`.
- **Consumed by:** `orders-service`.
- **Notable:** deliberately *not* clean architecture — a single project, with the discount rules in
  `Core/Services/CustomerDiscountsService.cs`. One of the two service repositories with no
  `LICENSE` file.
- **Confidence: high** for the component; **medium** for placing it inside a "customer and
  commercial" grouping, since it shares none of the platform's conventions and reads equally well
  as a standalone utility.
- **Port:** `5008`.

### 2.5 Orchestration and observation

#### `ordermaker-service`

`ordermaker-service` (also known as: OrderMaker Service, `Pacco.Services.OrderMaker`, AI Order
Maker, Docker image `devmentors/pacco.services.ordermaker`) orchestrates end-to-end order creation
as a saga. Repository: `Pacco.Services.OrderMaker`, path: `src/Pacco.Services.OrderMaker`.

- **Owning domain:** none — it is a **process orchestrator across four other contexts** (Orders,
  Parcels, Vehicles, Availability). It owns no aggregate.
- **Write model owned:** none observable. There is no `mongo` section, no
  `Convey.Persistence.MongoDB` package, and `Extensions.cs:43` calls `builder.Services.AddChronicle()`
  with no persistence backend configured; `Extensions.cs:37` registers `.AddRedis()`. Where saga
  state actually lives is **unknown — requires runtime validation** (G3).
- **Contract surface:** exchange `ordermaker`; publishes `make_order_completed` and
  `make_order_rejected`, and issues commands onto **other services' exchanges** (`create_order`,
  `add_parcel_to_order`, `assign_vehicle_to_order`, `approve_order`, `reserve_resource`,
  `cancel_order`). Subscribes `order_created`, `parcel_added_to_order`,
  `vehicle_assigned_to_order`, `order_approved`, `resource_reserved`. HTTP: `POST /orders`,
  `GET /`.
- **Dependencies (synchronous):** `availability-service` (`GET /resources/{resourceId}`),
  `vehicles-service` (`GET /vehicles`).
- **Notable:** the only broker-participating service with **no Jaeger package** — the saga is
  invisible to distributed tracing. `Program.cs` calls `UseApp()` rather than
  `UseInfrastructure()`, and the saga constructs an **empty `CorrelationContext.UserContext`**, so
  it acts with no caller identity.
- **Confidence: medium.** The saga choreography in `Sagas/AIOrderMakingSaga.cs` is unambiguous, but
  the service is defined in both Compose stacks (`Pacco/compose/services.yml:79-80`, port `5015`)
  and scraped by Prometheus, while being **absent from both PM2 manifests and from all four
  `ntrada*.yml` gateway configurations**. Whether it is a live path is not answerable from code
  (G2 / B3).

#### `operations-service`

`operations-service` (also known as: Operations Service, `Pacco.Services.Operations`, Docker image
`devmentors/pacco.services.operations`, PM2 app `operations`) projects platform-wide operation
status to clients. Repository: `Pacco.Services.Operations`, path:
`src/Pacco.Services.Operations.Api`.

- **Owning domain:** none — it is a **read-only cross-cutting projection**. It writes no domain
  state and issues no commands.
- **Write model owned:** ambiguous. `appsettings.json` configures `mongo.database:
  operations-service`, but a workspace search finds **no `AddMongoRepository` call anywhere in the
  repository**; `Infrastructure/Extensions.cs:72,75` calls `.AddRedis()` and `requests.expirySeconds`
  is `300`. The effective store is **unknown — requires runtime validation** (G4).
- **Contract surface:** subscribes to **every command, event and rejected event on the platform**
  — 8 exchanges, 26 commands, 30 events, 31 rejected events — driven by
  `src/Pacco.Services.Operations.Api/messages.json`, with subscription types emitted at runtime by
  `Infrastructure/Subscriptions.cs` using `System.Reflection.Emit`. Exposes three protocols:
  HTTP `GET /operations/{operationId}`, the SignalR hub `/pacco` (`Hubs/PaccoHub.cs`), and gRPC
  `GrpcOperationsService.GetOperation` / `.SubscribeOperations` (`Operations.proto`).
- **Notable:** `messages.json` is the platform's **de facto message contract catalogue** and the
  single most decision-bearing file in the workspace — yet it is a hand-maintained copy of names
  owned by eight other repositories, with no generation or validation step. This repository is also
  the only one on the platform with **frontend assets**: `wwwroot/ui/index.html`,
  `wwwroot/ui/js/app.js` and a bundled `wwwroot/ui/js/signalr.js`, with Bootstrap 4.0.0 from CDN,
  no build tooling and no `package.json`.
- **Confidence: high** for the role; **low** for the state store; **unknown** for the wire payloads
  it receives, because the runtime-emitted types are field-less (G5).
- **Port:** `5005`.

#### `Pacco.Services.Operations.GrpcClient`

`Pacco.Services.Operations.GrpcClient` (also known as: Operations gRPC demo client) is a .NET
console application that subscribes to the `operations-service` gRPC stream. Repository:
`Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.GrpcClient`.

- **Owning domain:** none. It is a **sample/diagnostic client, not a platform deployable** — it has
  no `container_name`, no image, no compose entry and no PM2 entry.
- **Dependencies:** `operations-service` over gRPC.
- **Confidence: high** that it exists and is not deployed; its intended audience (demo versus
  operational tooling) is **not observed** in any file.

### 2.6 Platform infrastructure and unclassified

#### `Pacco`

`Pacco` (also known as: Pacco platform root, Pacco solution repository) owns no deployable of its
own and owns every environment definition. Repository: `Pacco`, path: `/` (repository root).

- **Responsibility:** the aggregate solution `Pacco.sln` (which references the other repositories'
  projects by relative path), six Docker Compose stacks under `compose/`, the PM2-style process
  manifests `services.yml` and `prod-services.yml`, the Prometheus scrape config, the RabbitMQ
  image build, and the clone/pull scripts.
- **Deployables defined:** 11 compose services and 10 PM2 apps — enumerated in §7.
- **Backing infrastructure defined:** `mongo`, `rabbitmq`, `redis`, `consul`, `fabio`, `vault`,
  `jaeger`, `seq`, `prometheus`, `grafana` on the `pacco-network` Docker network.
- **Notable:** there is **no Kubernetes, Helm or Terraform anywhere in the workspace**.
  `docker-images.txt` contains five Vault unseal keys and a root token in plaintext (B1).
- **Confidence: high.**

#### `Pacco.Web`

`Pacco.Web` (also known as: Pacco Web) contains **no service and no component**. Repository:
`Pacco.Web`, path: `/` (repository root).

- The clone holds exactly one tracked file, `README.md`, whose entire content is `# Pacco.Web`, on
  a single commit `b3bf026 Initial commit`. There is no `src/`, no `*.csproj`, no `package.json`,
  no `Dockerfile` and no configuration of any kind.
- No other repository references it: it is absent from `Pacco/README.md`'s clone list, from
  `Pacco.sln`, from every `compose/*.yml`, from both PM2 manifests and from every `ntrada*.yml`.
- **Status: Unverifiable — Missing Source Evidence.** Whether a Pacco web client exists outside
  this workspace cannot be determined from the repositories (B2).
- **Confidence: unknown.** No service or component is inferred; inferring one would be a guess.

---

## 3. Repository → service / component table

One row per deployable service or component. Confidence is **high** when the deployable name is
corroborated by at least three independent artefacts (compose `container_name`, Docker image name,
host `*.csproj`, PM2 app name); **medium** when the component exists unambiguously but its role,
reachability or state store is not settled by static reading; **unknown** when no evidence exists.
Evidence paths are repository-relative to the repository in the first column, except where a
`Pacco/` prefix names the platform repository.

| Repo | Inferred service or component | Confidence | Evidence (file path or section) |
|---|---|---|---|
| `Pacco.APIGateway` | `api-gateway` — north-south edge gateway, no domain state | High | `src/Pacco.APIGateway/Pacco.APIGateway.csproj`; `src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml`; `Pacco/compose/services.yml:5-6` (`devmentors/pacco.apigateway`, `container_name: api-gateway`); `Pacco/services.yml` PM2 app `api`; `repo-summary/Pacco.APIGateway.md` |
| `Pacco.Services.Availability` | `availability-service` — bounded context: resource availability and reservation | High | `src/Pacco.Services.Availability.Api/Pacco.Services.Availability.Api.csproj`; `src/Pacco.Services.Availability.Core/Entities/Resource.cs` (aggregate root); `appsettings.json` → `mongo.database: availability-service`, exchange `availability`; `Pacco/compose/services.yml:16-17`; `repo-summary/Pacco.Services.Availability.md` |
| `Pacco.Services.Customers` | `customers-service` — bounded context: customer profile, state, VIP | High | `src/Pacco.Services.Customers.Api/Pacco.Services.Customers.Api.csproj`; `src/Pacco.Services.Customers.Core/Entities/Customer.cs`, `State.cs`; `appsettings.json` → `mongo.database: customers-service`, `security.certificate.acl`; `Pacco/compose/services.yml:25-26`; `repo-summary/Pacco.Services.Customers.md` |
| `Pacco.Services.Deliveries` | `deliveries-service` — bounded context: delivery lifecycle and registrations | High | `src/Pacco.Services.Deliveries.Api/Pacco.Services.Deliveries.Api.csproj`; `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs`, `DeliveryStatus.cs`; `appsettings.json` → `mongo.database: deliveries-service`, exchange `deliveries`; `Pacco/compose/services.yml:34-35`; `repo-summary/Pacco.Services.Deliveries.md` |
| `Pacco.Services.Identity` | `identity-service` — bounded context: authentication, JWT issuance, roles | High | `src/Pacco.Services.Identity.Api/Pacco.Services.Identity.Api.csproj`; `src/Pacco.Services.Identity.Core/Entities/User.cs`, `RefreshToken.cs`, `Role.cs`; `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs`; `Pacco/compose/services.yml:43-44`; `repo-summary/Pacco.Services.Identity.md` |
| `Pacco.Services.Operations` | `operations-service` — cross-cutting read-only operation projection over HTTP, SignalR and gRPC | High (role) / Low (state store) | `src/Pacco.Services.Operations.Api/Pacco.Services.Operations.Api.csproj`; `messages.json` (platform message catalogue); `Infrastructure/Subscriptions.cs` (`Reflection.Emit`); `Hubs/PaccoHub.cs`; `Operations.proto`; `Infrastructure/Extensions.cs:72,75,106` (`AddRedis`, SignalR Redis backplane) with **no `AddMongoRepository` in the repository**; `Pacco/compose/services.yml:52-53` |
| `Pacco.Services.Operations` | `Pacco.Services.Operations.GrpcClient` — console gRPC demo client, **not a platform deployable** | High | `src/Pacco.Services.Operations.GrpcClient/Pacco.Services.Operations.GrpcClient.csproj`; absent from `Pacco/compose/services.yml`, `Pacco/services.yml`, `Pacco/prod-services.yml`; `repo-inventory.md` §2.3 row `Pacco.Services.Operations` |
| `Pacco.Services.OrderMaker` | `ordermaker-service` — cross-context saga orchestrator for order creation, owns no aggregate | Medium | `src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj`; `Sagas/AIOrderMakingSaga.cs`, `Sagas/AIMakingOrderData.cs`; `Extensions.cs:37` (`AddRedis`), `Extensions.cs:43` (`AddChronicle()` with no persistence); `Pacco/compose/services.yml:79-80` (port `5015`); **absent from `Pacco/services.yml`, `Pacco/prod-services.yml` and all four `ntrada*.yml`** |
| `Pacco.Services.Orders` | `orders-service` — bounded context: order aggregate, largest contract surface on the platform | High | `src/Pacco.Services.Orders.Api/Pacco.Services.Orders.Api.csproj`; `src/Pacco.Services.Orders.Core/Entities/Order.cs`, `Parcel.cs`, `OrderStatus.cs`, `Customer.cs`; `Orders.Infrastructure/Services/Clients/{Parcels,Pricing,Vehicles}ServiceClient.cs`; `Pacco/compose/services.yml:70-71`; `repo-summary/Pacco.Services.Orders.md` |
| `Pacco.Services.Parcels` | `parcels-service` — bounded context: parcel catalogue, size/variant, volume | High | `src/Pacco.Services.Parcels.Api/Pacco.Services.Parcels.Api.csproj`; `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`, `Size.cs`, `Variant.cs`; `tests/Pacco.Services.Parcels.PactProviderTests/`; `Pacco/compose/services.yml:88-89`; `repo-summary/Pacco.Services.Parcels.md` |
| `Pacco.Services.Pricing` | `pricing-service` — stateless pricing/discount calculator; **not a bounded context** (no write model, no exchange) | High | `src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj` (the repository's **only** `*.csproj`); `Core/Services/CustomerDiscountsService.cs`; `Queries/Handlers/GetOrderPricingHandler.cs`; `appsettings.json` — **zero occurrences of `rabbitMq`**, no `mongo`, no `redis`; `Pacco/compose/services.yml:97-98` |
| `Pacco.Services.Vehicles` | `vehicles-service` — bounded context: vehicle catalogue and pricing attributes | High | `src/Pacco.Services.Vehicles.Api/Pacco.Services.Vehicles.Api.csproj`; `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs`, `Variants.cs`; `appsettings.json` → `mongo.database: vehicles-service`, exchange `vehicles`; `Pacco/compose/services.yml:106-107`; `repo-summary/Pacco.Services.Vehicles.md` |
| `Pacco` | **No deployable of its own.** Platform-infrastructure and environment-definition repository | High | `Pacco.sln` (references other repositories' projects by relative path only); `compose/infrastructure.yml`, `compose/services.yml`, `compose/services-local.yml`; `services.yml`, `prod-services.yml`; `compose/prometheus/prometheus.yml`; `repo-summary/Pacco.md`. **No `*.csproj` exists in this repository** |
| `Pacco.Web` | **Unknown — no service or component inferred** | Unknown (Unverifiable — Missing Source Evidence) | Entire clone is `README.md` containing the single line `# Pacco.Web`, at commit `b3bf026 Initial commit`; no `src/`, no `*.csproj`, no `package.json`, no `Dockerfile`; `repo-summary/Pacco.Web.md`; not referenced by `Pacco/README.md`, `Pacco.sln`, any `compose/*.yml`, either PM2 manifest, or any `ntrada*.yml` |

**`Pacco.Context`** — the writable artifact repository holding this document — is excluded from the
table by the same rule the inventory applies: it contains no platform source code. Its tracked
content is a one-line `README.md` plus the `docs/architecture-inventory/` artifacts produced by
this discovery. It is also not in the backlog's thirteen-repository scope list.

---

## 4. Bounded-context boundaries and coupling surfaces

**Boundary rule observed in code:** a bounded context owns exactly one RabbitMQ topic exchange and
exactly one MongoDB logical database, and no service reads another's database. Eight contexts
satisfy this (`availability`, `customers`, `deliveries`, `identity`, `operations`, `orders`,
`parcels`, `vehicles`). The exceptions are `pricing-service` (no exchange, no database) and
`ordermaker-service` (an exchange but no database).

**There is no first-party shared library and no shared code repository in the workspace.** No
service references another service's project or a common `Pacco.*` NuGet package. Sharing happens
three other ways, and each is a coupling surface a later stage must reckon with:

| # | Coupling surface | What is shared | Kept consistent by | Risk visible in code |
|---|---|---|---|---|
| C1 | Replicated `customers` collection | `customers-service` owns `CustomerDocument`; `orders-service` and `parcels-service` each keep their own copy (`Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs`, `Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs`) | The `customer_created` event **only** | `customer_state_changed` and `customer_became_vip` are published but handled by no `Events/External/Handlers/` class in any repository, so both replicas drift permanently after creation |
| C2 | Duplicated event contracts | `CustomerCreated` exists as four independent C# classes (Customers, Availability, Orders, Parcels); `ResourceReserved` exists in Availability, Orders and OrderMaker | Nothing — the `snake_case` name on the wire is the only binding contract | A field added by a publisher reaches no consumer until each copy is hand-edited |
| C3 | `messages.json` catalogue | `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` lists 8 exchanges, 26 commands, 30 events, 31 rejected events owned by eight other repositories | Hand maintenance — no generation step, no validation step | A message added elsewhere is silently invisible to `operations-service` until someone edits this file |
| C4 | Shared backing infrastructure | One `mongo`, one `rabbitmq`, one `redis`, one `consul`, one `fabio`, one `vault`, one `jaeger`, one `seq`, one `prometheus`, one `grafana` on `pacco-network` (`Pacco/compose/infrastructure.yml`) | Compose definitions in the `Pacco` repository | Isolation is by database name and Redis key prefix only, so blast radius is platform-wide |
| C5 | Convention-by-template | Every service pins `Convey` `0.4.*` and reproduces the same `Infrastructure/Extensions.cs` composition root, `Decorators/`, `Contexts/`, `Exceptions/`, `Logging/`, `Mongo/` layout and near-identical `appsettings.json` | Copy-paste | Drift is already visible: `parcels-service` has no `AggregateRoot.cs`, `deliveries-service` has no `Convey.WebApi.Security`, `ordermaker-service` has no Jaeger, `pricing-service` has none of the template at all |
| C6 | Pact contract between Orders and Parcels | `tests/Pacco.Services.Orders.PactConsumerTests/PACT/` and `tests/Pacco.Services.Parcels.PactProviderTests/PACT/` | **Unknown** — no Pact Broker configuration exists in either repository or in either `.travis.yml` | If the pact file is copied by hand, the consumer test proves nothing about the provider's current behaviour (Q5) |

---

## 5. Documentation versus source code

Source code is the source of truth for every statement in this document. Three
documentation-versus-code discrepancies were found and are recorded here rather than reconciled.

**D1 — `operations-service` start path. Stale doc; code wins.**

- *Documentation claims:* `Pacco.Services.Operations/README.md:21` — "Service can be started
  locally via `dotnet run` command (executed in the `/src/Pacco.Services.Operations` directory)".
- *Source code shows:* no such directory exists. The only host project is
  `src/Pacco.Services.Operations.Api/Pacco.Services.Operations.Api.csproj`; the sibling project is
  `src/Pacco.Services.Operations.GrpcClient/`.
- *Resolution:* the path in this document is `src/Pacco.Services.Operations.Api`. The README
  instruction does not work as written.

**D2 — `Pacco.Web` scope mismatch. Unverifiable — Missing Source Evidence.**

- *Documentation claims:* the product backlog (issue 12998) lists `Pacco.Web` as repository #2 of
  thirteen in the discovery scope, implying a Pacco web client exists.
- *Source code shows:* `Pacco/README.md:31-46` lists the repositories to clone and **`Pacco.Web` is
  not among them**; the `Pacco.Web` clone itself contains only a one-line `README.md` at commit
  `b3bf026`; and no `compose/*.yml`, PM2 manifest, `ntrada*.yml` or `Pacco.sln` entry references
  it.
- *Resolution:* no service or component is recorded for `Pacco.Web`. This is not reconciled — it is
  raised as blocker B2.

**D3 — the platform README under-describes the platform. Not a contradiction, an omission.**

- *Documentation claims:* `Pacco/README.md` describes microservices, event-driven integration,
  Convey, clean architecture + DDD, and CNCF tooling.
- *Source code shows:* seven substantial architectural facts the README never mentions — the
  gateway's dual sync/async operating modes, Consul discovery, Fabio load balancing, Vault PKI and
  client-certificate service-to-service auth, the transactional outbox, the Chronicle saga in
  `ordermaker-service`, the Pact contract tests between `orders-service` and `parcels-service`, and
  the SignalR/gRPC real-time surface in `operations-service`.
- *Resolution:* every one of the seven is documented in §2 from code. Ten of the twelve
  service/gateway repositories carry a byte-similar boilerplate README that names no module,
  entity, event, endpoint or dependency the repository actually contains, so per-repo READMEs were
  treated as non-evidence throughout.

**On the .NET version — no conflict.** Every README states .NET Core 3.1 and the tree agrees
(`Dockerfile` uses `mcr.microsoft.com/dotnet/core/sdk:3.1`, `.travis.yml` pins `dotnet: 3.1.100`,
publish paths are `netcoreapp3.1`).

**Future/Intended State (Not Implemented).** Nothing in this document describes an intended future
state. Every entry in §2 and §3 reflects code present on `feature/12998/aidlc` at the time of
analysis.

---

## 6. Gaps — unknown or missing summary data

Per-repo summaries exist for **all thirteen in-scope repositories**, so no repository is missing a
summary. The gaps below are areas where the summaries — and the code they were derived from — do
not settle a question. Each is mirrored into the consolidated section at the end of this document.

| # | Repo / area | Gap | Why it is unknown |
|---|---|---|---|
| G1 | `Pacco.Web` | No service or component can be inferred. **Unverifiable — Missing Source Evidence** | The clone holds one file, `README.md`, with the text `# Pacco.Web`, on a single commit. Nothing in the workspace references the repository |
| G2 | `ordermaker-service` | How callers reach it. It is deployed by both Compose stacks (port `5015`) and scraped by Prometheus, but is in **neither** PM2 manifest and in **none** of the four `ntrada*.yml` gateway configurations, so `POST /orders` has no edge route | No caller path exists anywhere in the workspace; whoever invokes it does so in-network by a route not present in these repositories |
| G3 | `ordermaker-service` | Saga state persistence backend | `Extensions.cs:43` calls `AddChronicle()` with no persistence configuration and no `Chronicle.Persistence.*` package. Chronicle's default is in-memory, which would lose in-flight sagas on restart, but nothing states this. **Needs runtime validation** |
| G4 | `operations-service` | Which store holds operation state | `appsettings.json` configures `mongo.database: operations-service`, yet no `AddMongoRepository` call exists anywhere in the repository, while `AddRedis()` is called (`Infrastructure/Extensions.cs:72,75`) and `requests.expirySeconds` is `300`. Static reading cannot settle it |
| G5 | `operations-service` | Wire payloads of the messages it consumes | `Infrastructure/Subscriptions.cs` builds subscription types at runtime with `System.Reflection.Emit` from bare names in `messages.json`; the generated types are field-less. **Unknown — requires runtime capture** |
| G6 | `api-gateway` | Which of the four `ntrada*.yml` configurations is authoritative per environment | The pairs differ architecturally, not by hostname: the async pair converts 20 write routes from HTTP `downstream` to RabbitMQ `publish`. `Pacco/compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml`, but that is a local-development file |
| G7 | `deliveries-service` | What triggers `start_delivery` | The service subscribes to no external event and no service on the platform publishes a `deliveries` command. The only entry point is the gateway's `POST /deliveries` |
| G8 | `customers-service` → all consumers | Consumers of `customer_state_changed` and `customer_became_vip` | Both are published and declared in `messages.json`, but no `Events/External/Handlers/` class in any repository handles either. Only `operations-service` receives them, and only to display them |
| G9 | `vehicles-service` → all consumers | Consumers of `vehicle_added` and `vehicle_updated` | Only `vehicle_deleted` has a subscriber (`availability-service`, a cleanup handler). The other two vehicle events have no consumer in the workspace |
| G10 | All thirteen repositories | Team ownership | There is no `CODEOWNERS`, no `CONTRIBUTING.md` and no team metadata in any clone. The upstream `devmentors/*` origin implies devmentors.io maintenance, but the analysed clones are `hianshul100/*` forks with no ownership record. No finding in this document can be routed to a responsible team |
| G11 | All eight MongoDB-backed services | Schema evolution strategy | No migration tooling of any kind exists in the workspace. The only schema action found is one startup unique-index creation in `Identity.Infrastructure/Mongo/Extensions.cs`. Whether the absence is deliberate is undocumented |
| G12 | `orders-service`, `parcels-service` | How the Pact contract file crosses the repository boundary | Both `PACT/` directories exist and both projects reference `Pactify` 1.1.0, but no Pact Broker is configured in either repository or in either `.travis.yml` |
| G13 | `Pacco`, `Pacco.APIGateway`, `Pacco.Services.Operations` | Whether the committed secrets are live | `Pacco/docker-images.txt` contains five Vault unseal keys and a root token; the same symmetric JWT `issuerSigningKey` is committed in all four `ntrada*.yml` files and in `Operations/appsettings.json`; Seq `apiKey: secret` and RabbitMQ `guest/guest` are committed platform-wide. Whether these are demo values cannot be determined from the repositories |
| G14 | `Pacco.Services.Operations.GrpcClient` | Whether it is a demo or an operational tool | It is a console project with no container, no compose entry and no PM2 entry, and no file in the workspace states its purpose. **Not observed** |

---

## 7. Coverage

**Enumeration method.** .NET has no `pnpm-workspace.yaml` / `nx.json` / `go.work` equivalent, so
three independent manifests were used as the checklist and cross-checked against one another:

1. Every `*.csproj` on disk across the thirteen clones — **40 projects**, matching
   `repo-inventory.md` §7. `Pacco.sln` references the other repositories' projects by relative
   path; those are counted once, under their owning repository.
2. `container_name` entries in `Pacco/compose/services.yml` (identical set in
   `compose/services-local.yml`) — **11 compose deployables**.
3. `name` entries in the PM2 manifests `Pacco/services.yml` and `Pacco/prod-services.yml` —
   **10 PM2 apps**.

### 7.1 Project accounting

| Category | Count | Written up in this document |
|---|---|---|
| Executable host projects (`*.csproj` producing a runnable process) | **12** | **12 / 12** — §2 and §3 |
| Class-library projects (`.Application`, `.Core`, `.Infrastructure`) | 21 | 0 — excluded, see §7.3 |
| Test / contract-test projects | 7 | 0 — excluded, see §7.3 |
| **Total `*.csproj`** | **40** | — |
| Repositories with no `*.csproj` (`Pacco`, `Pacco.Web`) | 2 | **2 / 2** — §2.6 and §3 |
| **Entries written in §3** | — | **14** (12 executable projects + 2 project-less repositories) |

### 7.2 Deployable checklist — every manifest entry accounted for

| # | Compose deployable (`Pacco/compose/services.yml`) | Port | PM2 app | Host project | Written up |
|---|---|---|---|---|---|
| 1 | `api-gateway` | 5000 | `api` | `Pacco.APIGateway/src/Pacco.APIGateway` | §2.1 |
| 2 | `availability-service` | 5001 | `availability` | `…Availability/src/…Availability.Api` | §2.3 |
| 3 | `customers-service` | 5002 | `customers` | `…Customers/src/…Customers.Api` | §2.4 |
| 4 | `deliveries-service` | 5003 | `deliveries` | `…Deliveries/src/…Deliveries.Api` | §2.2 |
| 5 | `identity-service` | 5004 | `identity` | `…Identity/src/…Identity.Api` | §2.1 |
| 6 | `operations-service` | 5005 | `operations` | `…Operations/src/…Operations.Api` | §2.5 |
| 7 | `orders-service` | 5006 | `orders` | `…Orders/src/…Orders.Api` | §2.2 |
| 8 | `ordermaker-service` | 5015 | **absent from both PM2 manifests** | `…OrderMaker/src/Pacco.Services.OrderMaker` | §2.5 |
| 9 | `parcels-service` | 5007 | `parcels` | `…Parcels/src/…Parcels.Api` | §2.2 |
| 10 | `pricing-service` | 5008 | `pricing` | `…Pricing/src/…Pricing.Api` | §2.4 |
| 11 | `vehicles-service` | 5009 | `vehicles` | `…Vehicles/src/…Vehicles.Api` | §2.4 |

**11 / 11 compose deployables written up. 10 / 10 PM2 apps written up.** The single difference
between the two manifests is `ordermaker-service` (G2 / B3).

The twelfth executable project, `Pacco.Services.Operations.GrpcClient`, appears in **no** manifest
and is written up in §2.5 as a component rather than as a deployable — it is documented, not
excluded.

`Pacco/compose/infrastructure.yml` additionally defines ten backing containers (`consul`, `fabio`,
`grafana`, `jaeger`, `mongo`, `prometheus`, `rabbitmq`, `redis`, `seq`, `vault`). These are
infrastructure, not platform deployables; they are covered as coupling surface C4 in §4.

### 7.3 Exclusions

| Excluded | Count | Reason (one line) |
|---|---|---|
| `Pacco.Services.{Availability,Customers,Deliveries,Identity,Orders,Parcels,Vehicles}.{Application,Core,Infrastructure}` | 21 | Class libraries compiled into their sibling `.Api` host — not independently deployable; their contents are summarised under the owning service in §2 |
| `Pacco.Services.Availability.Tests.{Unit,Integration,EndToEnd,Performance,Shared}` | 5 | Test projects, not deployables; this stage explicitly does not generate a test inventory |
| `Pacco.Services.Orders.PactConsumerTests` | 1 | Contract-test project, not a deployable; its architectural significance is recorded as coupling surface C6 |
| `Pacco.Services.Parcels.PactProviderTests` | 1 | Contract-test project, not a deployable; counterpart to the above, recorded as C6 |
| `Pacco.Context` (repository) | 1 repo | The writable artifact repository holding this document; contains no platform source code and is not in the backlog's thirteen-repository scope |

**Every project present in the workspace manifests is either written up in §2/§3 or listed in the
exclusion table above with a reason. There are no unaccounted entries.**

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | A bounded context on this platform is exactly one service that owns one RabbitMQ exchange and one MongoDB database | Eight of the ten services fit this shape exactly, and no service reads another service's database anywhere in the code | If a context actually spans two services, the ten-context split in §2 is wrong and any later decomposition or ownership decision built on it inherits the error | Walk each service's `appsettings.json` and `Infrastructure/Mongo/` with a domain owner and confirm the boundary matches how the business thinks about it |
| A2 | The thirteen repositories in this workspace are the whole Pacco platform | They match the repository list in backlog issue 12998 exactly, and no clone references a repository outside the set | A missing repository would mean an undiscovered service, exchange or data store, and the catalogue in §2 would be incomplete | Confirm the repository list with the platform owner; re-run discovery if any repository is added |
| A3 | Every service runs against one shared MongoDB, RabbitMQ and Redis instance, isolated only by database name and Redis key prefix | `Pacco/compose/infrastructure.yml` defines exactly one container each, and every service's `appsettings.json` points at the same compose hostname | If production uses separate instances per service, coupling surface C4 in §4 and its blast-radius implication are overstated | Compare against the production deployment configuration, which is not in this workspace |
| A4 | `Convey` 0.4.* behaves as its package names describe — transactional outbox, Consul discovery, Fabio balancing, Jaeger tracing | The framework source is not in the workspace; only package references and configuration sections are visible | Any behaviour attributed to Convey in §2 could be wrong, particularly the outbox delivery guarantees claimed for `availability-service` | Read the Convey 0.4 source, or capture broker behaviour at runtime |
| A5 | `messages.json` in `operations-service` is an accurate catalogue of every message on the platform | Cross-checking it against each service's `Events/` and `Commands/` folders found no message present in a service but absent from the catalogue | The contract surfaces listed per service in §2 would be incomplete, and a consumer could be missed | The catalogue is hand-maintained with no generation step — re-verify it whenever a service adds a message |
| A6 | The absence of any migration tooling means schema evolution relies on document-model tolerance | No migration files or tools exist for the eight MongoDB databases; the only schema action found is one startup index creation in `Identity.Infrastructure/Mongo/Extensions.cs` | If an out-of-band migration process exists, the data-ownership picture in §2 is incomplete | Ask the platform owner whether schema changes are applied by a process outside these repositories |
| A7 | `pricing-service` belongs with `customers-service` in a customer-and-commercial grouping | Its only dependency is `customers-service` and its only logic is a function of customer state | If it is really a standalone utility, the grouping in §2.4 misleads any later decomposition or team-allocation decision | Confirm with the domain owner whether pricing rules are owned by the customer domain or independently |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** `Pacco/docker-images.txt` contains five Vault unseal keys and a Vault root token in plaintext, and the same symmetric JWT signing key is committed in all four `ntrada*.yml` gateway configs and in `Operations/appsettings.json`. Nobody on this side can tell whether these are throwaway demo values or credentials that currently protect something real | Any decision to treat these repositories as safe to publish, fork or share, and any security work in later stages | Platform owner / whoever administers the Pacco Vault instance | A person with access to the running Vault must check whether these keys still unseal it. If they do, rotate them and purge the values from git history before anything else proceeds | TBD |
| B2 | **[ACTION NOW]** `Pacco.Web` is an empty repository but is on the discovery scope list. We cannot tell whether a Pacco web client exists somewhere we were not given, or whether the repository is an abandoned placeholder | Completing the service catalogue — if a real web client exists, this document is missing a component, and the gateway's CORS (`allowedOrigins: ['*']` with `allowCredentials: true`) and auth surface has an unexamined consumer | Platform owner | Someone must state whether a Pacco web client exists. If it does, provide the repository and re-run discovery for it; if it does not, drop `Pacco.Web` from the scope list | TBD |
| B3 | **[ACTION NOW]** `ordermaker-service` is started by both Compose stacks and scraped by Prometheus, but sits behind no gateway route and appears in neither PM2 process manifest. How anything reaches its `POST /orders` is not answerable from the code | Deciding whether the order-creation saga described in §2.5 is a live path or dead code — this changes how order creation should be described and whether it needs governing at all | Platform owner / operations | Someone must state how `POST /orders` on `ordermaker-service` is invoked given there is no gateway route, and whether its omission from `services.yml` and `prod-services.yml` is deliberate | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is `api-gateway` meant to run in synchronous or asynchronous mode in production? | The two modes are architecturally different systems. In async mode 20 write endpoints stop returning a result and become fire-and-forget RabbitMQ publishes, so clients must poll `operations-service` instead. Every client contract depends on the answer | `Pacco/compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml`, so async appears to be the intended default — but that is a local-development file, not a production one | Platform architect |
| Q2 | **[ACTION NOW]** `orders-service` and `parcels-service` each keep their own `customers` collection, populated once from `customer_created` and never updated. Is that intentional? | `customer_state_changed` and `customer_became_vip` are published but consumed by nobody, so both copies drift from `customers-service` as customers change state. If pricing or eligibility ever reads a stale copy, customers get wrong outcomes | Likely a deliberate "only the creation snapshot matters" choice, but nothing in the code says so | Domain owner for Orders and Parcels |
| Q3 | **[handled later by HLD]** Where does `ordermaker-service` keep its saga state? | `AddChronicle()` is called with no persistence backend and no persistence package, which normally means in-memory. An in-flight order saga would then be lost on restart or on any second instance | Confirm the Chronicle default and either document in-memory as accepted or add a persistence backend | Platform architect |
| Q4 | **[handled later by HLD]** Does `operations-service` store operation state in MongoDB or in Redis? | It configures both, calls no `AddMongoRepository`, and sets a 300-second Redis expiry. If Redis with a five-minute expiry is the real store, any saga or workflow running longer than five minutes loses its status before it finishes | Read `Services/OperationsService.cs` end to end and confirm at runtime | Platform architect |
| Q5 | **[ACTION NOW]** How does the Pact contract file travel between `orders-service` (consumer) and `parcels-service` (provider)? | Both repositories have a `PACT/` directory and `Pactify` 1.1.0, but no Pact Broker is configured in either repository or either Travis pipeline. If the file is copied by hand, the contract test proves nothing about the other side's current behaviour | Someone who has run these tests must say whether a broker exists outside the repositories | Whoever owns the Orders/Parcels contract tests |
| Q6 | **[ACTION NOW]** Who or what triggers `start_delivery`? | `deliveries-service` subscribes to no event and no service publishes a `deliveries` command — the only entry point is a human calling `POST /deliveries` through the gateway. If an external dispatch system is meant to drive it, that integration is entirely undiscovered | Confirm whether deliveries are started manually or by a system not present in this workspace | Domain owner for Deliveries |
| Q7 | **[handled later by HLD]** Should the duplicated event contracts become a shared package? | `CustomerCreated` exists as four independent classes and `ResourceReserved` as three; the only real contract is the `snake_case` name on the wire. A field added by a publisher reaches no consumer until each copy is hand-edited | Record the current state as an explicit choice, or introduce a shared contracts package — this is an evolution decision, not a discovery finding | Platform architect |
| Q8 | **[ACTION NOW]** Who owns each service? | There is no `CODEOWNERS`, no `CONTRIBUTING.md` and no team metadata anywhere in the thirteen clones, so the ownership column of this catalogue is empty and no finding here can be routed to a responsible team | Supply an owner per service, or per grouping as organised in §2 | Delivery lead |
| Q9 | **[ACTION NOW]** Is the `customers-service` certificate ACL enforced at runtime, or is it advisory configuration? | `security.certificate.acl` grants `availability-service` the `customers:read` permission with `allowedDomains: ['pacco.io']`. If it is enforced, it is a real trust boundary that later designs must honour; if it is inert, `customers-service` is effectively open to any authenticated caller | The package `Convey.WebApi.Security` is referenced by `availability-service` and `customers-service` but not by `deliveries-service`, which suggests enforcement is real but uneven | Platform architect |
| Q10 | **[ACTION NOW]** Are `vehicle_added` and `vehicle_updated` meant to have consumers? | `vehicles-service` publishes three events; only `vehicle_deleted` has a subscriber, and only as a cleanup handler in `availability-service`. If a vehicle's attributes change, no other context learns about it, so `availability-service` and `orders-service` can act on stale vehicle data | Either these events are speculative and should be documented as such, or a consumer is missing | Domain owner for Vehicles |
| Q11 | **[handled later by HLD]** Is `Pacco.Services.Operations.GrpcClient` a throwaway demo or an operational tool people rely on? | It is the only executable project in the workspace with no container image and no manifest entry. If anyone depends on it operationally, it needs a deployment story; if not, it is sample code that should be labelled as such | Nothing in the workspace states its purpose — treat it as a demo until someone says otherwise | Platform architect |
| Q12 | **[ACTION NOW]** How do the eight MongoDB databases evolve their schemas? | No migration tooling of any kind exists. If schema changes are applied by an out-of-band process nobody has described, later data-model work will be built on an incomplete picture | Likely reliance on document-model tolerance, but this is inferred from absence, not stated anywhere | Platform owner |
| Q13 | **[handled later by HLD]** Should `pricing-service` be treated as part of a customer-and-commercial domain, or as a standalone utility? | It shares none of the platform's conventions — no broker, no database, no clean-architecture split, no `LICENSE`. The grouping affects how it is owned, deployed and evolved | Its only dependency and only logic are customer-driven, which argues for the grouping; its shape argues against it | Platform architect |
| Q14 | **[handled later by HLD]** What are the actual wire payloads of the messages `operations-service` consumes? | `Infrastructure/Subscriptions.cs` emits field-less types at runtime from bare names in `messages.json`, so the payloads it receives cannot be read from the code. Any later work that projects, stores or displays message content needs the real shapes | Capture the messages off RabbitMQ at runtime; the publisher-side DTOs in each service's `Events/` folder are the likely ground truth | Platform architect |
