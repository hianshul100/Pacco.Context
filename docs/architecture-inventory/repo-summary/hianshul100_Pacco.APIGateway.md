# Repository summary — `hianshul100_Pacco.APIGateway`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.APIGateway` (also known as: Pacco.APIGateway, `api-gateway`, the Pacco API gateway)
**Deployable:** `Pacco.APIGateway` (also known as: `api-gateway` — its container name and Consul/Jaeger service name — and `devmentors/pacco.apigateway`, its published image). Repository: `hianshul100_Pacco.APIGateway`, path: `src/Pacco.APIGateway`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

Single entry point for every client of the platform. It terminates authentication, applies CORS, and then either proxies the request to a service over HTTP or turns it into a message published to RabbitMQ. Almost all of its behaviour is data, not code: the routing table lives in YAML.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, `Microsoft.NET.Sdk.Web`, hosting the **Ntrada** gateway (`Ntrada 0.4.*`). Evidence: `src/Pacco.APIGateway/Pacco.APIGateway.csproj`, `src/Pacco.APIGateway/Program.cs`.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.APIGateway/Program.cs` — `Main` → `CreateHostBuilder(args).Build().RunAsync()` |
| Routing table (sync profile) | `src/Pacco.APIGateway/ntrada.yml`, `src/Pacco.APIGateway/ntrada.docker.yml` |
| Routing table (async profile) | `src/Pacco.APIGateway/ntrada-async.yml`, `src/Pacco.APIGateway/ntrada-async.docker.yml` |
| Local run | `scripts/start.sh` (sync profile), `scripts/start-async.sh` (async profile) |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.APIGateway.dll` |

Configuration selection, from `Program.cs`: the YAML file is taken from `args[0]`, else the `NTRADA_CONFIG` environment variable, else `ntrada.yml`; a missing `.yml` suffix is appended. This one line decides whether the whole platform's write path is synchronous or asynchronous.

## 4. Important modules / packages

Only one project: `src/Pacco.APIGateway/Pacco.APIGateway.csproj`.

Packages: `Ntrada 0.4.*` with extensions `Ntrada.Extensions.Cors`, `.CustomErrors`, `.Jwt`, `.RabbitMq`, `.Swagger`, `.Tracing`; `Convey.Logging`, `Convey.Metrics.AppMetrics`, `Convey.Security`; `NetEscapades.Configuration.Yaml 2.0.0`.

Hand-written code is four small classes under `src/Pacco.APIGateway/Infrastructure/`:

- `CorrelationContext.cs` and `CorrelationContextBuilder.cs` — build the correlation payload carried into messages.
- `SpanContextBuilder.cs` — propagates the trace span into published messages.
- `HttpRequestHook.cs` — hooks outbound proxied requests.

These are registered in `Program.cs` as `IContextBuilder`, `ISpanContextBuilder` and `IHttpRequestHook`.

## 5. External integrations

- **RabbitMQ** (async profile only) — connection name `api-gateway`, host `localhost`, port `5672`, virtual host `/`, user `guest`, topic exchanges, `messageContext.header: message_context`, `spanContextHeader: span_context`.
- **Jaeger** — `tracing.serviceName: api-gateway`, UDP `localhost:6831`, `sampler: const`.
- **Seq** — `appsettings.json` sets `logger.seq.enabled: true`.
- **Prometheus** — `appsettings.json` sets `metrics.enabled: true` with `prometheus: true`, `influx: false`.
- **The ten Pacco services** — reached by service name (`availability-service`, `customers-service`, `deliveries-service`, `identity-service`, `operations-service`, `orders-service`, `parcels-service`, `pricing-service`, `vehicles-service`), each with a `localUrl` of `localhost:500X` for local running.

The gateway's own `loadBalancer` block is `enabled: false` with `url: localhost:9999`, so Fabio is configured but switched off at the edge.

## 6. Data stores and state handling

None. The gateway holds no database, no cache and no session store. There is no ORM, no migration tool, no table and no collection. Its only persistent state is the YAML routing table in the repository.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ, topic exchanges, via `Ntrada.Extensions.RabbitMq`. Active only under `ntrada-async.yml` / `ntrada-async.docker.yml`.

