# Repository Summary — `hianshul100_Pacco.APIGateway`

**Primary name:** `api-gateway` (aliases used in this file: `Pacco.APIGateway` — the .NET project and assembly name; `devmentors/pacco.apigateway` — the published Docker image name).

**Repository:** `hianshul100_Pacco.APIGateway`, path: `src/Pacco.APIGateway`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

The single public entry point for the Pacco platform. It terminates client HTTP traffic, validates JWTs, and then either forwards the request to a downstream service over HTTP or converts it into a RabbitMQ message. Almost all of its behaviour is declared in YAML rather than written in C#.

## 2. Main runtime / service type

ASP.NET Core 3.1 web host (`netcoreapp3.1`) built on **Ntrada** `0.4.*`, a configuration-driven API gateway library. The C# in this repository is roughly 100 lines of host wiring plus four small infrastructure classes; the routing table is data.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.APIGateway/Program.cs` |
| Synchronous routing config | `src/Pacco.APIGateway/ntrada.yml` (455 lines) |
| Asynchronous routing config | `src/Pacco.APIGateway/ntrada-async.yml` |
| Docker variants of both | `src/Pacco.APIGateway/ntrada.docker.yml`, `src/Pacco.APIGateway/ntrada-async.docker.yml` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.APIGateway.dll` |
| Local run scripts | `scripts/start.sh` (sync mode), `scripts/start-async.sh` (async mode) |

`Program.cs` reads the YAML path from the first command-line argument or the `NTRADA_CONFIG` environment variable, defaulting to `ntrada.yml`. It then calls `AddNtrada()`, registers `CorrelationContextBuilder` as `IContextBuilder`, `SpanContextBuilder` as `ISpanContextBuilder` and `HttpRequestHook` as `IHttpRequestHook`, chains `AddConvey().AddMetrics().AddSecurity()`, and finishes with `app.UseNtrada()` and `.UseLogging()`.

## 4. Important modules / packages

- `src/Pacco.APIGateway/Infrastructure/CorrelationContextBuilder.cs` and `CorrelationContext.cs` — build the correlation payload propagated to downstream services.
- `src/Pacco.APIGateway/Infrastructure/SpanContextBuilder.cs` — extracts the tracing span context.
- `src/Pacco.APIGateway/Infrastructure/HttpRequestHook.cs` — hook invoked on each proxied HTTP request.
- NuGet packages (`src/Pacco.APIGateway/Pacco.APIGateway.csproj`): `Ntrada` `0.4.*` plus `Ntrada.Extensions.Cors`, `.CustomErrors`, `.Jwt`, `.RabbitMq`, `.Swagger`, `.Tracing`; `Convey.Logging`, `Convey.Metrics.AppMetrics`, `Convey.Security`, all `0.4.*`; `NetEscapades.Configuration.Yaml` `2.0.0`.

## 5. External integrations

- **RabbitMQ** — via `Ntrada.Extensions.RabbitMq`, configured in `ntrada-async.yml`.
- **Jaeger** — UDP agent on port `6831`, service name `api-gateway`.
- **Seq** — `http://localhost:5341` (`src/Pacco.APIGateway/appsettings.json`).
- **Prometheus** — metrics enabled; InfluxDB configured but disabled.
- **Fabio** — a `loadBalancer` block exists in `ntrada.yml` pointing at `localhost:9999` but is set `enabled: false`.
- All ten Pacco services, as HTTP downstreams.

## 6. Data stores & state

**None.** The gateway is stateless. No database client, no ORM, no query mechanism, no migration tool, no collection or table names, no foreign keys.

## 7. Messaging / async / events

**System:** RabbitMQ, configured in `src/Pacco.APIGateway/ntrada-async.yml` under the `rabbitmq` extension: `connectionName: api-gateway`, port `5672`, vhost `/`, username and password `guest`, exchange type `topic`, `messageContext.header: message_context`, `spanContextHeader: span_context`.

In async mode every mutating HTTP route is marked `use: rabbitmq` and is published to an exchange with a routing key instead of being proxied. Exchange and routing key names, verbatim:

| Exchange | Routing keys published |
|---|---|
| `availability` | `add_resource`, `reserve_resource`, `release_resource`, `delete_resource` |
| `customers` | `complete_customer_registration`, `change_customer_state` |
| `deliveries` | `start_delivery`, `fail_delivery`, `complete_delivery`, `add_delivery_registration` |
| `orders` | `create_order`, `delete_order`, `add_parcel_to_order`, `delete_parcel_from_order`, `assign_vehicle_to_order` |
| `parcels` | `add_parcel`, `delete_parcel` |
| `vehicles` | `add_vehicle`, `update_vehicle`, `delete_vehicle` |

