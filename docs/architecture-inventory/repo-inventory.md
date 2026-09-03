# Pacco — Architecture Inventory (Discovery Baseline)

**Project Key:** Common Architecture
**Stage:** `architecture_discovery` — evidence-based inventory. **No ADRs, no recommendations.**
**Date of analysis:** 2026-09-03
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`

## Scope

Thirteen in-scope platform/source repositories were analysed. `Pacco.Context` is the writable
artifact repository and is the output destination for this document; it is **excluded** from the
inventory table and per-repo summaries because it contains no platform source code (its entire
tracked content is a one-line `README.md`, commit `c3f2843 Add README`).

The in-scope set matches the repository list in the product backlog attachment
(`.attachments/01_product_backlog_20260903_170135_37cf143b.xlsx`, issue **12998** "Pacco - Discovery - Attempt-2").

| # | Repository | Primary deployable(s) |
|---|------------|-----------------------|
| 1 | `Pacco` | none (orchestration/infrastructure repo) |
| 2 | `Pacco.APIGateway` | `api-gateway` |
| 3 | `Pacco.Services.Availability` | `availability-service` |
| 4 | `Pacco.Services.Customers` | `customers-service` |
| 5 | `Pacco.Services.Deliveries` | `deliveries-service` |
| 6 | `Pacco.Services.Identity` | `identity-service` |
| 7 | `Pacco.Services.Operations` | `operations-service` |
| 8 | `Pacco.Services.OrderMaker` | `ordermaker-service` |
| 9 | `Pacco.Services.Orders` | `orders-service` |
| 10 | `Pacco.Services.Parcels` | `parcels-service` |
| 11 | `Pacco.Services.Pricing` | `pricing-service` |
| 12 | `Pacco.Services.Vehicles` | `vehicles-service` |
| 13 | `Pacco.Web` | none — repository is empty |

## Table of contents

1. [Platform at a glance](#1-platform-at-a-glance)
2. [Repository inventory — 14 dimensions](#2-repository-inventory--14-dimensions)
3. [Cross-repo relationships](#3-cross-repo-relationships)
4. [Suspected platform subsystems](#4-suspected-platform-subsystems)
5. [Documentation vs tree](#5-documentation-vs-tree)
6. [Gaps / unknowns](#6-gaps--unknowns)
7. [Coverage](#7-coverage)
8. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Platform at a glance

Pacco is a .NET Core 3.1 microservices platform for exclusive parcel delivery built around the
concept of limited resource availability. Every service is built on the **Convey** framework
(`Convey` 0.4.*, [convey-stack.github.io](https://convey-stack.github.io)), which supplies CQRS
dispatching, RabbitMQ message brokering, MongoDB persistence, Consul discovery, Fabio load
balancing, Vault secrets, Jaeger tracing, and App.Metrics/Prometheus metrics as composable NuGet
packages. Evidence: every `src/**/*.Infrastructure/*.csproj` and
`src/**/*.Api/*.csproj` (see per-repo files).

Runtime topology observed in code and config:

- **North-south:** clients → `api-gateway` (Ntrada 0.4.*, YAML-declared routes) → either an HTTP
  `downstream` call to a service, or a RabbitMQ `publish` to a service exchange
  (`Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` vs `ntrada-async.yml`).
- **East-west, async:** a RabbitMQ topic exchange per service (`availability`, `customers`,
  `deliveries`, `identity`, `ordermaker`, `orders`, `parcels`, `vehicles`, `operations`), with
  `snakeCase` message conventions and queue template
  `<service>/{{exchange}}.{{message}}` (each service's `appsettings.json` → `rabbitMq`).
- **East-west, sync:** a small number of direct HTTP calls via `Convey.HTTP` `IHttpClient`,
  resolved through Fabio/Consul (`httpClient.services` maps in `appsettings.json`).
- **Real-time:** `operations-service` fans every observed command/event/rejected-event out to
  browsers over SignalR (`/pacco` hub) and to gRPC subscribers
  (`Pacco.Services.Operations.Api/Operations.proto`).

There is **no Kubernetes, Helm, or Terraform anywhere in the workspace.** Deployment is
Docker/Docker Compose plus PM2-style process manifests (`Pacco/services.yml`,
`Pacco/prod-services.yml`) and Travis CI (`.travis.yml` in each service repo).

---

## 2. Repository inventory — 14 dimensions

Columns are the 14 required dimensions at summary level. Full detail and evidence paths are in
`repo-summary/<repo-name>.md`.

### 2.1 Purpose, runtime type, entrypoints, modules

| Repo | 1. Primary purpose | 2. Runtime/service type | 3. Key entrypoints | 4. Important modules/packages |
|---|---|---|---|---|
| `Pacco` | Solution aggregator + infra orchestration; no deployable of its own | Not a runtime — Docker Compose stacks, PM2 app manifests, clone/pull scripts | `compose/infrastructure.yml`, `compose/services.yml`, `compose/services-local.yml`, `services.yml`, `prod-services.yml`, `scripts/git-clone.sh` | `Pacco.sln` (aggregates all 12 service solutions), `compose/prometheus`, `compose/rabbitmq` |
| `Pacco.APIGateway` | Single north-south edge for the platform | ASP.NET Core 3.1 web host embedding **Ntrada** 0.4.* | `src/Pacco.APIGateway/Program.cs`; config selected by `NTRADA_CONFIG` env var | `Ntrada`, `Ntrada.Extensions.{Cors,CustomErrors,Jwt,RabbitMq,Swagger,Tracing}`, `Convey.{Logging,Metrics.AppMetrics,Security}`, `NetEscapades.Configuration.Yaml`, `Infrastructure/{CorrelationContextBuilder,SpanContextBuilder,HttpRequestHook}.cs` |
| `Pacco.Services.Availability` | Resource availability + reservation (the platform's "limited resources" core domain) | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Availability.Api/Program.cs` | Clean-architecture 4-project split: `.Api`, `.Application`, `.Core`, `.Infrastructure`; `Core/Entities/Resource.cs` is the aggregate root |
| `Pacco.Services.Customers` | Customer profile, VIP state, registration completion | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Customers.Api/Program.cs` | `.Api`/`.Application`/`.Core`/`.Infrastructure`; `Core/Entities/Customer.cs`, `Core/Entities/State.cs` |
| `Pacco.Services.Deliveries` | Delivery lifecycle and delivery registrations (tracking scans) | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Deliveries.Api/Program.cs` | `.Api`/`.Application`/`.Core`/`.Infrastructure`; `Core/Entities/Delivery.cs`, `DeliveryStatus.cs` |
| `Pacco.Services.Identity` | Authentication, JWT issuance, refresh tokens, roles | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Identity.Api/Program.cs` | `.Api`/`.Application`/`.Core`/`.Infrastructure`; `Infrastructure/Auth/{JwtProvider,PasswordService,Rng}.cs` |
| `Pacco.Services.Operations` | Cross-cutting operation/saga status projection + real-time push to clients | ASP.NET Core 3.1 host running **three** protocols: HTTP, SignalR hub, gRPC service | `src/Pacco.Services.Operations.Api/Program.cs`; second deployable `src/Pacco.Services.Operations.GrpcClient/Program.cs` (console demo client) | `Handlers/Generic{Command,Event,RejectedEvent}Handler.cs`, `Infrastructure/Subscriptions.cs` (runtime type emission from `messages.json`), `Hubs/PaccoHub.cs`, `Operations.proto` |
| `Pacco.Services.OrderMaker` | Orchestrates end-to-end order creation as a **saga** ("uber AI order maker") | ASP.NET Core 3.1 HTTP API + RabbitMQ pub/sub + Chronicle saga host | `src/Pacco.Services.OrderMaker/Program.cs` | Single-project (no clean-architecture split); `Sagas/AIOrderMakingSaga.cs`, `Sagas/AIMakingOrderData.cs`, `Handlers/AIOrderMakingHandler.cs`, `Chronicle_` 3.2.1 |
| `Pacco.Services.Orders` | Order aggregate: parcels on order, vehicle assignment, approval, delivery state | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Orders.Api/Program.cs` | `.Api`/`.Application`/`.Core`/`.Infrastructure`; `Core/Entities/{Order,Parcel,Customer,OrderStatus}.cs` |
| `Pacco.Services.Parcels` | Parcel catalogue, size/variant, volume aggregation | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Parcels.Api/Program.cs` | `.Api`/`.Application`/`.Core`/`.Infrastructure`; `Core/Entities/{Parcel,Size,Variant,Customer}.cs` |
| `Pacco.Services.Pricing` | Stateless order-price/discount calculation | ASP.NET Core 3.1 HTTP API — **query-only, no message broker** | `src/Pacco.Services.Pricing.Api/Program.cs` | Single-project; `Core/Services/CustomerDiscountsService.cs`, `Queries/Handlers/GetOrderPricingHandler.cs` |
| `Pacco.Services.Vehicles` | Vehicle catalogue and per-service pricing attributes | ASP.NET Core 3.1 HTTP API + RabbitMQ subscriber | `src/Pacco.Services.Vehicles.Api/Program.cs` | `.Api`/`.Application`/`.Core`/`.Infrastructure`; `Core/Entities/{Vehicle,Variants}.cs` |
| `Pacco.Web` | **Unknown** — repository contains only `README.md` with the text `# Pacco.Web` | None — no runtime present | None | None |

### 2.2 Integrations, data, messaging, APIs

| Repo | 5. External integrations | 6. Data stores / state (ORM + migrations) | 7. Messaging / async | 8. APIs exposed / consumed |
|---|---|---|---|---|
| `Pacco` | Declares the full infra estate: RabbitMQ, MongoDB, Redis, Consul, Fabio, Vault, Jaeger, Seq, Prometheus, Grafana, InfluxDB, Elasticsearch/Kibana/Logstash, SQL Server, PostgreSQL (`docker-images.txt`) | None owned. `compose/infrastructure.yml` provisions `mongo` and `redis` named volumes | RabbitMQ broker image built from `compose/rabbitmq/Dockerfile` (management + prometheus plugins ports `15672`, `15692`) | None |
| `Pacco.APIGateway` | RabbitMQ (`Ntrada.Extensions.RabbitMq`), Jaeger, Seq, Prometheus, Fabio (`loadBalancer.url: fabio:9999` in docker configs) | None — stateless | **Publishes** commands to 8 service exchanges in async mode: `availability`, `customers`, `deliveries`, `orders`, `parcels`, `vehicles` (routing keys listed in §3.2). Sync mode publishes nothing | **Exposes** `/availability`, `/customers`, `/deliveries`, `/identity`, `/operations`, `/orders`, `/parcels`, `/pricing`, `/vehicles`, `/` (home), `/docs` (Swagger). **Consumes** all nine services' HTTP APIs |
| `Pacco.Services.Availability` | `customers-service` over HTTP; Vault PKI client certificates; Consul; Fabio; Jaeger; Seq; Prometheus; Redis | MongoDB `availability-service`, collection **`resources`** + Convey outbox collections `inbox`/`outbox`. Access via `Convey.Persistence.MongoDB` `IMongoRepository<ResourceDocument, Guid>` — a document mapper, **not** a relational ORM. **No migration tool present** | Exchange `availability`. Publishes `resource_added`, `resource_deleted`, `resource_reserved`, `resource_reservation_released`, `resource_reservation_canceled` + 4 `*_rejected`. Subscribes `customer_created`, `vehicle_deleted`. Transactional outbox enabled (`outbox.enabled: true`) | Exposes `GET/POST /resources`, `GET/DELETE /resources/{resourceId}`, `POST/DELETE /resources/{resourceId}/reservations/{dateTime}`. Consumes `GET customers-service/customers/{id}/state` |
| `Pacco.Services.Customers` | Consul, Fabio, Jaeger, Seq, Prometheus, Redis, Vault | MongoDB `customers-service`, collection **`customers`** + `inbox`/`outbox`. `IMongoRepository<CustomerDocument, Guid>`. **No migration tool** | Exchange `customers`. Publishes `customer_created`, `customer_became_vip`, `customer_state_changed` + 2 `*_rejected`. Subscribes `signed_up` (identity), `order_completed` (orders) | Exposes `GET /customers`, `GET /customers/{customerId}`, `GET /customers/{customerId}/state`, `POST /customers`, `PUT /customers/{customerId}/state/{state}`. Consumes none (`httpClient.services: {}`) |
| `Pacco.Services.Deliveries` | Consul, Fabio, Jaeger, Seq, Prometheus, Redis, Vault | MongoDB `deliveries-service`, collection **`deliveries`** + `inbox`/`outbox`. `IMongoRepository<DeliveryDocument, Guid>`. **No migration tool** | Exchange `deliveries`. Publishes `delivery_started`, `delivery_completed`, `delivery_failed`, `registration_added_to_delivery` + 3 `*_rejected` | Exposes `GET /deliveries/{deliveryId}`, `POST /deliveries`, `POST /deliveries/{deliveryId}/fail`, `POST /deliveries/{deliveryId}/complete`, `POST /deliveries/{deliveryId}/registrations`. Consumes none |
| `Pacco.Services.Identity` | Consul, Fabio, Jaeger, Seq, Prometheus, Redis (access-token blacklist), Vault | MongoDB `identity-service`, collections **`users`**, **`refreshTokens`** + `inbox`/`outbox`. `IMongoRepository<UserDocument, Guid>` / `<RefreshTokenDocument, Guid>`. **No migration tool** — `Infrastructure/Mongo/Extensions.cs` creates a unique index on `users` at startup instead | Exchange `identity`. Publishes `signed_up`, `signed_in` + `sign_in_rejected`, `sign_up_rejected`. Subscribes command `sign_up` (`SubscribeCommand<SignUp>()`) | Exposes `GET /users/{userId}`, `GET /me`, `POST /sign-in`, `POST /sign-up`, `POST /access-tokens/revoke`, `POST /refresh-tokens/use`, `POST /refresh-tokens/revoke`. Consumes none |
| `Pacco.Services.Operations` | Consul, Fabio, Jaeger, Seq, Prometheus, Redis (SignalR backplane via `Microsoft.AspNetCore.SignalR.Redis`), Vault | MongoDB `operations-service` configured; **no `AddMongoRepository` call found** — operation state is held in Redis with `requests.expirySeconds: 300` (`appsettings.json`). Cross-check `Services/OperationsService.cs` at runtime | Exchange `operations` for its own queue; **subscribes to every message declared in `messages.json`** across 8 services (types emitted at runtime by `Infrastructure/Subscriptions.cs` via `System.Reflection.Emit`) | Exposes `GET /operations/{operationId}` (HTTP), SignalR hub `/pacco`, and gRPC `GrpcOperationsService.GetOperation` / `.SubscribeOperations` (server-streaming) |
| `Pacco.Services.OrderMaker` | `availability-service` and `vehicles-service` over HTTP; Consul; Fabio; Seq; Prometheus; Redis | **No MongoDB.** `appsettings.json` has no `mongo` section and no `Convey.Persistence.MongoDB` package. Redis is registered (`AddRedis()`), Chronicle saga state persistence backend is **Unknown — needs validation** (default is in-memory) | Exchange `ordermaker`. Publishes commands onto **other services'** exchanges (`create_order`, `add_parcel_to_order`, `assign_vehicle_to_order`, `approve_order`, `reserve_resource`, `cancel_order`) and its own `make_order_completed` / `make_order_rejected`. Subscribes `order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved`, `resource_reserved` | Exposes `POST /orders` and `GET /`. Consumes `GET availability-service/resources/{resourceId}`, `GET vehicles-service/vehicles`. **Not routed through the API gateway** |
| `Pacco.Services.Orders` | `parcels-service`, `pricing-service`, `vehicles-service` over HTTP; Consul; Fabio; Jaeger; Seq; Prometheus; Redis; Vault | MongoDB `orders-service`, collections **`orders`** and **`customers`** + `inbox`/`outbox`. `IMongoRepository<OrderDocument, Guid>`, `<CustomerDocument, Guid>`. **No migration tool** | Exchange `orders`. Publishes 9 events + 10 rejected events (full list §3.2). Subscribes `customer_created`, `parcel_deleted`, `resource_reserved`, `resource_reservation_canceled`, `delivery_started`, `delivery_completed`, `delivery_failed` | Exposes `GET /orders`, `GET/DELETE /orders/{orderId}`, `POST /orders`, `POST/DELETE /orders/{orderId}/parcels/{parcelId}`, `POST /orders/{orderId}/vehicles/{vehicleId}`. Consumes 3 services (see §3.1) |
| `Pacco.Services.Parcels` | Consul, Fabio, Jaeger, Seq, Prometheus, Redis, Vault | MongoDB `parcels-service`, collections **`parcels`** and **`customers`** + `inbox`/`outbox`. `IMongoRepository<ParcelDocument, Guid>`, `<CustomerDocument, Guid>`. **No migration tool** | Exchange `parcels`. Publishes `parcel_added`, `parcel_deleted` + 2 `*_rejected`. Subscribes `customer_created`, `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order` | Exposes `GET /parcels`, `GET /parcels/{parcelId}`, `GET /parcels/volume`, `POST /parcels`, `DELETE /parcels/{parcelId}`. Consumes none. Acts as **Pact provider** |
| `Pacco.Services.Pricing` | `customers-service` over HTTP; Consul; Fabio; Jaeger; Seq; Prometheus | **No data store at all.** No `mongo`/`redis` section, no persistence package. Fully stateless computation | **None.** No `Convey.MessageBrokers.*` package, no `rabbitMq` config section. This is the only service with no broker participation | Exposes `GET /pricing?customerId=&orderPrice=`. Consumes `GET customers-service/customers/{id}` |
| `Pacco.Services.Vehicles` | Consul, Fabio, Jaeger, Seq, Prometheus, Redis, Vault | MongoDB `vehicles-service`, collection **`vehicles`** + `inbox`/`outbox`. `IMongoRepository<VehicleDocument, Guid>`. **No migration tool** | Exchange `vehicles`. Publishes `vehicle_added`, `vehicle_updated`, `vehicle_deleted` + 3 `*_rejected`. Subscribes: none found | Exposes `GET /vehicles` (paged search), `GET /vehicles/{vehicleId}`, `POST /vehicles`, `PUT /vehicles/{vehicleId}`, `DELETE /vehicles/{vehicleId}`. Consumes none |
| `Pacco.Web` | None | None | None | None |

### 2.3 Deployment, security, observability, decisions, questions, frontend

| Repo | 9. Deployment/runtime clues | 10. Security/auth clues | 11. Observability | 12. Architecture-decision files + feature flags | 13. Open questions | 14. Frontend stack |
|---|---|---|---|---|---|---|
| `Pacco` | `compose/infrastructure.yml`, `compose/services.yml` (pulls `devmentors/pacco.*` images), `compose/services-local.yml`, `compose/host-infrastructure.yml`, `compose/consul-fabio-vault.yml`, `compose/grafana-seq-jaeger-prometheus.yml`, `compose/mongo-rabbit-redis.yml`; PM2-style `services.yml` / `prod-services.yml` (ports 5000–5009). **No Helm/K8s/Terraform** | `docker-images.txt` documents Vault init, unseal, `userpass` policy, PKI roles for `availability-service` / `customers-service`, and Mongo dynamic DB credentials. **It also contains live-looking Vault unseal keys and a root token in plaintext** | `compose/prometheus/prometheus.yml` scrapes `docker.for.mac.localhost:5000/metrics-text`; Grafana, Jaeger, Seq, ELK all declared | `README.md` states the two governing choices: microservices + event-driven, and "clean architecture + DDD … or another style that is the best fit". `assets/clean_architecture.png`, `assets/infrastructure.png`, `assets/pacco_overview.png`. **No feature-flag system** | Which compose file is authoritative for a given environment; whether the committed Vault credentials are live | No frontend assets — checked `assets/` (PNG diagrams only), `compose/`, `scripts/`. No `public/`, `src/`, `static/`, `wwwroot/`, `web/`, no `package.json` |
| `Pacco.APIGateway` | `Dockerfile` (SDK 3.1 → aspnet 3.1, `ENTRYPOINT dotnet Pacco.APIGateway.dll`), `.travis.yml` → `scripts/build.sh` + `scripts/dockerize.sh` → Docker Hub `$DOCKER_USERNAME/pacco.apigateway`. Compose sets `NTRADA_CONFIG=ntrada-async.docker.yml` | JWT validation at the edge (`validIssuer: pacco`, `validateAudience: false`); `role: admin` claim gates on 5 routes; `@user_id` claim binding injects the caller's id into downstream URLs and message payloads. **A symmetric `issuerSigningKey` is committed in all four `ntrada*.yml` files.** CORS `allowedOrigins: ['*']` with `allowCredentials: true` | Jaeger (`serviceName: api-gateway`), Seq, Prometheus via `Convey.Metrics.AppMetrics`; `generateRequestId`/`generateTraceId`; exposes `Request-ID`, `Trace-ID`, `Resource-ID`, `Total-Count` headers | The four `ntrada*.yml` files **are** the architecture decision: sync-vs-async edge is a config switch, not a code change. `Infrastructure/HttpRequestHook.cs`, `CorrelationContextBuilder.cs`, `SpanContextBuilder.cs`. **No feature-flag system** | Which of sync/async is intended for production; whether `*` CORS is deliberate | No frontend assets — checked `src/Pacco.APIGateway/` (no `wwwroot/`, `public/`, `static/`, `assets/`), no `package.json`, no view templates |
| `Pacco.Services.Availability` | `Dockerfile`, `.travis.yml`, `scripts/{build,dockerize,start,test}.sh`; image `pacco.services.availability`; port 5001 | JWT bearer; **mTLS-style client-certificate auth** via Vault PKI (`security.certificate.header: Certificate`), certificate attached in `CustomersServiceClient` ctor; `Convey.WebApi.Security` | Jaeger `serviceName: availability`, Seq, Prometheus, request masking, 12 secret-bearing properties excluded from logs | Clean-architecture 4-project split is the decision, visible in `Pacco.Services.Availability.sln`. `Infrastructure/Extensions.cs` is the composition root. **No feature-flag system** | Whether the outbox `disableTransactions: true` is intentional given a single-node Mongo | No frontend assets — checked `src/**/` (only `certs/`, `Properties/`); no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Customers` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.customers`; port 5002 | JWT bearer + **certificate ACL**: `security.certificate.acl` grants `availability-service` the `customers:read` permission, `allowedDomains: ['pacco.io']` | Jaeger `serviceName: customers`, Seq, Prometheus, request masking | Clean-architecture split; `Infrastructure/Extensions.cs`. **No feature-flag system** | Whether the ACL is enforced or advisory | No frontend assets — checked `src/**/`; no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Deliveries` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.deliveries`; port 5003 | JWT bearer; `security.certificate.header: Certificate`; **no `Convey.WebApi.Security` package** unlike Availability/Customers | Jaeger `serviceName: deliveries`, Seq, Prometheus | Clean-architecture split; `Infrastructure/Extensions.cs`. **No feature-flag system** | Who publishes `start_delivery` in practice — no in-repo publisher of the trigger | No frontend assets — checked `src/**/`; no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Identity` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.identity`; port 5004 | **The platform's auth origin.** `Infrastructure/Auth/JwtProvider.cs`, `PasswordService.cs` (ASP.NET `IPasswordHasher`), `Rng.cs`; `UseAccessTokenValidator()` gives Redis-backed token revocation; `Core/Entities/Role.cs` | Jaeger, Seq, Prometheus; sensitive properties excluded from logs | Clean-architecture split; `Infrastructure/Extensions.cs`. **No feature-flag system** | Whether the gateway's symmetric `issuerSigningKey` and this service's certificate-based signing are the same trust root | No frontend assets — checked `src/**/`; no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Operations` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.operations`; port 5005. Second project `Pacco.Services.Operations.GrpcClient` is a console app, not containerised | JWT — the SignalR hub authenticates by token passed to `initializeAsync`; `jwt.allowAnonymousEndpoints: ['/sign-in','/sign-up']`. **The symmetric `issuerSigningKey` is committed in `appsettings.json`** | Jaeger `serviceName: operations`, Seq, Prometheus; this service *is* the platform's operation-level observability projection | `messages.json` is the **platform-wide message contract catalogue** — the single most decision-bearing file in the workspace. `Infrastructure/Subscriptions.cs` (runtime type emission). **No feature-flag system** | Whether operation state is Mongo- or Redis-backed; how `requests.expirySeconds: 300` interacts with long sagas | **Yes** — `wwwroot/ui/index.html` + `wwwroot/ui/js/app.js` + bundled `wwwroot/ui/js/signalr.js` (SignalR JS client, webpack UMD bundle). Bootstrap 4.0.0 from CDN. **No build tooling, no `package.json`, no MFE/federation** |
| `Pacco.Services.OrderMaker` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.ordermaker`; port 5015. **Absent from `Pacco/services.yml`, `Pacco/prod-services.yml`, and `Pacco/compose/services.yml`** | JWT config present but `Program.cs` calls `UseApp()` (not `UseInfrastructure()`); the saga constructs an **empty `CorrelationContext.UserContext`**, i.e. it acts without a caller identity | Seq, Prometheus, Convey logging. **No Jaeger package** — the only broker-participating service without distributed tracing | `Sagas/AIOrderMakingSaga.cs` encodes the whole order-creation choreography; `Chronicle_` 3.2.1 is the saga library choice. **No feature-flag system** | Saga persistence backend; whether this service is deployed at all | No frontend assets — checked `src/**/` (only `certs/`, `Properties/`); no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Orders` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.orders`; port 5006 | JWT bearer; `GetOrdersHandler` filters by `IAppContext` (caller identity), so list scoping is identity-driven | Jaeger `serviceName: orders`, Seq, Prometheus | Clean-architecture split; `Infrastructure/Extensions.cs`. **Pact consumer** contract tests in `tests/Pacco.Services.Orders.PactConsumerTests` (`Pactify` 1.1.0) — a live contract-testing decision. **No feature-flag system** | Whether the local `customers` collection is a projection or a duplicate source of truth | No frontend assets — checked `src/**/`; no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Parcels` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.parcels`; port 5007 | JWT bearer; `GetParcelsHandler` scopes by `IAppContext` | Jaeger `serviceName: parcels`, Seq, Prometheus | Clean-architecture split; **Pact provider** tests in `tests/Pacco.Services.Parcels.PactProviderTests` — the counterpart to Orders' consumer tests. **No feature-flag system** | How the Pact file is shared between the two repos (no broker config found) | No frontend assets — checked `src/**/`; no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Pricing` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.pricing`; port 5008. **No `LICENSE` file** (every other service repo has one) | JWT bearer via `Convey.Security`; no certificate ACL | Jaeger `serviceName: pricing`, Seq, Prometheus | Single-project layout is itself the decision — deliberately *not* clean architecture, matching the README's "another style that is the best fit". `Core/Services/CustomerDiscountsService.cs` holds the discount rules. **No feature-flag system** | Whether discount tiers are hard-coded or configurable | No frontend assets — checked `src/**/` (only `certs/`, `Properties/`, `.idea/`); no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Services.Vehicles` | `Dockerfile`, `.travis.yml`, `scripts/*`; image `pacco.services.vehicles`; port 5009. **No `LICENSE` file** | JWT bearer; gateway gates `POST/PUT/DELETE /vehicles` — role claims per `ntrada*.yml` | Jaeger, Seq, Prometheus | Clean-architecture split; `Infrastructure/Extensions.cs`. **No feature-flag system** | Why `vehicle_deleted` is consumed by Availability but no other vehicle event is consumed anywhere | No frontend assets — checked `src/**/`; no `wwwroot/`, `public/`, `static/`, no `package.json` |
| `Pacco.Web` | None — `git log` shows a single commit `b3bf026 Initial commit` | None | None | None | Is this a placeholder for a planned web client, or an abandoned repo? | No frontend assets — checked the entire clone. The repository contains exactly one tracked file, `README.md` |

---

## 3. Cross-repo relationships

### 3.1 Synchronous HTTP calls (service → service)

Every edge below is proven by a `Convey.HTTP` `IHttpClient` call whose base URL comes from
`httpClient.services` in `appsettings.json`. Logical keys resolve to Consul/Fabio service names.

| Caller | Callee | Physical call | Evidence |
|---|---|---|---|
| `api-gateway` | all 9 services | `downstream:` entries in `ntrada*.yml` | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` |
| `availability-service` | `customers-service` | `GET {customers}/customers/{id}/state` | `Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` |
| `orders-service` | `parcels-service` | `GET {parcels}/parcels/{id}` | `.../Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs` |
| `orders-service` | `pricing-service` | `GET {pricing}/pricing?customerId={id}&orderPrice={price}` | `.../Orders.Infrastructure/Services/Clients/PricingServiceClient.cs` |
| `orders-service` | `vehicles-service` | `GET {vehicles}/vehicles/{id}` | `.../Orders.Infrastructure/Services/Clients/VehiclesServiceClient.cs` |
| `pricing-service` | `customers-service` | `GET {customers}/customers/{id}` | `Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs` |
| `ordermaker-service` | `availability-service` | `GET {availability}/resources/{resourceId}` | `.../OrderMaker/Services/Clients/AvailabilityServiceClient.cs` |
| `ordermaker-service` | `vehicles-service` | `GET {vehicles}/vehicles` | `.../OrderMaker/Services/Clients/VehiclesServiceClient.cs` |

