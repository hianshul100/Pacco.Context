# Repository summary — `Pacco.APIGateway`

**Repository:** `Pacco.APIGateway` (workspace clone path: `hianshul100_Pacco.APIGateway`)
**Deployable:** `api-gateway` (also known as: `Pacco.APIGateway`, the Pacco API Gateway, image `devmentors/pacco.apigateway`). **Repository: `Pacco.APIGateway`, path: `src/Pacco.APIGateway`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.APIGateway
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

The single public entry point for the Pacco platform. It is a **configuration-driven API gateway** built on the `Ntrada` library: routes, authentication, CORS, tracing, and Swagger are declared in YAML rather than in code. It fronts all ten backend services, terminates JWT authentication, and — in its asynchronous profile — converts write requests into RabbitMQ messages instead of proxying them downstream.

**Evidence:** `src/Pacco.APIGateway/Program.cs`, `src/Pacco.APIGateway/ntrada.yml`, `src/Pacco.APIGateway/ntrada-async.yml`, `README.md`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP service. Listens on port `5000` locally and on container port `80`. It has no domain layer, no database, and no `Application`/`Core`/`Infrastructure` project split — a single `.csproj` plus four small infrastructure classes.

**Evidence:** `src/Pacco.APIGateway/Pacco.APIGateway.csproj` (`<TargetFramework>netcoreapp3.1</TargetFramework>`), `Dockerfile` (`ENV ASPNETCORE_URLS http://*:80`), `compose/services.yml` in the `Pacco` repository (`5000:80`).

## 3. Key entrypoints

| Entrypoint | File | Notes |
|---|---|---|
| `Program.Main` | `src/Pacco.APIGateway/Program.cs` | Reads the config file name from the `NTRADA_CONFIG` environment variable, else the first CLI argument, else defaults to `ntrada.yml`; calls `AddNtrada(...)` with the correlation/span/hook builders, `AddConvey().AddMetrics().AddSecurity()`, then `app.UseNtrada()` and `.UseLogging()` |
| `ntrada.yml` | `src/Pacco.APIGateway/ntrada.yml` | **Synchronous profile** — all routes proxy downstream over HTTP (`use: downstream`) |
| `ntrada-async.yml` | `src/Pacco.APIGateway/ntrada-async.yml` | **Asynchronous profile** — write routes publish to RabbitMQ (`use: rabbitmq`) |
| `ntrada.docker.yml`, `ntrada-async.docker.yml` | same directory | Container variants (hostnames point at container names instead of `localhost`) |
| `scripts/start.sh`, `scripts/start-async.sh` | `scripts/` | Select the sync or async profile when running locally |
| `Dockerfile` | repo root | `ENTRYPOINT dotnet Pacco.APIGateway.dll`, `ENV NTRADA_CONFIG ntrada.docker` |

The environment variable `NTRADA_CONFIG` is the **single most important runtime switch in the platform** — it decides whether writes are synchronous HTTP or asynchronous messages. `compose/services.yml` (in the `Pacco` repository) sets it to `ntrada-async.docker.yml`.

## 4. Important modules/packages

- `Program.cs` — composition root (≈40 lines; the whole gateway).
- `Infrastructure/CorrelationContext.cs`, `Infrastructure/CorrelationContextBuilder.cs` — build the `Correlation-Context` value (correlation id, user id, resource id, trace id, span context, connection id, name, created-at) attached to every forwarded request and published message.
- `Infrastructure/SpanContextBuilder.cs` — extracts the Jaeger span context so downstream services join the same trace.
- `Infrastructure/HttpRequestHook.cs` — hook invoked per proxied request.
- `certs/localhost.cer` — the public certificate used to validate JWTs in the container profile.
- `Pacco.rest`, `Pacco-sample-scenario.rest` — REST Client scratch files; `Pacco-sample-scenario.rest` is the end-to-end happy-path walkthrough the root README points readers at.