Under the async profile every write route publishes a message instead of proxying. Exchange and routing key, copied character-for-character from `src/Pacco.APIGateway/ntrada-async.yml`:

| Exchange | Routing keys |
|---|---|
| `availability` | `add_resource`, `reserve_resource`, `release_resource`, `delete_resource` |
| `customers` | `complete_customer_registration`, `change_customer_state` |
| `deliveries` | `start_delivery`, `fail_delivery`, `complete_delivery`, `add_delivery_registration` |
| `orders` | `create_order`, `delete_order`, `add_parcel_to_order`, `delete_parcel_from_order`, `assign_vehicle_to_order` |
| `parcels` | `add_parcel`, `delete_parcel` |
| `vehicles` | `add_vehicle`, `update_vehicle`, `delete_vehicle` |

Payload key fields are assembled by the `bind` blocks in the same file. Examples: `reserve_resource` binds `resourceId`, `customerId` (from `@user_id`), `dateTime`; `create_order` binds `customerId` from `@user_id` and generates `orderId`; `add_parcel_to_order` binds `orderId`, `parcelId`, `customerId`; `change_customer_state` binds `customerId`, `state`; `delete_parcel` binds `parcelId`, `customerId`. Fields the caller supplies in the request body are merged in by Ntrada and are not enumerated in the file, so the complete published payload for each routing key is **unknown — requires runtime capture**.

Routes that generate an identity before publishing use `resourceId: {property: …, generate: true}` — `deliveryId`, `orderId`, `parcelId`, `vehicleId`, and `userId` on sign-up. The generated value is returned to the caller in the `Resource-ID` response header, which is one of the exposed CORS headers.

Only `complete_customer_registration` carries a declared contract: `payload: complete_customer_registration` with `schema: complete_customer_registration.schema`. The referenced payload and schema files are not present in the repository — see *README vs repository*.

## 8. APIs exposed and consumed

**Exposed.** Base paths, from the `modules` blocks (module `path` prefixes the upstream route):

| Prefix | Routes |
|---|---|
| `/` | `GET /` → literal text (`Welcome to Pacco API!` sync, `Welcome to Pacco API [async]!` async) |
| `/availability` | `GET /resources`, `GET /resources/{resourceId}`, `POST /resources`, `POST /resources/{resourceId}/reservations/{dateTime}`, `DELETE /resources/{resourceId}/reservations/{dateTime}`, `DELETE /resources/{resourceId}` |
| `/customers` | `GET /` (role `admin`), `GET /me`, `GET /{customerId}` (role `admin`), `GET /{customerId}/state` (role `admin`), `POST /`, `PUT /{customerId}/state/{state}` (role `admin`) |
| `/deliveries` | `GET /{deliveryId}`, `POST /`, `POST /{deliveryId}/fail`, `POST /{deliveryId}/complete`, `POST /{deliveryId}/registrations` |
| `/identity` | `GET /users/{userId}` (role `admin`), `GET /me`, `POST /sign-up` (`auth: false`), `POST /sign-in` (`auth: false`) |
| `/operations` | `GET /{operationId}` (`auth: false`) |
| `/orders` | `GET /`, `GET /{orderId}`, `POST /`, `DELETE /{orderId}`, `POST /{orderId}/parcels/{parcelId}`, `DELETE /{orderId}/parcels/{parcelId}`, `POST /{orderId}/vehicles/{vehicleId}` |
| `/parcels` | `GET /`, `GET /{parcelId}`, `GET /volume`, `POST /`, `DELETE /{parcelId}` |
| `/pricing` | `GET /` |
| `/vehicles` | `GET /`, `GET /{vehicleId}`, `POST /`, `PUT /{vehicleId}`, `DELETE /{vehicleId}` |

Swagger is served at route prefix `docs` (`extensions.swagger`, title `Pacco API`, version `v1`, `includeSecurity: true`).