`customers-service` is the most-called service (2 inbound sync callers) and calls nothing itself —
a leaf in the sync graph.

### 3.2 Asynchronous messaging topology

**Broker:** RabbitMQ. **Exchange type:** `topic`, durable, per service. **Casing:** `snakeCase`.
**Queue naming:** `<service>/{{exchange}}.{{message}}`. **Context propagation:** `message_context`
header; **trace propagation:** `span_context` header.

The authoritative catalogue of every message name on the platform is
`Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` — it enumerates
8 exchanges, 26 commands, 30 events, and 31 rejected events. Names below are copied verbatim.

| Exchange | Commands | Events | Rejected events |
|---|---|---|---|
| `availability` | `add_resource`, `delete_resource`, `release_resource`, `reserve_resource` | `resource_added`, `resource_deleted`, `resource_reservation_released`, `resource_reservation_canceled`, `resource_reserved` | `add_resource_rejected`, `delete_resource_rejected`, `release_resource_rejected`, `reserve_resource_rejected` |
| `customers` | `change_customer_state`, `complete_customer_registration` | `customer_created`, `customer_became_vip`, `customer_state_changed` | `change_customer_state_rejected`, `complete_customer_registration_rejected` |
| `deliveries` | `add_delivery_registration`, `complete_delivery`, `fail_delivery`, `start_delivery` | `delivery_completed`, `delivery_failed`, `delivery_started`, `registration_added_to_delivery` | `complete_delivery_rejected`, `fail_delivery_rejected`, `start_delivery_rejected` |
| `identity` | `sign_in`, `sign_up` | `signed_up`, `signed_in` | `sign_in_rejected`, `sign_up_rejected` |
| `ordermaker` | *(none declared)* | `make_order_completed` | `make_order_rejected` |
| `orders` | `add_parcel_to_order`, `approve_order`, `assign_vehicle_to_order`, `cancel_order`, `create_order`, `delete_order`, `delete_parcel_from_order` | `order_approved`, `order_canceled`, `order_completed`, `order_created`, `order_deleted`, `order_delivering`, `parcel_added_to_order`, `parcel_deleted_from_order`, `vehicle_assigned_to_order` | `add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`, `cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`, `delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found` |
| `parcels` | `add_parcel`, `delete_parcel` | `parcel_added`, `parcel_deleted` | `add_parcel_rejected`, `delete_parcel_rejected` |
| `vehicles` | `add_vehicle`, `delete_vehicle`, `update_vehicle` | `vehicle_added`, `vehicle_deleted`, `vehicle_updated` | `add_vehicle_rejected`, `delete_vehicle_rejected`, `update_vehicle_rejected` |