**NuGet packages** (`Pacco.APIGateway.csproj`): `Ntrada 0.4.*`, `Ntrada.Extensions.Cors`, `Ntrada.Extensions.CustomErrors`, `Ntrada.Extensions.Jwt`, `Ntrada.Extensions.RabbitMq`, `Ntrada.Extensions.Swagger`, `Ntrada.Extensions.Tracing` (all `0.4.*`), `Convey.Logging`, `Convey.Metrics.AppMetrics`, `Convey.Security`, `NetEscapades.Configuration.Yaml 2.0.0`.

## 5. External integrations

| Integration | How | Evidence |
|---|---|---|
| Ten backend services over HTTP | `use: downstream` with `service:` + `localUrl:` per module | `ntrada.yml` modules block |
| RabbitMQ | `extensions.rabbitmq` — `connectionName: api-gateway`, `hostnames: [localhost]`, port `5672`, vhost `/`, `exchange.type: topic`, `messageContext.enabled: true` (header `message_context`), `spanContextHeader: span_context` | `ntrada-async.yml` |
| Jaeger | `extensions.tracing` — `serviceName: api-gateway`, UDP `localhost:6831`, `sampler: const` | `ntrada.yml` |
| Seq | `logger.seq.serverUrl: http://localhost:5341` | `appsettings.json` |
| Prometheus | `metrics.prometheusEnabled: true`, `database: pacco`, `env: local`, 5-second interval; `influxEnabled: false` | `appsettings.json` |
| Fabio (load balancer) | `loadBalancer.enabled: false`, `url: localhost:9999` — **declared but disabled** in the committed config | `ntrada.yml` |

## 6. Data stores / state handling

**None.** The gateway is stateless: no database client, no cache client, no repository code, no ORM, no migration tool, no table or collection names. The only persistent artefact it writes is the rolling log file `logs/logs.txt`.

**Evidence:** `Pacco.APIGateway.csproj` has no persistence package; `appsettings.json` has no connection string.

## 7. Messaging / async / event mechanisms

**System: RabbitMQ**, enabled only in the asynchronous profile (`ntrada-async.yml` / `ntrada-async.docker.yml`).

The gateway is **publish-only** — it declares no queues and subscribes to nothing. Each write route sets `use: rabbitmq` and `config.exchange: <exchange>`; Ntrada derives the routing key from the route's `payload`/`routingKey` convention. The exchanges it publishes to, by module:

| Module | Exchange | Routes that publish |
|---|---|---|
| `availability` | `availability` | `POST /resources`, `POST /resources/{resourceId}/reservations/{dateTime}`, `DELETE /resources/{resourceId}/reservations/{dateTime}`, `DELETE /resources/{resourceId}` |
| `customers` | `customers` | `POST /`, `PUT /{customerId}/state/{state}` |
| `deliveries` | `deliveries` | `POST /`, `POST /{deliveryId}/fail`, `POST /{deliveryId}/complete`, `POST /{deliveryId}/registrations` |
| `identity` | `identity` | `POST /sign-up` |
| `orders` | `orders` | `POST /`, `DELETE /{orderId}`, `POST /{orderId}/parcels/{parcelId}`, `DELETE /{orderId}/parcels/{parcelId}`, `POST /{orderId}/vehicles/{vehicleId}` |
| `parcels` | `parcels` | `POST /`, `DELETE /{parcelId}` |
| `vehicles` | `vehicles` | `POST /`, `PUT /{vehicleId}`, `DELETE /{vehicleId}` |

Read routes (`GET`) and `POST /sign-in` remain `use: downstream` HTTP even in the async profile, because they must return data synchronously.

**Message payload key fields:** the gateway forwards the HTTP body plus the values bound by `bind:` (notably `customerId:@user_id`) and `resourceId.generate: true`. The concrete command contracts live in the owning service repositories; the canonical name catalogue is `messages.json` in `Pacco.Services.Operations`.

**Correlation/trace propagation:** every published message carries the `message_context` header (built by `CorrelationContextBuilder`) and the `span_context` header.

## 8. APIs exposed or consumed

**Exposed — the complete public HTTP surface of the Pacco platform** (upstream path → downstream target, from `ntrada.yml`). `auth: true` unless noted.

