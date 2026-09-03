---
title: "Repository Summary — Pacco.APIGateway"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.APIGateway"
status: "evidence-based inventory"
---

# Pacco.APIGateway

**Primary name:** `Pacco.APIGateway` (aliases used in this file: `api-gateway` — the Docker Compose service name and the Jaeger `serviceName`; `api` — the app name in `services.yml` / `prod-services.yml`).
Repository: `Pacco.APIGateway`, path: `hianshul100_Pacco.APIGateway/`
Deployable project: `Pacco.APIGateway`, path: `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Pacco.APIGateway.csproj`

---

## 1. Primary purpose

The single public entry point for the platform. It terminates client HTTP calls, validates JWTs, and either proxies the request to a downstream service or publishes it to RabbitMQ. Routing is entirely declarative: there are no controllers, only YAML.

Evidence: `src/Pacco.APIGateway/ntrada.yml`, `src/Pacco.APIGateway/ntrada-async.yml`, `src/Pacco.APIGateway/Program.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` process hosting **Ntrada** `0.4.*`, a YAML-configured API gateway. Long-running HTTP server, listening on `http://*:80` in the container and `5000` on the host.

Evidence: `src/Pacco.APIGateway/Pacco.APIGateway.csproj`, `Dockerfile`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` (`Main` → `CreateWebHostBuilder` → `UseNtrada()`) | `src/Pacco.APIGateway/Program.cs` |
| Container entrypoint `dotnet Pacco.APIGateway.dll` | `Dockerfile` |
| `scripts/start.sh` (synchronous mode) | `scripts/start.sh` |
| `scripts/start-async.sh` (asynchronous mode) | `scripts/start-async.sh` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh` | `scripts/` |

## 4. Modules / packages

Single project. Source folders: `Infrastructure/` containing `CorrelationContext.cs`, `CorrelationContextBuilder.cs`, `HttpRequestHook.cs`, `SpanContextBuilder.cs`.

NuGet packages (`Pacco.APIGateway.csproj`): `Ntrada 0.4.*`, `Ntrada.Extensions.Cors 0.4.*`, `Ntrada.Extensions.CustomErrors 0.4.*`, `Ntrada.Extensions.Jwt 0.4.*`, `Ntrada.Extensions.RabbitMq 0.4.*`, `Ntrada.Extensions.Swagger 0.4.*`, `Ntrada.Extensions.Tracing 0.4.*`, `Convey.Logging 0.4.*`, `Convey.Metrics.AppMetrics 0.4.*`, `Convey.Security 0.4.*`, `NetEscapades.Configuration.Yaml 2.0.0`.

## 5. External integrations

- **RabbitMQ** — in async mode only (`ntrada-async.yml`, `extensions.rabbitmq`).
- **Jaeger** — `extensions.tracing`, `serviceName: api-gateway`, `udpHost: localhost`, `udpPort: 6831`, `sampler: const`.
- **Prometheus** — via `Convey.Metrics.AppMetrics`, `appsettings.json` `metrics` block.
- **Fabio load balancer** — `loadBalancer.enabled: false`, `url: localhost:9999` (present but switched off in the committed configuration).
- **All ten downstream services** — reached by `localUrl` `localhost:5001` … `localhost:5009`.

## 6. Data stores / state

None. The gateway is stateless: no database, no ORM, no migration tool, no collections or tables, no cache. There is therefore no cross-domain foreign-key coupling to report.

Evidence: `appsettings.json` contains only a `metrics` block; the project references no persistence package.

## 7. Messaging / async / events

**System:** RabbitMQ, enabled only in the asynchronous configuration `ntrada-async.yml` / `ntrada-async.docker.yml`.

Settings: `connectionName: api-gateway`, `hostnames: localhost`, `port: 5672`, `virtualHost: /`, `username: guest`, `password: guest`, `exchange.type: topic`, `messageContext.enabled: true` with `header: message_context`, `spanContextHeader: span_context`.

Exchanges published to, taken from the `config.exchange` value on each write route: `availability`, `customers`, `deliveries`, `identity`, `orders`, `parcels`, `vehicles`.