Cross-service **event subscription** edges proven by `Events/External/Handlers/*.cs`:

| Publisher (exchange) | Event | Subscriber(s) |
|---|---|---|
| `identity` | `signed_up` | `customers-service` |
| `customers` | `customer_created` | `availability-service`, `orders-service`, `parcels-service` |
| `vehicles` | `vehicle_deleted` | `availability-service` |
| `availability` | `resource_reserved` | `orders-service`, `ordermaker-service` |
| `availability` | `resource_reservation_canceled` | `orders-service` |
| `parcels` | `parcel_deleted` | `orders-service` |
| `orders` | `order_created` | `ordermaker-service` |
| `orders` | `order_approved` | `ordermaker-service` |
| `orders` | `order_completed` | `customers-service` |
| `orders` | `order_canceled`, `order_deleted` | `parcels-service` |
| `orders` | `parcel_added_to_order` | `parcels-service`, `ordermaker-service` |
| `orders` | `parcel_deleted_from_order` | `parcels-service` |
| `orders` | `vehicle_assigned_to_order` | `ordermaker-service` |
| `deliveries` | `delivery_started`, `delivery_completed`, `delivery_failed` | `orders-service` |
| *all 8 exchanges* | *every command, event and rejected event* | `operations-service` (via `messages.json`) |