| Upstream | Method | Downstream | Notes |
|---|---|---|---|
| `/` | GET | — | `use: return_value`, "Welcome to Pacco API!" (async profile: "Welcome to Pacco API [async]!"); `auth: false` |
| `/availability/resources` | GET | `availability-service/resources` | |
| `/availability/resources/{resourceId}` | GET | `availability-service/resources/{resourceId}` | |
| `/availability/resources` | POST | `availability-service/resources` | |
| `/availability/resources/{resourceId}/reservations/{dateTime}` | POST | same | binds `customerId: @user_id` |
| `/availability/resources/{resourceId}/reservations/{dateTime}` | DELETE | same | |
| `/availability/resources/{resourceId}` | DELETE | same | |
| `/customers` | GET | `customers-service/customers` | claim `role: admin` |
| `/customers/me` | GET | `customers-service/customers/@user_id` | |
| `/customers/{customerId}` | GET | `customers-service/customers/{customerId}` | claim `role: admin` |
| `/customers/{customerId}/state` | GET | `customers-service/customers/{customerId}/state` | claim `role: admin` |
| `/customers` | POST | `customers-service/customers` | binds `customerId: @user_id`; `payload: create_customer`; `schema: create_customer.schema` |
| `/customers/{customerId}/state/{state}` | PUT | same | claim `role: admin` |
| `/deliveries/{deliveryId}` | GET/POST(`/fail`,`/complete`,`/registrations`) | `deliveries-service/deliveries/...` | `POST /deliveries` generates `deliveryId` |
| `/identity/users/{userId}` | GET | `identity-service/users/{userId}` | claim `role: admin` |
| `/identity/me` | GET | `identity-service/me` | |
| `/identity/sign-up` | POST | `identity-service/sign-up` | **`auth: false`**, generates `userId` |
| `/identity/sign-in` | POST | `identity-service/sign-in` | **`auth: false`**, response `content-type: application/json` |
| `/operations/{operationId}` | GET | `operations-service/operations/{operationId}` | **`auth: false`** |
| `/orders` | GET | `orders-service/orders?customerId=@user_id` | |
| `/orders/{orderId}` | GET / DELETE | `orders-service/orders/{orderId}` | |
| `/orders` | POST | `orders-service/orders` | generates `orderId`, binds `customerId: @user_id` |
| `/orders/{orderId}/parcels/{parcelId}` | POST / DELETE | same | |
| `/orders/{orderId}/vehicles/{vehicleId}` | POST | same | |
| `/parcels` | GET | `parcels-service/parcels?customerId=@user_id` | |
| `/parcels/volume` | GET | `parcels-service/parcels/volume` | |
| `/parcels/{parcelId}` | GET / DELETE | `parcels-service/parcels/{parcelId}` | |
| `/parcels` | POST | `parcels-service/parcels` | generates `parcelId`, binds `customerId: @user_id` |
| `/pricing` | GET | `pricing-service/pricing?customerId=@user_id` | |
| `/vehicles` | GET | `vehicles-service/vehicles` | `onSuccess.data: response.data.items` (unwraps the paged result) |
| `/vehicles/{vehicleId}` | GET / PUT / DELETE | `vehicles-service/vehicles/{vehicleId}` | |
| `/vehicles` | POST | `vehicles-service/vehicles` | generates `vehicleId` |
| `/docs` | GET | — | Swagger UI (`extensions.swagger.routePrefix: docs`) |

**Consumed:** the ten service HTTP APIs above, addressed by Consul service name (`availability-service` … `vehicles-service`) with `localUrl` fallbacks `localhost:5001`–`localhost:5009`. No service is registered for `ordermaker-service` in `ntrada.yml` — **`Pacco.Services.OrderMaker` is not reachable through the gateway.**

**Notable API behaviours:** `useLocalUrl: true`, `passQueryString: true`, `forwardRequestHeaders: true`, `forwardResponseHeaders: true`, `generateRequestId: true`, `generateTraceId: true`, `useForwardedHeaders: true`; HTTP retries `2` with exponential backoff; exposed response headers `Request-ID`, `Resource-ID`, `Trace-ID`, `Total-Count`.