**Payloads:** the gateway does not define message payload classes. It forwards the request body, adding route-bound values declared per route — for example `bind: customerId:@user_id` and generated identifiers such as `resourceId: {property: orderId, generate: true}`. The concrete field set of each published message is owned by the receiving service. The exact wire payload the gateway emits is **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed modules and routes (`ntrada.yml`; the async file mirrors the same paths):

| Module | Path prefix | Routes |
|---|---|---|
| `home` | `/` | `GET /` returns the literal value `Welcome to Pacco API!` |
| `availability` | `availability` | proxied to `availability-service` |
| `customers` | `customers` | `GET /` (role `admin`), `GET /me` → `customers-service/customers/@user_id`, `GET /{customerId}` (role `admin`), `GET /{customerId}/state` (role `admin`), `POST /` (binds `customerId:@user_id`, payload `create_customer`, schema `create_customer.schema`), `PUT /{customerId}/state/{state}` (role `admin`) |
| `deliveries` | `deliveries` | `POST /` with `resourceId: {property: deliveryId, generate: true}` |
| `identity` | `identity` | `GET /users/{userId}` (role `admin`), `GET /me`, `POST /sign-up` (`auth: false`, generates `userId`), `POST /sign-in` (`auth: false`) |
| `operations` | `operations` | `GET /{operationId}` with **`auth: false`** |
| `orders` | `orders` | `GET /` → `orders-service/orders?customerId=@user_id`, `POST /` (binds `customerId:@user_id`, generates `orderId`), `POST`/`DELETE /{orderId}/parcels/{parcelId}`, `POST /{orderId}/vehicles/{vehicleId}` |
| `parcels` | `parcels` | `GET /` → `parcels-service/parcels?customerId=@user_id`, `GET /volume`, `POST /` (generates `parcelId`, binds `customerId:@user_id`) |
| `pricing` | `pricing` | `GET /` → `pricing-service/pricing?customerId=@user_id` |
| `vehicles` | `vehicles` | `GET /` (`onSuccess.data: response.data.items`), `GET /{vehicleId}`, `POST /` (generates `vehicleId`), `PUT /{vehicleId}`, `DELETE /{vehicleId}` |

Consumed: the ten service base URLs above. Swagger is published at route prefix `docs` with name `Pacco`, title `Pacco API`, version `v1`.

Sample request collections are committed at `Pacco.rest` and `Pacco-sample-scenario.rest`.

## 9. Deployment / runtime clues

`Dockerfile`: two-stage build from `mcr.microsoft.com/dotnet/core/sdk:3.1` publishing to `mcr.microsoft.com/dotnet/core/aspnet:3.1`, `ENV ASPNETCORE_URLS http://*:80`, `ENV ASPNETCORE_ENVIRONMENT docker`, `ENV NTRADA_CONFIG ntrada.docker`, `ENTRYPOINT dotnet Pacco.APIGateway.dll`.

CI: `.travis.yml` runs `./scripts/build.sh` then `./scripts/test.sh`, and on success `./scripts/dockerize.sh`, on dotnet `3.1.100`, dist `xenial`, branches `master` and `develop`.