**Payload structure:** observable for every event above from the publisher/subscriber DTO classes
in `Events/` and `Events/External/` (e.g. `ResourceReserved(ResourceId, CustomerId, DateTime)`).
The `operations-service` subscriptions are the exception: `Infrastructure/Subscriptions.cs` emits
**field-less** types at runtime with `Reflection.Emit`, so the wire payload it receives is
**unknown — requires runtime capture**.

### 3.3 Shared libraries

There is **no first-party shared library** and **no shared code repository** in the workspace. No
service references another service's project or a common `Pacco.*` NuGet package. Sharing is
achieved by:

- **Convention over code** — every service pins `Convey` `0.4.*` and reproduces the same
  `Infrastructure/Extensions.cs` composition root, the same `Decorators/`, `Contexts/`,
  `Exceptions/`, `Logging/`, `Mongo/` folder structure, and near-identical `appsettings.json`.
  This is duplication-by-template, not a shared library.
- **Contract duplication** — an event consumed across a boundary is redeclared in the consumer.
  `CustomerCreated` exists as four independent C# classes (Customers, Availability, Orders,
  Parcels); `ResourceReserved` exists in Availability, Orders, and OrderMaker. The RabbitMQ
  `snake_case` message name is the only binding contract.
- **The `messages.json` catalogue** in `Pacco.Services.Operations`, which is a *copy* of names
  owned by eight other repositories with no generation or validation step.