## 9. Deployment/runtime clues

- `Dockerfile`: multi-stage, `mcr.microsoft.com/dotnet/core/sdk:3.1` → `mcr.microsoft.com/dotnet/core/aspnet:3.1`; `ENV ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`, `NTRADA_CONFIG ntrada.docker`; `ENTRYPOINT ["dotnet", "Pacco.APIGateway.dll"]`.
- Composed as `api-gateway` on `5000:80` with `NTRADA_CONFIG=ntrada-async.docker.yml` (`Pacco/compose/services.yml`), i.e. **the composed platform runs the asynchronous profile**, overriding the Dockerfile default.
- CI: `.travis.yml` — `language: csharp`, `mono: none`, `dist: xenial`, `dotnet: 3.1.100`, `branches.only: [master, develop]`, `script: ./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`. **No GitHub Actions workflow exists.**
- Helper scripts: `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh`, `scripts/start-async.sh`.
- **No Kubernetes manifests, no Helm chart, no Terraform.**

## 10. Security/auth clues

- **JWT bearer authentication** via `Ntrada.Extensions.Jwt`. `auth.enabled: true`, `auth.global: false` — authentication is opt-in **per route**, so any route that omits `auth: true` is public.
- `validIssuer: pacco`; role claim type `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`; `validateAudience: false`; `validateLifetime: true`.
- **A symmetric `issuerSigningKey` literal is committed in `ntrada.yml` and `ntrada-async.yml`.** The identical literal also appears in `Pacco.Services.Identity/src/Pacco.Services.Identity.Api/appsettings.json` (the token issuer) and `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json`. The value is not reproduced here. This is a committed shared signing secret; anyone with repository read access can mint valid platform tokens.
- `certs/localhost.cer` is committed for the certificate-based validation path.
- **CORS is fully open:** `extensions.cors.allowedOrigins: ['*']`, with `allowCredentials` and the exposed-header list above.
- `extensions.customErrors.includeExceptionMessage: true` — exception messages are returned to callers.
- **Unauthenticated routes:** `GET /`, `POST /identity/sign-up`, `POST /identity/sign-in` (expected), and **`GET /operations/{operationId}` (`auth: false`)** — operation status, including `userId`, `name`, `state`, `code`, and `reason`, is readable by anyone who can guess or observe an operation id.
- Role-gated routes (`role: admin`): customer listing and lookup, customer state read and write, and `GET /identity/users/{userId}`.
- `@user_id` binding is used to force `customerId` from the token rather than the request body on customer, order, and parcel creation — a deliberate anti-IDOR measure.

## 11. Observability/logging/tracing

