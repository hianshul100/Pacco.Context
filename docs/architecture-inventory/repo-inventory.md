# Pacco — Architecture Inventory

**Project:** Common Architecture
**Scope:** the Pacco platform as checked out in this workspace — 13 in-scope repositories.
**Branch inspected in every repository:** `feature/13106/aidlc`.
**Nature of this document:** an evidence-based inventory of what exists. It records findings, not recommendations, and contains no architecture decision records.

---

## Contents

1. [Scope and method](#1-scope-and-method)
2. [Repository inventory](#2-repository-inventory)
3. [Cross-repository relationships](#3-cross-repository-relationships)
4. [Suspected platform subsystems](#4-suspected-platform-subsystems)
5. [Documentation versus tree — platform-level patterns](#5-documentation-versus-tree--platform-level-patterns)
6. [Gaps and unknowns](#6-gaps-and-unknowns)
7. [Coverage](#7-coverage)
8. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Scope and method

### 1.1 What was inspected

Fourteen repositories are present in the workspace. Thirteen are in scope for this inventory. The fourteenth, `hianshul100_Pacco.Context`, is the destination for these documents and is deliberately excluded from the inventory table, from the relationship map and from the per-repository summaries.

| # | Repository | Role | Per-repository summary |
|---|---|---|---|
| 1 | `hianshul100_Pacco` | Composition and infrastructure definitions; no deployable | [`repo-summary/hianshul100_Pacco.md`](repo-summary/hianshul100_Pacco.md) |
| 2 | `hianshul100_Pacco.APIGateway` | `api-gateway` | [`repo-summary/hianshul100_Pacco.APIGateway.md`](repo-summary/hianshul100_Pacco.APIGateway.md) |
| 3 | `hianshul100_Pacco.Services.Availability` | `availability-service` | [`repo-summary/hianshul100_Pacco.Services.Availability.md`](repo-summary/hianshul100_Pacco.Services.Availability.md) |
| 4 | `hianshul100_Pacco.Services.Customers` | `customers-service` | [`repo-summary/hianshul100_Pacco.Services.Customers.md`](repo-summary/hianshul100_Pacco.Services.Customers.md) |
| 5 | `hianshul100_Pacco.Services.Deliveries` | `deliveries-service` | [`repo-summary/hianshul100_Pacco.Services.Deliveries.md`](repo-summary/hianshul100_Pacco.Services.Deliveries.md) |
| 6 | `hianshul100_Pacco.Services.Identity` | `identity-service` | [`repo-summary/hianshul100_Pacco.Services.Identity.md`](repo-summary/hianshul100_Pacco.Services.Identity.md) |
| 7 | `hianshul100_Pacco.Services.Operations` | `operations-service` | [`repo-summary/hianshul100_Pacco.Services.Operations.md`](repo-summary/hianshul100_Pacco.Services.Operations.md) |
| 8 | `hianshul100_Pacco.Services.OrderMaker` | `ordermaker-service` | [`repo-summary/hianshul100_Pacco.Services.OrderMaker.md`](repo-summary/hianshul100_Pacco.Services.OrderMaker.md) |
| 9 | `hianshul100_Pacco.Services.Orders` | `orders-service` | [`repo-summary/hianshul100_Pacco.Services.Orders.md`](repo-summary/hianshul100_Pacco.Services.Orders.md) |
| 10 | `hianshul100_Pacco.Services.Parcels` | `parcels-service` | [`repo-summary/hianshul100_Pacco.Services.Parcels.md`](repo-summary/hianshul100_Pacco.Services.Parcels.md) |
| 11 | `hianshul100_Pacco.Services.Pricing` | `pricing-service` | [`repo-summary/hianshul100_Pacco.Services.Pricing.md`](repo-summary/hianshul100_Pacco.Services.Pricing.md) |
| 12 | `hianshul100_Pacco.Services.Vehicles` | `vehicles-service` | [`repo-summary/hianshul100_Pacco.Services.Vehicles.md`](repo-summary/hianshul100_Pacco.Services.Vehicles.md) |
| 13 | `hianshul100_Pacco.Web` | Empty placeholder; no deployable | [`repo-summary/hianshul100_Pacco.Web.md`](repo-summary/hianshul100_Pacco.Web.md) |

**Deployables:** 11 — `api-gateway` and ten services. Two repositories produce nothing.

### 1.2 How each repository was examined

Every repository was read twice and then reconciled.

- **Documentation pass** — root `README.md`, `CONTRIBUTING.md`, `docs/` and `CHANGELOG.md`. Across all thirteen repositories, **only `README.md` ever exists**. There is no `CONTRIBUTING.md`, no `CHANGELOG.md` and no `docs/` directory anywhere in the workspace. The three architecture diagrams in `hianshul100_Pacco/assets/` were opened and read as part of this pass; they are the only documentation artefact in the workspace that is not a `README.md`.
- **Repository pass** — top-level directories, solution and project manifests (`.sln`, `.csproj`), deployment clues (`Dockerfile`, Docker Compose files, continuous-integration configuration), and process entrypoints.
- **Reconciliation** — recorded in a "README vs repository" section in every per-repository summary, and summarised platform-wide in [section 5](#5-documentation-versus-tree--platform-level-patterns).

### 1.3 Source of truth

Source code and configuration on disk are authoritative. Where a document and the code disagree, the code is reported as the fact and the document's claim is recorded alongside it as a claim. Nothing has been silently reconciled. Two labels are used where they apply:

- **Future/Intended State (Not Implemented)** — a document describes something the code does not do.
- **Unverifiable — Missing Source Evidence** — a repository or component referenced somewhere is not present in the workspace.

### 1.4 The platform in one paragraph

Pacco is a .NET Core 3.1 microservices platform for exclusive parcel delivery, built around the idea of limited resource availability. Eleven deployables run behind a configuration-driven API gateway. Reads go over HTTP; almost every write is turned into a RabbitMQ message on a topic exchange and answered asynchronously, with the outcome pushed back to the client over SignalR. Each service owns its own MongoDB database. Consul provides discovery, Fabio load balancing, Vault secrets and dynamic database credentials, and Jaeger, Prometheus, Grafana and Seq provide observability. Every service is built on the same shared library, **Convey** `0.4.*`, which supplies the command and query dispatchers, the message broker abstraction, the transactional outbox and all of the cross-cutting wiring.

---

## 2. Repository inventory

One row per in-scope repository. Because fourteen dimensions do not fit in a readable table, the single logical table is presented as three panels — 2.1 covers dimensions 1 to 5, 2.2 covers 6 to 9, and 2.3 covers 10 to 14. Row order is identical in all three.

### 2.1 Panel A — purpose, runtime, entrypoints, modules, integrations

| Repository | 1. Primary purpose | 2. Runtime / service type | 3. Key entrypoints | 4. Important modules | 5. External integrations |
|---|---|---|---|---|---|
| `hianshul100_Pacco` | Composition, infrastructure definitions, clone and run scripts | None — no deployable | `compose/infrastructure.yml`, `compose/services.yml`, `scripts/git-clone.sh`, `Pacco.sln`, `services.yml` | `compose/prometheus/`, `compose/rabbitmq/`, `docker-images.txt` | Declares Consul, Fabio, Grafana, Jaeger, MongoDB, Prometheus, RabbitMQ, Redis, Seq, Vault |
| `hianshul100_Pacco.APIGateway` | Public entry point; routes or converts every client request | ASP.NET Core 3.1 on Ntrada `0.4.*` | `src/Pacco.APIGateway/Program.cs`, `ntrada.yml`, `ntrada-async.yml` | `Infrastructure/{CorrelationContextBuilder,SpanContextBuilder,HttpRequestHook}.cs` | RabbitMQ, Jaeger, Seq, Prometheus; all ten services as HTTP downstreams |
| `hianshul100_Pacco.Services.Availability` | Owns resources and their reservations | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Availability.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure` + 5 test projects | `customers-service` over HTTP; RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger |
| `hianshul100_Pacco.Services.Customers` | Owns customer profile and state lifecycle | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Customers.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure` | RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger; no outbound HTTP |
| `hianshul100_Pacco.Services.Deliveries` | Tracks delivery from start to completion or failure | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Deliveries.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure` | RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger; no outbound HTTP |
| `hianshul100_Pacco.Services.Identity` | Authentication authority; issues every JWT | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Identity.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure`; `JwtProvider`, `PasswordService`, `RefreshTokenService` | RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger; no outbound HTTP |
| `hianshul100_Pacco.Services.Operations` | Tracks every operation and pushes outcomes to clients | ASP.NET Core 3.1 on Convey; HTTP + SignalR + gRPC + RabbitMQ consumer | `src/Pacco.Services.Operations.Api/Program.cs`, `Infrastructure/Subscriptions.cs`, `messages.json`, `Operations.proto` | `.Api` and `.GrpcClient` only — no domain layering | RabbitMQ (all exchanges), Redis, MongoDB (configured, unused), Consul, Fabio, Jaeger, Vault (kv + PKI + dynamic Mongo credentials for a database it never writes to) |
| `hianshul100_Pacco.Services.OrderMaker` | Saga orchestrator for end-to-end order creation | ASP.NET Core 3.1 on Convey + Chronicle `3.2.1` | `src/Pacco.Services.OrderMaker/Program.cs`, `Sagas/AIOrderMakingSaga.cs` | Single project; `AvailabilityServiceClient`, `VehiclesServiceClient`, `ResourceReservationsService` | `availability-service`, `vehicles-service` over HTTP; RabbitMQ, Redis, Consul, Fabio; **no Mongo, Vault or Jaeger** |
| `hianshul100_Pacco.Services.Orders` | Owns the order aggregate and lifecycle | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Orders.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure` + `PactConsumerTests` | `parcels-service`, `pricing-service`, `vehicles-service` over HTTP; RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger |
| `hianshul100_Pacco.Services.Parcels` | Owns parcels and computes parcel volume | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Parcels.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure` + `PactProviderTests` | RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger; no outbound HTTP |
| `hianshul100_Pacco.Services.Pricing` | Calculates order price with customer discount | ASP.NET Core 3.1 on Convey; HTTP only | `src/Pacco.Services.Pricing.Api/Program.cs` | **Single project**; domain nested at `src/Pacco.Services.Pricing.Api/Core/` | `customers-service` over HTTP; Consul, Fabio, Jaeger, Prometheus, Seq, Vault (kv + PKI only — the one Vault user with no `lease` block). **No RabbitMQ, MongoDB or Redis** |
| `hianshul100_Pacco.Services.Vehicles` | Owns the vehicle fleet catalogue | ASP.NET Core 3.1 on Convey; HTTP + RabbitMQ consumer | `src/Pacco.Services.Vehicles.Api/Program.cs` | `.Api`, `.Application`, `.Core`, `.Infrastructure` | RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger; no outbound HTTP |
| `hianshul100_Pacco.Web` | **Unknown** — empty placeholder | None | None — one file, `README.md` | None | None |

### 2.2 Panel B — data, messaging, APIs, deployment

| Repository | 6. Data stores and state | 7. Messaging, async, events | 8. APIs exposed and consumed | 9. Deployment and runtime |
|---|---|---|---|---|
| `hianshul100_Pacco` | None. Declares MongoDB (`27017`, volume `mongo:/data/db`) and Redis (`6379`). No ORM, no migration tool | Declares the RabbitMQ broker (`5672`, `15672`, `15692`). No exchanges or events defined here | None | Compose files only; images `devmentors/pacco.*`; network `pacco-network`. No continuous integration |
| `hianshul100_Pacco.APIGateway` | **None** — stateless. No ORM, no migration tool, no collections | RabbitMQ topic exchanges; publishes 20 routing keys across 6 exchanges. Config: `connectionName api-gateway`, `message_context`, `span_context` | Exposes 10 public modules (`availability`, `customers`, `deliveries`, `identity`, `operations`, `orders`, `parcels`, `pricing`, `vehicles`, `home`); Swagger at `docs`. Consumes all 10 services | Docker `sdk:3.1` → `aspnet:3.1`; `NTRADA_CONFIG=ntrada.docker`; Travis; image `devmentors/pacco.apigateway` |
| `hianshul100_Pacco.Services.Availability` | MongoDB `availability-service`; collection `resources`; `outbox`/`inbox`; Redis prefix `availability:`. Convey `IMongoRepository`, no ORM, no migrations | Exchange `availability`. Consumes 4 commands + `customer_created`, `vehicle_deleted`. Publishes `resource_added`, `resource_deleted`, `resource_reserved`, `resource_reservation_released`, `resource_reservation_canceled` + 4 rejections | 7 routes under `resources`; Swagger at `docs`. Consumes `customers-service` | Port `5001`; Travis; image `devmentors/pacco.services.availability` |
| `hianshul100_Pacco.Services.Customers` | MongoDB `customers-service`; collection `customers`; `outbox`/`inbox`; Redis prefix `customers:`. **Source of the `customers` replicas in two other databases** | Exchange `customers`. Consumes `complete_customer_registration`, `change_customer_state`, `signed_up`, `order_completed`. Publishes `customer_created`, `customer_became_vip`, `customer_state_changed` + 2 rejections | 5 routes under `customers`; the 3 read routes require `role: admin` at the gateway | Port `5002`; Travis; image `devmentors/pacco.services.customers` |
| `hianshul100_Pacco.Services.Deliveries` | MongoDB `deliveries-service`; collection `deliveries`; `outbox`/`inbox`; Redis prefix `deliveries:` | Exchange `deliveries`. Consumes 4 commands, **no events**. Publishes `delivery_started`, `delivery_completed`, `delivery_failed`, `registration_added_to_delivery` + 3 rejections | 5 routes under `deliveries`; only `GET deliveries/{deliveryId}` is public | Port `5003`; Travis; image `devmentors/pacco.services.deliveries` |
| `hianshul100_Pacco.Services.Identity` | MongoDB `identity-service`; collections `users`, `refreshTokens`; `outbox`/`inbox`; Redis prefix `identity:` holds revoked access tokens | Exchange `identity`. Consumes `sign_up` only. Publishes `signed_up`, `signed_in` + 2 rejections. `sign_in` is catalogued but unsubscribed | 7 routes via plain `UseEndpoints`; only `sign-up` and `sign-in` are public, both `auth: false` | Port `5004`; Travis; image `devmentors/pacco.services.identity` |
| `hianshul100_Pacco.Services.Operations` | **Redis is the store** — `IDistributedCache`, `requests.expirySeconds: 300`. MongoDB `operations-service` is configured but **no repository is registered and nothing is written**. No outbox | Subscribes to **every** command, event and rejection on all 8 exchanges, from `messages.json`, with types generated at runtime by `System.Reflection.Emit`. **Publishes nothing** | HTTP `GET operations/{operationId}` (public, `auth: false`); SignalR hub `/pacco`; gRPC `GrpcOperationsService.GetOperation` and `SubscribeOperations` | Port `5005`; Travis; image `devmentors/pacco.services.operations`; Redis SignalR backplane allows horizontal scaling |
| `hianshul100_Pacco.Services.OrderMaker` | **No MongoDB.** Redis registered — presumed Chronicle saga store. **No outbox** | Exchange `ordermaker`. Consumes `order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved`, `resource_reserved`. Publishes `make_order_completed` + `make_order_rejected`, and commands into 2 other exchanges. `Saga` header carries `SagaStates` | `GET ""`, `POST orders`. **Not exposed through `api-gateway`.** Consumes `vehicles-service`, `availability-service` | Port `5015` (outside the `5001`–`5009` block); Travis; image `devmentors/pacco.services.ordermaker` |
| `hianshul100_Pacco.Services.Orders` | MongoDB `orders-service`; collections `orders` **and a `customers` replica**; `outbox`/`inbox`; Redis prefix `orders:` | Exchange `orders`. **14 subscriptions** (7 commands, 7 events). Publishes 9 events + 10 rejections — the largest message surface on the platform | 7 routes under `orders`; `GET orders?customerId=@user_id` is public. Consumes `parcels-service`, `pricing-service`, `vehicles-service`. PACT **consumer** of `parcels-service` | Port `5006`; Travis; image `devmentors/pacco.services.orders` |
| `hianshul100_Pacco.Services.Parcels` | MongoDB `parcels-service`; collections `parcels` **and a `customers` replica**; `outbox`/`inbox`; Redis prefix `parcels:` | Exchange `parcels`. Consumes `add_parcel`, `delete_parcel`, `customer_created`, `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order`. Publishes `parcel_added`, `parcel_deleted` + 2 rejections | 5 routes under `parcels`; `GET parcels?customerId=@user_id` and `GET parcels/volume` are public. PACT **provider** for `orders-service` | Port `5007`; Travis; image `devmentors/pacco.services.parcels`; provider test needs a live MongoDB |
| `hianshul100_Pacco.Services.Pricing` | **None.** No MongoDB, no Redis, no outbox. No ORM, no collections, no migrations | **None.** No broker, no exchange, no subscriptions. The only service absent from `messages.json` — correctly so | `GET pricing` only. Public as `GET pricing?customerId=@user_id`. Consumes `customers-service` | Port `5008`; Travis; image `devmentors/pacco.services.pricing`; **no `LICENSE`**; committed `.idea/` folder |
| `hianshul100_Pacco.Services.Vehicles` | MongoDB `vehicles-service`; collection `vehicles`; `outbox`/`inbox`; Redis prefix `vehicles:` | Exchange `vehicles`. Consumes `add_vehicle`, `update_vehicle`, `delete_vehicle`, **no events**. Publishes `vehicle_added`, `vehicle_updated`, `vehicle_deleted` + 3 rejections — the most symmetric set on the platform | 5 routes under `vehicles`; `GET vehicles` returns `PagedResult<VehicleDto>`, the only paged endpoint. Read by `orders-service` and `ordermaker-service` | Port `5009`; Travis; image `devmentors/pacco.services.vehicles`; **no `LICENSE`** |
| `hianshul100_Pacco.Web` | None | None | None | None — no `Dockerfile`, no continuous integration, absent from every compose file |

### 2.3 Panel C — security, observability, decisions, questions, frontend

| Repository | 10. Security and auth | 11. Observability | 12. Decision files and feature flags | 13. Open questions | 14. Frontend |
|---|---|---|---|---|---|
| `hianshul100_Pacco` | Vault dev mode, `VAULT_DEV_ROOT_TOKEN_ID=secret`; PKI roles for `availability-service` and `customers-service` with `allowed_domains=pacco.io`; sample unseal keys in `docker-images.txt` | Declares Jaeger, Prometheus, Grafana, Seq | `README.md`, `compose/infrastructure.yml`, `Pacco.sln`, and `assets/pacco_overview.png`, `clean_architecture.png`, `infrastructure.png` — the platform's only diagrams. **No feature flags** | Dangling `Pacco.APIGateway.Ocelot` reference; unused relational databases and ELK in `docker-images.txt`; the diagrams omit `ordermaker-service`, draw `pricing-service` on RabbitMQ, and advertise Kubernetes, Istio and Rancher that do not exist | None detected — checked `/`, `assets/`, `compose/`, `scripts/` |
| `hianshul100_Pacco.APIGateway` | JWT `validIssuer: pacco`, role claim on `customers` reads; **hardcoded `issuerSigningKey`**; CORS `*`; committed `certs/localhost.cer`; `sign-up`, `sign-in`, `operations/{id}` are `auth: false` | Jaeger `serviceName: api-gateway`; correlation and span builders; Seq; Prometheus | `ntrada.yml`, `ntrada-async.yml`, `Program.cs`, `Pacco-sample-scenario.rest`. **No feature flags** | `loadBalancer.enabled: false` while Consul and Fabio are configured; sync vs async deployment shape | None detected — checked `/`, `src/`, `Infrastructure/`, `Properties/`, `certs/`, `scripts/` |
| `hianshul100_Pacco.Services.Availability` | **Certificate authentication** (`AddCertificateAuthentication`); granted `customers:read` by `customers-service`; Vault kv + PKI + dynamic Mongo credentials; `vault.token: "secret"` committed | Jaeger `availability` + Jaeger decorators; **`MetricsJob` and `CustomMetricsMiddleware` — unique to this service**; Seq; Prometheus | `Infrastructure/Extensions.cs` — the platform's canonical composition root. **No feature flags** | Purpose of the unique metrics components; performance-test thresholds | None detected — checked `/`, `src/`, all four projects, `tests/`, `scripts/` |
| `hianshul100_Pacco.Services.Customers` | **The platform's only access-control list** — grants `availability-service` the permission `customers:read`; `certificate.enabled`, `allowedDomains ["pacco.io"]`; Vault kv + PKI + dynamic Mongo credentials | Jaeger `customers`; Seq; Prometheus; handler logging | `appsettings.json` `security` block; `Infrastructure/Extensions.cs`. **No feature flags** | `pricing-service` calls this service but is absent from the access-control list; VIP promotion rule undocumented | None detected — checked `/`, `src/`, all four projects, `scripts/` |
| `hianshul100_Pacco.Services.Deliveries` | JWT only; no access-control list, no certificate authentication; Vault kv + PKI + dynamic Mongo credentials | Jaeger `deliveries`; Seq; Prometheus; handler logging | `Infrastructure/Extensions.cs`, `Program.cs`. **No feature flags** | `add_delivery_registration` has no rejection event; delivery state machine undocumented | None detected — checked `/`, `src/`, all four projects, `scripts/` |
| `hianshul100_Pacco.Services.Identity` | **Security root of the platform.** Shared `issuerSigningKey`; `certificate.location: certs/localhost.pfx` with password `test`; **`localhost.pfx`, `.key`, `.pem`, `.cer` all committed**; `validateIssuer: false`; `expiryMinutes: 60`; revocation via `UseAccessTokenValidator` | Jaeger `identity`; Seq; Prometheus; redaction list covers `Password`, `Email`, `Login`, `Token` | `appsettings.json` `jwt` block; `Infrastructure/Extensions.cs`. **No feature flags** | No public refresh-token route at the gateway; `sign_in` catalogued but unsubscribed; `validateIssuer: false` | None detected — checked `/`, `src/`, all four projects, `certs/`, `scripts/` |
| `hianshul100_Pacco.Services.Operations` | Shared `issuerSigningKey`; SignalR authenticates by passing a JWT as a hub argument; **`GET operations/{id}` is public with `auth: false`**; Vault kv + PKI + dynamic Mongo credentials (`Program.cs:50` `.UseVault()`); `vault.token: "secret"` committed | Jaeger `operations` — the broadest trace coverage on the platform; correlation context round-tripped as JSON; Seq; Prometheus | `Infrastructure/Subscriptions.cs` (runtime type generation), `messages.json`, `Operations.proto`, `Services/OperationsService.cs`. **No feature flags** | MongoDB configured but unused; 300-second expiry on operation records; unauthenticated read of user-attributable data; `messages.json` drift | **Yes — the only frontend in the workspace.** `wwwroot/ui/`: vanilla JavaScript, SignalR client `1.1.0`, Bootstrap `4.0.0` from CDN. No `package.json`, no build tooling |
| `hianshul100_Pacco.Services.OrderMaker` | **No Vault, no JWT validation, no access-control list.** Not reachable through the gateway; publishes commands with only broker credentials | **No Jaeger at all** — the one service that spans five others is invisible to tracing. Seq; Prometheus; `Saga` header carries saga state | `Sagas/AIOrderMakingSaga.cs` — the most decision-dense file in the workspace; `Extensions.cs`. **No feature flags** | Where Chronicle stores saga state; compensation only for one step; no outbox; "AI" naming with no AI code | None detected — checked `/`, `src/`, `Sagas/`, `scripts/` |
| `hianshul100_Pacco.Services.Orders` | JWT only; customer scoping applied by the gateway, not the service; Vault kv + PKI + dynamic Mongo credentials | Jaeger `orders`; forwards the `Saga` header; Seq; Prometheus; handler logging across 14 handlers | `Infrastructure/Extensions.cs`, `Program.cs`, `PACT/ParcelsApiPactConsumerTests.cs`. **No feature flags** | PACT test missing from the repository's own solution; order state machine undocumented; nothing consumes `order_delivering` | None detected — checked `/`, `src/`, all four projects, `tests/`, `scripts/` |
| `hianshul100_Pacco.Services.Parcels` | JWT only; customer scoping applied by the gateway; Vault kv + PKI + dynamic Mongo credentials; a second `appsettings.json` inside the test project | Jaeger `parcels`; Seq; Prometheus; handler logging | `Infrastructure/Extensions.cs`, `Program.cs`, `PACT/ParcelsApiPactProviderTests.cs` + Mongo fixtures. **No feature flags** | PACT provider test missing from the repository's own solution; scope of `parcels/volume`; fragile route ordering | None detected — checked `/`, `src/`, all four projects, `tests/`, `scripts/` |
| `hianshul100_Pacco.Services.Pricing` | JWT only; Vault kv + PKI, no `lease` block (`Program.cs:33` `.UseVault()`); `vault.token: "secret"` committed; calls `customers-service` without a client certificate and is absent from its access-control list | Jaeger `pricing` (HTTP traces only); Seq; Prometheus. **No handler logging** | `Infrastructure/Extensions.cs` (notable for what it omits), `Core/Services/CustomerDiscountsService.cs`. **No feature flags** — and this is the service most likely to have wanted them | Whether its `customers-service` calls are authorised; discount rule undocumented; single-project structure | None detected — checked `/`, `src/`, `Core/`, `Infrastructure/`, `Services/`, `scripts/` |
| `hianshul100_Pacco.Services.Vehicles` | JWT only; **no role restriction on fleet mutations at the gateway**; Vault kv + PKI + dynamic Mongo credentials | Jaeger `vehicles`; forwards the `Saga` header; Seq; Prometheus; handler logging | `Infrastructure/Extensions.cs`, `Program.cs`. **No feature flags** | Which endpoint `GetBestAsync` calls and how vehicles are ranked; nothing consumes `vehicle_added` or `vehicle_updated` | None detected — checked `/`, `src/`, all four projects, `scripts/` |
| `hianshul100_Pacco.Web` | None — the only repository with no credentials of any kind | None | None. **No feature flags** | What it was meant to be; whether it is in scope at all | None detected — checked `/`, the only directory present. **A repository named `.Web` contains no web assets** |

---

## 3. Cross-repository relationships

Three distinct integration mechanisms are in use. They are described separately because they fail differently and are operated differently.

### 3.1 Synchronous HTTP calls

Nine service-to-service call paths exist. Every one goes through Fabio with 3 retries and request masking (`maskTemplate: "*****"`), configured in each caller's `httpClient` block.

| Caller | Callee | Declared in | Purpose |
|---|---|---|---|
| `api-gateway` | all 10 services | `src/Pacco.APIGateway/ntrada.yml` `services:` block | Proxying every public read |
| `availability-service` | `customers-service` | `httpClient.services: {customers: customers-service}` | `CustomersServiceClient` |
| `pricing-service` | `customers-service` | `httpClient.services: {customers: customers-service}` | `CustomersServiceClient`, for discounts |
| `orders-service` | `parcels-service` | `httpClient.services: {parcels: parcels-service}` | Parcel lookup — **the only contract-tested path** |
| `orders-service` | `pricing-service` | `httpClient.services: {pricing: pricing-service}` | Order pricing |
| `orders-service` | `vehicles-service` | `httpClient.services: {vehicles: vehicles-service}` | Vehicle lookup |
| `ordermaker-service` | `availability-service` | `httpClient.services: {availability: availability-service}` | `IResourceReservationsService.GetBestAsync` |
| `ordermaker-service` | `vehicles-service` | `httpClient.services: {vehicles: vehicles-service}` | `IVehiclesServiceClient.GetBestAsync` |

`customers-service`, `deliveries-service`, `identity-service`, `operations-service`, `parcels-service` and `vehicles-service` make no outbound HTTP calls at all — their `httpClient.services` maps are empty.

**Observation.** `orders-service` has the widest synchronous fan-out at three dependencies. `customers-service` has the widest synchronous fan-in at three callers, of which only one (`availability-service`) is named in its access-control list.

### 3.2 Asynchronous messaging

Every message travels over a RabbitMQ **topic** exchange with `conventionsCasing: snakeCase`, durable exchanges, the `message_context` header and the `span_context` header. Each service owns one exchange named after its domain, and binds queues using the template `<service>/{{exchange}}.{{message}}`.

Eight exchanges carry traffic: `availability`, `customers`, `deliveries`, `identity`, `orders`, `ordermaker`, `parcels`, `vehicles`. `pricing-service` has no exchange.

**Who publishes commands.** `api-gateway` in async mode is the primary command publisher, turning 20 mutating HTTP routes into published messages across six exchanges. `ordermaker-service` is the only other command publisher, and it publishes into exchanges it does not own.

**Event flows between services**, read from the `SubscribeEvent<>` calls in each service's `Extensions.cs`:

| Event | Published by | Consumed by |
|---|---|---|
| `customer_created` | `customers-service` | `availability-service`, `orders-service`, `parcels-service` |
| `signed_up` | `identity-service` | `customers-service` |
| `order_completed` | `orders-service` | `customers-service` |
| `order_created` | `orders-service` | `ordermaker-service` |
| `order_approved` | `orders-service` | `ordermaker-service` |
| `order_canceled` | `orders-service` | `parcels-service` |
| `order_deleted` | `orders-service` | `parcels-service` |
| `parcel_added_to_order` | `orders-service` | `ordermaker-service`, `parcels-service` |
| `parcel_deleted_from_order` | `orders-service` | `parcels-service` |
| `vehicle_assigned_to_order` | `orders-service` | `ordermaker-service` |
| `parcel_deleted` | `parcels-service` | `orders-service` |
| `delivery_started` | `deliveries-service` | `orders-service` |
| `delivery_completed` | `deliveries-service` | `orders-service` |
| `delivery_failed` | `deliveries-service` | `orders-service` |
| `resource_reserved` | `availability-service` | `orders-service`, `ordermaker-service` |
| `resource_reservation_canceled` | `availability-service` | `orders-service` |
| `vehicle_deleted` | `vehicles-service` | `availability-service` |

**Events with no consumer other than `operations-service`:** `customer_became_vip`, `customer_state_changed`, `signed_in`, `order_delivering`, `parcel_added`, `registration_added_to_delivery`, `resource_added`, `resource_deleted`, `resource_reservation_released`, `vehicle_added`, `vehicle_updated`, `make_order_completed`.

**`operations-service` subscribes to everything.** It is not a peer in the flows above; it is an observer of all of them, and it is the only consumer that reads `messages.json` to decide what to subscribe to.

### 3.3 Data replication

The platform replicates one entity across service boundaries.

| Owner | Collection | Replicated into | Kept in step by |
|---|---|---|---|
| `customers-service` | `customers` in database `customers-service` | `customers` in database `orders-service`; `customers` in database `parcels-service` | The `customer_created` event only |

Customer records therefore exist in three separate MongoDB databases. There are no foreign keys — MongoDB does not enforce them and the databases are separate — and **no reconciliation process exists anywhere in the workspace**. A dropped or missed `customer_created` event leaves a permanent inconsistency that nothing detects or repairs. The transactional outbox in each service reduces the chance of a message being lost on publish, but it does nothing about a consumer that fails to apply one.

Related identifier coupling without replication: order documents hold parcel and vehicle identifiers; delivery documents hold an order identifier; the saga state object in `ordermaker-service` holds order, parcel, vehicle and resource identifiers simultaneously. None of these are enforced.

### 3.4 The order-making saga — the one flow that crosses everything

`ordermaker-service` is the only component that composes the platform end to end. From `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`:

```
MakeOrder                     → publishes create_order            → orders-service
order_created                 → publishes add_parcel_to_order (×N) → orders-service
parcel_added_to_order (all)   → HTTP GetBestAsync                  → vehicles-service
                              → HTTP GetBestAsync                  → availability-service
                              → publishes assign_vehicle_to_order   → orders-service
vehicle_assigned_to_order     → publishes reserve_resource          → availability-service
order_approved                → publishes make_order_completed, CompleteAsync()
```

Correlation key: `OrderId`. Every published message carries the header `Saga` set to `SagaStates.Pending`, `Completed` or `Rejected`, and every other service explicitly forwards that header (`GetHeadersToForward` in each `Extensions.cs`). Compensation exists for one step only — `CompensateAsync(ParcelAddedToOrder)` publishes `CancelOrder(orderId, "Because I'm saga")`.

This single flow touches four services synchronously or asynchronously, and is the reason `ordermaker-service`'s absence from Jaeger matters: it is the only place where a distributed trace would show the whole business transaction, and it is the only service that does not emit one.

### 3.5 Shared infrastructure

Every deployable except `hianshul100_Pacco` and `hianshul100_Pacco.Web` registers with Consul, sits behind Fabio, reports to Prometheus, and logs to Seq. Nine of eleven publish traces to Jaeger; `ordermaker-service` does not, and `hianshul100_Pacco.Web` does not exist. **Nine of eleven use Vault** — every deployable except `api-gateway` and `ordermaker-service`, neither of which has a `vault` block in its settings file or a `.UseVault()` call. Of those nine, eight also declare `lease.mongo` for dynamic MongoDB credentials; `pricing-service` is the exception, using Vault for key-value settings and PKI only because it has no database.

Verification method for this paragraph: `vault` key read from each deployable's `src/**/appsettings.json`, and `.UseVault()` located by grep across `**/*.cs` — nine matches, one per Vault-enabled service (`Program.cs` in each `.Api` project, plus `Pacco.Services.Pricing.Api/Program.cs:33` and `Pacco.Services.Operations.Api/Program.cs:50`).

### 3.6 Shared code

**There is none.** No service references another service's assembly. Contracts are duplicated by convention: a class named `CustomerCreated` in one service and a class named `CustomerCreated` in another are matched only by the snake_case wire name `customer_created` that Convey derives from the type name. `operations-service` takes this to its conclusion by generating message types at runtime rather than referencing any of them.

The only shared dependency is the **Convey** `0.4.*` package family from NuGet, which every deployable uses, plus **Ntrada** `0.4.*` in the gateway and **Chronicle** `3.2.1` in `ordermaker-service`.

---

## 4. Suspected platform subsystems

These groupings are inferred from message flows, call graphs and naming. They are not declared anywhere in the codebase — no manifest, README or diagram in the workspace names them — so they are offered as observed structure rather than documented design.

### 4.1 Edge and access

`api-gateway`, `identity-service`.

Everything a client touches. `api-gateway` owns the public surface, terminates JWTs and decides whether a request becomes an HTTP proxy call or a RabbitMQ message. `identity-service` mints the tokens it validates. The two share a JWT signing key literal.

### 4.2 Order management — the core domain

`orders-service`, `parcels-service`, `pricing-service`.

The order aggregate, the parcels attached to it, and what it costs. `orders-service` is the hub: 14 subscriptions, 19 published messages, three synchronous dependencies. The platform's only contract test sits between `orders-service` and `parcels-service`.

### 4.3 Fulfilment resources

`vehicles-service`, `availability-service`, `deliveries-service`.

The physical means of fulfilment. `vehicles-service` owns the fleet, `availability-service` owns bookable resources and their reservation calendar, `deliveries-service` tracks the delivery itself. These are the services the "limited availability" premise of the platform actually rests on.

### 4.4 Customer

`customers-service`.

A subsystem of one, but a distinctive one: it is the only source of replicated data, the only holder of an access-control list, and the only service whose reads are role-restricted at the gateway.

### 4.5 Orchestration

`ordermaker-service`.

Also a subsystem of one. It owns no domain and no data. Its job is to sequence the other four subsystems into a single business outcome.

### 4.6 Client feedback

`operations-service`.

Sits outside the domain entirely. Because every write is asynchronous, this is the only way a caller learns what happened. It observes all eight exchanges, holds results in Redis for five minutes, and pushes them over SignalR, gRPC or HTTP.

### 4.7 Platform composition

`hianshul100_Pacco`.

No runtime. Holds the infrastructure definitions, the aggregate solution and the developer scripts that make the other twelve repositories usable together.

### 4.8 Unassigned

`hianshul100_Pacco.Web` — **Unverifiable — Missing Source Evidence.** It cannot be placed in a subsystem because it contains nothing.

---

## 5. Documentation versus tree — platform-level patterns

Per-repository reconciliations live in each summary's "README vs repository" section. What follows are the patterns that repeat across the workspace.

### 5.1 The documentation surface is one file per repository

Across all thirteen in-scope repositories, the only documentation file that ever exists is `README.md`.

- **No `CONTRIBUTING.md` anywhere.**
- **No `CHANGELOG.md` anywhere.**
- **No `docs/` directory anywhere** — including in `hianshul100_Pacco`, whose own `README.md` refers to project documentation.

The documentation pass for every repository therefore reduced to reading a single file.

### 5.2 The service READMEs are a template with the name substituted

Ten service READMEs and the gateway README share an identical structure: the Pacco logo, "What is Pacco?", "What is `<Name>`?" answered with "`<Name>` is the microservice being part of Pacco solution", two Travis build badges, start-up instructions, a default port, Docker commands, and a pointer to a `.rest` file.

They describe **how to start the service** and nothing else. Not one of them mentions messaging, the outbox, Vault, the security model, or what the service actually does in the domain. The most consequential facts about each service are undocumented in every case.

### 5.3 Nine of eleven READMEs give a `dotnet run` path that does not exist

Every service README says the service starts with `dotnet run` executed in `/src/Pacco.Services.<Name>`. For nine services the actual project directory is `src/Pacco.Services.<Name>.Api`, so the documented command fails.

The two exceptions are accidents of naming rather than corrections: `ordermaker-service` lives in `src/Pacco.Services.OrderMaker` and `api-gateway` lives in `src/Pacco.APIGateway`, both without an `.Api` suffix.

**Verdict: Stale doc**, repeated eleven times from one template.

### 5.4 Ports and image names are accurate everywhere

Every README's stated default port matches the Consul port in that service's `appsettings.json`, and every documented `docker pull` image name matches an image in `hianshul100_Pacco/compose/services.yml`.

| Service | README port | `appsettings.json` | Match |
|---|---|---|---|
| `availability-service` | 5001 | 5001 | yes |
| `customers-service` | 5002 | 5002 | yes |
| `deliveries-service` | 5003 | 5003 | yes |
| `identity-service` | 5004 | 5004 | yes |
| `operations-service` | 5005 | 5005 | yes |
| `orders-service` | 5006 | 5006 | yes |
| `parcels-service` | 5007 | 5007 | yes |
| `pricing-service` | 5008 | 5008 | yes |
| `vehicles-service` | 5009 | 5009 | yes |
| `ordermaker-service` | 5015 | 5015 | yes |

**Verdict: Confirmed.** Where the templated documentation makes a checkable factual claim, it is right.

### 5.5 The platform README's repository list is incomplete

`hianshul100_Pacco/README.md` lists twelve repositories to clone: `Pacco`, `Pacco.APIGateway`, and ten `Pacco.Services.*`. The workspace contains fourteen. `hianshul100_Pacco.Web` and `hianshul100_Pacco.Context` are present and unlisted.

**Verdict: Stale doc.**

### 5.6 The aggregate solution references a repository that does not exist

`hianshul100_Pacco/Pacco.sln` contains a project reference to `..\Pacco.APIGateway.Ocelot\src\Pacco.APIGateway.Ocelot\Pacco.APIGateway.Ocelot.csproj`. No such repository is in the workspace and none is in the README clone list. The running gateway uses Ntrada, not Ocelot.

**Verdict: Conflict — dangling reference. Unverifiable — Missing Source Evidence.** The aggregate solution cannot open cleanly as checked out.

### 5.7 `docker-images.txt` documents infrastructure nothing uses

`hianshul100_Pacco/docker-images.txt` is a free-text runbook covering SQL Server 2017, PostgreSQL, InfluxDB, Elasticsearch, Kibana and Logstash alongside the components that are genuinely in use.

Against the code: no service references a relational database at all. `metrics.influxEnabled` is `false` in every `appsettings.json`, and `logger.elk.enabled` is `false` in every `appsettings.json` while pointing at `http://localhost:9200`.

**Verdict: Conflict — docs-only claim.** Label the relational and ELK sections **Future/Intended State (Not Implemented)**.

### 5.8 Test projects exist on disk but not in their own solutions

| Project | Repository | In that repository's `.sln`? | Referenced from |
|---|---|---|---|
| `Pacco.Services.Orders.PactConsumerTests` | `hianshul100_Pacco.Services.Orders` | **No** | `hianshul100_Pacco/Pacco.sln` only |
| `Pacco.Services.Parcels.PactProviderTests` | `hianshul100_Pacco.Services.Parcels` | **No** | `hianshul100_Pacco/Pacco.sln` only |

Both halves of the platform's only contract test are invisible to their own repository's solution file. Whether `./scripts/test.sh` under Travis discovers them is unresolved.

**Verdict: Disk-only components. Needs validation.**

### 5.9 Configured-but-unused infrastructure inside services

Three cases where a service's configuration claims a dependency the code does not exercise:

- **`operations-service` and MongoDB.** `appsettings.json` declares `mongo.database: operations-service` and `AddInfrastructure()` calls `.AddMongo()`, but **no `AddMongoRepository<>` call exists in the repository**. Nothing is written. Operation state lives in Redis with a 300-second expiry.
- **`operations-service` and Vault's MongoDB credential lease.** The same `appsettings.json` declares `vault.lease.mongo` with `type: database`, `roleName: operations-service`, `enabled: true` and `autoRenewal: true`, and `Program.cs:50` calls `.UseVault()`. The service therefore obtains — and continuously renews — dynamic database credentials for the database established above as never written to. This is the strongest form of the pattern in the workspace: the configuration does not merely name an unused dependency, it maintains live credentials for it.
- **`api-gateway` and Fabio.** `ntrada.yml` contains a `loadBalancer` block pointing at `localhost:9999` with `enabled: false`, while every downstream service registers with Consul and is configured behind Fabio.

**Verdict: Conflict — configuration overstates the code.**

### 5.10 A repository named for a capability that does not exist

`hianshul100_Pacco.Web` contains one file, `README.md`, holding the single line `# Pacco.Web`. Its history is one commit. It is absent from the platform README's clone list, from `Pacco.sln`, and from every compose file.

**Verdict: Unverifiable — Missing Source Evidence.**

### 5.11 "AI" appears in names but not in code

`ordermaker-service` names its saga `AIOrderMakingSaga`, its `GET ""` route returns `Welcome to Pacco uber AI order maker Service!`, and the README carries the same framing. No model, inference library, training data or scoring code exists in the repository. Vehicle and resource selection are ordinary HTTP calls to `GetBestAsync` endpoints on other services, and what "best" means is decided downstream.

**Verdict: Conflict — docs-only claim.** Label the AI framing **Future/Intended State (Not Implemented)**.

### 5.12 The PNG diagrams — the platform's only visual documentation

`hianshul100_Pacco/assets/` holds four images. `pacco_logo.png` is branding. The other three were opened and reconciled against the code.

**`pacco_overview.png` — "Pacco: Architecture overview", by Piotr Gankiewicz & Dariusz Pawlukiewicz.** A single-page system diagram with a legend distinguishing synchronous HTTP calls (solid) from asynchronous broker calls (dashed), and colouring edges by message kind: red **Command**, green **Integration Event**, blue **Query**.

| What the diagram asserts | What the tree shows | Verdict |
|---|---|---|
| Seven domain service boxes: Availability, Customers, Deliveries, Orders, Parcels, Pricing, Vehicles, plus Identity Service, API Gateway and Operations Service. | All ten exist as repositories and deployables. | **Confirmed.** |
| **`ordermaker-service` does not appear at all.** | It is a running deployable with its own exchange, the Chronicle saga, and command publication into five other services' exchanges. | **Stale doc.** The one component that orchestrates the platform's only cross-service flow is absent from the platform's only diagram. |
| **Pricing Service has both a red Command and a green Integration Event connector to RabbitMQ**, drawn identically to the six other domain services. | `pricing-service` has no `rabbitMq` block, no `.AddRabbitMq()` call, no exchange, no subscriptions, and is the only service absent from `messages.json`. | **Conflict — the diagram overstates the code.** See [section 3.2](#32-asynchronous-messaging). |
| All seven domain service boxes have an identical solid two-way arrow into a shared Infrastructure band containing MongoDB, Vault, Grafana, Prometheus, Jaeger, Consul and Fabio. | Holds for six of the seven. `pricing-service` has no MongoDB and no Redis. Of the deployables the diagram does not draw into the band, `operations-service` uses Redis rather than MongoDB and `ordermaker-service` has neither MongoDB nor Jaeger nor Vault. | **Partly stale.** The band is drawn as uniform across services that are not uniform. |
| "Vault stores microservice's **secrets** (such as connection strings, production credentials) and allows to **inject** them on application startup." | Nine of eleven deployables configure Vault key-value and PKI and call `.UseVault()` at start-up; eight also lease dynamic MongoDB credentials. | **Confirmed** — and it is the only documentation anywhere in the workspace that states Vault's role. |
| "Operations Service **subscribes** to all messages and informs the user about operations status via **Web Sockets push notifications**"; Operations Service is drawn outside the "System's visibility boundary". | `Infrastructure/Subscriptions.cs` subscribes to every entry in `messages.json`; the SignalR hub at `/pacco` pushes `operation_pending` / `operation_completed`. | **Confirmed.** The diagram documents the hub's existence, though not its Redis backplane and not the gRPC surface. |
| "Exchange name = namespace; Queue name = {assembly}/{namespace}.{message}; Routing key = {namespace}.{message}". | Matches the `rabbitMq` conventions and `queueTemplate` in every service's `appsettings.json`. | **Confirmed.** The only place the naming convention is written down. |
| "API Gateway and all microservices communicate with each other **synchronously** through Fabio Load Balancer … available at `http://localhost:9999`." | Every service configures `fabio.url: http://localhost:9999`, but the gateway's own `ntrada.yml` sets `loadBalancer.enabled: false`. | **Partly stale.** See [section 5.9](#59-configured-but-unused-infrastructure-inside-services). |
| "Identity Service creates JWT … token is then put into API Gateway request Authorization header"; "API sends command with CorrelationContext to RabbitMQ". | Matches `identity-service` and the gateway's async mode. | **Confirmed.** |

**`clean_architecture.png`.** A concentric-ring diagram — Core (aggregate, entity, value object, domain events, repository interfaces, exceptions) inside Application (command/event handlers, integration events, dispatchers, queries, DTO, service interfaces) inside Infrastructure (MongoDB documents and repository implementation, exception-to-message mapper, event mapper, RabbitMQ client, Jaeger agent, Consul client, HTTP clients, App Metrics), all inside an "Internal API" boundary. It is a template, not a map of any particular service: it matches the four-project `.Api` / `.Application` / `.Core` / `.Infrastructure` layout of the eight layered services, and does not match `pricing-service` (one project, nested `Core/`), `operations-service` (`.Api` and `.GrpcClient`, no domain layering) or `ordermaker-service` (one project). **Verdict: Confirmed for eight of eleven deployables; silent about the three exceptions.**

**`infrastructure.png`.** A logo wall of eighteen technologies: Docker, Kubernetes, Vault, Fabio, .NET Core, Consul, RabbitMQ, Istio, Prometheus, Grafana, Jaeger, Seq, Rancher, Redis, MongoDB, gRPC. Fifteen are present in the workspace. **Three are not:** searching for Kubernetes manifests, Helm charts, Istio configuration and Rancher configuration across all fourteen repositories returned zero matches — the only deployment mechanism committed anywhere is Docker Compose, which is also recorded at [section 6.3](#63-structural-gaps-in-the-platform). **Verdict: Conflict — docs-only claim** for Kubernetes, Istio and Rancher. Label them **Future/Intended State (Not Implemented)**.

### 5.13 What is documented nowhere at all

The following are architecturally significant, are visible only by reading code, and appear in no README, diagram or comment anywhere in the workspace — the three PNG diagrams having now been read and reconciled in [section 5.12](#512-the-png-diagrams--the-platforms-only-visual-documentation):

- The transactional outbox and inbox pattern, applied in eight services.
- Runtime message-type generation by reflection in `operations-service`.
- `messages.json` as a runtime input rather than documentation.
- The Chronicle saga and the `Saga` header protocol — and `ordermaker-service` itself, which no diagram or README outside its own repository mentions.
- The `customers` read-model replicas in two databases.
- The certificate-plus-access-control-list authorisation between `availability-service` and `customers-service`.
- The Redis backplane behind the SignalR hub, and the gRPC surface on `operations-service`. (The hub itself *is* drawn, as "Web Socket push message", in `pacco_overview.png`.)
- Vault's dynamic MongoDB credential leasing, configured by eight of the nine Vault-enabled services. `pacco_overview.png` states that Vault holds secrets and injects them at start-up, but not that credentials are generated per-lease and renewed.

---

## 6. Gaps and unknowns

### 6.1 Determinable only at runtime

- **Message payload field sets.** Convey derives wire names from type names, and no schema registry, no shared contract package and no OpenAPI or AsyncAPI document exists. The only payload fields readable from code are those the `ordermaker-service` saga constructs (`create_order` → `OrderId`, `CustomerId`; `assign_vehicle_to_order` → `OrderId`, `VehicleId`, `ReservationDate`; `reserve_resource` → `VehicleId`, `CustomerId`, `ReservationDate`, `ReservationPriority`; `make_order_completed` → `OrderId`) and the six gRPC fields in `Operations.proto` (`id`, `userId`, `name`, `state`, `code`, `reason`). Everything else is **unknown — requires runtime capture**.
- **Whether Chronicle persists saga state.** `ordermaker-service` registers Redis and nothing else. If Chronicle defaults to in-memory state, every restart loses in-flight orders.
- **Whether `pricing-service` is authorised to call `customers-service`.** It is absent from that service's access-control list and presents no client certificate.
- **How traffic actually reaches services.** Consul and Fabio are configured everywhere, but the gateway sets `loadBalancer.enabled: false`, and Docker Compose service names resolve directly on `pacco-network`.

### 6.2 Domain logic not enumerated

The `*.Core` projects were identified but their contents were not read class by class. The following remain **Unknown**: the order state machine, the delivery state machine, valid customer states, the VIP promotion rule, the discount rule in `CustomerDiscountsService`, the vehicle ranking behind `GetBestAsync`, and what `GET parcels/volume` aggregates over.

### 6.3 Structural gaps in the platform

- **No feature-flag system.** Searching `*.cs`, `*.json`, `*.csproj` and `*.yml` across all fourteen repositories for `launchdarkly`, `unleash`, `flagsmith`, `split.io`, `featureflag`, `feature_flag`, `featuremanagement` and `IFeatureManager` returned zero matches. **There are no flag keys to list.** Every behavioural change requires a redeploy.
- **No API documentation artefact.** Swagger UI is generated at runtime at `docs` on every service, but no OpenAPI document is committed anywhere. `.rest` files are the only checked-in API examples.
- **No architecture decision records.** No `docs/adr/`, no `decisions/`, no numbered decision files. The architecture is recorded only in code and configuration.
- **No reconciliation for replicated data.** Three databases hold customer records and nothing detects or repairs divergence.
- **Test coverage is uneven.** `availability-service` has five test projects; `orders-service` and `parcels-service` have one each, and both are missing from their own solutions; the other eight deployables have **no tests at all**.
- **Continuous integration is branch-limited.** Every `.travis.yml` filters to `master` and `develop`. The branch inspected here, `feature/13106/aidlc`, is not built by continuous integration.
- **No infrastructure-as-code beyond Docker Compose.** No Kubernetes manifests, no Helm charts, no Terraform, no Istio configuration and no Rancher configuration anywhere in the workspace — although `assets/infrastructure.png` advertises Kubernetes, Istio and Rancher. See [section 5.12](#512-the-png-diagrams--the-platforms-only-visual-documentation).

### 6.4 Security observations recorded as facts

These are stated as findings, not recommendations, and each is evidenced in the relevant per-repository summary.

| Finding | Evidence |
|---|---|
| One JWT signing key literal — byte-identical in all nine occurrences — is checked into three repositories, across nine files | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/{ntrada.yml, ntrada-async.yml, ntrada.docker.yml, ntrada-async.docker.yml}`; `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/{appsettings.json, appsettings.docker.json, appsettings.local.json}`; `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/{appsettings.json, appsettings.docker.json}` |
| Private key material is committed | `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/certs/{localhost.pfx,localhost.key,localhost.pem,localhost.cer}`, with the `.pfx` password `test` in the adjacent `appsettings.json` |
| `vault.token: "secret"` is committed in nine services | each service's `src/**/appsettings.json`, including `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json` and `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json`. Only `api-gateway` and `ordermaker-service` have no `vault` block |
| RabbitMQ `guest`/`guest` is committed in ten deployables | the nine `appsettings.json` files that have a `rabbitMq` block — every service except `pricing-service` — plus `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada-async.yml:65-66` |
| Seq `apiKey: secret` is committed in all eleven deployables | each service's `appsettings.json` and `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/appsettings.json` |
| CORS allows every origin at the gateway | `ntrada.yml`, `cors.allowedOrigins: '*'` |
| One public read route needs no token and returns user-attributable data | `ntrada.yml` `operations` module, `auth: false`; `Operations.proto` `GetOperationResponse.userId` |
| Fleet mutations require no role | `ntrada.yml` `vehicles` module has no `claims` requirement, unlike the `customers` module |
| `identity-service` sets `validateIssuer: false` | its `appsettings.json`, against `true` in every other service |
| Example Vault unseal keys and root tokens sit in a runbook | `hianshul100_Pacco/docker-images.txt` |

### 6.5 What could not be checked at all

- `ntrada.docker.yml` and `ntrada-async.docker.yml` were not diffed against their non-Docker counterparts.
- No repository was built, started or exercised. Every finding is from static reading.
- `Pacco.APIGateway.Ocelot` cannot be assessed — it is not present.
- The single workspace attachment, `.attachments/01_product_backlog_20260910_211746_9ab63577.xlsx`, holds one backlog row (`13106`, Story, "Run Discovery - Test2", project key `Common Architecture`) and contains no architectural content.

---

## 7. Coverage

Projects were enumerated from the authoritative manifests — each repository's `.sln` file — and cross-checked against `.csproj` files on disk. Every enumerated project is either analysed or excluded with a reason.

**Totals: 31 projects enumerated across 13 repositories. 31 accounted for — 24 analysed, 7 excluded with reason. Two repositories contain no projects.**

| Repository | Projects enumerated (source) | Analysed | Excluded, with reason |
|---|---|---|---|
| `hianshul100_Pacco` | 0 — no `.csproj` in this repository. `Pacco.sln` aggregates projects from sibling repositories via `..\` paths | — | `Pacco.APIGateway.Ocelot` — referenced by `Pacco.sln` but **not present in the workspace**; **Unverifiable — Missing Source Evidence** |
| `hianshul100_Pacco.APIGateway` | 1 (`Pacco.APIGateway.sln`): `Pacco.APIGateway` | 1 | 0 |
| `hianshul100_Pacco.Services.Availability` | 9 (`Pacco.Services.Availability.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure`, `.Tests.Unit`, `.Tests.Integration`, `.Tests.EndToEnd`, `.Tests.Performance`, `.Tests.Shared` | 4 (the `src/` projects) | 5 test projects — enumerated and named, but their assertions were not read; excluded from dimension-level analysis as test scaffolding rather than runtime architecture |
| `hianshul100_Pacco.Services.Customers` | 4 (`Pacco.Services.Customers.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure` | 4 | 0 |
| `hianshul100_Pacco.Services.Deliveries` | 4 (`Pacco.Services.Deliveries.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure` | 4 | 0 |
| `hianshul100_Pacco.Services.Identity` | 4 (`Pacco.Services.Identity.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure` | 4 | 0 |
| `hianshul100_Pacco.Services.Operations` | 2 (`Pacco.Services.Operations.sln`): `.Api`, `.GrpcClient` | 2 | 0 — `.GrpcClient` is analysed and recorded as a developer tool, not a deployable |
| `hianshul100_Pacco.Services.OrderMaker` | 1 (`Pacco.Services.OrderMaker.sln`): `Pacco.Services.OrderMaker` | 1 | 0 |
| `hianshul100_Pacco.Services.Orders` | 4 (`Pacco.Services.Orders.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure` | 4 | 1 — `Pacco.Services.Orders.PactConsumerTests` exists on disk at `tests/` but is **absent from this repository's `.sln`**; recorded as a disk-only component, not analysed dimension by dimension |
| `hianshul100_Pacco.Services.Parcels` | 4 (`Pacco.Services.Parcels.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure` | 4 | 1 — `Pacco.Services.Parcels.PactProviderTests` exists on disk at `tests/` but is **absent from this repository's `.sln`**; recorded as a disk-only component, not analysed dimension by dimension |
| `hianshul100_Pacco.Services.Pricing` | 1 (`Pacco.Services.Pricing.sln`): `Pacco.Services.Pricing.Api` | 1 | 0 |
| `hianshul100_Pacco.Services.Vehicles` | 4 (`Pacco.Services.Vehicles.sln`): `.Api`, `.Application`, `.Core`, `.Infrastructure` | 4 | 0 |
| `hianshul100_Pacco.Web` | 0 — no `.sln`, no `.csproj`, no manifest of any kind | 0 | Nothing to exclude; the repository contains one file, `README.md` |

**Depth of analysis, stated plainly.** For every analysed project, the following were read: the project manifest (`.csproj`), the entrypoint (`Program.cs`), the composition root (`Extensions.cs`), and the full `appsettings.json`. Domain classes inside the `*.Core` projects were **not** read individually — see [section 6.2](#62-domain-logic-not-enumerated). Two files were read in full because of their platform-wide significance: `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` and `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`.

**Excluded from this inventory by instruction:** `hianshul100_Pacco.Context` — the repository these documents are written into.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `messages.json` in `operations-service` is an accurate catalogue of every message the platform publishes. | It is read at runtime to build subscriptions, so a wrong entry would break client notifications. It is the only complete list that exists. | The platform event catalogue in this document would inherit any drift, and downstream design work would target messages that do not exist or miss ones that do. | Compare it against the event classes in each service's `.Application/Events/` directory. |
| A2 | The credentials committed in `appsettings.json` and `ntrada.yml` — `vault.token: "secret"`, RabbitMQ `guest`/`guest`, Seq `apiKey: secret`, the shared JWT signing key — are local-development defaults overridden at deployment. | The same literals appear identically across every deployable repository that carries them — `vault.token: "secret"` in nine, RabbitMQ `guest`/`guest` in ten, Seq `apiKey: secret` in all eleven — which is the signature of a shared template, and Vault key-value and PKI paths are configured for each service to supply real values. | Live credentials would be in source control, identical everywhere, and anyone with repository access could mint valid tokens for any user. | Check the deployment pipeline and the Vault key-value paths `<service>/settings`. |
| A3 | Each service's MongoDB database is private to that service, with no shared database or cross-database query. | Each `appsettings.json` names a distinct database, and every data access goes through Convey's `IMongoRepository` scoped to that database. | Data ownership boundaries would be wrong, and the replication finding in section 3.3 would be misstated. | Inspect a running MongoDB instance for the actual database and collection layout. |
| A4 | The `customers` collections in the `orders-service` and `parcels-service` databases are read-model replicas, not systems of record. | `customers-service` owns the domain and publishes `customer_created`; both other services subscribe to it and register a `customers` repository. | Any statement about the customer system of record would be wrong. | Read the `CustomerCreated` handlers in both services. |
| A5 | `Pacco.APIGateway.Ocelot` is an earlier gateway implementation that was replaced by the Ntrada-based `api-gateway`. | The live gateway uses Ntrada; the Ocelot project is absent from the workspace and from the platform README's clone list. | A live deployable would be missing from the platform inventory entirely. | Ask the repository owner; check the remote organisation for the repository. |
| A6 | No repository was built or run; every finding is from static reading of code and configuration. | Stated method — see section 1.2. | Runtime behaviour could differ from what configuration implies, particularly around routing, Fabio and Chronicle persistence. | Stand the platform up with `compose/infrastructure.yml` and `compose/services-local.yml` and observe. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** A working JWT signing key, private key material (`localhost.pfx`, `.key`, `.pem`) and its password `test` are committed to source control, and the same signing key literal appears in three repositories. | Any statement that the platform's authentication is production-ready, and any security review of the platform. | Security owner | Confirm these are development-only. If they are used in any reachable environment, rotate the key material and move it into Vault, which is already configured with key-value and PKI engines for every service. | TBD |
| B2 | **[ACTION NOW]** `hianshul100_Pacco.Web` is empty — one file, one commit, no code — and `Pacco.sln` references `Pacco.APIGateway.Ocelot`, which is not in the workspace. Neither can be inventoried. | Completing the platform component list, and any architecture description of the user-facing layer. | Platform architect | Confirm whether each is abandoned, planned, or located outside this workspace. Then populate, remove, or record as intentionally reserved. | TBD |
| B3 | **[ACTION NOW]** It cannot be determined from the repositories whether in-flight order-making sagas survive a restart of `ordermaker-service`. Chronicle is registered but its persistence store is not explicitly configured; Redis is the only store present. | Any statement about platform resilience, and any decision to run more than one instance of `ordermaker-service`. | Service owner for `ordermaker-service` | Read Chronicle `3.2.1`'s configured persistence, or inspect Redis while a saga is in flight. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is `pricing-service` authorised to call `customers-service`? It is absent from that service's access-control list and presents no client certificate, unlike `availability-service`. | If the list is enforced, order pricing is silently broken. If it is not, the platform's only service-to-service authorisation rule is decorative. Either answer changes how the security model should be described. | The list is probably checked only when a certificate is presented, so unauthenticated calls pass. The model is inconsistent either way. | Security owner |
| Q2 | **[ACTION NOW]** How does a client refresh an expiring token? `identity-service` exposes `refresh-tokens/use`, `refresh-tokens/revoke`, `access-tokens/revoke` and `me`, but none of them appears in `ntrada.yml`. | Access tokens expire after 60 minutes. Without a public refresh route, every session ends at the hour mark. | Either these routes are reached directly, bypassing the gateway, or the gateway configuration is incomplete. | Platform architect |
| Q3 | **[ACTION NOW]** Is a 300-second expiry on operation records in `operations-service` acceptable? | It is the only way a caller learns the outcome of any write on the platform. A client away for more than five minutes can never find out what happened. | Likely a deliberate trade-off for a demonstration platform, but it should be a recorded decision rather than a default. | Platform architect |
| Q4 | **[ACTION NOW]** Does continuous integration run the two PACT projects, given both are missing from their own repositories' solution files and Travis builds only `master` and `develop`? | They are the platform's only contract test, one on each side. If they do not run, the single guarantee about inter-service compatibility is not enforced. | Add both projects to their repositories' solution files and confirm `./scripts/test.sh` discovers them. | Platform architect |
| Q5 | **[ACTION NOW]** Should `ordermaker-service` be brought under Jaeger tracing? | It is the only service that spans five others, and the only one invisible to the tracing system. Every end-to-end trace of an order breaks at that hop. | Bringing it into line with the other nine deployables would mean the `.AddJaeger()` registration and the RabbitMQ Jaeger plugin that they all carry; whether the omission is deliberate is unknown. | Platform architect |
| Q6 | **[handled later by the platform inventory review]** What corrects customer replicas in the `orders-service` and `parcels-service` databases when a `customer_created` event is missed? | Three databases hold customer records with no reconciliation process. A dropped message leaves a permanent inconsistency that nothing detects. | No mechanism exists today. The outbox reduces publish-side loss but does nothing about consumer-side failure. | Platform architect |
| Q7 | **[handled later by the platform inventory review]** How does traffic actually reach services in a deployed environment, given `loadBalancer.enabled: false` at the gateway while Consul and Fabio are configured everywhere? | Determines whether Consul and Fabio are on the real request path or merely registered. | Docker Compose service names resolve directly on `pacco-network`, so Fabio may be bypassed entirely. | Platform architect |
| Q8 | **[handled later by the platform inventory review]** Are SQL Server, PostgreSQL, InfluxDB and the ELK stack in `docker-images.txt` still intended parts of the platform? | They appear in the platform runbook but no service touches them; `influxEnabled` and `elk.enabled` are `false` everywhere. | They look like leftovers from earlier experiments. | Platform architect |
| Q9 | **[handled later by the platform inventory review]** Should the platform adopt a feature-flag mechanism? | There is none. Every behavioural change — including the discount rule in `pricing-service` — requires a code change and a redeploy. | Out of scope for this inventory; recorded because the absence is total and was explicitly checked. | Platform architect |
| Q10 | **[handled later by the platform inventory review]** Should the certificate-plus-access-control-list model used between `availability-service` and `customers-service` become the platform-wide standard? | Exactly one service defines an access-control list and exactly one other authenticates with a certificate. That is either an unfinished pilot or an oversight. | It looks like a pilot on a single call path. | Platform architect |
| Q11 | **[handled later by the platform inventory review]** Should fleet mutations at the gateway require an administrator role? | `add_vehicle`, `update_vehicle` and `delete_vehicle` carry no claims requirement, while customer reads require `role: admin`. Any authenticated user can change the fleet. | Add a `claims: role: admin` requirement to the `vehicles` mutation routes in `ntrada.yml`. | Security owner |
| Q12 | **[ACTION NOW]** `pacco_overview.png` omits `ordermaker-service` entirely and draws `pricing-service` as a RabbitMQ participant, and `infrastructure.png` advertises Kubernetes, Istio and Rancher, none of which exist in the workspace. Which is authoritative — the diagrams or the code? | The diagrams are the platform's only visual documentation and the only place the exchange and queue naming convention and Vault's role are written down, so they will be read as authoritative by anyone onboarding. Three of their claims are contradicted by the tree. | The code is authoritative per [section 1.3](#13-source-of-truth); the diagrams appear to predate `ordermaker-service` and to describe an intended rather than a built deployment topology. Confirmation is needed on whether Kubernetes, Istio and Rancher are planned or abandoned. | Platform architect |