### 3.4 Shared infrastructure

All services share one instance of each backing service on the `pacco-network` Docker network
(`Pacco/compose/infrastructure.yml`): `mongo`, `rabbitmq`, `redis`, `consul`, `fabio`, `vault`,
`jaeger`, `seq`, `prometheus`, `grafana`.

### 3.5 Shared data stores

**One MongoDB server, one logical database per service** — `availability-service`,
`customers-service`, `deliveries-service`, `identity-service`, `operations-service`,
`orders-service`, `parcels-service`, `vehicles-service`. No service reads another's database.
**One Redis server** partitioned by key prefix: `availability:`, `customers:`, `deliveries:`,
`identity:`, `operations:`, `ordermaker:`, `orders:`, `parcels:`.

**Cross-domain coupling points.** MongoDB has no foreign keys, so the relational-FK test does not
apply. The equivalent coupling is a **replicated `customers` collection owned by another domain**:

| Collection | Owning service | Replicated into | Kept in sync by |
|---|---|---|---|
| `customers` | `customers-service` (`CustomerDocument`) | `orders-service` (`Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs`), `parcels-service` (`Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs`) | `customer_created` event only — no update or delete event is consumed by either replica |

This is the platform's principal data-coupling risk: two services hold a customer copy that is
written once at creation and never reconciled, while `customer_state_changed` and
`customer_became_vip` are published but consumed by neither.