**Payload key fields:** the gateway does not define payloads in C#. `ntrada-async.yml` shapes them declaratively. The `customers` module route names a payload and schema pair `complete_customer_registration` / `complete_customer_registration.schema`. Elsewhere, `bind` entries inject values into the payload — observed bindings are `customerId:@user_id`, `resourceId:{resourceId}` and `dateTime:{dateTime}`. The full field set of each published message is defined by the consuming service's command class, not here.

## 8. APIs exposed & consumed

**Exposed** (from `ntrada.yml` modules; each module's `path` is the public prefix):

| Module | Public path prefix | Notable routes |
|---|---|---|
| `home` | `/` | `GET /` returns the literal string `Welcome to Pacco API!` |
| `availability` | `availability` | `GET resources`, `GET resources/{resourceId}`, plus the mutating routes listed above |
| `customers` | `customers` | `GET /`, `GET /{customerId}`, `GET /{customerId}/state` |
| `deliveries` | `deliveries` | `GET /{deliveryId}` |
| `identity` | `identity` | `POST sign-up`, `POST sign-in` |
| `operations` | `operations` | `GET /{operationId}` |
| `orders` | `orders` | `GET /` |
| `parcels` | `parcels` | `GET /`, `GET volume` |
| `pricing` | `pricing` | `GET /` |
| `vehicles` | `vehicles` | `GET /` |

Swagger UI is served at `routePrefix: docs`.

**Consumed** — downstream service addresses from the `services:` block in `ntrada.yml`. Each entry has a `localUrl` for local runs and a `url` used inside Docker, for example `availability-service: localUrl: localhost:5001, url: availability-service` and `customers-service: localhost:5002`. Sample downstream paths seen in routes: `availability-service/resources`, `customers-service/customers/{customerId}/state`, `deliveries-service/deliveries/{deliveryId}`, `identity-service/sign-up`, `identity-service/sign-in`, `operations-service/operations/{operationId}`, `orders-service/orders?customerId=@user_id`, `parcels-service/parcels?customerId=@user_id`, `parcels-service/parcels/volume`, `pricing-service/pricing?customerId=@user_id`, `vehicles-service/vehicles`.

HTTP behaviour: 2 retries with exponential backoff.

## 9. Deployment & runtime clues

- `Dockerfile`: multi-stage, `mcr.microsoft.com/dotnet/core/sdk:3.1` to build, `mcr.microsoft.com/dotnet/core/aspnet:3.1` to run. Sets `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`, `NTRADA_CONFIG=ntrada.docker`.
- `.travis.yml`: `language: csharp`, `dotnet: 3.1.100`, branch filter `master` and `develop` only, `script: ./scripts/build.sh` then `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- `scripts/`: `start.sh`, `start-async.sh`, `build.sh`, `test.sh`, `dockerize.sh`.
- Published image: `devmentors/pacco.apigateway`, referenced from `hianshul100_Pacco/compose/services.yml`.
- Sync and async modes are the same binary with a different YAML file; they are not separate deployables.

## 10. Security & auth clues

- JWT bearer auth: `ntrada.yml` sets `auth.enabled: true` with `auth.global: false`, so authentication is opt-in per route. `validIssuer: pacco`.
- Role claim type: `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`. The `customers` module routes `GET /`, `GET /{customerId}` and `GET /{customerId}/state` require `claims: role: admin`.
- `POST identity/sign-up`, `POST identity/sign-in` and `GET operations/{operationId}` are explicitly `auth: false`.
- **The JWT signing key is hardcoded in `ntrada.yml` and `ntrada-async.yml`** as `jwt.issuerSigningKey`, value `eiquief5phee9pazo0Faegaez9gohThailiur5woy2befiech1oarai4aiLi6ahVecah3ie9Aiz6Peij`. It is checked into source control in plain text.
- RabbitMQ credentials `guest`/`guest` are in `ntrada-async.yml`.
- CORS `allowedOrigins: '*'`. Exposed headers: `Request-ID`, `Resource-ID`, `Trace-ID`, `Total-Count`.
- A development certificate is committed at `src/Pacco.APIGateway/certs/localhost.cer`.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger via `Ntrada.Extensions.Tracing`; `serviceName: api-gateway`, UDP agent port `6831`. `SpanContextBuilder` propagates the span context downstream, and the `span_context` header carries it onto RabbitMQ messages.
- **Correlation:** `CorrelationContextBuilder` produces the correlation payload; `Request-ID` and `Trace-ID` are exposed to browsers via CORS.
- **Logging:** `src/Pacco.APIGateway/appsettings.json` configures console, rolling file, and Seq at `http://localhost:5341`.
- **Metrics:** AppMetrics with `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, environment `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.APIGateway/ntrada.yml` — the platform's public API surface, auth policy and downstream address book, all in one file. The single most decision-dense file in the repository.
- `src/Pacco.APIGateway/ntrada-async.yml` — the decision that mutations go through RabbitMQ rather than synchronous HTTP.
- `src/Pacco.APIGateway/Program.cs` — the decision to use Ntrada rather than hand-written routing.
- `Pacco.rest`, `Pacco-sample-scenario.rest` — executable request samples that document intended end-to-end flows.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- `ntrada.yml` sets `loadBalancer.enabled: false` while every downstream service is registered in Consul and configured to sit behind Fabio. How traffic actually reaches services in a deployed environment — direct DNS, Fabio, or Docker network aliases — is **Unknown** from the configuration alone. **Needs validation.**
- Sync mode and async mode are mutually exclusive per process. Whether production runs one, the other, or two gateway deployments is **Unknown**; `compose/services.yml` starts a single `api-gateway` container.
- The exact payload shape of each published RabbitMQ message is **unknown — requires runtime capture** or cross-reading of the consuming service's command class.
- `ntrada.docker.yml` and `ntrada-async.docker.yml` were not diffed against their non-Docker counterparts; differences beyond host names are **Unknown**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.APIGateway/`, `src/Pacco.APIGateway/Infrastructure/`, `src/Pacco.APIGateway/Properties/`, `src/Pacco.APIGateway/certs/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript. The Swagger UI at `/docs` is generated at runtime by `Ntrada.Extensions.Swagger`; no assets for it are stored in the repository.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| The gateway is part of the Pacco solution and is started with `dotnet run` or `./scripts/start.sh`. | Both exist. `scripts/start.sh` is present. | **Confirmed.** |
| HTTP requests are listed in a `.rest` file at the repository root. | `Pacco.rest` and `Pacco-sample-scenario.rest` are both at the root. | **Confirmed.** |
| The README does not mention the async RabbitMQ mode. | `ntrada-async.yml`, `ntrada-async.docker.yml` and `scripts/start-async.sh` exist and implement a materially different runtime behaviour. | **Disk-only component.** A significant capability is undocumented. |
| The README does not mention that the JWT signing key is embedded in configuration. | `ntrada.yml` and `ntrada-async.yml` both contain a literal `issuerSigningKey`. | **Disk-only.** Worth surfacing to whoever owns deployment. |

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The hardcoded `issuerSigningKey` in `ntrada.yml` is a local-development value that is overridden in real deployments. | The same repository ships a `certs/localhost.cer` development certificate, and the services validate JWTs against a certificate rather than a shared key. | If it is used as-is anywhere reachable, anyone with the repository can mint valid tokens for any user and role. | Check the deployment pipeline and Vault for an override; confirm with whoever owns deployment. |
| A2 | Sync mode (`ntrada.yml`) and async mode (`ntrada-async.yml`) are alternative configurations of one deployable, not two services. | One `.csproj`, one `Dockerfile`, one image, and a single `NTRADA_CONFIG` variable selecting the file. | The platform's public surface would be split across two runtimes, changing every downstream diagram. | Inspect the running environment or ask whoever operates the gateway. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is the committed JWT signing key in use in any environment other than a developer laptop? | It is a working signing key sitting in source control. If it is live, it is a serious exposure. | It is most likely development-only, but that has not been proven from the repository alone. | Security owner / deployment owner |
| Q2 | **[handled later by the platform inventory review]** With `loadBalancer.enabled: false`, how does the gateway actually resolve downstream services in a deployed environment? | Determines whether Consul and Fabio are on the real request path or only registered and unused. | Docker Compose service names (`availability-service`, etc.) resolve on the `pacco-network`, so Fabio may be bypassed entirely. | Platform architect |
| Q3 | **[handled later by the platform inventory review]** Should CORS `allowedOrigins: '*'` remain, given there is no browser client in this workspace? | A wide-open CORS policy on a public gateway is worth a deliberate decision rather than a default. | — | Security owner |
