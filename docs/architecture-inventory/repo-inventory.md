# Pacco — Repository Inventory

**Project:** Common Architecture
**Stage:** Architecture Discovery (evidence-based inventory)
**Backlog item:** `12915 — Pacco - Discovery` (Story, Priority High, Project Key `Common Architecture`)
**Date of analysis:** 2026-09-03 (second pass; supersedes the 2026-09-02 pass)
**Repositories analysed:** 13 (the writable artifact repository `Pacco.Context` is excluded from this inventory — it is the destination for these files, not a subject of analysis)
**Branch analysed in every clone:** `feature/12915/aidlc`

> **Corrections made in this pass.** Re-reading source that the first pass had inferred rather than opened produced four factual corrections, all in the direction of code over documentation and code over inference. They are listed here because a reader who saw the earlier version needs to know what changed, and each is applied in full in the sections and per-repository files below.
>
> | # | Earlier claim | What the code shows | Evidence |
> |---|---|---|---|
> | C1 | `operations-service` stores operations in a MongoDB collection named `operations`, which grows without bound. | It stores them **in Redis**, under key `requests:{id}`, with a **300-second sliding expiry**. There is no `operations` collection and no Mongo document type; `.AddMongo()` is called but no repository is registered. The risk is the **opposite** of unbounded growth: asynchronous outcomes become unreadable five idle minutes after they are written. | `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Services/OperationsService.cs`; `appsettings.json` `requests.expirySeconds: 300`; `Infrastructure/Extensions.cs` |
> | C2 | `identity-service` alone sets `validateIssuer: false`. | **Two** services do: `identity-service` and `operations-service`. The other eight set it to `true`. | `grep -rn '"validateIssuer"' hianshul100_Pacco.Services.*/src/*/appsettings.json` |
> | C3 | Whether CI runs `availability-service`'s tests is Unknown. | It does not. Its `.travis.yml` is the **only** one of eleven that omits `- ./scripts/test.sh`. The one repository with a real test suite is the one whose CI never executes it. | `Pacco.Services.Availability/.travis.yml` versus the ten sibling `.travis.yml` files |
> | C4 | The gRPC contract lives at `Protos/Operations.proto`. | It lives at the project root: `src/Pacco.Services.Operations.Api/Operations.proto`, with a second copy in `src/Pacco.Services.Operations.GrpcClient/`. | `find src -type f` in `Pacco.Services.Operations` |

This document records **what the code shows**. Where documentation and code disagree, both are stated and the code is treated as authoritative. Where evidence is thin, the entry says **Unknown** or **Needs validation** rather than guessing. Per-repository detail — all fourteen dimensions, the README-versus-tree reconciliation, and repository-specific assumptions, blockers, and open questions — is in `repo-summary/<repo-name>.md`.

**Knowledge-graph note on naming:** each deployable is named here by its exact deployable name (`availability-service`, `api-gateway`, and so on), with its repository and path stated on first mention. Technical identifiers — exchange names, message names, API paths, collection names, package names — are copied character-for-character from source.

---

## 1. Platform in one paragraph

Pacco is an **event-driven .NET Core 3.1 microservices platform** for exclusive parcel delivery, built around the problem of **limited resource availability**. Eleven deployable components sit behind one configuration-driven API gateway. Services integrate almost entirely through **RabbitMQ topic exchanges** using a command/event/rejection triad per operation, with six synchronous HTTP calls where a caller needs an immediate answer. Every service is built on the **Convey** framework (`0.4.*`); eight store state in **MongoDB** (no ORM, no migrations), `operations-service` stores its records in **Redis** with a five-minute expiry, and `pricing-service` and `ordermaker-service` persist nothing at all. All register with **Consul**, load-balances through **Fabio**, draws secrets and PKI certificates from **HashiCorp Vault**, and reports to **Jaeger**, **Prometheus/Grafana**, and **Seq**. The platform is choreographed, not orchestrated — with exactly one exception, a Chronicle saga in `ordermaker-service`. There is no Kubernetes, Helm, or Terraform anywhere; the only deployment mechanisms are Docker Compose and a pair of process-manager manifests.

---

## 2. Repository inventory — summary table

The table covers all fourteen required dimensions across three parts, one row per in-scope repository.

### 2a. Purpose, runtime, entrypoints, modules (dimensions 1–4)

| Repository | Deployable(s) & path | 1. Primary purpose | 2. Runtime/service type | 3. Key entrypoints | 4. Important modules/packages |
|---|---|---|---|---|---|
| `Pacco` | *none* (composition repo) | Umbrella repository: aggregate solution, Compose stacks, clone/pull scripts | Not a runtime — no `.csproj` anywhere | `compose/infrastructure.yml`, `compose/services.yml`, `compose/services-local.yml`, `services.yml`, `prod-services.yml`, `scripts/git-clone*.sh` | `Pacco.sln`, `compose/` (7 stacks), `docker-images.txt`, `assets/` (4 PNGs) |
| `Pacco.APIGateway` | `api-gateway` — `src/Pacco.APIGateway` | Single public entry point; YAML-configured routing, JWT termination, sync→async switch | ASP.NET Core, port `5000`→`80` | `Program.cs` (reads `NTRADA_CONFIG`), `ntrada.yml`, `ntrada-async.yml`, `+ .docker.yml` variants | `Ntrada 0.4.*` + 6 Ntrada extensions; `Convey.Logging/Metrics/Security`; `Infrastructure/CorrelationContextBuilder.cs`, `SpanContextBuilder.cs`, `HttpRequestHook.cs` |
| `Pacco.Services.Availability` | `availability-service` — `src/Pacco.Services.Availability.Api` | Limited-resource availability and reservations — the platform's core domain | ASP.NET Core + RabbitMQ consumer, port `5001` | `Program.cs` (public `CreateWebHostBuilder`), `Infrastructure/Extensions.cs`, `Metrics/MetricsJob.cs` | 4-project clean architecture + **5 test projects** incl. **NBomber** performance tests |
| `Pacco.Services.Customers` | `customers-service` — `src/Pacco.Services.Customers.Api` | Customer profile, lifecycle state, VIP policy | ASP.NET Core + RabbitMQ consumer, port `5002` | `Program.cs`, `Infrastructure/Extensions.cs` | 4-project clean architecture; `Core/Services/VipPolicy.cs`. **No tests** |
| `Pacco.Services.Deliveries` | `deliveries-service` — `src/Pacco.Services.Deliveries.Api` | Physical delivery execution: start, registrations, complete, fail | ASP.NET Core + RabbitMQ consumer, port `5003` | `Program.cs`, `Infrastructure/Extensions.cs` | 4-project clean architecture. **No tests** |
| `Pacco.Services.Identity` | `identity-service` — `src/Pacco.Services.Identity.Api` | Authentication authority: credentials, JWT issuance, refresh, revocation | ASP.NET Core + RabbitMQ consumer, port `5004` | `Program.cs` (**raw `UseEndpoints`**, not the dispatcher), `Infrastructure/Auth/` | 4-project clean architecture; `Convey.Auth` as issuer. **No tests** |
| `Pacco.Services.Operations` | `operations-service` — `src/Pacco.Services.Operations.Api`; plus non-deployed `Pacco.Services.Operations.GrpcClient` — `src/Pacco.Services.Operations.GrpcClient` | Asynchronous operation tracker: records every message, serves status by REST, gRPC, and SignalR | ASP.NET Core serving **HTTP + gRPC + SignalR + RabbitMQ**, port `5005` | `Program.cs` (`MapHub<PaccoHub>("/pacco")`, `MapGrpcService`), `Infrastructure/Subscriptions.cs`, `messages.json` | **Single-project**; `Grpc.AspNetCore 2.28.0`, `Google.Protobuf 3.11.4`, `Microsoft.AspNetCore.SignalR 1.1.0`, `SignalR.Redis 1.1.5`. **No tests** |
| `Pacco.Services.OrderMaker` | `ordermaker-service` — `src/Pacco.Services.OrderMaker` | Saga orchestrating end-to-end order making across four services | ASP.NET Core + RabbitMQ consumer + Chronicle saga, port **`5015`** | `Program.cs` (**`UseApp()`**, no `UseVault()`), `Sagas/AIOrderMakingSaga.cs`, `Extensions.cs` | **Single-project**; **`Chronicle 3.2.1`** (used nowhere else). **No tests** |
| `Pacco.Services.Orders` | `orders-service` — `src/Pacco.Services.Orders.Api` | Order aggregate and lifecycle — the platform's hub | ASP.NET Core + RabbitMQ consumer, port `5006` | `Program.cs`, `Infrastructure/Extensions.cs` | 4-project clean architecture; 3 HTTP clients; **`Pactify 1.1.0` consumer tests** |
| `Pacco.Services.Parcels` | `parcels-service` — `src/Pacco.Services.Parcels.Api` | Parcel catalogue and volume calculation | ASP.NET Core + RabbitMQ consumer, port `5007` | `Program.cs` (public `GetWebHostBuilder`), `Infrastructure/Extensions.cs`, `scripts/start-test.sh` | 4-project clean architecture; **`Pactify 1.1.0` provider tests** (the only tests here) |
| `Pacco.Services.Pricing` | `pricing-service` — `src/Pacco.Services.Pricing.Api` | Stateless discount calculation | ASP.NET Core only — **no broker, no database, no cache**, port `5008` | `Program.cs` (no `.AddApplication()`), `Core/Services/CustomerDiscountsService.cs` | **Single-project**; trimmed Convey set. **No tests**. Committed `.idea/`; **no `LICENSE`** |
| `Pacco.Services.Vehicles` | `vehicles-service` — `src/Pacco.Services.Vehicles.Api` | Vehicle catalogue (fleet master data) | ASP.NET Core + RabbitMQ consumer, port `5009` | `Program.cs`, `Infrastructure/Extensions.cs` | 4-project clean architecture; only paged query in the platform. **No tests**; **no `LICENSE`** |
| `Pacco.Web` | *none* | **Unverifiable — Missing Source Evidence** | **Unverifiable** — no code | **None exist** | **None exist** — repository contains only `README.md` (`# Pacco.Web`) |