---

## 4. Suspected platform subsystems

Groupings are inferred from exchange ownership, HTTP call direction, and MongoDB database
ownership. Confidence is stated per grouping.

### S1 — Edge & Access (confidence: high)
`api-gateway` (repo `Pacco.APIGateway`), `identity-service` (repo `Pacco.Services.Identity`).

The only two components that terminate untrusted traffic. The gateway validates JWTs it does not
issue; `identity-service` issues them. Every other service trusts the resulting bearer token.

### S2 — Order Fulfilment Core (confidence: high)
`orders-service` (`Pacco.Services.Orders`), `parcels-service` (`Pacco.Services.Parcels`),
`deliveries-service` (`Pacco.Services.Deliveries`).

`orders-service` is the hub: it owns the largest contract surface on the platform (7 commands,
9 events, 10 rejected events), calls three services synchronously, and subscribes to seven
external events. Parcels and Deliveries exist almost entirely to serve the order lifecycle —
`parcels-service` subscribes to four `orders` events and `deliveries-service`'s three events are
consumed only by `orders-service`.

### S3 — Resource & Capacity (confidence: high)
`availability-service` (`Pacco.Services.Availability`), `vehicles-service`
(`Pacco.Services.Vehicles`).

This is the "limited resources availability" concept the platform README names as the domain's
organising idea. `availability-service` reserves abstract resources against dates;
`vehicles-service` supplies the catalogue of the things reserved. The `vehicle_deleted` →
`availability-service` subscription is the only edge between them, and it is a cleanup edge.

### S4 — Customer & Commercial (confidence: medium)
`customers-service` (`Pacco.Services.Customers`), `pricing-service`
(`Pacco.Services.Pricing`).

Grouped because `pricing-service`'s only dependency is `customers-service` and its only logic
(`Core/Services/CustomerDiscountsService.cs`) is a function of customer state. Confidence is
medium rather than high: `pricing-service` shares none of the platform's other conventions — no
broker, no database, no clean-architecture split, no `LICENSE` — so it may be better read as a
standalone utility than as a member of a customer subsystem.

### S5 — Orchestration & Observation (confidence: medium)
`ordermaker-service` (`Pacco.Services.OrderMaker`), `operations-service`
(`Pacco.Services.Operations`).

Both are cross-cutting: neither owns a domain, both listen to many exchanges and act across
service boundaries. `ordermaker-service` writes (drives a saga across Orders, Parcels, Vehicles,
Availability); `operations-service` only reads (projects every message to SignalR/gRPC clients).
Confidence is medium because `ordermaker-service` is absent from every deployment manifest, so
whether this subsystem is operationally real is unverified.

### S6 — Platform Infrastructure (confidence: high)
`Pacco` repository.

Owns no service but owns the definition of every environment: the compose stacks, the process
manifests, the aggregate solution file, and the infrastructure runbook.

### S7 — Unclassified
`Pacco.Web`. Empty repository; cannot be placed in any subsystem. **Unverifiable — Missing
Source Evidence.**

---

## 5. Documentation vs tree

Two recurring platform-level patterns were observed across the eleven repositories that have a
substantive README.

**Pattern 1 — READMEs are a boilerplate template, not a description of the repository.**
Ten of the twelve service/gateway repos share a byte-similar README: logo, the same
"What is Pacco?" paragraph, a Travis badge table, "How to start the application?", and a pointer
to a `.rest` file. None of them names a single module, entity, event, endpoint, or dependency
that the repository actually contains. Consequently, for every service repo, the entire
Application/Core/Infrastructure structure, the MongoDB collections, the RabbitMQ exchange, the
event catalogue, and the outbound HTTP dependencies are **disk-only** — present in the tree, absent
from the README. The per-repo "README vs repository" sections record this individually.

**Pattern 2 — the platform README under-describes the platform it aggregates.**
`Pacco/README.md` names microservices, event-driven integration, Convey, clean architecture + DDD,
and CNCF tooling. It does not mention: the API gateway's dual sync/async operating modes; Consul
service discovery; Fabio load balancing; Vault PKI and client-certificate service-to-service auth;
the transactional outbox; the Chronicle saga in `ordermaker-service`; the Pact contract tests
between Orders and Parcels; or the SignalR/gRPC real-time surface in `operations-service`. All
seven are substantial architectural facts visible in the tree.

**Stale doc markers.**

- `Pacco/README.md` lists twelve repositories to clone. `Pacco.Web` is **not** among them, yet
  `Pacco.Web` is in the backlog's repository list for this discovery. Neither the platform README
  nor any config references it. **Stale doc / scope mismatch — needs validation.**
- `Pacco/README.md` and every service README state ".NET Core 3.1"; the tree agrees
  (`Dockerfile` uses `mcr.microsoft.com/dotnet/core/sdk:3.1`, `.travis.yml` pins `dotnet: 3.1.100`,
  publish paths are `netcoreapp3.1`). **No conflict.**