Three routes quietly rewrite the request so a caller can only see their own data: `GET /orders` → `orders-service/orders?customerId=@user_id`, `GET /parcels` → `parcels-service/parcels?customerId=@user_id`, `GET /pricing` → `pricing-service/pricing?customerId=@user_id`, and `GET /customers/me` → `customers-service/customers/@user_id`. `GET /vehicles` unwraps the paged result with `onSuccess: data: response.data.items`.

**Consumed.** Under the sync profile, HTTP to all nine services listed above. Under the async profile, the read routes stay HTTP and the write routes become RabbitMQ publishes. Gateway HTTP retry policy: `http.retries: 2`, `interval: 2.0`, `exponential: true`.

## 9. Deployment and runtime clues

- `Dockerfile`: build on `mcr.microsoft.com/dotnet/core/sdk:3.1`, run on `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`, `NTRADA_CONFIG ntrada.docker`.
- **Conflict.** The image defaults to the **synchronous** profile (`NTRADA_CONFIG ntrada.docker`), but `hianshul100_Pacco/compose/services.yml` overrides it to `ntrada-async.docker.yml`. Whichever one a reader looks at first gives them the wrong answer about how writes flow.
- CI: `.travis.yml` — csharp, dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success. `dockerize.sh` tags master as `latest` plus the build number and develop as `dev` plus `dev-<build number>`, then pushes to Docker Hub using `$DOCKER_USERNAME` / `$DOCKER_PASSWORD`.
- Published on host port 5000 in the composed stack.

## 10. Security and auth clues

- JWT bearer validation via `Ntrada.Extensions.Jwt`. `validIssuer: pacco`, `validateIssuer: true`, `validateLifetime: true`, `validateAudience: false`.
- `auth.enabled: true` with `auth.global: false` — authentication is **opt-in per route**, not applied by default. Every route in both YAML files carries an explicit `auth` value, so nothing is currently unprotected by accident, but a new route added without `auth: true` would be public.
- Role claim type: `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`. Routes requiring `role: admin`: all four admin customer routes and `GET /identity/users/{userId}`.
- Deliberately anonymous: `POST /identity/sign-up`, `POST /identity/sign-in`, `GET /operations/{operationId}`.
- **Hard-coded secret.** `extensions.jwt.issuerSigningKey` is a symmetric signing key written directly into `ntrada.yml` and `ntrada-async.yml`. The same value appears in the Identity and Operations service settings. Anyone with repository access can mint a valid token for any user and any role.
- CORS: `allowedOrigins: ['*']` together with `allowCredentials: true`, `allowedHeaders: ['*']`. Exposed headers: `Request-ID`, `Resource-ID`, `Trace-ID`, `Total-Count`.
- `customErrors.includeExceptionMessage: true` returns exception text to callers.
- `certs/localhost.cer` is committed under `src/Pacco.APIGateway/`.

## 11. Observability, logging and tracing clues

`generateRequestId: true` and `generateTraceId: true` mint the correlation and trace identifiers at the edge; `SpanContextBuilder` and `CorrelationContextBuilder` carry them into published messages under the `span_context` and `message_context` headers, which is what makes an end-to-end trace possible across the async hops. Tracing goes to Jaeger as service `api-gateway`; logging goes to Seq; metrics go to Prometheus via `Convey.Metrics.AppMetrics`. `useForwardedHeaders: true`.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.APIGateway/ntrada.yml` and `ntrada-async.yml` are the most decision-dense files in the whole platform: they define the public API surface, the authorisation model, the sync-versus-async write path, and the message contracts at the edge. `Program.cs` defines the configuration-selection rule. The `Dockerfile` and `hianshul100_Pacco/compose/services.yml` decide which profile actually runs.

**Feature flag system: none.** No flag library and no flag store. The one runtime switch is the `NTRADA_CONFIG` environment variable, which selects a whole routing table rather than toggling a feature; its observed values are `ntrada.docker` (in the `Dockerfile`) and `ntrada-async.docker.yml` (in `compose/services.yml`). Flag keys: none exist.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** `hianshul100_Pacco.APIGateway/` (repository root), `src/Pacco.APIGateway/`, `src/Pacco.APIGateway/Infrastructure/`, `src/Pacco.APIGateway/certs/`, `scripts/`. There is no `package.json`, no `wwwroot`, no template file, no bundler configuration and no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** the gateway is built with Ntrada; it runs via `dotnet run` or `./scripts/start.sh`; it is available on `http://localhost:5000`; it can be built from the local `Dockerfile` or pulled as `devmentors/pacco.apigateway`.