### 2b. Integrations, data, messaging, APIs (dimensions 5–8)

| Repository | 5. External integrations | 6. Data stores / state (ORM, migrations, collections, cross-domain coupling) | 7. Messaging / async / events | 8. APIs exposed / consumed |
|---|---|---|---|---|
| `Pacco` | Declares all infra images: Consul, Fabio, Grafana, Jaeger, Mongo, Prometheus, RabbitMQ, Redis, Seq, Vault | Declares the shared `mongo` (vol `mongo:/data/db`) and `redis` (vol `redis:/data`) containers. **No ORM, no migrations, no collections owned** | Declares the RabbitMQ broker (custom image + plugins, `5672`/`15672`/`15692`). Publishes/consumes nothing | Exposes none; consumes none |
| `Pacco.APIGateway` | 10 backend services (HTTP); RabbitMQ (async profile); Jaeger; Seq; Prometheus; Fabio (**`loadBalancer.enabled: false`**) | **Stateless.** No store, no ORM, no migrations, no collections, no cross-domain coupling | **RabbitMQ, publish-only** (async profile). Publishes to exchanges `availability`, `customers`, `deliveries`, `identity`, `orders`, `parcels`, `vehicles`. Headers `message_context`, `span_context` | **Exposes the platform's entire public API** (see §5). Consumes all 10 service APIs. **No `ordermaker` module** |
| `...Availability` | `customers-service` over HTTP **with a client certificate**; RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger | Mongo db `availability-service`, collection **`resources`** + `inbox`/`outbox`. Convey `IMongoRepository<>` over `MongoDB.Driver` — **no ORM, no migration tool**. `ResourceDocument` has `Version` (optimistic concurrency). **No local replica of any foreign aggregate** | Exchange `availability`. Cmds in: `add_resource`, `delete_resource`, `release_resource`, `reserve_resource`. Evts in: `customer_created`, `vehicle_deleted`. Evts out: `resource_added`, `resource_deleted`, `resource_reserved`, `resource_reservation_released`, `resource_reservation_canceled` + 4 `*_rejected` | Exposes `GET/POST resources`, `GET/DELETE resources/{resourceId}`, `POST/DELETE resources/{resourceId}/reservations/{dateTime}`. Consumes `GET /customers/{id}/state`. Called by `ordermaker-service` (`GET /resources/{id}`) |
| `...Customers` | RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger. **`httpClient.services` empty** | Mongo db `customers-service`, collection **`customers`** + `inbox`/`outbox`. **No ORM, no migrations.** Origin of the `customer_created` fan-out that three services replicate | Exchange `customers`. Cmds in: `change_customer_state`, `complete_customer_registration`. Evts in: `signed_up`, `order_completed`. Evts out: `customer_created`, `customer_became_vip`, `customer_state_changed` + 2 `*_rejected` | Exposes `GET customers`, `GET customers/{id}`, `GET customers/{id}/state`, `POST customers`, `PUT customers/{id}/state/{state}`. Consumes none. Called by `availability-service` and `pricing-service` |
| `...Deliveries` | RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger. **`httpClient.services` empty.** **No carrier/tracking/geocoding/notification integration** | Mongo db `deliveries-service`, collection **`deliveries`** + `inbox`/`outbox`. **No ORM, no migrations.** Holds `OrderId` as a logical reference; **no local replica, no `customer_created` subscription** | Exchange `deliveries`. Cmds in: `add_delivery_registration`, `complete_delivery`, `fail_delivery`, `start_delivery`. **Evts in: none.** Evts out: `delivery_started`, `delivery_completed`, `delivery_failed`, `registration_added_to_delivery` + 3 `*_rejected` (**none for `add_delivery_registration`**) | Exposes `GET deliveries/{id}`, `POST deliveries`, `POST deliveries/{id}/{fail,complete,registrations}`. Consumes none. Called by nothing |
| `...Identity` | RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger. **No external IdP** — no OAuth2/OIDC/SAML/LDAP/MFA | Mongo db `identity-service`, collections **`users`**, **`refreshTokens`** + `inbox`/`outbox`. **No ORM, no migrations.** Passwords hashed via `IPasswordHasher<T>`. **The `UserId` minted here is used as `CustomerId` platform-wide** — an implicit, unenforced identity contract | Exchange `identity`. Cmds in: `sign_up` (**`sign_in` declared in `messages.json` but unsubscribed**). Evts out: `signed_up`, `signed_in` + `sign_up_rejected`, `sign_in_rejected` (**payloads carry `Email`**) | Exposes `POST sign-up`, `POST sign-in`, `GET me`, `GET users/{userId}`, `POST access-tokens/revoke`, `POST refresh-tokens/use`, `POST refresh-tokens/revoke`. **The last three are not routed at the gateway** |
| `...Operations` | **Every exchange** in `messages.json`; **Redis as both the operation store and the SignalR backplane**; Consul; Fabio; Vault; Jaeger; browsers (WebSockets); gRPC clients. **Mongo configured but never used** | **Redis only.** `IDistributedCache`, key `requests:{id}` (effective key `operations:requests:{id}`), value = JSON `OperationDto`, **`SlidingExpiration = 300s`**. **No ORM, no migrations, no collections, no query mechanism** — fetch by exact id only. **Mongo is configured** (`mongo.database: operations-service`, plus a Vault dynamic-credential lease) **and never opened**: no `AddMongoRepository<>`, no document type. **No outbox/inbox block** → no dedup, though `TrySetAsync` refuses to modify an already-`Completed`/`Rejected` operation. Holds a projection of every domain's activity; no FK | **Subscribes to everything, publishes nothing.** `Infrastructure/Subscriptions.cs` reads `messages.json` and **emits message types at runtime** via `AssemblyBuilder`/`TypeBuilder` + `MessageAttribute`, so no `Subscribe*<>` call appears in source | Exposes `GET operations/{operationId}` (**`auth: false`**); gRPC `GrpcOperationsService.GetOperation` + **`SubscribeOperations` (server-streaming, unauthenticated, unfiltered)**; SignalR hub `/pacco` (per-user groups). Consumes none |
| `...OrderMaker` | `availability-service` (`GET /resources/{id}`), `vehicles-service` (`GET /vehicles`); RabbitMQ; Redis; Consul; Fabio; Jaeger *(configured, never registered)*. **No Mongo, no Vault, no outbox**; `httpClient.type: ""` | **No database at all.** No ORM, no migrations, no collections. Saga state in `AIMakingOrderData` held by **Chronicle with no persistence provider → in-memory**; lost on restart, not shareable across instances | Exchange `ordermaker`. Evts in: `order_created`, `parcel_added_to_order`, `vehicle_assigned_to_order`, `order_approved`, `resource_reserved`. Cmds out onto **other services'** exchanges: `create_order`, `add_parcel_to_order`, `assign_vehicle_to_order`, `cancel_order`, `reserve_resource`. Evts out: `make_order_completed`, `make_order_rejected`. Sets the **`Saga` header** | Exposes `GET ""`, `POST orders`. **Not routed at the gateway.** Consumes Availability + Vehicles. Called by nothing in the workspace |
| `...Orders` | `parcels-service`, `pricing-service`, `vehicles-service` over HTTP (**3 — the most**); RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger. **No payment integration** | Mongo db `orders-service`, collections **`orders`**, **`customers`** + `inbox`/`outbox`. **No ORM, no migrations.** `CustomerDocument` is `public Guid Id { get; set; }` only — an **id-only replica** fed by `customer_created`. Order documents hold `VehicleId` and parcel ids as logical references | Exchange `orders`. **7 cmds in**, **7 evts in** (`customer_created`, `parcel_deleted`, `resource_reserved`, `resource_reservation_canceled`, `delivery_started`, `delivery_completed`, `delivery_failed`), **9 evts out**, **10 rejections** incl. `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found` | Exposes `GET orders`, `GET/DELETE orders/{orderId}`, `POST orders`, `POST/DELETE orders/{orderId}/parcels/{parcelId}`, `POST orders/{orderId}/vehicles/{vehicleId}`. **`ApproveOrder` has no HTTP route** — driven by `resource_reserved` |
| `...Parcels` | RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger. **`httpClient.services` empty** | Mongo db `parcels-service`, collections **`parcels`**, **`customers`** + `inbox`/`outbox`. **No ORM, no migrations.** `CustomerDocument` is id-only — the same replica pattern as Orders | Exchange `parcels`. Cmds in: `add_parcel`, `delete_parcel`. Evts in: `customer_created`, `parcel_added_to_order`, `parcel_deleted_from_order`, `order_canceled`, `order_deleted`. Evts out: `parcel_added`, `parcel_deleted` + 2 `*_rejected` | Exposes `GET parcels`, `GET parcels/volume`, `GET parcels/{parcelId}`, `POST parcels`, `DELETE parcels/{parcelId}`. Called by `orders-service` (`GET /parcels/{id}` — the Pact-covered contract) |
| `...Pricing` | `customers-service` (`GET /customers/{id}`); Consul; Fabio; Vault (**no `lease.mongo`**); Jaeger | **No data store of any kind.** No ORM, no query mechanism, no migrations, no collections, no cross-domain replica. Reads customer data per request and keeps nothing | **None.** No `rabbitMq` block, no broker packages. **The only service absent from `messages.json`** — no exchange, no messages. This is a confirmed absence, not a runtime-capture gap | Exposes **`GET pricing`** only (`?customerId=&orderPrice=`). Consumes `GET /customers/{id}`. Called by `orders-service` |
| `...Vehicles` | RabbitMQ; Mongo; Redis; Consul; Fabio; Vault; Jaeger. **`httpClient.services` empty.** No telematics/GPS/fleet integration | Mongo db `vehicles-service`, collection **`vehicles`** + `inbox`/`outbox`. **No ORM, no migrations.** No local replica; coupling runs outward only. **`VehicleDocument` has no capacity attribute** | Exchange `vehicles`. Cmds in: `add_vehicle`, `delete_vehicle`, `update_vehicle`. **Evts in: none.** Evts out: `vehicle_added`, `vehicle_updated`, `vehicle_deleted` + 3 `*_rejected`. `vehicle_deleted` **cascades** into `availability-service` resource removal | Exposes `GET vehicles` (**`PagedResult<VehicleDto>`**), `GET/PUT/DELETE vehicles/{vehicleId}`, `POST vehicles`. Called by `orders-service` and `ordermaker-service` |
| `Pacco.Web` | **None exist** | **None exist** — no store, ORM, migrations, collections, or coupling | **None exist** — no broker config, no publisher, no subscriber, no message names | **None exist.** Not in `ntrada*.yml`, no `httpClient.services` entry anywhere, absent from all compose and process manifests |