- `Pacco/README.md` says services can be started "all at once using Docker:
  `docker-compose -f services-local.yml up`". `compose/services-local.yml` exists, but neither it
  nor `compose/services.yml` nor `services.yml` nor `prod-services.yml` contains
  `ordermaker-service`. The documented "start them all" path therefore does not start every
  service in the tree. **Documented claim not fully reflected in the tree.**
- `Pacco.Services.Operations/README.md` says the service starts from
  `/src/Pacco.Services.Operations`. The actual directory is
  `/src/Pacco.Services.Operations.Api`. **Stale doc.**
- `Pacco.Web/README.md` (`# Pacco.Web`) and `Pacco.Context/README.md` (`# Pacco.Context`) assert
  nothing and are contradicted by nothing. **Unknown.**

**No documentation/code conflict of substance was found**, because the documentation is too thin
to conflict with anything. The one directional conflict — the Operations start path — is recorded
above with code as the source of truth.

---

## 6. Gaps / unknowns

| # | Gap | Why it could not be determined |
|---|---|---|
| G1 | Purpose and status of `Pacco.Web` | The clone contains one file, `README.md`, with the text `# Pacco.Web`, on a single commit `b3bf026 Initial commit`. No manifest, no source, no config, no reference from any other repo. **Unverifiable — Missing Source Evidence** |
| G2 | Whether `ordermaker-service` is deployed | It has a `Dockerfile`, a `.travis.yml` and port 5015, but appears in **none** of `Pacco/services.yml`, `Pacco/prod-services.yml`, `Pacco/compose/services.yml`, `compose/services-local.yml`, or any `ntrada*.yml` gateway module. Nothing in the workspace routes traffic to it |
| G3 | Chronicle saga state persistence in `ordermaker-service` | `Extensions.cs` calls `builder.Services.AddChronicle()` with no persistence configuration and no `Chronicle.Persistence.*` package. Chronicle's default is in-memory, which would lose saga state on restart, but this is not stated anywhere. **Needs validation at runtime** |
| G4 | `operations-service` state store | `appsettings.json` configures `mongo.database: operations-service`, but no `AddMongoRepository<...>` call exists in the repo, while `redis` and `requests.expirySeconds: 300` are configured. Which store actually holds operation state is not determinable from static reading of `Services/OperationsService.cs` alone |
| G5 | Wire payloads of the messages `operations-service` consumes | `Infrastructure/Subscriptions.cs` builds subscription types at runtime with `Reflection.Emit` from bare names in `messages.json`; the generated types have no fields. **Unknown — requires runtime capture** |
| G6 | Which gateway configuration is authoritative per environment | Four configs exist (`ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml`) and they differ *architecturally*, not just by hostname: the async pair converts 21 write routes from HTTP `downstream` to RabbitMQ `publish`. `compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml`; nothing states the production intent |
| G7 | Trigger for `start_delivery` | `deliveries-service` subscribes to no external event and no other service publishes a `deliveries` command. The only route is the gateway's `POST /deliveries`. Whether a human operator or an unseen system drives deliveries is undetermined |
| G8 | Consumers of `customer_state_changed` and `customer_became_vip` | Both are published by `customers-service` and declared in `messages.json`, but no `Events/External/Handlers/` class in any repo handles them. Only `operations-service` receives them, and only to display them |
| G9 | Whether committed secrets are live | `Pacco/docker-images.txt` contains five Vault unseal keys and a root token; four `ntrada*.yml` files and `Operations/appsettings.json` contain the same symmetric JWT `issuerSigningKey`; Seq `apiKey: secret` and RabbitMQ `guest/guest` are committed platform-wide. Whether these are demo values or real credentials cannot be determined from the repositories |
| G10 | Ownership and team assignment for every repository | No `CODEOWNERS`, no `CONTRIBUTING.md`, no team metadata in any clone. The upstream `devmentors/*` origin implies devmentors.io maintenance, but the analysed clones are `hianshul100/*` forks with no ownership record |
| G11 | Database migration strategy | No migration tooling of any kind was found in the workspace — no Alembic, Flyway, Doctrine, ActiveRecord, Liquibase, or EF Core migrations. Schema evolution for the eight MongoDB databases appears to rely on document-model tolerance and the one startup index creation in `Identity.Infrastructure/Mongo/Extensions.cs`. Whether this is deliberate is undocumented |
| G12 | Feature-flag system | A workspace-wide search for `LaunchDarkly`, `Unleash`, `Flagsmith`, `Split`, `featureFlag`, `feature_flag`, and `featureToggle` across `*.cs`, `*.json`, and `*.yml` returned zero matches. **No feature flag system is in use, and therefore no flag keys exist to list** |
| G13 | Pact contract sharing between Orders and Parcels | `tests/Pacco.Services.Orders.PactConsumerTests/PACT/` and `tests/Pacco.Services.Parcels.PactProviderTests/PACT/` both exist, but no Pact Broker configuration was found in either repo or in either `.travis.yml`. How the contract file crosses the repository boundary is undetermined |

---

## 7. Coverage

Project enumeration method per repository, in priority order: (1) the `.sln` solution file where
present, (2) every `*.csproj` on disk, (3) top-level `src/` and `tests/` directories. .NET has no
`pnpm-workspace.yaml` / `nx.json` / `go.work` equivalent; the `.sln` + `*.csproj` pair is the
authoritative project list and was used as the coverage checklist.

`Pacco.sln` in the `Pacco` repository aggregates the *other* repositories' projects by relative
path (`../Pacco.Services.*/src/...`); those projects are counted once, under their owning
repository, not twice.

