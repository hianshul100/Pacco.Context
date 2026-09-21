# Repository: `Pacco.APIGateway`

`api-gateway` (also known as: Pacco API Gateway, `Pacco.APIGateway`, Docker image
`devmentors/pacco.apigateway`) is the single north-south entry point for the Pacco platform.

- **Repository:** `Pacco.APIGateway`, path: `src/Pacco.APIGateway`
- **Base ref analysed:** `feature/12998/aidlc`

---

## README vs repository

`README.md` is the platform boilerplate: logo, the shared "What is Pacco?" paragraph, a Travis
badge, and start instructions. It does not name Ntrada, the YAML route files, the sync/async
duality, or any route.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1 (`Dockerfile` uses
`mcr.microsoft.com/dotnet/core/sdk:3.1`); Travis CI (`.travis.yml`); the `scripts/` start path.

**Present on disk, absent from README (disk-only) — all of the following are undocumented:**

- That the gateway is **Ntrada**, a configuration-driven reverse proxy, not hand-written routing.
- That **four** gateway configurations exist and that two of them (`ntrada-async.yml`,
  `ntrada-async.docker.yml`) convert every write route from an HTTP call into a RabbitMQ publish.
  This is the single largest architectural fact in the repository.
- That `NTRADA_CONFIG` selects the configuration at startup.
- Every route, every downstream service, every routing key, every role gate.
- `Infrastructure/CorrelationContextBuilder.cs`, `Infrastructure/SpanContextBuilder.cs`,
  `Infrastructure/HttpRequestHook.cs` — the correlation, tracing and header-propagation seams.
- The JWT configuration, the `role: admin` gates, and the CORS policy.
- `certs/localhost.cer`.

**Stale doc.** `Pacco/README.md` points to `Pacco-sample-scenario.rest` "in the APIGateway repo";
the file was not found in this repository. **Needs validation.**

**Unknown.** Nothing in the repository states which of the four configurations is intended for
production.

---

## 1. Primary purpose

Terminate all external client traffic; validate JWTs; enforce coarse role-based access on a subset
of routes; bind caller identity (`@user_id`) into downstream URLs and message payloads; and route
each request either to a backend service over HTTP or onto a RabbitMQ exchange as a command —
without any per-route C# code.

## 2. Main runtime / service type

ASP.NET Core 3.1 web host hosting **Ntrada** `0.4.*` as an in-process API gateway. Single
container, single process. Stateless.

## 3. Key entrypoints

- `src/Pacco.APIGateway/Program.cs` — the only code entrypoint. Reads `NTRADA_CONFIG` (default
  `ntrada.yml`), loads it with `builder.AddYamlFile(configPath, false)`, then registers
  `AddNtrada()` together with `IContextBuilder → CorrelationContextBuilder`,
  `ISpanContextBuilder → SpanContextBuilder`, `IHttpRequestHook → HttpRequestHook`, plus
  `AddConvey().AddMetrics().AddSecurity()`; the pipeline is `app.UseNtrada()` and `.UseLogging()`.