Environment-specific settings files: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`; gateway configurations `ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml`.

## 10. Security / auth clues

- JWT bearer validation via `Ntrada.Extensions.Jwt`: `validIssuer: pacco`, `validateIssuer: true`, `validateAudience: false`, `validateLifetime: true`.
- `auth.enabled: true`, `auth.global: false` — authentication is applied per route, not globally.
- Role claim type `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`; `admin` role required on the customer administration routes and `identity/users/{userId}`.
- A certificate is committed at `src/Pacco.APIGateway/certs/localhost.cer`.

**Security findings** (reported, not remediated in this stage):

1. A symmetric JWT signing key is committed in clear text under `extensions.jwt.issuerSigningKey` in both `ntrada.yml` and `ntrada-async.yml`. Anyone with repository access can mint valid tokens for the whole platform.
2. CORS is configured with `allowedOrigins: '*'` together with `allowCredentials: true` (`extensions.cors`).
3. RabbitMQ credentials `guest` / `guest` are committed in `ntrada-async.yml`.
4. `GET operations/{operationId}` is exposed with `auth: false`, so operation status is readable without a token by anyone who knows or guesses an operation identifier.

## 11. Observability / logging / tracing

- Distributed tracing to Jaeger, with `generateRequestId` and `generateTraceId` enabled and `Request-ID`, `Resource-ID`, `Trace-ID`, `Total-Count` exposed as CORS response headers.
- `SpanContextBuilder.cs` and `CorrelationContextBuilder.cs` construct the correlation and span context propagated to downstream services and to RabbitMQ.
- Logging via `Convey.Logging`.
- Metrics via `Convey.Metrics.AppMetrics` with `prometheusEnabled: true`, `database: "pacco"`, `env: "local"`, `interval: 5`.
- HTTP retry policy: `http.retries: 2`, `interval: 2.0`, `exponential: true`.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.APIGateway/ntrada.yml` — the synchronous routing contract for the whole platform.
- `src/Pacco.APIGateway/ntrada-async.yml` — the asynchronous variant that turns writes into RabbitMQ publishes.
- `src/Pacco.APIGateway/Infrastructure/HttpRequestHook.cs` — request mutation hook.
- `src/Pacco.APIGateway/Infrastructure/SpanContextBuilder.cs` and `CorrelationContextBuilder.cs` — trace propagation decision.
- `Dockerfile` — which configuration ships by default.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are configuration booleans such as `auth.enabled`, `auth.global`, `loadBalancer.enabled`, `extensions.rabbitmq.enabled`, `messageContext.enabled`, `useLocalUrl`, `passQueryString`, `useForwardedHeaders`, `forwardRequestHeaders`, `forwardResponseHeaders`, `generateRequestId`, `generateTraceId`, `metrics.prometheusEnabled`. These are deployment configuration, not runtime feature flags.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` holds only the C# project. There is no `package.json`, no bundler configuration, no HTML and no view templates. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README presents the gateway as the platform's API entry point built on Ntrada | Confirmed by `Program.cs`, the csproj and the YAML routing files | Confirmed |
| `Dockerfile` sets `NTRADA_CONFIG ntrada.docker` (the **synchronous** configuration) | `hianshul100_Pacco/compose/services.yml` overrides the same variable to `ntrada-async.docker.yml` (the **asynchronous** configuration) | Needs validation — the image default and the compose deployment disagree on the gateway's integration style |
| `loadBalancer.url: localhost:9999` matches the Fabio container in `compose/infrastructure.yml` | `loadBalancer.enabled: false`, so Fabio is provisioned but not used by the gateway | Stale doc |
| `.travis.yml` runs `./scripts/test.sh` | There is no test project in this repository, so the step has nothing to execute | Needs validation |

**Docs-only claims:** none beyond the above.
**Disk-only components:** `Pacco-sample-scenario.rest`, `appsettings.local.json`, `certs/localhost.cer`, `Properties/launchSettings.json` — present on disk, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The asynchronous configuration is the one actually used, because the shared Compose file selects it. | The Compose file overrides the image's own default, and an override is normally deliberate. |
| A2 | The `localhost:500X` addresses in the routing files are for running everything on one developer machine, not for container deployment. | The container-specific files with the `.docker` suffix exist separately for that purpose. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | A token signing key is stored in clear text in the routing files, so anyone with repository access can create valid logins for every service. | **[ACTION NOW]** Report to the security owner of the platform; this stage records the location but does not change any file. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Should the gateway run in synchronous or asynchronous mode? The container image and the shared Compose file choose differently. | **[ACTION NOW]** Confirm with the requesting team, because the answer changes how every write request reaches the services. |
| Q2 | Why can operation status be read without logging in? | **[handled later by the ADR authoring stage]** Decide whether this is intentional for the demo user interface or an oversight, and record the decision. |
| Q3 | Is the Fabio load balancer meant to be used by the gateway? It is started by the shared stack but switched off here. | **[handled later by the ADR authoring stage]** Record whether load balancing sits in front of the gateway or is unused. |