| Repository | Enumerated | Documented | Excluded | Exclusion reasons |
|---|---|---|---|---|
| `Pacco` | 0 C# projects (`Pacco.sln` references only other repos' projects) | 0 | 0 | — (repo documented as a non-code orchestration repository; its 8 compose files and 2 process manifests are all covered in §2.3) |
| `Pacco.APIGateway` | 1 | 1 | 0 | — |
| `Pacco.Services.Availability` | 9 | 4 | 5 | `Tests.Unit`, `Tests.Integration`, `Tests.EndToEnd`, `Tests.Performance`, `Tests.Shared` — excluded: test projects, not deployables (NBomber load harness and xunit suites) |
| `Pacco.Services.Customers` | 4 | 4 | 0 | — |
| `Pacco.Services.Deliveries` | 4 | 4 | 0 | — |
| `Pacco.Services.Identity` | 4 | 4 | 0 | — |
| `Pacco.Services.Operations` | 2 | 2 | 0 | — (`Pacco.Services.Operations.GrpcClient` is documented as a non-containerised console demo client, not excluded) |
| `Pacco.Services.OrderMaker` | 1 | 1 | 0 | — |
| `Pacco.Services.Orders` | 5 | 4 | 1 | `Pacco.Services.Orders.PactConsumerTests` — excluded: contract-test project, not a deployable (its architectural significance is recorded in §2.3 dimension 12) |
| `Pacco.Services.Parcels` | 5 | 4 | 1 | `Pacco.Services.Parcels.PactProviderTests` — excluded: contract-test project, not a deployable (significance recorded in §2.3 dimension 12) |
| `Pacco.Services.Pricing` | 1 | 1 | 0 | — |
| `Pacco.Services.Vehicles` | 4 | 4 | 0 | — |
| `Pacco.Web` | 0 | 0 | 0 | — (no projects exist; the empty state is itself documented) |
| **Total** | **40** | **33** | **7** | 7 test/contract-test projects |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The thirteen repositories in this workspace are the whole Pacco platform | They match the repository list in backlog issue 12998 exactly, and no clone references a repository outside the set | A missing repository would mean an undiscovered service, exchange, or data store, making the subsystem groupings in §4 wrong | Confirm the repository list with the platform owner; re-run discovery if any repo is added |
| A2 | Every service runs against one shared MongoDB, RabbitMQ, and Redis instance, isolated only by database name and key prefix | `Pacco/compose/infrastructure.yml` defines exactly one container each, and every service's `appsettings.json` points at `localhost`/the compose hostname | If production actually uses separate instances per service, the shared-infrastructure analysis in §3.4 and the blast-radius implications are overstated | Compare against the production deployment configuration, which is not in this workspace |
| A3 | `Convey` 0.4.* behaves as its package names describe (outbox, Consul discovery, Fabio balancing, Jaeger tracing) | The framework source is not in the workspace; only the package references and configuration sections are visible | Any behaviour attributed to Convey in this document could be wrong, particularly the outbox delivery guarantees | Read the Convey 0.4 source at convey-stack.github.io, or capture broker behaviour at runtime |
| A4 | `messages.json` in `operations-service` is an accurate catalogue of every message on the platform | Cross-checking it against each service's `Events/` and `Commands/` folders found no message present in a service but absent from the catalogue | The event topology in §3.2 would be incomplete; a downstream consumer could be missed | The catalogue is hand-maintained with no generation step — re-verify it whenever a service adds a message |
| A5 | The absence of any migration tooling means schema evolution is handled by document-model tolerance | No migration files or tools of any kind exist for the eight MongoDB databases; only one startup index creation was found | If an out-of-band migration process exists, the data-evolution picture in §2.2 is incomplete | Ask the platform owner whether schema changes are applied by an external process |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** `Pacco/docker-images.txt` contains five Vault unseal keys and a Vault root token in plaintext, and the same symmetric JWT signing key is committed in four gateway config files and in `Operations/appsettings.json`. Nobody on this side can tell whether these are throwaway demo values or credentials that currently protect something real | Any decision to treat these repositories as safe to publish, fork, or share; and any security discussion in later stages | Platform owner / whoever administers the Pacco Vault instance | A person with access to the running Vault must check whether these keys still unseal it. If they do, rotate them and purge the values from git history before anything else proceeds | TBD |
| B2 | **[ACTION NOW]** `Pacco.Web` is an empty repository, but it is on the discovery scope list. We cannot tell whether a web client exists somewhere we were not given, or whether the repo is an abandoned placeholder | Completing the platform picture — if a real web client exists, the entire frontend dimension of this inventory is missing a component, and the gateway's CORS and auth surface has an unexamined consumer | Platform owner | Someone must state whether a Pacco web client exists. If it does, provide the repository and re-run discovery for it; if it does not, drop `Pacco.Web` from the scope list | TBD |
| B3 | **[ACTION NOW]** `ordermaker-service` has a Dockerfile, a CI pipeline and a port, but is in no deployment manifest and behind no gateway route. Whether it runs in production is not answerable from the code | Deciding whether the saga in §4/S5 is a live orchestration path or dead code — this changes how the order-creation flow should be described and whether it needs governing | Platform owner / operations | Someone must confirm whether `ordermaker-service` is deployed and, if so, how callers reach it | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is the API gateway meant to run in synchronous mode or asynchronous mode in production? | The two modes are architecturally different systems. In async mode 21 write endpoints stop returning a result and become fire-and-forget RabbitMQ publishes, so clients must poll `operations-service` instead. Any client contract depends on the answer | The compose file sets `NTRADA_CONFIG=ntrada-async.docker.yml`, so async appears to be the intended default — but that is a local-development file, not a production one | Platform architect |
| Q2 | **[ACTION NOW]** `orders-service` and `parcels-service` each keep their own `customers` collection, populated once from `customer_created` and never updated. Is that intentional? | `customer_state_changed` and `customer_became_vip` are published but consumed by nobody, so both copies silently drift from `customers-service` as customers change state. If pricing or eligibility ever reads those stale copies, customers get wrong outcomes | Likely a deliberate "only the creation snapshot matters" choice, but nothing in the code says so | Domain owner for Orders and Parcels |
| Q3 | **[handled later by HLD]** Where does `ordermaker-service` keep its saga state? | `AddChronicle()` is called with no persistence backend and no persistence package, which normally means in-memory. An in-flight order saga would then be lost on restart or on any second instance | Confirm the Chronicle default and either document in-memory as accepted or add a persistence backend | Platform architect |
| Q4 | **[handled later by HLD]** Does `operations-service` store operation state in MongoDB or in Redis? | It configures both, calls no `AddMongoRepository`, and sets a 300-second Redis expiry. If Redis with a 5-minute expiry is the real store, any saga or workflow running longer than five minutes loses its status before it finishes | Read `Services/OperationsService.cs` end-to-end and confirm at runtime | Platform architect |
| Q5 | **[ACTION NOW]** How does the Pact contract file travel between `Pacco.Services.Orders` (consumer) and `Pacco.Services.Parcels` (provider)? | Both repos have a `PACT/` directory and `Pactify` 1.1.0, but no Pact Broker is configured in either repo or either Travis pipeline. If the file is copied by hand, the contract test proves nothing about the other side's current behaviour | Someone who has run these tests must say whether a broker exists outside the repositories | Whoever owns the Orders/Parcels contract tests |
| Q6 | **[ACTION NOW]** Who or what triggers `start_delivery`? | `deliveries-service` subscribes to no event and no service publishes a `deliveries` command — the only entry point is a human calling `POST /deliveries` through the gateway. If an external dispatch system is meant to drive it, that integration is entirely undiscovered | Confirm whether deliveries are started manually or by a system not present in this workspace | Domain owner for Deliveries |
| Q7 | **[handled later by HLD]** Should the duplicated event contracts become a shared package? | `CustomerCreated` exists as four independent classes and `ResourceReserved` as three; the only real contract is the `snake_case` name on the wire. A field added by a publisher reaches no consumer until each one is edited by hand | Record the current state as an explicit choice or introduce a shared contracts package — this is an evolution decision, not a discovery finding | Platform architect |
| Q8 | **[ACTION NOW]** Who owns each repository? | There is no `CODEOWNERS`, no `CONTRIBUTING.md`, and no team metadata anywhere in the thirteen clones, so no finding in this document can be routed to a responsible team | Supply an owner per repository, or per subsystem as grouped in §4 | Delivery lead |