- **Logging:** Convey logging (`.UseLogging()`), `appsettings.json` → console enabled, rolling file `logs/logs.txt` (daily), Seq at `http://localhost:5341` with an API key committed in the file.
- **Tracing:** Jaeger via `Ntrada.Extensions.Tracing`, `serviceName: api-gateway`, UDP `localhost:6831`, `sampler: const` (sample everything). `SpanContextBuilder` propagates the span downstream; in async mode the span rides on the `span_context` message header.
- **Correlation:** `CorrelationContextBuilder` emits a `Correlation-Context` structure carrying correlation id, user id, resource id, trace id, connection id and timestamp — the platform's cross-service request identity.
- **Metrics:** App.Metrics via `Convey.Metrics.AppMetrics`, Prometheus formatter enabled, InfluxDB disabled.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.APIGateway/ntrada.yml` | The entire public API contract, per-route authorisation model, and downstream service map |
| `src/Pacco.APIGateway/ntrada-async.yml` | The decision that writes are fire-and-forget messages rather than synchronous calls — the platform's central CQRS/eventual-consistency choice, expressed in configuration |
| `src/Pacco.APIGateway/Program.cs` | Config-file selection precedence: `NTRADA_CONFIG` → argv[0] → `ntrada.yml` |
| `src/Pacco.APIGateway/Infrastructure/CorrelationContextBuilder.cs` | The shape of cross-service correlation identity |
| `Dockerfile` | Default profile is the **synchronous** one; compose overrides it to async |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. The nearest equivalents are static YAML/JSON switches evaluated once at startup: `auth.enabled`, `auth.global`, `loadBalancer.enabled`, `extensions.<name>.enabled` (`cors`, `customErrors`, `jwt`, `rabbitmq`, `swagger`, `tracing`), `metrics.enabled`, `metrics.prometheusEnabled`, `metrics.influxEnabled`, `logger.console.enabled`, `logger.file.enabled`, `logger.seq.enabled`. Plus the `NTRADA_CONFIG` environment variable, which behaves as a deployment-time profile switch. None of these can be toggled at runtime.

## 13. Open questions / ambiguities

- **Async response contract is undefined here.** In the async profile a write returns immediately; the caller is expected to poll `GET /operations/{operationId}` or subscribe to the Operations SignalR hub. Nothing in this repository documents that contract. **Needs validation.**
- **`loadBalancer.enabled: false`** while the rest of the platform registers with Consul and expects Fabio on `9999`. Whether Fabio is meant to sit in front of, or behind, the gateway is **Unknown**.
- **`ordermaker-service` has no gateway module**, so `POST /orders` at the gateway goes to `orders-service`, not to the saga. How the AI order-maker flow is triggered in a deployed system is **Unknown** — see `repo-summary/Pacco.Services.OrderMaker.md`.
- `HttpRequestHook.cs` behaviour was not traced end-to-end. **Needs validation.**
- Differences between `ntrada.yml` and `ntrada.docker.yml` (and their async twins) were confirmed to exist at the hostname level but not diffed route-by-route. **Needs validation** that the four files stay in sync.
- No tests of any kind exist in this repository (`scripts/test.sh` exists but there is no test project).

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (contains only the C# project), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these directories exist except `src/`, which holds no web assets. No `package.json`, no bundler configuration. The gateway serves a Swagger UI at `/docs`, which is generated by `Ntrada.Extensions.Swagger` at runtime and is not checked-in frontend code.

---

## README vs repository

**What the README claims:**
- The gateway is the Pacco API Gateway built with Ntrada. — **Confirmed** (`Pacco.APIGateway.csproj`, `Program.cs`).
- Run with `dotnet run` from `/src/Pacco.APIGateway`, or `./scripts/start.sh`, or the Docker image. — **Confirmed**; unlike the ten service repositories, this repo's documented source path is **correct**, because the project directory really is `src/Pacco.APIGateway` (there is no `.Api` suffix here).
- Available at `http://localhost:5000`. — **Confirmed** against `Pacco/compose/services.yml` (`5000:80`) and `launchSettings.json`.
- Travis badge and Docker Hub image `devmentors/pacco.apigateway`. — Files exist, but they point at the **upstream `devmentors` organisation**, not the `hianshul100` fork under analysis. **Stale doc.**

**Components on disk but not in the README:**
- **The asynchronous profile.** `ntrada-async.yml`, `ntrada-async.docker.yml`, and `scripts/start-async.sh` are not mentioned, even though `Pacco/compose/services.yml` runs the gateway in exactly that mode. This is the most significant documentation gap in the repository: the README describes a synchronous proxy, and the composed system runs an asynchronous publisher.
- `Infrastructure/CorrelationContextBuilder.cs`, `SpanContextBuilder.cs`, `HttpRequestHook.cs`, `CorrelationContext.cs` — the correlation and tracing model is undocumented.
- `certs/localhost.cer` — undocumented.
- `Pacco.rest` and `Pacco-sample-scenario.rest` — the root `Pacco` README points at `Pacco-sample-scenario.rest`, but this repository's own README does not mention either file.
- The `NTRADA_CONFIG` environment variable — undocumented, despite being the switch that changes the platform's integration style.

**README claims not reflected in the clone — Stale doc:**
- All links and badges resolve to `github.com/devmentors/Pacco.APIGateway` and `api.travis-ci.org/devmentors/...`; the analysed clone is `hianshul100/Pacco.APIGateway` on `feature/12915/aidlc`, where Travis is not configured to run. **Stale doc.**