### 2c. Deployment, security, observability, decisions, questions, frontend (dimensions 9–14)

| Repository | 9. Deployment/runtime | 10. Security/auth | 11. Observability | 12. Decision files & feature flags | 13. Open questions | 14. Frontend stack |
|---|---|---|---|---|---|---|
| `Pacco` | 7 Compose stacks; `services.yml`/`prod-services.yml` process manifests; **no CI, no K8s/Helm/Terraform** | Vault dev mode `VAULT_DEV_ROOT_TOKEN_ID=secret`; Mongo/RabbitMQ/Redis started **without auth**; `docker-images.txt` contains sample Vault unseal keys | Prometheus + Grafana + Jaeger + Seq declared; ELK and InfluxDB documented but not started | `README.md`, `compose/infrastructure.yml`, `docker-images.txt`, `compose/services.yml`, `assets/*.png`. **No feature-flag system** — only startup booleans | Production target unknown; `devmentors` vs `hianshul100` images; diagram claims unverified | **None detected** — checked `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, view templates. `assets/` holds 4 PNGs only |
| `Pacco.APIGateway` | Docker (`NTRADA_CONFIG ntrada.docker`); composed on `5000:80` with **`ntrada-async.docker.yml`**; `.travis.yml`; **no K8s/Helm/Terraform** | **Committed JWT `issuerSigningKey`** (same value in 3 other files); `auth.global: false` (opt-in per route); **CORS `allowedOrigins: ['*']`**; `includeExceptionMessage: true`; `GET /operations/{id}` **`auth: false`**; `@user_id` binding prevents customer-id spoofing | Jaeger `serviceName: api-gateway`; `CorrelationContextBuilder`; Prometheus; Seq | `ntrada.yml`, `ntrada-async.yml`, `Program.cs`, `CorrelationContextBuilder.cs`, `Dockerfile`. **No feature-flag system** — `NTRADA_CONFIG` acts as a deploy-time profile switch | Async response contract undefined; Fabio disabled; no `ordermaker` route | **None detected** — checked all listed directories; Swagger UI at `/docs` is runtime-generated |
| `...Availability` | Docker; `5001:80`; in all three manifests; **no K8s/Helm/Terraform**. **`.travis.yml` is the only one of eleven that omits `- ./scripts/test.sh`** — the platform's only test suite is compiled by CI and never executed | JWT via `certs/localhost.cer`; **presents a client certificate** to Customers (`security.certificate.header: "Certificate"`); Vault PKI CN `availability-service.pacco.io`; `vault.token: "secret"`; **no in-service authorisation** | Jaeger + **RabbitMQ Jaeger plugin**; `.AddHandlersLogging()`; **`CustomMetricsMiddleware` + `MetricsJob`** (only custom metrics in the platform); Seq | `Infrastructure/Extensions.cs` (the platform template), `Core/Entities/Resource.cs`, `ResourceDocument.cs`, both exception mappers, `appsettings.json`. **No feature-flag system** | `disableTransactions: true`; Redis use unproven; reservation-conflict rule unverified | **None detected** — checked all listed directories |
| `...Customers` | Docker; `5002:80`; `.travis.yml`; in all three manifests | JWT; **the platform's only certificate *verifier*** — `security.certificate.acl: { "availability-service": { permissions: ["customers:read"] } }`; `vault.token: "secret"`; `role: admin` enforced **at the gateway only** | Jaeger + RabbitMQ plugin; handler logging; Seq; Prometheus | **`Core/Services/VipPolicy.cs` — VIP threshold hard-coded at 20 completed orders**; `Entities/Customer.cs`; `appsettings.json` ACL. **No feature-flag system** | VIP threshold not configurable; `customer_became_vip` has no subscriber; no tests | **None detected** — checked all listed directories |
| `...Deliveries` | Docker; `5003:80`; `.travis.yml`; in all three manifests | JWT; **no certificate block**; `vault.token: "secret"`; **no role gate and no ownership check on any delivery write route** | Jaeger + RabbitMQ plugin; handler logging; Seq; Prometheus | `Core/Entities/Delivery.cs`, `DeliveryRegistration.cs`, `Infrastructure/Extensions.cs`, `ExceptionToMessageMapper.cs`. **No feature-flag system** | **Nothing publishes `start_delivery`**; no rejection event for registrations; no tests | **None detected** — checked all listed directories |
| `...Identity` | Docker; `5004:80`; `.travis.yml`; in all three manifests; **`certs/localhost.pfx` baked into the image** | **Committed `issuerSigningKey`**; **`certificate.password: "test"`**; `expiryMinutes: 60`; **`validateIssuer: false`** (shared with `operations-service`; the other eight set `true`); `allowAnonymousEndpoints: ["/sign-in","/sign-up"]`; no rate limit or lockout | Jaeger + RabbitMQ plugin; handler logging; Seq; Prometheus. **No auth-specific metrics** | `appsettings.json`, `Program.cs` (raw `UseEndpoints`), `Infrastructure/Auth/`, `Core/Entities/RefreshToken.cs`, `PasswordService.cs`. **No feature-flag system** | Self-assigned `admin` role unverified; refresh endpoints unrouted; `sign_in` unsubscribed; no tests | **None detected** — checked all listed directories; no hosted login page |
| `...Operations` | Docker; `5005:80` with the platform's **only `depends_on` list** (8 services); `.travis.yml` (runs `test.sh`; no test projects exist); `scripts/proto/{lin,mac,win}-compile.sh`; HTTP/2 required for gRPC | **Committed `issuerSigningKey`**; **`validateIssuer: false`** (one of only two services); `GET /operations/{id}` **`auth: false`**; **gRPC `SubscribeOperations` unauthenticated and unfiltered**; SignalR correctly scoped to per-user groups; `allowAnonymousEndpoints: ["/sign-in","/sign-up"]` copied from Identity for paths that do not exist here; Bootstrap 4.0.0 from CDN with SRI | Jaeger `serviceName: "operations"`; Seq; Prometheus. **No `AddJaegerDecorators()`, no `AddHandlersLogging()`.** **No SignalR/gRPC connection metrics.** **Is itself the platform's operation-tracking tool** | **`messages.json`** (the platform's message catalogue), `Infrastructure/Subscriptions.cs`, **`Services/OperationsService.cs`** (operations are cache entries, not records), `Infrastructure/RequestsOptions.cs` + `requests.expirySeconds: 300` (**the platform's 5-minute outcome-retention window**), `Hubs/PaccoHub.cs`, `Operations.proto`, `Program.cs`, `wwwroot/ui/`. **No feature-flag system** (`messages.json` is the closest analogue, read at start-up) | **5-minute expiry on every operation record**; Redis as sole store; unfiltered gRPC stream; unused Mongo configuration; `.AddRedis()` called twice; purpose of `GrpcClient`; behaviour when down | **The only frontend in the workspace.** `wwwroot/ui/index.html` (Bootstrap 4.0.0 CDN), `wwwroot/ui/js/app.js` (vanilla IIFE, hard-codes `http://localhost:5005/pacco`), `wwwroot/ui/js/signalr.js` (**vendored 180,968-byte webpack UMD bundle**). **No `package.json`, no build tooling, no framework, no micro-frontend markers** |
| `...OrderMaker` | Docker; in `compose/services.yml` and Operations' `depends_on`; **absent from `services.yml` and `prod-services.yml`**; `.travis.yml`; **single instance only** (in-memory saga) | JWT config present; **no Vault at all** (only service); no certificate; **`POST /orders` takes a caller-supplied `CustomerId` with no auth or ownership check** | **No Jaeger registered despite a `jaeger` config block** — the orchestrator emits no traces. No `.AddHandlersLogging()`. No saga metrics | `Sagas/AIOrderMakingSaga.cs`, `Extensions.cs`, `Program.cs`, `Services/ResourceReservationsService.cs`, `Services/Clients/VehiclesServiceClient.cs`, `appsettings.json`. **No feature-flag system** | **4 of 5 compensations empty**; in-memory saga state; not routed anywhere; **"AI" is Future/Intended State (Not Implemented)** | **None detected** — checked all listed directories |
| `...Orders` | Docker; `5006:80`; `.travis.yml`; in all three manifests; Pact publication unproven | JWT; **no certificate block** (3 outbound calls unauthenticated); `vault.token: "secret"`; `@user_id` binding on create/list but **not on per-order routes** | Jaeger + RabbitMQ plugin; handler logging; Seq; Prometheus. No counters for the two `*_not_found` divergence events | `Core/Entities/Order.cs`, `Infrastructure/Extensions.cs`, `CustomerDocument.cs`, `PricingServiceClient.cs`, `Events/Rejected/`, the Pact consumer test, `appsettings.json`. **No feature-flag system** | Ownership checks unverified; **no payment anywhere**; 3 sync deps with no fallback; `*_not_found` unhandled | **None detected** — checked all listed directories |
| `...Parcels` | Docker; `5007:80`; `.travis.yml`; in all three manifests; **`scripts/start-test.sh`** for provider verification | JWT; **no certificate block**; `vault.token: "secret"`; `@user_id` on collection routes only; **`GET parcels/volume` has no customer scope** | Jaeger + RabbitMQ plugin; handler logging; Seq; Prometheus | `Core/Entities/Parcel.cs` (**volume calculation**), `CustomerDocument.cs`, `Infrastructure/Extensions.cs`, `Program.cs` (route ordering protects `parcels/volume`), the Pact provider test. **No feature-flag system** | Volume query scope; ownership checks; **volume calculation untested**; Pact publication unproven | **None detected** — checked all listed directories |
| `...Pricing` | Docker; `5008:80`; `.travis.yml`; in all three manifests; **trivially horizontally scalable**; committed `.idea/`; **no `LICENSE`** | JWT; no certificate (its Customers call is unauthenticated, unlike Availability's); `vault.token: "secret"`; **no authorisation — `customerId` taken from the query string as given** | Jaeger (no RabbitMQ plugin — no broker); handler logging; Seq; Prometheus. **Invisible to `operations-service`** | **`Core/Services/CustomerDiscountsService.cs` — the entire pricing policy hard-coded**: `>=10`→`0.1`, `>3 && <10`→`0.05`, `<=3 && >0`→`0.02`, `+0.1` if VIP (max 20%). `Program.cs`; `appsettings.json`. **No feature-flag system** | Rates not configurable; VIP stacking; boundary values; hard dependency on Customers; no tests | **None detected** — checked all listed directories |
| `...Vehicles` | Docker; `5009:80`; `.travis.yml`; in all three manifests; **no `LICENSE`** | JWT; **no certificate block**; `vault.token: "secret"`; **no role gate on fleet mutation** — any authenticated user can add/update/delete a vehicle | Jaeger + RabbitMQ plugin; handler logging; Seq; Prometheus. No deletion counter | `Core/Entities/Vehicle.cs` (**no capacity attribute**), `Infrastructure/Extensions.cs`, `Queries/BrowseVehicles.cs`, `Events/VehicleDeleted.cs`, and cross-repo `ntrada.yml`'s `onSuccess.data` rewrite. **No feature-flag system** | **No vehicle capacity anywhere**; deletion cascades destructively; OrderMaker reads only page 1; paging metadata lost at the gateway | **None detected** — checked all listed directories |
| `Pacco.Web` | **None exist** — no Dockerfile, no CI, no compose entry, no manifest entry. The only repository with no CI configuration at all | **None exist** | **None exist** | **None exist** — no ADR, no `docs/`. **No feature-flag system**; no flag keys | Whether the repository should hold a web client at all; branch discrepancy | **None detected** — checked `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, view templates. **None of these directories exist**; the repository contains only `README.md` |

---

## 3. Cross-repo relationships

### 3a. Synchronous HTTP calls between services (six, all confirmed in source)

| Caller | Callee | Exact call | Authenticated? | Evidence |
|---|---|---|---|---|
| `availability-service` | `customers-service` | `GET {customers-service}/customers/{customerId}/state` | **Yes — client certificate in the `Certificate` header**, matched by an ACL granting `customers:read` | `Availability/.../Services/CustomersServiceClient.cs`; `Customers/.../appsettings.json` `security.certificate.acl` |
| `orders-service` | `parcels-service` | `GET {parcels-service}/parcels/{id}` | No | `Orders/.../Services/ParcelsServiceClient.cs` |
| `orders-service` | `pricing-service` | `GET {pricing-service}/pricing?customerId={customerId}&orderPrice={orderPrice}` | No | `Orders/.../Services/PricingServiceClient.cs` |
| `orders-service` | `vehicles-service` | `GET {vehicles-service}/vehicles/{id}` | No | `Orders/.../Services/VehiclesServiceClient.cs` |
| `ordermaker-service` | `vehicles-service` | `GET {vehicles-service}/vehicles` → `PagedResult<VehicleDto>` | No | `OrderMaker/.../Clients/VehiclesServiceClient.cs` |
| `ordermaker-service` | `availability-service` | `GET {availability-service}/resources/{resourceId}` | No | `OrderMaker/.../Clients/AvailabilityServiceClient.cs` |
| `pricing-service` | `customers-service` | `GET {customers-service}/customers/{id}` | No | `Pricing/.../Services/Clients/CustomersServiceClient.cs` |

Service addresses are resolved by Consul service name, with `httpClient.type: "fabio"` in every service except `ordermaker-service` (`""`). **Only one of these seven calls carries a caller credential** — the platform applies its mutual-authentication pattern to exactly one edge.

### 3b. Event flows (who publishes what to whom)

```
identity-service ──signed_up──────────────► customers-service
customers-service ──customer_created──────► availability-service, orders-service, parcels-service
                  ──customer_became_vip───► (no subscriber found)
                  ──customer_state_changed► (no subscriber found)

orders-service ──order_created────────────► ordermaker-service
               ──parcel_added_to_order────► parcels-service, ordermaker-service
               ──parcel_deleted_from_order► parcels-service
               ──vehicle_assigned_to_order► ordermaker-service
               ──order_approved───────────► ordermaker-service
               ──order_canceled───────────► parcels-service
               ──order_deleted────────────► parcels-service
               ──order_completed──────────► customers-service   (drives the VIP policy)
               ──order_delivering─────────► (no subscriber found)

parcels-service ──parcel_deleted──────────► orders-service

availability-service ──resource_reserved──► orders-service (approves the order), ordermaker-service
                     ──resource_reservation_canceled► orders-service
                     ──resource_added / resource_deleted / resource_reservation_released ► (no subscriber found)

vehicles-service ──vehicle_deleted────────► availability-service   (removes the resource — cascading)
                 ──vehicle_added / vehicle_updated ► (no subscriber found)

deliveries-service ──delivery_started─────► orders-service
                   ──delivery_completed───► orders-service
                   ──delivery_failed──────► orders-service
                   ──registration_added_to_delivery ► (no subscriber found)

ordermaker-service ──create_order, add_parcel_to_order, assign_vehicle_to_order, cancel_order ► orders-service
                   ──reserve_resource─────► availability-service
                   ──make_order_completed / make_order_rejected ► (no subscriber found)

api-gateway (async profile) ──► availability, customers, deliveries, identity, orders, parcels, vehicles exchanges

operations-service ◄── every message on every exchange (dynamic subscription)
pricing-service ── no messaging at all
```

**Events with no subscriber in the workspace (nine):** `customer_became_vip`, `customer_state_changed`, `order_delivering`, `resource_added`, `resource_deleted`, `resource_reservation_released`, `registration_added_to_delivery`, `vehicle_added`, `vehicle_updated`, plus `make_order_completed` and `make_order_rejected`. All are recorded by `operations-service`, which records everything; none drives behaviour anywhere. Whether they exist for future use, for external consumers, or by oversight is **Unknown**.

### 3c. The order lifecycle end to end (the platform's principal flow)

1. `POST /identity/sign-up` → `identity-service` publishes **`signed_up`**.
2. `customers-service` creates the customer and publishes **`customer_created`**; `availability-service`, `orders-service`, and `parcels-service` each record the customer id locally.
3. `POST /parcels` → `parcels-service` publishes **`parcel_added`**.
4. `POST /orders` → `orders-service` creates the order (calling `pricing-service`, which calls `customers-service`) and publishes **`order_created`**.
5. `POST /orders/{orderId}/parcels/{parcelId}` → `orders-service` calls `parcels-service` (`GET /parcels/{id}`) and publishes **`parcel_added_to_order`**.
6. `POST /orders/{orderId}/vehicles/{vehicleId}` → `orders-service` calls `vehicles-service` and publishes **`vehicle_assigned_to_order`**.
7. A reservation is made on `availability-service`, which publishes **`resource_reserved`**; `orders-service` consumes it and **approves the order**, publishing `order_approved`.
8. A delivery is started on `deliveries-service` — **by an actor not present in any of the thirteen repositories** — which publishes `delivery_started` → `order_delivering`, then `delivery_completed` → **`order_completed`**.
9. `customers-service` consumes `order_completed`, increments the completed-order count, and applies the VIP policy at 20 orders.

**The saga alternative:** `POST /orders` on `ordermaker-service` (port `5015`, not routed at the gateway) runs steps 4–7 automatically via `AIOrderMakingSaga`, correlating everything on `OrderId` and stamping a `Saga` header that all services forward.

**Two breaks in this chain, both confirmed:** nothing in the workspace publishes `start_delivery` or calls `POST /deliveries` (step 8 has no known trigger), and `ordermaker-service` is unreachable through the gateway and absent from both process-manager manifests (the saga path is not exercised by either documented start-up).

### 3d. Shared infrastructure

| Component | Shared by | Isolation |
|---|---|---|
| MongoDB (one `mongo` container) | **8 services in fact; 9 by configuration** | **Separate logical database per service**, named `<service>-service`. No shared collections, no cross-database joins, no foreign keys. `operations-service` declares `mongo.database: operations-service` and a Vault dynamic-credential lease for it but **never opens it** — no repository, no document type, no collection |
| RabbitMQ (one broker) | 9 services + the gateway | **One topic exchange per service**, named after the service without the `-service` suffix. Queue template `<service>/{{exchange}}.{{message}}` |
| Redis (one instance) | 9 services | Instance prefix per service (`<service>:`). Only `operations-service` has an identified functional use, and it has **two**: the SignalR backplane **and** the operation store itself (`operations:requests:{id}`, 300-second sliding expiry). In the other eight services Redis is configured and no read or write was found. **This makes one un-replicated Redis instance the system of record for every in-flight asynchronous outcome on the platform** |
| Consul | 10 services + gateway lookups | Service names `<name>-service`; `pingEndpoint: ping`, interval 3s |
| Vault | 9 services (**not `ordermaker-service`**) | KV v2 `kv/<service>-service/settings`; PKI role `<service>-service`, CN `<service>-service.pacco.io`; dynamic Mongo credentials with auto-renewal |
| Jaeger, Prometheus, Grafana, Seq | all services | Per-service `serviceName`; ELK and InfluxDB sinks configured but disabled everywhere |

### 3e. Cross-domain data coupling

| Pattern | Where | Nature |
|---|---|---|
| **Id-only customer replica** | `orders-service` and `parcels-service` each own a `customers` collection whose `CustomerDocument` is `public Guid Id { get; set; }` and nothing more | Populated solely by `customer_created`. Not a foreign key — no relational constraint exists anywhere — but a hard operational dependency: a missed event makes the customer unknown and their work rejected |
| **Shared identity GUID** | `identity-service`'s `UserId` is used as `CustomerId` by Customers, Orders, and Parcels, and is what the gateway binds via `@user_id` | Implicit and unenforced. Nothing validates the correspondence |
| **Logical references without enforcement** | `deliveries-service` holds `OrderId`; `orders-service` holds `VehicleId` and parcel ids; `availability-service` resources correspond to vehicles by id | No existence checks, no cascade rules except the destructive `vehicle_deleted` → resource removal |
| **Vehicle deletion cascade** | `vehicles-service` → `availability-service` | Deleting a vehicle removes its resource and every reservation on it, with no in-use check and no notification to `orders-service`, which retains `VehicleId` references |

---

## 4. Suspected platform subsystems

Grouped by responsibility. These groupings are inferred from code structure and message flow; they are not declared anywhere in the repositories.

| Subsystem | Components | Evidence | Confidence |
|---|---|---|---|
| **Edge / API** | `api-gateway` | The sole public entry; `ntrada*.yml` defines the whole external contract | High |
| **Identity & access** | `identity-service`, plus JWT validation in all ten services and the certificate ACL in `customers-service` | Token issuance here, validation everywhere, one mutual-auth edge | High |
| **Customer domain** | `customers-service`, with id-only replicas in `orders-service` and `parcels-service` | `customer_created` fan-out; `VipPolicy` | High |
| **Ordering core** | `orders-service`, `parcels-service` | 7 commands, 7 event subscriptions, 9 events; bidirectional parcel-to-order events; the platform's only Pact pair | High |
| **Capacity & fleet** | `availability-service`, `vehicles-service` | Resource reservations; `vehicle_deleted` → resource removal | High |
| **Fulfilment** | `deliveries-service` | Closes the order lifecycle via three delivery events | High |
| **Commercial policy** | `pricing-service`, plus `VipPolicy.cs` in `customers-service` | The discount table and the VIP threshold — the platform's entire commercial logic, both hard-coded, in two different repositories | Medium — the grouping is real, the co-location is not |
| **Process orchestration** | `ordermaker-service` | The only saga; the only Chronicle dependency; the `Saga` header protocol | High, though the subsystem's operational status is **Unknown** |
| **Operational visibility** | `operations-service` | Subscribes to everything; REST + gRPC + SignalR; owns `messages.json` | High |
| **Platform infrastructure** | `Pacco` (compose stacks), Consul, Fabio, Vault, RabbitMQ, MongoDB, Redis, Jaeger, Prometheus, Grafana, Seq | `compose/infrastructure.yml`, `docker-images.txt` | High |
| **Web client** | `Pacco.Web` (empty), plus the demonstration page in `operations-service` | **Unverifiable — Missing Source Evidence** for `Pacco.Web`; the only real page is a development demo | Low — no delivered web client exists |

---

## 5. Public API surface (from `ntrada.yml` / `ntrada-async.yml`)

All routes require a bearer token except where noted. In the asynchronous profile, write routes publish to RabbitMQ instead of proxying downstream.

| Prefix | Routes | Notes |
|---|---|---|
| `/` | `GET` | `return_value`, anonymous |
| `/availability` | `GET/POST /resources`, `GET/DELETE /resources/{resourceId}`, `POST/DELETE /resources/{resourceId}/reservations/{dateTime}` | reservation binds `customerId: @user_id` |
| `/customers` | `GET /` (admin), `GET /me`, `GET /{customerId}` (admin), `GET /{customerId}/state` (admin), `POST /`, `PUT /{customerId}/state/{state}` (admin) | `POST` binds `customerId: @user_id`, schema-validated |
| `/deliveries` | `GET /{deliveryId}`, `POST /`, `POST /{deliveryId}/fail`, `POST /{deliveryId}/complete`, `POST /{deliveryId}/registrations` | **no role gate, no ownership binding** |
| `/identity` | `GET /users/{userId}` (admin), `GET /me`, `POST /sign-up` (anonymous), `POST /sign-in` (anonymous) | refresh and revocation endpoints **not routed** |
| `/operations` | `GET /{operationId}` | **anonymous** |
| `/orders` | `GET /`, `GET/DELETE /{orderId}`, `POST /`, `POST/DELETE /{orderId}/parcels/{parcelId}`, `POST /{orderId}/vehicles/{vehicleId}` | list and create bound to `@user_id`; per-order routes not bound |
| `/parcels` | `GET /`, `GET /volume`, `GET/DELETE /{parcelId}`, `POST /` | list and create bound to `@user_id` |
| `/pricing` | `GET /` | bound to `@user_id` at the gateway only |
| `/vehicles` | `GET /`, `GET/PUT/DELETE /{vehicleId}`, `POST /` | **no role gate**; `GET /` unwrapped via `onSuccess.data: response.data.items` |
| `/docs` | `GET` | Swagger UI |

**Not exposed at the gateway:** `ordermaker-service` entirely; Identity's three token-management endpoints; the Operations gRPC service and SignalR hub.

---

## 6. Documentation vs tree — platform-level patterns

Recurring documentation problems observed across the thirteen repositories. Per-repository detail is in each `repo-summary/<repo-name>.md` under "README vs repository".

| # | Pattern | Scope | Assessment |
|---|---|---|---|
| D1 | **The documented source path is wrong in nine of the ten service repositories.** Each README says the run command is executed "in the `/src/Pacco.Services.<X>` directory", but the host project on disk is `/src/Pacco.Services.<X>.Api`. Affects Availability, Customers, Deliveries, Identity, Operations, Orders, Parcels, Pricing, Vehicles. **`Pacco.Services.OrderMaker` is the only correct one**, because it is genuinely a single project without the `.Api` suffix. | 9 repos | **Stale doc.** Following the README verbatim fails in nine repositories |
| D2 | **Every README points at the upstream `devmentors` organisation** — links, Travis badges (`api.travis-ci.org/devmentors/...`), and Docker Hub images (`devmentors/pacco.*`) — while the analysed code is a `hianshul100` fork on `feature/12915/aidlc`, where Travis is not configured to run. `Pacco/compose/services.yml` also pulls `devmentors/*` images. | 12 repos + compose | **Stale doc**, with a real consequence: the composed platform runs third-party images, not the analysed code |
| D3 | **Message contracts are undocumented everywhere.** Not one README lists the commands a service consumes or the events it publishes. The only complete catalogue is `messages.json`, which lives inside `Pacco.Services.Operations` and is mentioned in no README, including its own. | All 10 services | **Documentation gap.** The platform's integration contract is discoverable only by reading source |
| D4 | **The asynchronous gateway profile is undocumented.** `ntrada-async.yml` and `scripts/start-async.sh` are not mentioned in the gateway's README, yet `compose/services.yml` runs exactly that profile. The README describes a synchronous proxy; the composed system is an asynchronous publisher. | `Pacco.APIGateway`, `Pacco` | **Documentation gap**, and the most consequential one — it changes the platform's integration style |
| D5 | **`docker-images.txt` documents infrastructure that no service uses:** SQL Server 2017, PostgreSQL, InfluxDB, Elasticsearch, Kibana, Logstash, Mongo Express. No service configuration references any of them; `logger.elk.enabled` and `metrics.influxEnabled` are `false` everywhere. | `Pacco` | **Future/Intended State (Not Implemented)** — or leftovers from a different setup |
| D6 | **The root README omits `Pacco.Web`,** listing twelve repositories to clone while the discovery request names thirteen. `Pacco.Web` itself contains only a one-line README. | `Pacco`, `Pacco.Web` | **Conflict**, recorded and not reconciled |
| D7 | **Branch discrepancy:** the discovery request specifies branch `master` for all thirteen repositories; every clone in this workspace is on `feature/12915/aidlc`. | All 13 | **Conflict**, recorded and not reconciled. This inventory describes `feature/12915/aidlc` |
| D8 | **Deviations from the platform norm are never explained.** `pricing-service` has no database or broker; `ordermaker-service` has no Vault, no Mongo, no outbox, and no Jaeger registration; `operations-service` has no layering and no outbox; `identity-service` alone sets `validateIssuer: false`. Each is a deliberate-looking choice, and none is documented anywhere. | 4 repos | **Documentation gap** |
| D9 | **No ADRs and no `docs/` directory exist in any of the thirteen repositories.** Architectural decisions are recorded only as code and configuration, plus four undocumented PNG diagrams in `Pacco/assets/`. | All 13 | **Documentation gap** |
| D10 | **Test assets are undocumented,** including the platform's only performance tests (NBomber, in Availability) and its only contract-testing pair (Pactify, Orders ↔ Parcels, plus `scripts/start-test.sh`). | 3 repos | **Documentation gap** |
| D11 | **CI runs the test step everywhere except the one repository that has tests.** Ten of the eleven `.travis.yml` files (the gateway and nine services) are byte-identical and run `./scripts/build.sh` then `./scripts/test.sh`. `Pacco.Services.Availability`'s differs by exactly one missing line — `- ./scripts/test.sh` — and it is the only repository with unit, integration, end-to-end, and performance suites. Eight repositories that contain no test project run `dotnet test` on nothing; the one with five test projects never runs it. `Pacco` and `Pacco.Web` have no CI at all. | 13 repos | **Conflict.** The Travis badges in every README imply CI verification the pipeline does not perform |
| D12 | **`operations-service` carries configuration for a datastore it never uses.** `mongo.connectionString`, `mongo.database: operations-service`, `mongo.seed`, `.AddMongo()`, and a Vault dynamic-credential lease with auto-renewal all exist for a MongoDB database the service never opens; its real store is Redis. Similarly, `jwt.allowAnonymousEndpoints: ["/sign-in","/sign-up"]` names two paths this service does not serve. | `Pacco.Services.Operations` | **Conflict** between configuration and code. Configuration blocks appear to have been copied service to service without pruning |
| D13 | **Repository hygiene:** `Pacco.Services.Pricing` and `Pacco.Services.Vehicles` have **no `LICENSE`** while the other eleven do; `Pacco.Services.Pricing` has a committed JetBrains Rider `.idea/` directory at `src/Pacco.Services.Pricing.Api/.idea/`. | 2 repos | **Needs validation** |

---

## 7. Gaps, unknowns, and platform-level observations

### 7a. Confirmed absences (verified across all thirteen repositories)

| Absence | How it was verified |
|---|---|
| **No ORM and no migration tooling** | Eight services use Convey's `IMongoRepository<>` over `MongoDB.Driver`; `operations-service` uses `IDistributedCache` over Redis; `pricing-service` and `ordermaker-service` persist nothing. No Entity Framework Core, NHibernate, Dapper, Flyway, Liquibase, Alembic, or EF migrations anywhere. Collections are created implicitly on first write |
| **No feature-flag system** | No LaunchDarkly, Unleash, Flagsmith, Split, or OpenFeature dependency or configuration in any repository. The only switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `vault.enabled`, `outbox.enabled`, `jaeger.enabled`, `metrics.enabled`, `swagger.enabled`, `logger.*.enabled`, `security.certificate.enabled`) plus the `NTRADA_CONFIG` deploy-time profile switch. **There are no runtime-toggleable flag keys to record** |
| **No Kubernetes, Helm, or Terraform** | `find` across all thirteen clones returned no chart, manifest, or `.tf` file |
| **No GitHub Actions** | CI is Travis only (`.travis.yml` in eleven repositories — the gateway and the ten services; none in `Pacco` or `Pacco.Web`) |
| **The platform's main test suite is never executed by CI** | Ten `.travis.yml` files run `./scripts/test.sh` (`dotnet test`); eight of those repositories contain no test project, so the step passes on nothing. The two that do — `Pacco.Services.Orders` and `Pacco.Services.Parcels`, holding only the Pact pair — are therefore the **only** tests any pipeline invokes. `Pacco.Services.Availability`, with all five real test projects, is the single repository whose `.travis.yml` omits the line |
| **No `package.json` anywhere** | The only frontend assets in the workspace are a static page and a vendored webpack bundle inside `operations-service` |
| **No payment, invoicing, or settlement** | Orders carry a total price; nothing collects it |
| **No vehicle capacity model** | `parcels-service` computes parcel volume; `VehicleDocument` has no capacity attribute; no code compares the two |
| **No carrier, tracking, geocoding, or notification integration** | `deliveries-service` has an empty `httpClient.services` and no external client |
| **No external identity provider** | No OAuth2/OIDC/SAML/LDAP/MFA in `identity-service` |
| **No ADRs** | No `docs/` directory or decision record in any repository |

### 7b. Platform-level unknowns

| # | Unknown | Why it could not be resolved |
|---|---|---|
| U1 | The production deployment target | Only Compose and two process-manager manifests exist; nothing describes a production runtime |
| U2 | Which container images the target deployment should run | Compose pulls `devmentors/*`; the source is a `hianshul100` fork |
| U3 | What triggers a delivery | Nothing publishes `start_delivery` or calls `POST /deliveries` |
| U4 | Whether `ordermaker-service` is operational | Absent from the gateway and from both process-manager manifests |
| U5 | What the four diagrams in `Pacco/assets/` assert | PNG images; content not machine-readable |
| U6 | Whether the Pact contract is published and verified automatically | No broker URL or publish step in either repository |
| U7 | Whether Redis is used at all outside `operations-service` | Configured in nine services; a functional use was identified in only one |
| U8 | The response contract for asynchronous writes | The gateway returns immediately; nothing documents how a caller learns the outcome. What *is* now established is the window: whatever the contract is, it expires **300 seconds** after the last touch (`operations-service`, `requests.expirySeconds`) |
| U11 | Why `operations-service` is configured for a MongoDB database it never opens, including a Vault dynamic-credential lease | The configuration is present and complete; the code that would use it does not exist. Nothing records whether the feature was removed, never built, or copied in |
| U12 | Whether omitting `- ./scripts/test.sh` from `Pacco.Services.Availability/.travis.yml` was deliberate | The integration and end-to-end suites need MongoDB and RabbitMQ, which the Travis job never starts — a plausible reason, but one written down nowhere, and it would not explain skipping the unit tests |
| U9 | Whether services enforce ownership on per-resource routes | Gateway binds `@user_id` on collection routes only; handlers were not traced exhaustively |
| U10 | Whether the platform assumes a trusted internal network | No service performs authorisation; all role gating is at the gateway |

### 7c. Security observations (evidence only — no recommendations at this stage)

These are recorded as findings. Remediation belongs to a later stage.

1. **A shared JWT signing key is committed in plain text in four files across three repositories** — `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml`, `Pacco.Services.Identity/src/Pacco.Services.Identity.Api/appsettings.json`, and `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json`. The literal value is deliberately not reproduced in this inventory. Anyone with repository read access can mint tokens the whole platform accepts, including `admin` tokens.
2. **`vault.token: "secret"`** appears in nine services' `appsettings.json` — the dev-mode root token from `compose/infrastructure.yml`.
3. **`certificate.password: "test"`** in `Pacco.Services.Identity`'s settings, with `certs/localhost.pfx` committed and baked into the image.
4. **`Pacco/docker-images.txt` contains sample Vault unseal keys and a root token** in plain text.
5. **CORS `allowedOrigins: ['*']`** and **`includeExceptionMessage: true`** at the gateway.
6. **Anonymous route:** `GET /operations/{operationId}` (`auth: false`) returns `userId`, `state`, `code`, and `reason`.
7. **Unauthenticated, unfiltered gRPC stream:** `SubscribeOperations` delivers every user's operations to any caller with network access.
8. **No role gate on destructive routes:** any authenticated user can start, complete, or fail any delivery, and can add, update, or delete any vehicle — the latter cascading into resource and reservation deletion.
9. **Only one of seven cross-service HTTP calls is authenticated** (Availability → Customers, by certificate with an ACL).
10. **Identity's `POST /sign-in` has no rate limit, lockout, or CAPTCHA**, and the service emits no authentication metrics.
11. **Two services set `validateIssuer: false`** — `identity-service` and `operations-service`. The other eight set it to `true`. Any token signed with the shared key is accepted by those two regardless of issuer.
12. **Rejection events `sign_up_rejected` and `sign_in_rejected` carry `Email`** into the broker and into subscribers' logs.
13. **MongoDB, RabbitMQ, and Redis run without authentication** in `compose/infrastructure.yml`.
14. **`ordermaker-service` has no Vault integration**, so no PKI identity and no managed secrets.

### 7d. Reliability observations

1. **`outbox.disableTransactions: true` in all seven outbox-enabled services** — state writes and message enqueues are not atomic.
2. **`operations-service` and `ordermaker-service` have no outbox or inbox**, so neither deduplicates redelivered messages. In `operations-service` this is partly mitigated: `TrySetAsync` refuses to modify an operation already in a terminal state, so a redelivered message cannot flip a completed operation back to `pending` — but it does restart the 5-minute expiry clock on one still pending.
3. **`ordermaker-service` holds saga state in memory** — a restart strands in-flight orders; only one instance can run.
4. **Four of the saga's five compensations are empty** — a mid-saga failure can permanently hold a vehicle assignment and a resource reservation.
5. **Asynchronous outcomes survive five minutes.** `operations-service` keeps every operation record in Redis alone, with `SlidingExpiration = 300s` (`requests.expirySeconds: 300`), and writes it nowhere else. Since the composed gateway runs the asynchronous profile, this is the only channel through which a caller can learn whether a write succeeded — so five idle minutes after any write, its outcome is unrecoverable and `GET /operations/{id}` returns `404`, indistinguishable from an id that never existed. A Redis restart loses every in-flight operation across the platform at once. *(This replaces an earlier, mistaken note about the `operations` collection growing without bound — there is no such collection; see C1 at the head of this document.)*
6. **`orders-service` has three synchronous dependencies** with only Convey's two HTTP retries; no cache, fallback, or circuit breaker was found.
7. **No service performs a health check beyond Consul's `ping`**; no readiness or dependency check exists.
8. **`ordermaker-service` emits no traces** despite carrying a full Jaeger configuration block.

### 7e. Testing coverage

| Repository | Tests present | Does CI run them? |
|---|---|---|
| `Pacco.Services.Availability` | Unit, Integration, EndToEnd, **Performance (NBomber 0.16.0)**, Shared fixtures — **the only full pyramid** | **No.** Its `.travis.yml` is the only one of eleven that omits `- ./scripts/test.sh`. `scripts/test.sh` exists and contains `dotnet test`; nothing invokes it |
| `Pacco.Services.Orders` | Pact **consumer** tests (`Pactify 1.1.0`) — one endpoint | Yes — `./scripts/test.sh` runs |
| `Pacco.Services.Parcels` | Pact **provider** tests (`Pactify 1.1.0`) — the same endpoint; no domain tests | Yes — `./scripts/test.sh` runs, though provider verification also needs a running service (`scripts/start-test.sh`), which CI does not start. Whether the verification actually executes is **Unknown** |
| `Pacco.APIGateway`, `...Customers`, `...Deliveries`, `...Identity`, `...Operations`, `...OrderMaker`, `...Pricing`, `...Vehicles` | **None** | The step runs and tests nothing |
| `Pacco`, `Pacco.Web` | **None** | **No CI at all** — neither repository has a `.travis.yml` |

Ten of thirteen repositories have no tests, including the service that owns authentication, the service that owns the platform's pricing policy, and the only distributed-transaction coordinator. The distribution of the CI test step is exactly inverted against where the tests are: it runs in eight repositories that have nothing to test and is absent from the one that has the most.

---

## 8. Coverage

Every project enumerated from the authoritative manifest (`*.sln` / `*.csproj`, or the repository tree where no solution exists) is either analysed under the fourteen dimensions or excluded with a stated reason.

| Repository | Enumerated | Documented | Excluded (with reason) |
|---|---|---|---|
| `Pacco` | 0 projects (no `.csproj`); 7 compose stacks, 2 process manifests, 5 scripts, 1 solution file, `docker-images.txt`, `assets/` | All analysed as infrastructure assets in `repo-summary/Pacco.md` | **1 — `Pacco.sln`:** an aggregate solution referencing projects in sibling clones; it defines no code of its own, and each referenced project is analysed in its own repository |
| `Pacco.APIGateway` | 1 project — `src/Pacco.APIGateway` | 1 | 0 |
| `Pacco.Services.Availability` | 9 projects — 4 `src/*` + 5 `tests/*` | 9 (the four source projects under all fourteen dimensions; the five test projects under dimensions 4 and 9) | 0 |
| `Pacco.Services.Customers` | 4 projects — `src/*` | 4 | 0 |
| `Pacco.Services.Deliveries` | 4 projects — `src/*` | 4 | 0 |
| `Pacco.Services.Identity` | 4 projects — `src/*` | 4 | 0 |
| `Pacco.Services.Operations` | 2 projects — `src/Pacco.Services.Operations.Api`, `src/Pacco.Services.Operations.GrpcClient` | 2 (the `.Api` project under all fourteen dimensions; `GrpcClient` under dimensions 4, 8, and 13 as a non-deployed client) | 0 |
| `Pacco.Services.OrderMaker` | 1 project — `src/Pacco.Services.OrderMaker` | 1 | 0 |
| `Pacco.Services.Orders` | 5 projects — 4 `src/*` + 1 `tests/*` | 5 | 0 |
| `Pacco.Services.Parcels` | 5 projects — 4 `src/*` + 1 `tests/*` | 5 | 0 |
| `Pacco.Services.Pricing` | 1 project — `src/Pacco.Services.Pricing.Api` | 1 | 0 |
| `Pacco.Services.Vehicles` | 4 projects — `src/*` | 4 | 0 |
| `Pacco.Web` | 0 projects — no manifest of any kind exists | 0 | **Whole repository — "Unverifiable — Missing Source Evidence":** the repository's entire tracked content is `README.md` containing the single line `# Pacco.Web`. All fourteen dimensions are recorded as unverifiable in `repo-summary/Pacco.Web.md` |
| **Totals** | **40 projects** across 13 repositories | **40 documented** | **1 solution file excluded with reason; 1 repository recorded as unverifiable** |

**Repository excluded from this inventory by instruction:** `Pacco.Context` — the writable artifact repository that receives these files. It is the destination for the inventory, not a subject of it.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The branch present in every clone, `feature/12915/aidlc`, is the code intended for review, even though the discovery request names `master`. | It is what was provided in the workspace, and it is the only branch with content in all thirteen clones. | The entire inventory could describe code that differs from what the requester meant, across all thirteen repositories. | Compare `feature/12915/aidlc` with `master` in each repository and confirm which is authoritative. |
| A2 | `messages.json` in `Pacco.Services.Operations` is an accurate and current catalogue of every message the platform emits. | Every name in it was cross-checked against the publishing and subscribing code in the other repositories without contradiction, and it is the only single-file inventory that exists. | Messages renamed or added elsewhere would be missing from this inventory, and `operations-service` would silently stop tracking them. | Add a build-time check comparing each service's message contracts against `messages.json`. |
| A3 | The composed platform runs the asynchronous gateway profile, so writes are fire-and-forget and callers learn outcomes from `operations-service`. | `Pacco/compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml`, overriding the Dockerfile's synchronous default. | Every conclusion about eventual consistency, operation tracking, and the role of `operations-service` would be inverted. | Confirm with the platform owner which profile production runs. |
| A4 | Each service is deployed as a single logical instance per environment, and the platform relies on a trusted internal network. | No service performs authorisation of its own; all role gating is at the gateway; `ordermaker-service` cannot run more than one instance; no network policy or mesh configuration exists. | Direct access to any service port bypasses every access control the platform has, and the saga would misbehave under multiple instances. | Ask the platform owner about network segmentation and the intended instance count per service. |
| A5 | The identifier issued by `identity-service` as `UserId` is the same GUID used as `CustomerId` throughout the platform. | The gateway binds `customerId: @user_id` on customer, order, and parcel creation, and `customers-service` creates its record directly from the `signed_up` event. | Every ownership binding and `@user_id` substitution across the platform would associate work with the wrong person. | Trace a sign-up end to end and compare the id in `users` with the id in `customers`. |
| A6 | Docker Compose plus the two process-manager manifests represent the whole of the platform's deployment story. | No Kubernetes manifest, Helm chart, or Terraform configuration exists in any of the thirteen repositories. | A production topology exists that this inventory has not seen, and every deployment and infrastructure conclusion here is incomplete. | Ask the platform owner for the production deployment definition. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** A shared JWT signing key is committed in plain text in four files across three repositories (the gateway's two Ntrada configurations, Identity's settings, and Operations' settings). Anyone who can read the repositories can issue tokens the entire platform accepts, including `admin` tokens. | Any deployment to an environment reachable by untrusted users; security sign-off on the platform as a whole. | Security owner | Move the key into Vault — every service already has a `vault` configuration block — rotate it, and remove it from all four files and from git history. | TBD |
| B2 | **[ACTION NOW]** `Pacco.Web` was named as one of thirteen repositories to analyse but contains no source code, only a one-line README. There is nothing to inventory and no one has confirmed whether that is expected. | Any statement about the platform's web client; completeness of this inventory. | Platform owner / the requester of this discovery | Confirm whether the repository is a placeholder, whether the code exists elsewhere, or whether it should leave scope — then supply the source or record the exclusion. | TBD |
| B3 | **[ACTION NOW]** The order lifecycle has no known trigger for its delivery stage. Nothing in the thirteen repositories publishes `start_delivery` or calls `POST /deliveries`, so no order can reach completion through any path visible here. | Any end-to-end validation of the platform's principal flow; any claim that the happy path works. | Product owner / platform owner | Identify the actor that starts deliveries — a courier application, an operator console, or a manual call — and document it, or confirm the capability is genuinely missing. | TBD |
| B4 | **[ACTION NOW]** `ordermaker-service` holds saga state in memory and implements only one of five compensations. A restart strands in-flight orders, and a mid-saga failure permanently holds a vehicle assignment and a resource reservation — the scarce resource the product exists to manage. | Any decision to run the saga in an environment with real customers or real inventory. | Service owner / product owner | Give Chronicle a persistence provider, implement the missing compensations (`availability-service` already accepts `release_resource`), or keep the service disabled until both are done. | TBD |
| B5 | **[ACTION NOW]** Destructive operations are reachable by any authenticated user with no role requirement: any customer can complete or fail any delivery, and can delete any vehicle — which cascades into removing its availability resource and every reservation on it. | Security sign-off on the public API; any exposure of the gateway to real customers. | Security owner | Decide which routes require an operator or administrator role, add the role claims at the gateway, and add in-service checks for the cascading vehicle deletion. | TBD |
| B6 | **[ACTION NOW]** The only record of an asynchronous write's outcome lives in Redis for five idle minutes and nowhere else. Because the composed gateway publishes writes rather than proxying them, `operations-service` is the sole channel by which any caller learns whether their command succeeded — and `requests.expirySeconds: 300` deletes that answer, with no durable copy, no archive, and no distinction at the API between "expired" and "never existed". A Redis restart erases every in-flight operation on the platform simultaneously. | Any claim that the asynchronous API is usable by real clients; any client that retries, reconnects, or investigates a failure; every post-incident analysis. | API owner / platform owner | Decide the retention the flow requires, then either raise the expiry, persist operations durably — the Mongo configuration and its Vault lease already exist unused in that service — or publish the five-minute limit as part of the API contract so clients are built to it. | TBD |
| B7 | **[ACTION NOW]** The container images the platform composes (`devmentors/pacco.*`) are published by a third-party organisation this project does not control, while the analysed source is a `hianshul100` fork. Nobody has stated which images the target deployment should run. | Any build, release, or deployment work; any claim about what code is actually running. | Platform owner / release engineering | Decide whether to keep consuming upstream images or stand up a registry for the fork, then update `Pacco/compose/services.yml`. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is there a production deployment target beyond Docker Compose, and where is it defined? | Nothing in the thirteen repositories describes one, so every deployment, scaling, and resilience conclusion here is provisional. | No — Compose plus a process manager appears to be the whole story, but this cannot be proven from the repositories. | Platform owner |
| Q2 | **[ACTION NOW]** Which branch is authoritative: `master`, as the discovery request states, or `feature/12915/aidlc`, which every clone is on? | If the two differ, this inventory describes code other than the code intended for review. | The analysed branch was used throughout, since that is what was provided. | The requester of this discovery |
| Q3 | **[ACTION NOW]** When a write is published to RabbitMQ instead of proxied, what does the caller receive, how do they learn the outcome, **and is a five-minute window enough**? | The contract between the gateway, `operations-service`, and clients is undefined, so every API consumer must guess — and the guess now has a hard deadline. `operations-service` holds each record in Redis with a 300-second sliding expiry and nowhere else, so a client that polls late, retries after a network partition, or reconnects after a restart gets `404` and cannot tell a lost operation from an invented id. | Clients appear to poll `GET /operations/{operationId}` or listen on the SignalR hub, but nothing states it, and nothing states the deadline. | API owner |
| Q4 | **[ACTION NOW]** Is `ordermaker-service` part of the intended platform, and if so how is it meant to be reached? | It is absent from the gateway and from both process-manager manifests, so nothing in the documented set-up ever invokes it — which changes whether its other gaps matter. | Unknown — it appears only in `compose/services.yml` and in `operations-service`'s `depends_on`. | Platform owner |
| Q5 | **[ACTION NOW]** Where is vehicle capacity meant to be modelled and enforced? | `parcels-service` computes parcel volume and the product is about fitting parcels into vehicles, yet vehicles have no capacity attribute and nothing compares the two. It is the platform's most conspicuous domain gap. | Unknown — no capacity attribute or check exists in any repository. | Product owner |
| Q6 | **[ACTION NOW]** Is payment out of scope? Orders carry a total price and nothing collects it. | A commercial ordering platform with no payment path is either intentionally partial or missing a whole subsystem, and the answer changes the service map substantially. | Unknown — no payment code, service, or integration exists anywhere. | Product owner |
| Q7 | **[ACTION NOW]** Should the platform's commercial policy — the discount table in `pricing-service` and the VIP threshold of 20 completed orders in `customers-service` — be configurable rather than compiled in? | Every rate or threshold change currently requires a code change, a build, an image push, and a redeploy of a service, and none of it is tested. | Move them to configuration if the business expects to tune them; otherwise document them as fixed policy. | Product owner |
| Q8 | **[handled later by architecture_evolution_generation]** Should the certificate-based service-to-service authentication used on the Availability → Customers call be extended to the other six internal calls, or dropped? | The platform authenticates exactly one of seven internal edges, which is the worst of both worlds: the cost of the mechanism without its protection. | Extend it platform-wide or rely on network isolation, but not the current half-application. | Architecture team |
| Q9 | **[handled later by architecture_evolution_generation]** Should `outbox.disableTransactions: true` remain, given it means state writes and outgoing messages are not atomic in seven services? | It undermines the reliability guarantee the outbox pattern exists to provide, and it is set identically everywhere, which suggests a copied default rather than a decision. | Run MongoDB as a replica set so transactions can be enabled, or document at-least-once delivery with compensations. | Architecture team |
| Q10 | **[handled later by architecture_evolution_generation]** Eleven published events have no subscriber anywhere in the workspace. Are they for future use, for consumers outside these repositories, or accidental? | They are recorded by `operations-service` but drive no behaviour, so they may be dead weight on the broker or an undocumented integration surface. | Unknown. | Architecture team |
| Q11 | **[ACTION NOW]** Is it acceptable that ten of thirteen repositories have no tests, including the services owning authentication, pricing policy, and the only distributed transaction — and that the one repository which does have tests is the only one whose CI never runs them? | These are the behaviours where a silent regression is most damaging, and nothing would catch one. The CI step is present in eight repositories with nothing to test and absent from the one with five test projects, so the pipeline's green badges certify a `dotnet build` and little else. | No — authentication, pricing, and the saga warrant tests at minimum, and the existing suite should at least be executed. | Engineering owner |
| Q12 | **[handled later by architecture_evolution_generation]** Should the READMEs be corrected — the nine wrong source paths, the `devmentors` links across twelve repositories, and the missing `Pacco.Web` entry? | New joiners following the documentation run commands that fail, clone the wrong repositories, and miss one entirely. | Yes, and the fix is mechanical. | Repository maintainers |
| Q13 | **[ACTION NOW]** What do the four diagrams in `Pacco/assets/` assert, and does any of it contradict the code? | They are the platform's only architecture diagrams, and if they disagree with the code, downstream design work could inherit a wrong picture. | Not determinable — the images are not machine-readable. | Architecture team |
| Q14 | **[ACTION NOW]** Was `- ./scripts/test.sh` removed from `Pacco.Services.Availability`'s CI on purpose, and if so what runs those five test projects instead? | It is the platform's only real test suite and its only performance baseline. Ten sibling pipelines run the step; this one does not, and nothing else in the workspace runs it either. | Unknown. The integration and end-to-end suites need MongoDB and RabbitMQ, which the Travis job never starts — plausible, undocumented, and no explanation for skipping the unit tests. | Service owner / engineering owner |
| Q15 | **[ACTION NOW]** Why is `operations-service` configured for a MongoDB database — connection string, database name, and a Vault dynamic-credential lease with auto-renewal — that it never opens? | It makes the service appear to depend on a datastore it does not use, which distorts capacity planning and failure-mode analysis, and it leaves an unused credential path open. It also suggests a store that was intended and never built, which would change how the five-minute expiry should be read. | Unknown. The likeliest reading is a configuration block copied from another service, since `allowAnonymousEndpoints` in the same file was demonstrably copied the same way and names paths this service does not serve. | Service owner |
| Q16 | **[handled later by architecture_evolution_generation]** Should the platform have a real web client, and what should it be built with? | There is no delivered frontend: `Pacco.Web` is empty, and the only page in the workspace is a development demonstration inside `operations-service` that hard-codes a localhost URL. No `package.json` exists anywhere to set a precedent. | No answer is available from the repositories. | Product owner / architecture team |