- **Configuration entrypoints** (these define the gateway's behaviour, not the code):
  `src/Pacco.APIGateway/ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`,
  `ntrada-async.docker.yml`.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.APIGateway.dll`, `ASPNETCORE_URLS http://*:80`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

**Packages** (`src/Pacco.APIGateway/Pacco.APIGateway.csproj`): `Ntrada`,
`Ntrada.Extensions.Cors`, `Ntrada.Extensions.CustomErrors`, `Ntrada.Extensions.Jwt`,
`Ntrada.Extensions.RabbitMq`, `Ntrada.Extensions.Swagger`, `Ntrada.Extensions.Tracing`,
`Convey`, `Convey.Logging`, `Convey.Metrics.AppMetrics`, `Convey.Security`,
`NetEscapades.Configuration.Yaml`.

**Code modules** — four files, all cross-cutting:

| File | Responsibility |
|---|---|
| `Infrastructure/CorrelationContext.cs` | The correlation payload shape carried across the platform |
| `Infrastructure/CorrelationContextBuilder.cs` | Builds the correlation context that becomes the `message_context` header on published commands |
| `Infrastructure/SpanContextBuilder.cs` | Builds the `span_context` header so a Jaeger trace survives the HTTP→AMQP hop |
| `Infrastructure/HttpRequestHook.cs` | Mutates outbound downstream requests — the seam where gateway-added headers are applied |

## 5. External integrations

RabbitMQ (`Ntrada.Extensions.RabbitMq` — publishes commands in async mode), Jaeger (tracing,
`serviceName: api-gateway`, UDP 6831), Seq (structured logs), Prometheus (via
`Convey.Metrics.AppMetrics`, `/metrics` and `/metrics-text`), and Fabio
(`loadBalancer.url: fabio:9999` in the `.docker.yml` variants). It integrates with all nine
backend services as HTTP downstreams.

## 6. Data stores / state

**None.** The gateway is stateless: no database section in `appsettings*.json`, no persistence
package, no cache. There is therefore **no ORM or query mechanism**, **no migration tool**, and
**no tables or collections**. **No cross-domain data coupling** — but see dimension 8 for the
*contract* coupling it introduces by naming other services' routing keys.

## 7. Messaging / async / events

In **asynchronous mode** the gateway is a message publisher. Each write route declares an
`exchange` and a `routing_key`; Ntrada publishes the request body as a command message instead of
proxying it. Message and exchange names copied verbatim from `ntrada-async.yml`:

| Exchange | Routing key | Upstream route |
|---|---|---|
| `availability` | `add_resource` | `POST /availability/resources` |
| `availability` | `reserve_resource` | `POST /availability/resources/{resourceId}/reservations/{dateTime}` |
| `availability` | `release_resource` | `DELETE /availability/resources/{resourceId}/reservations/{dateTime}` |
| `availability` | `delete_resource` | `DELETE /availability/resources/{resourceId}` |
| `customers` | `complete_customer_registration` | `POST /customers` |
| `customers` | `change_customer_state` | `PUT /customers/{customerId}/state/{state}` |
| `deliveries` | `start_delivery` | `POST /deliveries` |
| `deliveries` | `fail_delivery` | `POST /deliveries/{deliveryId}/fail` |
| `deliveries` | `complete_delivery` | `POST /deliveries/{deliveryId}/complete` |
| `deliveries` | `add_delivery_registration` | `POST /deliveries/{deliveryId}/registrations` |
| `orders` | `create_order` | `POST /orders` |
| `orders` | `delete_order` | `DELETE /orders/{orderId}` |
| `orders` | `add_parcel_to_order` | `POST /orders/{orderId}/parcels/{parcelId}` |
| `orders` | `delete_parcel_from_order` | `DELETE /orders/{orderId}/parcels/{parcelId}` |
| `orders` | `assign_vehicle_to_order` | `POST /orders/{orderId}/vehicles/{vehicleId}` |
| `parcels` | `add_parcel` | `POST /parcels` |
| `parcels` | `delete_parcel` | `DELETE /parcels/{parcelId}` |
| `vehicles` | `add_vehicle` | `POST /vehicles` |
| `vehicles` | `update_vehicle` | `PUT /vehicles/{vehicleId}` |
| `vehicles` | `delete_vehicle` | `DELETE /vehicles/{vehicleId}` |

**Payload fields.** Ntrada forwards the HTTP request body as the message body; the gateway adds
bindings declared per route — notably `customerId: '@user_id'` on `reserve_resource`, which
injects the authenticated caller's id into the published command. The full field set of each
message is owned by the receiving service's command class, not by the gateway.

**Headers.** `CorrelationContextBuilder` and `SpanContextBuilder` supply the `message_context` and
`span_context` values that every downstream consumer reads.

**In synchronous mode** (`ntrada.yml`, `ntrada.docker.yml`) the gateway publishes **nothing** —
every one of the routes above becomes a `downstream` HTTP call instead. The gateway subscribes to
no messages in either mode.

## 8. APIs exposed / consumed

**Exposed** — one module per backend domain, each with `path` prefix and `localUrl`:

| Module | Path prefix | Downstream service | `localUrl` | Notable routes |
|---|---|---|---|---|
| `home` | `/` | — | — | `GET /` returns the literal string `Welcome to Pacco API [async]!` |
| `availability` | `availability` | `availability-service` | `localhost:5001` | `GET /resources`, `GET /resources/{resourceId}` + 4 write routes |
| `customers` | `customers` | `customers-service` | `localhost:5002` | `GET /` (**role `admin`**), `GET /me` → `customers-service/customers/@user_id`, `PUT /{customerId}/state/{state}` (**role `admin`**) |
| `deliveries` | `deliveries` | `deliveries-service` | `localhost:5003` | `GET /{deliveryId}` + 4 write routes |
| `identity` | `identity` | `identity-service` | `localhost:5004` | `GET /users/{userId}` (**role `admin`**), `GET /me`, `POST /sign-up`, `POST /sign-in` — all HTTP downstream in both modes |
| `operations` | `operations` | `operations-service` | `localhost:5005` | `GET /{operationId}` |
| `orders` | `orders` | `orders-service` | `localhost:5006` | `GET /` → `orders-service/orders?customerId=@user_id` + 5 write routes |
| `parcels` | `parcels` | `parcels-service` | `localhost:5007` | `GET /` → `parcels-service/parcels?customerId=@user_id`, `GET /volume` + 2 write routes |
| `pricing` | `pricing` | `pricing-service` | `localhost:5008` | `GET /` → `pricing-service/pricing?customerId=@user_id` |
| `vehicles` | `vehicles` | `vehicles-service` | `localhost:5009` | `GET /{vehicleId}`, `GET /` + 3 write routes |

Swagger is exposed via `Ntrada.Extensions.Swagger`.

**Consumed:** the HTTP APIs of all nine backend services. In `.docker.yml` variants `useLocalUrl`
is `false` and calls are resolved through Fabio at `fabio:9999` instead of the `localUrl` values.

**Note.** `ordermaker-service` has **no gateway module** in any of the four configurations — it is
not reachable through the platform edge, even though `Pacco/compose/services.yml` runs it (port
5015) and lists it under this gateway's own `depends_on`.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`, `ENV ASPNETCORE_URLS http://*:80`,
  `ENV ASPNETCORE_ENVIRONMENT docker`, `ENTRYPOINT dotnet Pacco.APIGateway.dll`.
- `.travis.yml`: `language: csharp`, `dotnet: 3.1.100`, branches `master` and `develop`,
  `script: ./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`.
- `scripts/dockerize.sh`: maps `master`→`latest`, `develop`→`dev`; `docker login` with
  `$DOCKER_USERNAME`/`$DOCKER_PASSWORD`; pushes `$DOCKER_USERNAME/pacco.apigateway`.
- `Pacco/compose/services.yml` runs it on `5000:80` with `NTRADA_CONFIG=ntrada-async.docker.yml`.
- `Pacco/prod-services.yml` runs it as `dotnet Pacco.APIGateway.dll` on port 5000.
- **Environment-driven architecture switch:** `NTRADA_CONFIG` decides whether the platform's write
  path is synchronous or asynchronous.

## 10. Security / auth clues

- **JWT validation at the edge.** All four configs declare an `auth` section with
  `validIssuer: pacco`, `validateAudience: false`, and a symmetric `issuerSigningKey`.
- **A symmetric JWT signing key is committed in plaintext** in all four `ntrada*.yml` files. The
  same key value also appears in `Pacco.Services.Operations/src/.../appsettings.json`. Recorded as
  an observation; the value is not reproduced here.
- **Role gates.** `claims: {role: admin}` is applied to `GET /customers`,
  `PUT /customers/{customerId}/state/{state}`, `GET /identity/users/{userId}`, and the vehicle
  write routes — a small, explicitly enumerated admin surface.
- **Identity binding.** `@user_id` is substituted into downstream paths and query strings
  (`/customers/me`, `/orders`, `/parcels`, `/pricing`) and into the `reserve_resource` payload.
  Authorisation for reads is therefore partly *implemented at the gateway* by rewriting the
  request, not by the backend service checking the caller.
- **CORS.** `allowedOrigins: ['*']` together with `allowCredentials: true`. Recorded as an
  observation — whether this is deliberate is **Unknown**.
- `certs/localhost.cer` is present; `Convey.Security` is referenced.
- Anonymous routes: `POST /identity/sign-up`, `POST /identity/sign-in`, and `GET /`.

## 11. Observability / logging / tracing

- **Tracing:** `Ntrada.Extensions.Tracing` with Jaeger; `serviceName: api-gateway`, UDP host
  `localhost` locally and `jaeger` in the `.docker.yml` variants; `SpanContextBuilder` propagates
  the span across the HTTP→AMQP boundary via the `span_context` header.
- **Correlation:** `generateRequestId` and `generateTraceId` are enabled; the gateway returns
  `Request-ID`, `Trace-ID`, `Resource-ID` and `Total-Count` headers.
- **Logging:** `Convey.Logging` with console, file and Seq sinks; `.UseLogging()` in `Program.cs`.
- **Metrics:** `Convey.Metrics.AppMetrics` with Prometheus enabled; this is the endpoint
  `Pacco/compose/prometheus/prometheus.yml` scrapes at `:5000/metrics-text`.

## 12. Architecture-decision files and feature flags

**Files carrying architecture decisions:**

| File | Decision |
|---|---|
| `src/Pacco.APIGateway/ntrada.yml` / `ntrada.docker.yml` | The synchronous edge: every route proxies to a service and returns its response |
| `src/Pacco.APIGateway/ntrada-async.yml` / `ntrada-async.docker.yml` | The asynchronous edge: 20 write routes become RabbitMQ publishes and return no domain result. Clients must observe outcomes through `operations-service` |
| `src/Pacco.APIGateway/Program.cs` | That the mode is selected by the `NTRADA_CONFIG` environment variable — an architectural switch made at deploy time |
| `src/Pacco.APIGateway/Infrastructure/SpanContextBuilder.cs` | That trace context crosses the protocol boundary into AMQP |
| `src/Pacco.APIGateway/Infrastructure/CorrelationContextBuilder.cs` | That caller identity and correlation travel with published commands |

**Feature flag system:** **none detected.** No LaunchDarkly, Unleash, Flagsmith, Split, or
in-house flag mechanism appears in the code or configuration. **No flag keys exist.** The
`NTRADA_CONFIG` environment variable performs a similar job (deploy-time behaviour switching) but
is a configuration selector, not a feature-flag system.

## 13. Open questions / ambiguities

1. Which mode — sync or async — is intended for production.
2. Whether `allowedOrigins: ['*']` with `allowCredentials: true` is deliberate.
3. Whether the committed symmetric signing key is a live secret.
4. Why `ordermaker-service` has no gateway module.
5. Whether async-mode clients are expected to poll `operations-service`, and how they learn the
   operation id — this is not expressed anywhere in the gateway configuration.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.APIGateway/` and all subdirectories
(`Infrastructure/`, `certs/`, `Properties/`), and the repository root. There is no `wwwroot/`,
`public/`, `public/js/`, `static/`, `assets/`, `resources/js/`, or `web/` directory; there is no
`package.json`, no bundler configuration, and no view templates (no `.cshtml`, `.html`, or
Razor pages). The gateway's only browser-visible surface is the Ntrada Swagger UI, which is served
by `Ntrada.Extensions.Swagger` and ships no first-party frontend code.

---

## Evidence

| Fact | File |
|---|---|
| Host composition, `NTRADA_CONFIG` selection, DI registrations | `src/Pacco.APIGateway/Program.cs` |
| Synchronous route map | `src/Pacco.APIGateway/ntrada.yml`, `src/Pacco.APIGateway/ntrada.docker.yml` |
| Asynchronous route map, exchanges and routing keys | `src/Pacco.APIGateway/ntrada-async.yml`, `src/Pacco.APIGateway/ntrada-async.docker.yml` |
| JWT, CORS and role configuration | all four `ntrada*.yml` files |
| Fabio/Jaeger/RabbitMQ hostnames for containers | `src/Pacco.APIGateway/ntrada.docker.yml`, `ntrada-async.docker.yml` |
| Correlation and span propagation | `src/Pacco.APIGateway/Infrastructure/CorrelationContext.cs`, `CorrelationContextBuilder.cs`, `SpanContextBuilder.cs`, `HttpRequestHook.cs` |
| Package set | `src/Pacco.APIGateway/Pacco.APIGateway.csproj` |
| Logging, metrics and environment settings | `src/Pacco.APIGateway/appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json` |
| Signing certificate | `src/Pacco.APIGateway/certs/localhost.cer` |
| Container build and runtime | `Dockerfile` |
| CI and image publication | `.travis.yml`, `scripts/build.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |
| Deployed port and selected config | `../hianshul100_Pacco/compose/services.yml`, `../hianshul100_Pacco/prod-services.yml` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Ntrada 0.4.* behaves as its configuration keys describe — `downstream` proxies, `exchange`/`routing_key` publishes, `claims` gates, `bind` substitutes | The Ntrada source is not in this workspace; only the configuration and four small C# files are visible | Every routing, auth and messaging statement in this document is derived from configuration semantics; if they differ, the gateway's real behaviour differs | Read the Ntrada 0.4 source, or capture gateway behaviour against a running platform |
| A2 | The `.docker.yml` variants are the container-targeted forms of their non-Docker counterparts, differing only in hostnames and `useLocalUrl` | A file-by-file comparison showed the route sets identical and only the infrastructure addresses changed | A behavioural difference hidden in the Docker variants would mean the deployed gateway does not match the analysed one | Diff the four files as part of any change to routing |
| A3 | The gateway performs no authorisation beyond JWT validation and the enumerated `role: admin` gates | No other claim checks appear in any configuration, and the four C# files contain no authorisation logic | Read scoping would be weaker than described — routes that rely on `@user_id` rewriting would be the only thing separating one customer's data from another's | Confirm whether backend services independently verify the caller, rather than trusting the rewritten request |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** A symmetric JWT signing key is committed in plaintext in all four `ntrada*.yml` files and repeated in `Pacco.Services.Operations`. Anyone holding it can mint a token the gateway accepts, including one with the `admin` role. We cannot tell from here whether this key is live | Treating the platform's authentication as sound, and any later decision about token handling | Whoever owns Pacco authentication | Confirm whether this key is used by a running system. If it is, rotate it, move it into Vault, and purge the value from git history | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is the production gateway meant to run the synchronous or the asynchronous configuration? | These are two different systems. In async mode, twenty write endpoints stop returning a domain result and become fire-and-forget publishes, so every client must be written differently. Nothing in the repository states the intent | `Pacco/compose/services.yml` selects `ntrada-async.docker.yml`, so async looks like the default — but that is a local-development file | Platform architect |
| Q2 | **[ACTION NOW]** In asynchronous mode, how does a client learn the outcome of a write? | The gateway returns no domain result for twenty routes. `operations-service` exists to report outcomes, but nothing in the gateway configuration hands the client an operation id or tells it where to look | Clients appear to be expected to connect to the `operations-service` SignalR hub and correlate by request id, but this is inference, not documented behaviour | Platform architect |
| Q3 | **[ACTION NOW]** Is `allowedOrigins: ['*']` with `allowCredentials: true` intentional? | This combination allows any website to make credentialed calls to the gateway on a signed-in user's behalf | It is likely a development convenience that was never tightened, but that is a guess | Whoever owns Pacco authentication |
| Q4 | **[ACTION NOW]** Why is `ordermaker-service` absent from all four gateway configurations? | It exposes `POST /orders` and drives the order-creation saga, yet nothing at the platform edge can reach it. Callers must therefore reach it by a path outside the gateway | It is **not** an undeployed service: `Pacco/compose/services.yml:78-85` and `services-local.yml:78-85` run it on port 5015 and this gateway `depends_on` it. So the omission looks deliberate — an internal-only orchestrator — rather than an artefact of the service being unused | Platform owner |
| Q5 | **[handled later by HLD]** Should the gateway's twenty routing keys be validated against the services that own them? | The gateway hard-codes other services' exchange and routing-key names in YAML with no compile-time or startup check. A renamed command in a service silently becomes an unroutable publish at the edge | Introduce a shared contract source or a startup validation step | Platform architect |