**Unknown (neither pass yielded proof):**
- Whether the four `ntrada*.yml` files are kept consistent with one another.
- Whether the async profile's `operationId` correlation is actually surfaced to clients.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The asynchronous profile (`ntrada-async.docker.yml`) is the intended production behaviour, not an experiment. | `Pacco/compose/services.yml` explicitly sets `NTRADA_CONFIG=ntrada-async.docker.yml` for the composed platform, overriding the Dockerfile's synchronous default. | Every conclusion about eventual consistency, operation polling, and the role of `operations-service` would be inverted; the platform would actually be a synchronous request/response system. | Confirm with the platform owner which profile production runs, and record the answer next to the `NTRADA_CONFIG` default in the Dockerfile. |
| A2 | Routes in `ntrada.yml` without an explicit `auth: true` are genuinely intended to be public. | `auth.global: false` makes authentication opt-in per route, and `POST /sign-in` / `POST /sign-up` are correctly public, so the pattern looks deliberate. | `GET /operations/{operationId}` would be an unintended information-disclosure hole exposing operation state and user ids. | A security owner reviews every route lacking `auth: true` and confirms or corrects each one. |
| A3 | `ntrada.docker.yml` and `ntrada-async.docker.yml` differ from their non-`docker` twins only in hostnames and URLs. | Spot-checking showed the module and route structure to be the same shape, with `localhost` replaced by container names. | A route could be authenticated in one profile and public in another, so a security review of one file would not cover what actually runs. | Diff the four YAML files route-by-route and add a check that fails the build when their route sets diverge. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** A shared JWT signing secret is committed in plain text in `ntrada.yml` and `ntrada-async.yml`, and the same value appears in the Identity and Operations service settings. Anyone who can read the repository can issue tokens that the whole platform accepts. | Any deployment to an environment reachable by untrusted users; also any security sign-off on the authentication design. | Security owner / platform owner | Move the signing key into Vault (the platform already runs Vault and every service already has a `vault` config block), rotate the current value, and remove it from all four files and from git history. | TBD |
| B2 | **[ACTION NOW]** `GET /operations/{operationId}` is configured with `auth: false`, exposing operation state and the associated `userId` to unauthenticated callers. Nobody has confirmed whether this is deliberate. | Security review of the public API surface; any decision about exposing the gateway outside a trusted network. | Security owner | Decide whether operation status must stay anonymous (because the client polls before it has a token) or should require the bearer token; change `ntrada.yml`, `ntrada-async.yml`, and both `.docker.yml` variants accordingly. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** When a write is published to RabbitMQ instead of proxied, what does the client get back, and how is it told the work finished? | Without a documented answer, every API consumer has to guess, and the contract between the gateway, `operations-service`, and the client is undefined. | The client appears to poll `GET /operations/{operationId}` or listen on the Operations SignalR hub, but nothing in this repository states it. | API owner |
| Q2 | **[ACTION NOW]** Is `ordermaker-service` meant to be reachable through the gateway? It has no module in any `ntrada*.yml`, so `POST /orders` reaches `orders-service` directly and the saga is bypassed. | It determines whether the AI order-maker flow is live, dead code, or triggered by some path not visible in these repositories. | Unknown — no evidence either way in the gateway configuration. | Platform owner |
| Q3 | **[handled later by architecture_evolution_generation]** Should CORS remain `allowedOrigins: ['*']`, and should `includeExceptionMessage: true` stay on outside development? | Both are development-friendly settings that leak information or widen the attack surface when exposed publicly. | Restrict origins to the known web client and disable exception passthrough for non-development environments. | Security owner |
| Q4 | **[ACTION NOW]** Should Fabio sit in front of the gateway or behind it? The gateway has `loadBalancer.enabled: false` while every backend service registers with Consul and configures a Fabio URL. | It changes the network topology diagram and where TLS terminates. | Unknown from the repositories. | Platform owner |