**Present on disk but absent from the README:**

- The entire async profile. `ntrada-async.yml`, `ntrada-async.docker.yml` and `scripts/start-async.sh` exist, and the composed stack uses the async profile by default, yet the README never mentions that a second mode exists. This is the single biggest documentation gap in the repository: a reader following the README will believe every write is a synchronous HTTP proxy.
- `Pacco.rest` and `Pacco-sample-scenario.rest` — request collections in the repository root that the README does not point at (the service READMEs do point at their own `.rest` files).
- The four `Infrastructure/` classes that add correlation and span propagation.

**Conflicts to surface:**

- **Stale doc.** The README says `dotnet run` should be executed in `/src/Pacco.APIGateway`, and that directory does exist here — this one is correct, unlike the equivalent line in the service READMEs.
- **Configuration conflict.** `Dockerfile` sets `NTRADA_CONFIG ntrada.docker` (sync) while `hianshul100_Pacco/compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml` (async). Both are current; they disagree about the default write path.
- **Unverifiable — Missing Source Evidence.** `ntrada-async.yml` references `payload: complete_customer_registration` and `schema: complete_customer_registration.schema`, and `ntrada.yml` references `payload: create_customer` with `schema: create_customer.schema`. No payload or schema files exist anywhere in the repository, so those routes' request validation cannot be verified from source.
- **Unknown.** `ntrada.docker.yml` and `ntrada-async.docker.yml` were not compared line by line against their non-`.docker` counterparts; the differences are assumed to be host names only.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The `.docker.yml` routing tables differ from the plain ones only in host names and service URLs. | They are named as environment variants of the same file and are loaded by the same mechanism. |
| A2 | The composed stack, which selects the async profile, represents the intended production behaviour. | It is the only stack that runs published images. |
| A3 | Request bodies are merged into published messages by Ntrada alongside the `bind` values. | The `bind` blocks only ever add identity and user fields, never the business payload, yet the receiving commands need it. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** The JWT signing key is written in plain text in `ntrada.yml` and `ntrada-async.yml`, and the same value is repeated in the Identity and Operations service settings. | Anyone who can read the repository can forge an admin token for any user. | Platform security owner: rotate the key and move it into Vault, which the platform already runs. |
| B2 | **[ACTION NOW]** The payload and schema files named by the customer routes do not exist in the repository. | Those two routes cannot be validated or reasoned about from source, and they may be failing at runtime. | Gateway maintainer: add the missing files or remove the references. |
| B3 | **[ACTION NOW]** CORS allows every origin while also allowing credentials, and error responses include exception messages. | Both settings are unsafe outside a local sandbox. | Gateway maintainer: restrict origins and turn off exception message pass-through for non-local environments. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Which profile is the intended default — the synchronous one baked into the image, or the asynchronous one set by the composed stack? | The two answers give completely different reliability, latency and failure behaviour for every write in the platform. | Gateway maintainer. |
| Q2 | **[handled later by the API-contract stage]** What is the full published message body for each of the twenty routing keys? | The `bind` blocks show only part of it; consumers depend on the rest. | API-contract stage, by capturing messages from a running gateway. |
| Q3 | **[ACTION NOW]** Should the async profile be documented in the README? | Every reader currently forms an incorrect picture of how writes reach the services. | Gateway maintainer. |
| Q4 | **[handled later by the security-architecture stage]** With `auth.global: false`, is per-route opt-in the intended posture? | A future route added without `auth: true` becomes public with no warning. | Security-architecture stage. |
