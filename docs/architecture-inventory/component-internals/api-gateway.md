# Component Internals — `api-gateway`

**Component:** `api-gateway`
**Repository:** `hianshul100_Pacco.APIGateway` (`https://github.com/hianshul100/Pacco.APIGateway`)
**Scoped path:** `.` (whole repository; the deployable lives at `src/Pacco.APIGateway`)
**Base ref inspected:** `feature/12998/aidlc`
**Batch:** 1 of 7 (`api-gateway`, `availability-service`)
**Status:** new artifact — no prior `component-internals/api-gateway.md` existed in this repository at the time of writing, so nothing was adopted or superseded. Related, non-superseded baselines: `docs/architecture-inventory/baselines/service-summaries.md` §2.1, `docs/architecture-inventory/repo-summary/Pacco.APIGateway.md`, `docs/architecture-inventory/patterns/integration/declarative-configuration-driven-api-gateway.md`.

> **How to read this document.** This is a *working model of the component's internals*, not a
> catalogue of its surface. Every mechanism claim is cited to a file and a line or a config key in
> this repository. Where a mechanism is implemented inside the `Ntrada 0.4.*` NuGet package rather
> than in this repository, that is stated explicitly and the claim is marked
> **`Unverifiable — Missing Source Evidence`** unless this repository's own code or configuration
> proves it. The Ntrada source is **not** in this workspace.

---

## 1. Purpose & boundary

### 1.1 What this component is

`api-gateway` is the platform's single north–south edge. It is an ASP.NET Core 3.1 host whose
*entire* request-handling behaviour is supplied by the `Ntrada` middleware pipeline
(`src/Pacco.APIGateway/Program.cs:47` — `app.UseNtrada()`), driven by a **YAML routing manifest**
selected at process start. This repository contains **four** such manifests
(`ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml`) and exactly
**four C# types** (`src/Pacco.APIGateway/Infrastructure/`), which together supply the
correlation/tracing context that every downstream Pacco service depends on for caller identity.

Its responsibilities, all evidenced in this repository:

| Responsibility | Where it is implemented |
| --- | --- |
| Terminate untrusted HTTP, expose one public port | `Dockerfile:9` (`ASPNETCORE_URLS http://*:80`); `src/Pacco.APIGateway/Properties/launchSettings.json` (`http://localhost:5000` locally) |
| Validate a JWT it did not issue | `ntrada.yml:43-48` (`extensions.jwt`) — validation performed inside `Ntrada.Extensions.Jwt` |
| Gate routes on authentication | `ntrada.yml:1-5` (`auth.enabled: true`, `auth.global: false`) plus per-route `auth: true/false` |
| Gate routes on a `role` claim | `ntrada.yml:5` (claim alias) plus per-route `claims: {role: admin}` — 5 routes |
| Bind the token subject into downstream payloads | per-route `bind:` entries using the `@user_id` placeholder |
| Route to a downstream HTTP service **or** publish to a RabbitMQ exchange | per-route `use: downstream` vs `use: rabbitmq` |
| Manufacture request/trace identifiers | `ntrada.yml:16-17` (`generateRequestId`, `generateTraceId`) |
| Build and forward the platform's caller-identity envelope | `Infrastructure/CorrelationContextBuilder.cs:20`, `Infrastructure/HttpRequestHook.cs:20` |
| Propagate the OpenTracing span across the async boundary | `Infrastructure/SpanContextBuilder.cs:18` |
| Publish an aggregated Swagger document | `ntrada.yml:50-56` (`extensions.swagger`, `routePrefix: docs`) |
| Emit logs and Prometheus metrics | `Program.cs:45,48`; `appsettings.json:2-26` |

### 1.2 What this component explicitly is **not**

- **It owns no domain state.** There is no database client, no repository, no entity and no
  migration anywhere in this repository. The complete non-generated C# surface is four files under
  `src/Pacco.APIGateway/Infrastructure/` totalling 118 lines.
- **It is not an authorization engine.** The only authorization primitive is exact string equality
  on a single configured claim (`claims: {role: admin}`); there is no policy language, no scope
  model, no per-resource ownership check. Resource-level ownership is enforced *downstream* — see
  `component-internals/availability-service.md` §3.13 for how `availability-service` consumes the
  identity this gateway asserts.
- **It is not a token issuer.** `POST /identity/sign-in` and `POST /identity/sign-up` are proxied
  with `auth: false` (`ntrada.yml:254-269`); `identity-service` mints the JWT. The gateway only
  holds validation material.
- **It is not an aggregator/BFF.** Every route is 1:1 with a single downstream operation. The only
  response transformation available is a single JSON path projection (`onSuccess.data`), used once
  (`ntrada.yml:423-424`).
- **It is not a resilience boundary for the platform.** It has a retry policy
  (`ntrada.yml:7-10`) but there is no circuit breaker, bulkhead, rate limit or timeout budget in
  any manifest.
- **It does not participate in the transactional outbox.** In async mode it publishes to RabbitMQ
  directly at request time; there is no local store, so a publish failure is a lost write with no
  retry beyond the RabbitMQ client's own.

### 1.3 Boundary caveat that dominates every other fact here

The four manifests are **two architecturally different systems**, not four environment variants of
one system. The `ntrada.yml` / `ntrada.docker.yml` pair proxies every write over HTTP and returns
the downstream status code. The `ntrada-async.yml` / `ntrada-async.docker.yml` pair converts
**20 write routes** into fire-and-forget RabbitMQ publishes that return no result. Which pair is
authoritative in production is **not answerable from this repository** — see §8 (ABQ-1) and
`baselines/service-summaries.md` Q1/G6.

---

## 2. Core concepts (exhaustive)

Every significant concept this component defines or owns, in the order they are exercised by a
request. Concepts 1–4 are *configuration structure*; 5–11 are *request-time policy*; 12–16 are
*context manufacture*; 17–22 are *cross-cutting/host*.

| # | Concept | Owned by |
| --- | --- | --- |
| 1 | Routing manifest (the config document) | this repo (`ntrada*.yml`) + `Program.cs` selection logic |
| 2 | Manifest selection (`NTRADA_CONFIG` / argv / default) | `Program.cs:28-39` — **this repo's own code** |
| 3 | Module | `ntrada*.yml` `modules:` |
| 4 | Route | `ntrada*.yml` `routes:` entries |
| 5 | Route strategy (`use:`) | `ntrada*.yml` per route |
| 6 | Downstream service registry & URL resolution | `ntrada*.yml` `services:` + `useLocalUrl` + `loadBalancer` |
| 7 | Authentication gate | `auth.enabled` / `auth.global` / per-route `auth:` |
| 8 | Claim gate | `auth.claims` alias map + per-route `claims:` |
| 9 | Token-subject binding (`@user_id`) and `bind:` | per-route `bind:` and inline in `downstream:` |
| 10 | Payload template & JSON schema (`payload:` / `schema:`) | per-route — **referenced but not present** |
| 11 | Resource-ID generation (`resourceId:`) | per-route |
| 12 | Request ID | `generateRequestId: true` |
| 13 | Trace ID | `generateTraceId: true` |
| 14 | Correlation context envelope | `Infrastructure/CorrelationContext.cs`, `CorrelationContextBuilder.cs`, `HttpRequestHook.cs` — **this repo's own code** |
| 15 | Span context | `Infrastructure/SpanContextBuilder.cs` — **this repo's own code** |
| 16 | Async publish descriptor (`config.exchange` / `config.routing_key`) | per-route in the async manifests |
| 17 | Response shaping (`onSuccess.data`, `responseHeaders`) | per-route |
| 18 | CORS policy | `extensions.cors` |
| 19 | JWT validation parameters | `extensions.jwt` |
| 20 | HTTP retry policy | top-level `http:` |
| 21 | Swagger surface | `extensions.swagger` |
| 22 | Host observability (logging, metrics, tracing) | `Program.cs` + `appsettings*.json` + `extensions.tracing` |

---

## 3. Per concept

### 3.1 Routing manifest (the configuration document)

**Definition.** The single artifact that defines the gateway's entire behaviour: what routes exist,
who may call them, where each goes, and how the response is shaped. There is no code path in this
repository that adds, removes or modifies a route.

**Representation & storage.** Four YAML files in `src/Pacco.APIGateway/`:

| File | Lines | Write transport | Host resolution | Broker host | Jaeger host |
| --- | --- | --- | --- | --- | --- |
| `ntrada.yml` | 455 | HTTP `downstream` | `useLocalUrl: true` → `localUrl` | n/a | `localhost` |
| `ntrada.docker.yml` | 455 | HTTP `downstream` | `useLocalUrl: false` → `url` | n/a | `jaeger` |
| `ntrada-async.yml` | 534 | RabbitMQ `use: rabbitmq` | `useLocalUrl: true` → `localUrl` | `localhost` | `localhost` |
| `ntrada-async.docker.yml` | 534 | RabbitMQ `use: rabbitmq` | `useLocalUrl: false` → `url` | `rabbitmq` | `jaeger` |

The `.yml` → `.docker.yml` delta is genuinely only infrastructure hostnames — verified by diff, it
is 3 lines for the sync pair (`useLocalUrl`, `loadBalancer.url`, `tracing.udpHost`) and 4 for the
async pair (the same three plus `rabbitmq.hostnames`). The sync → async delta is the 20 rewritten
write routes plus the added `extensions.rabbitmq` block (`ntrada-async.yml:58-81`).

All four are copied to the output directory at build time
(`Pacco.APIGateway.csproj:25-37`, `CopyToOutputDirectory: Always`).

**Lifecycle.** Loaded **once**, at host construction, as an ASP.NET Core configuration source:
`Program.cs:38` — `builder.AddYamlFile(configPath, false)`. The second argument is `optional: false`,
so a missing file is a **hard startup failure**, not a silent empty routing table. There is no
reload-on-change (`reloadOnChange` is not passed), no admin API and no hot-reload endpoint: **a
route change requires a process restart.** Parsing is provided by
`NetEscapades.Configuration.Yaml 2.0.0` (`Pacco.APIGateway.csproj:13`).

**Invariants & enforcement.**
- File must exist → `FileNotFoundException` at startup (**fails loudly**), from `optional: false`.
- Schema validity beyond YAML well-formedness is enforced inside Ntrada's own option binding, which
  is not in this workspace. An unknown key (e.g. a typo'd `dowstream:`) binds to nothing and is
  **silently dropped by the .NET configuration binder** — this is standard `IConfiguration`
  behaviour, not Ntrada-specific. **`Unverifiable — Missing Source Evidence`** for the exact
  Ntrada-side consequence; what *is* certain is that no code in this repository validates manifest
  keys.

**Extension procedure — add a route.**
1. Choose the module in the target manifest (`modules.<module>.routes`). If the module is new, add
   `path:` and a `services:` block for it as well (§3.6).
2. Add the route entry with **at minimum** `upstream`, `method`, `use`, and — for
   `use: downstream` — `downstream`.
3. Decide `auth:` explicitly. Because `auth.global: false` (`ntrada.yml:3`), **omitting `auth`
   leaves the route public.** This is the single most common silent mistake on this component.
4. **Apply the same edit to all four manifests** unless you intend the route to exist only in one
   mode. Nothing cross-checks them; drift is invisible until a request 404s in one environment.
5. Restart the process.

**Failure modes.**
- Manifest absent/unreadable → process does not start.
- Route present in `ntrada.yml` but not `ntrada-async.yml` → the endpoint exists in sync mode and
  returns 404 in async mode, with no warning at startup.
- Two routes with the same `upstream` + `method` in one module → first-wins/last-wins is decided by
  Ntrada's route table construction. **`Unverifiable — Missing Source Evidence`.** Do not rely on
  either; keep `upstream`+`method` unique.

---

### 3.2 Manifest selection

**Definition.** The rule that picks *which* of the four manifests this process will run. This is
one of only two behaviours in the whole component implemented by this repository's own C# rather
than by Ntrada — it is therefore fully verifiable.

**Representation & storage.** `src/Pacco.APIGateway/Program.cs:28-39`:

```csharp
const string extension = "yml";
var ntradaConfig = Environment.GetEnvironmentVariable("NTRADA_CONFIG");
var configPath = args?.FirstOrDefault() ?? ntradaConfig ?? $"ntrada.{extension}";
if (!configPath.EndsWith($".{extension}")) { configPath += $".{extension}"; }
builder.AddYamlFile(configPath, false);
```

**Lifecycle.** Evaluated once per process start, inside `ConfigureAppConfiguration`. Precedence is
strictly: **first command-line argument** → **`NTRADA_CONFIG` environment variable** → literal
default `ntrada.yml`.

The `.yml` suffix is appended if absent, which is why `NTRADA_CONFIG=ntrada.docker`
(`Dockerfile:11`) and `NTRADA_CONFIG=ntrada-async` (`scripts/start-async.sh:3`) both work — they
resolve to `ntrada.docker.yml` and `ntrada-async.yml`.

Known settings of this switch in the workspace:

| Entry point | Value | Effective manifest |
| --- | --- | --- |
| `scripts/start.sh` | unset | `ntrada.yml` (default) |
| `scripts/start-async.sh:3` | `ntrada-async` | `ntrada-async.yml` |
| `Dockerfile:11` | `ntrada.docker` | `ntrada.docker.yml` |
| `Pacco/compose/services.yml` (sibling repo, read-only) | `ntrada-async.docker.yml` | `ntrada-async.docker.yml` — **overrides the image default** |

**Invariants & enforcement.** None beyond the `optional: false` file check. There is no allow-list
of permitted manifest names: **any path the process can read is accepted**, including one outside
the application directory. `configPath` is taken verbatim from `args[0]` or the environment with no
canonicalisation.

**Extension procedure — add a fifth mode.** Add the `.yml` file, set `NTRADA_CONFIG` (or pass argv)
in the launch surface (`Dockerfile`, `scripts/*.sh`, and the sibling `Pacco` repo's compose/PM2
manifests), and add it to `Pacco.APIGateway.csproj`'s `<Content Include=...>` list so it is copied
to the output directory — **omitting the csproj entry is a silent failure: the file builds fine and
the container starts with a `FileNotFoundException` at runtime.**

**Failure modes.** Typo in `NTRADA_CONFIG` → startup `FileNotFoundException` (**loud**). Because
the image default (`ntrada.docker`) is *sync* while the platform compose file sets *async*, running
the published image without setting `NTRADA_CONFIG` silently gives you the sync topology.

---

### 3.3 Module

**Definition.** A named grouping of routes that shares a URL path prefix and a downstream service
registry.

**Representation & storage.** `modules:` map in each manifest; the key is the module name and
`path:` is the prefix. Nine modules exist in every manifest:

| Module | `path:` | Routes (`ntrada.yml`) | Services declared |
| --- | --- | --- | --- |
| `home` | *(none)* | 1 | — |
| `availability` | `availability` | 6 | `availability-service` |
| `customers` | `customers` | 6 | `customers-service` |
| `deliveries` | `deliveries` | 5 | `deliveries-service` |
| `identity` | `identity` | 4 | `identity-service` |
| `operations` | `operations` | 1 | `operations-service` |
| `orders` | `orders` | 7 | `orders-service` |
| `parcels` | `parcels` | 5 | `parcels-service` |
| `pricing` | `pricing` | 1 | `pricing-service` |
| `vehicles` | `vehicles` | 5 | `vehicles-service` |

Total: 41 routes. The `home` module (`ntrada.yml:66-72`) has **no `path:`**, so its single route
`upstream: /` is the site root.

**Lifecycle.** Static; created and destroyed only by editing the manifest and restarting.

**Invariants & enforcement.** The effective URL is `/{module.path}{route.upstream}` — this is
Ntrada's composition rule, evidenced by the manifest's own shape (e.g. `customers` module,
`upstream: /me` → the platform's `GET /customers/me`, matching `downstream:
customers-service/customers/@user_id`). Nothing in this repository enforces prefix uniqueness across
modules.

**Extension procedure.** Add a `modules.<name>` key with `path:`, `routes:` and `services:`.
A module with `routes:` but no `services:` is only valid if every route uses `return_value` or
`rabbitmq`; a `downstream:` route naming a service that the module does not declare cannot be
resolved. **`Unverifiable — Missing Source Evidence`** whether Ntrada fails this at startup or at
first request; treat it as a request-time 500 and always declare the service.

**Failure modes.** Two modules with the same `path:` → overlapping URL space, resolution order
undefined here. `path:` omitted (as in `home`) → the module's routes are mounted at the root, which
is deliberate for `home` and almost certainly a bug anywhere else.

---

### 3.4 Route

**Definition.** One (method, upstream-path) pair and the complete policy applied to it.

**Representation & storage.** A YAML map inside `modules.<m>.routes`. The full key vocabulary
actually used across the four manifests:

| Key | Meaning | Example |
| --- | --- | --- |
| `upstream` | path suffix, may contain `{token}` segments | `ntrada.yml:96` |
| `method` | HTTP verb | `GET`/`POST`/`PUT`/`DELETE` |
| `use` | strategy — `downstream`, `rabbitmq`, `return_value` | §3.5 |
| `downstream` | target, `"<service-key>/<path>"`, may contain `{token}`, `@user_id` and a query string | `ntrada.yml:298` |
| `returnValue` | literal body for `use: return_value` | `ntrada.yml:72` |
| `auth` | boolean authentication gate | §3.7 |
| `claims` | required claim values | §3.8 |
| `bind` | payload injections | §3.9 |
| `payload` | payload template name | §3.10 |
| `schema` | JSON-schema name | §3.10 |
| `resourceId` | `{property, generate}` | §3.11 |
| `config` | `{exchange, routing_key}` for `use: rabbitmq` | §3.16 |
| `onSuccess` | `{data: <json path>}` response projection | §3.17 |
| `responseHeaders` | literal response headers | `ntrada.yml:268-269` |

**Lifecycle.** Purely declarative. Created/changed/removed by manifest edit + restart. There is no
per-route runtime state.

**Invariants & enforcement.** The only *enforced* invariants are those Ntrada applies at
request time (auth, claims, schema). Structural consistency — e.g. that every `{token}` in
`upstream` also appears in `downstream`, or that a `bind:` name matches a downstream payload
property — is **not checked anywhere**. A `bind: - foo:{bar}` where `{bar}` is not an `upstream`
token binds an unresolved placeholder; the downstream service then receives a value it cannot
parse, which surfaces as a *downstream* 400, not a gateway error.

**Extension procedure.** See §3.1. The per-route checklist that actually prevents defects:
`auth` set explicitly → `{token}`s balanced between `upstream` and `downstream` → `bind:` names
matching the downstream command's constructor parameter names (camelCase; e.g.
`customerId:@user_id` binds to `ReserveResource(… Guid customerId)`) → the same route added to all
four manifests.

**Failure modes.** A `{token}` present in `downstream` but not in `upstream` cannot be substituted;
the literal `{token}` text is forwarded. A route with `use: downstream` but no `downstream:` key
has no target.

---

### 3.5 Route strategy (`use:`)

**Definition.** Which execution engine handles the matched request. Three values appear in this
repository.

| `use:` | Behaviour | Occurrences |
| --- | --- | --- |
| `return_value` | Ntrada writes `returnValue` as the body; nothing leaves the process | 1 (`home`) |
| `downstream` | Proxy over HTTP to the resolved downstream URL, relay the status/body | 40 in `ntrada.yml`; 20 in `ntrada-async.yml` |
| `rabbitmq` | Serialize the bound payload and publish to `config.exchange` with `config.routing_key`; return without a domain result | 0 in `ntrada.yml`; 20 in `ntrada-async.yml` |

**Representation & storage.** Per-route `use:` key. `use: rabbitmq` is only functional when
`extensions.rabbitmq` is present — it exists **only** in the two async manifests
(`ntrada-async.yml:58-81`) and the `Ntrada.Extensions.RabbitMq` package is referenced
unconditionally (`Pacco.APIGateway.csproj:18`).

**Lifecycle.** Static per route.

**Invariants & enforcement.** The **critical, load-bearing consequence**: switching a route from
`downstream` to `rabbitmq` changes its contract from *synchronous, returns the domain result and
the downstream status code* to *fire-and-forget, returns no domain result*. In async mode the
caller learns the outcome only by polling `operations-service`
(`GET /operations/{operationId}`, `ntrada-async.yml` `operations` module, `auth: false`) keyed on
the `Request-ID` the gateway generated (§3.12). Client code written against the sync manifest
**silently breaks** against the async one: it still gets a 2xx, but the body no longer carries the
result, and the write may still be rejected downstream seconds later.

**Extension procedure — convert a write route to async.**
1. In the async manifests, replace `use: downstream` + `downstream: …` with
   `use: rabbitmq` + `config: {exchange: <service exchange>, routing_key: <snake_case command>}`.
2. The `routing_key` must equal the downstream command's type name in `snake_case` — that is the
   binding contract. For `availability-service`, `add_resource` ↔ `AddResource`
   (`ntrada-async.yml:121`; command type at
   `Pacco.Services.Availability/src/…Application/Commands/AddResource.cs:9`), matched by the
   consumer's `rabbitMq.conventionsCasing: "snakeCase"`
   (`Pacco.Services.Availability/src/…Api/appsettings.json:115`).
3. Add every field the command needs via `bind:` — the async path has no URL to carry path
   parameters into the payload, so `resourceId:{resourceId}` and `dateTime:{dateTime}` must be bound
   explicitly (`ntrada-async.yml:130-133`).
4. Ensure the downstream service actually calls `SubscribeCommand<T>()` for that type
   (for Availability: `…Infrastructure/Extensions.cs:107-110`). **If it does not, the message is
   published to a live exchange with no bound queue and is discarded by the broker — completely
   silently, with a 2xx returned to the caller.**

**Failure modes.** `use: rabbitmq` in a manifest without `extensions.rabbitmq` → no broker
connection is configured. Routing key that no consumer binds → message dropped by RabbitMQ (topic
exchange with no matching binding discards). Exchange name typo → same silent discard.

---

### 3.6 Downstream service registry & URL resolution

**Definition.** The per-module map from a service *key* used in `downstream:` to the two candidate
network addresses for that service.

**Representation & storage.** `modules.<m>.services.<key>` with two fields, e.g.
`ntrada.yml:123-126`:

```yaml
services:
  availability-service:
    localUrl: localhost:5001
    url: availability-service
```

The nine registrations and their local ports (identical in all four manifests):

| Service key | `localUrl` | `url` |
| --- | --- | --- |
| `availability-service` | `localhost:5001` | `availability-service` |
| `customers-service` | `localhost:5002` | `customers-service` |
| `deliveries-service` | `localhost:5003` | `deliveries-service` |
| `identity-service` | `localhost:5004` | `identity-service` |
| `operations-service` | `localhost:5005` | `operations-service` |
| `orders-service` | `localhost:5006` | `orders-service` |
| `parcels-service` | `localhost:5007` | `parcels-service` |
| `pricing-service` | `localhost:5008` | `pricing-service` |
| `vehicles-service` | `localhost:5009` | `vehicles-service` |

**Lifecycle.** Static. **There is no service discovery in this component**: no Consul client is
referenced in `Pacco.APIGateway.csproj`, and the address list is hard-coded per manifest. This is a
material difference from the downstream services, which *do* register with Consul
(e.g. `Pacco.Services.Availability/src/…Api/appsettings.json:7-17`).

**Invariants & enforcement.** Which field is used is decided by the top-level `useLocalUrl` flag:
`true` in `ntrada.yml`/`ntrada-async.yml` (line 18), `false` in both `.docker.yml` variants. Note
that the `loadBalancer` block is **`enabled: false` in all four manifests** (line 20), so the Fabio
address (`localhost:9999` / `fabio:9999`, line 21) is **configured but inert**. Any statement that
the gateway resolves downstreams through Fabio is contradicted by the manifests — the gateway
addresses services **directly**, by container/DNS name in Docker and by `localhost:<port>` locally.

**Extension procedure — add a downstream service.** Add the key under the owning module's
`services:`, set both `localUrl` (with the port from that service's `launchSettings.json`) and
`url` (its `container_name` in the platform compose file), in **all four** manifests. Then reference
the key as the first path segment of `downstream:`.

**Failure modes.** A `downstream:` naming an undeclared key cannot resolve — a request-time failure,
not a startup one. `localUrl` port drift against the service's actual `launchSettings.json` breaks
local development only, and silently (connection refused surfaces as a 5xx after the retry budget).
Enabling `loadBalancer` without a running Fabio makes every downstream call fail at once.

---

### 3.7 Authentication gate

**Definition.** Whether a route requires a valid bearer token before any other processing.

**Representation & storage.** Two levels:
- Global switch: `auth.enabled: true`, `auth.global: false` (`ntrada.yml:1-3`).
- Per route: `auth: true` / `auth: false`.

**Lifecycle.** Evaluated per request inside `Ntrada.Extensions.Jwt`.

**Invariants & enforcement — the most important default in this component.**
`auth.global: false` means authentication is **opt-in per route**. A route that omits `auth:`
entirely is **public**. Enforcement is therefore *by omission*, and a missing key is not an error.

Routes that are public in `ntrada.yml`, by explicit `auth: false`:

| Route | Line | Rationale |
| --- | --- | --- |
| `POST /identity/sign-up` | `ntrada.yml:258` | must precede having a token |
| `POST /identity/sign-in` | `ntrada.yml:267` | must precede having a token |
| `GET /operations/{operationId}` | `ntrada.yml:284` | **operation status is world-readable by id** |

The `home` route (`ntrada.yml:69-72`) omits `auth:` and is public by default — harmless, it returns
a constant string. Every other route in every manifest sets `auth: true` explicitly. That is 37 of
41 routes in `ntrada.yml`.

`GET /operations/{operationId}` being unauthenticated is a real, deliberate-looking property with a
consequence: in **async mode** it is the *only* channel through which a caller learns whether a
write succeeded, and anyone holding (or guessing) an operation id can read it. Operation ids are
the gateway-generated `Request-ID` (§3.12), so their unguessability is the only control.

**Extension procedure.** Set `auth: true` on the new route. If the route must be public, set
`auth: false` **explicitly** rather than omitting the key, so that the intent is reviewable.

**Failure modes.** Omitted `auth:` → **silently public**. Expired/invalid/absent token on an
`auth: true` route → 401 from the JWT middleware, before any hook runs, so no
`Correlation-Context` is built and nothing reaches the downstream. Setting `auth.global: true`
without auditing all 41 routes would break `sign-in`/`sign-up` and lock the platform out.

---

### 3.8 Claim gate

**Definition.** An additional per-route requirement that a named claim carry a specific value.

**Representation & storage.** Two parts:
- An **alias map** at `auth.claims` (`ntrada.yml:4-5`):
  `role: http://schemas.microsoft.com/ws/2008/06/identity/claims/role`. This maps the short name
  used in routes to the full WS-Federation claim URI that .NET's `ClaimTypes.Role` emits.
- Per route: `claims: {role: admin}`.

**Lifecycle.** Evaluated per request, after authentication.

**Invariants & enforcement.** Exactly **five** routes are admin-gated, identically in all four
manifests:

| Route | `ntrada.yml` | `ntrada-async.yml` |
| --- | --- | --- |
| `GET /customers/` | 136-138 | 169-171 |
| `GET /customers/{customerId}` | 150-152 | 183-185 |
| `GET /customers/{customerId}/state` | 158-160 | 191-193 |
| `PUT /customers/{customerId}/state/{state}` | 179-181 | 217-218 |
| `GET /identity/users/{userId}` | 244-246 | 290-291 |

Matching is exact string equality on the claim value. There is no role hierarchy, no "any of"
semantics, and no negative assertion. A claim name used in a route but **absent from the
`auth.claims` alias map is not translated** — whether Ntrada then matches the raw name or fails the
request is **`Unverifiable — Missing Source Evidence`**; the only mapped alias in this repository is
`role`.

Note the asymmetry this creates with the downstream services: the gateway's admin gate protects
*read* routes on Customers and Identity, but **no `availability` route is claim-gated**. Ownership
enforcement for reservations happens inside `availability-service`
(`…Application/Commands/Handlers/ReserveResourceHandler.cs:30`), driven by the `Role` field of the
correlation context this gateway builds (§3.14) — see
`component-internals/availability-service.md` §3.13.

**Extension procedure — add a new gated claim.**
1. Add the short-name → claim-URI entry under `auth.claims` in all four manifests.
2. Add `claims: {<shortName>: <value>}` to the route.
3. Confirm `identity-service` actually issues that claim; the gateway cannot mint claims and will
   simply reject every request if it does not.

**Failure modes.** A `claims:` block on a route that also has `auth: false` — the claim can never be
evaluated because there is no principal; this combination does not occur in the current manifests.
Value case-sensitivity: the manifests use lowercase `admin`, and `availability-service` compares
case-*insensitively* (`…Infrastructure/Contexts/IdentityContext.cs:29`), so the gateway is the
stricter of the two. Changing the issued role string to `Admin` would break the gateway's five
gates while leaving the downstream check working — a genuinely confusing split failure.

---

### 3.9 Token-subject binding (`@user_id`) and `bind:`

**Definition.** The mechanism by which the authenticated caller's identity, and URL path segments,
are injected into the outbound request — either into the downstream URL or into the message payload.

**Representation & storage.** Two distinct forms.

*(a) Inline in the `downstream:` string* — substitution into the proxied URL:

| Route | `downstream:` | Line |
| --- | --- | --- |
| `GET /customers/me` | `customers-service/customers/@user_id` | `ntrada.yml:143` |
| `GET /orders/` | `orders-service/orders?customerId=@user_id` | `ntrada.yml:298` |
| `GET /parcels/` | `parcels-service/parcels?customerId=@user_id` | `ntrada.yml:362` |
| `GET /pricing/` | `pricing-service/pricing?customerId=@user_id` | `ntrada.yml:406` |

*(b) A `bind:` list* — injection into the **request payload** as `name:value` pairs, where the value
is either `@user_id` or an `{upstream-token}`:

```yaml
bind:
  - resourceId:{resourceId}
  - customerId:@user_id
  - dateTime:{dateTime}
```
(`ntrada.yml:101-104`, `POST /availability/resources/{resourceId}/reservations/{dateTime}`)

**Lifecycle.** Applied per request, after auth, before dispatch.

**Invariants & enforcement — the security-critical property.** `@user_id` is resolved from the
validated token, **not from anything the client sends**. This is what makes
`POST /availability/resources/{resourceId}/reservations/{dateTime}` safe to expose without a claim
gate: the caller cannot reserve on behalf of someone else, because `customerId` is overwritten with
the token subject regardless of what the body contained. The same pattern secures order and parcel
creation (`ntrada.yml:316`, `386`).

Note the **binding is one-directional and unvalidated**: nothing checks that a `bind:` name
corresponds to a real property on the downstream command. A misspelled `customrId:@user_id` binds a
property nobody reads, and the *real* `customerId` then arrives as whatever the client sent (or
`Guid.Empty`). This is a **silent** authorization bypass in exactly the routes that depend on
binding for their safety. There is no test, schema or startup check anywhere in this repository that
would catch it.

Routes that bind `customerId:@user_id` (the ownership-critical set) in `ntrada.yml`:
`POST /availability/resources/{resourceId}/reservations/{dateTime}` (line 103),
`POST /customers/` (line 167), `POST /orders/` (line 316),
`POST /orders/{orderId}/parcels/{parcelId}` (line 330),
`POST /orders/{orderId}/vehicles/{vehicleId}` (line 348),
`POST /parcels/` (line 386).

The async manifest binds `customerId:@user_id` on strictly **more** routes, because the async
`DELETE` operations have no URL to carry it: e.g. `DELETE /orders/{orderId}`
(`ntrada-async.yml:372-373`) and `DELETE /parcels/{parcelId}` (`ntrada-async.yml:459-460`) bind it,
whereas their sync counterparts (`ntrada.yml:318-322`, `388-392`) bind **nothing at all** and rely
entirely on the downstream service to check ownership from the correlation context.

**Extension procedure.** Add `bind:` entries whose names exactly match (camelCase) the constructor
parameter names of the downstream command type. Verify against the command's source — for
Availability that is `…Application/Commands/*.cs`. If the value is a path token, the token must
appear in `upstream:`.

**Failure modes.** Misspelled bind name → **silent**; the value is dropped and the downstream sees
its default. `@user_id` on an `auth: false` route → no subject to bind. A bind that shadows a
client-supplied field always wins (that is the point); a bind that *should* shadow but is missing
means the client controls the field.

---

### 3.10 Payload template & JSON schema (`payload:` / `schema:`)

**Definition.** Ntrada's request-body validation and templating facility: `payload:` names a
template file describing the outbound body, `schema:` names a JSON-schema file used to validate the
inbound body before dispatch.

**Representation & storage.** Referenced on exactly **one route per manifest pair**:

| Manifest | Route | Keys | Line |
| --- | --- | --- | --- |
| `ntrada.yml` / `.docker.yml` | `POST /customers/` | `payload: create_customer`, `schema: create_customer.schema` | 169-170 |
| `ntrada-async.yml` / `.docker.yml` | `POST /customers/` | `payload: complete_customer_registration`, `schema: complete_customer_registration.schema` | 204-205 |

**These files do not exist in this repository.** A recursive search for `*payload*` and `*.schema*`
across the clone returns nothing; there is no `Payloads/` directory, and `Pacco.APIGateway.csproj`
copies only `ntrada*.yml` and `certs/**` to the output. So the four route-level references are
**dangling**.

**Lifecycle.** Would be loaded by Ntrada at startup or first use from a conventional directory. The
convention (`Payloads/<name>.json`) is Ntrada's, and is **`Unverifiable — Missing Source
Evidence`** — the package source is not in this workspace.

**Invariants & enforcement.** Because the referenced files are absent, one of two things is true at
runtime and this repository cannot tell you which: either Ntrada **fails loudly** at startup when it
cannot resolve `create_customer.schema`, or it **silently skips validation** for that route. This is
recorded as ABQ-2 (§8) and matters concretely: `POST /customers/` is the only route on the platform
with declared body validation, and it may be running with none.

**Extension procedure — add body validation to a route.**
1. Create the schema file in Ntrada's payloads directory (name it `<something>.schema`, matching the
   `schema:` value exactly).
2. Add a `<Content Include="Payloads\**"><CopyToOutputDirectory>Always</CopyToOutputDirectory>` item
   to `Pacco.APIGateway.csproj` — **otherwise the file is not deployed and behaves exactly like the
   two dangling references above.**
3. Add `schema: <name>.schema` to the route.
4. Repeat in all four manifests.

**Failure modes.** A dangling `schema:` reference is the failure mode currently present. A schema
that is stricter than the downstream contract rejects valid requests at the edge with no downstream
log line, making the rejection hard to trace.

---

### 3.11 Resource-ID generation (`resourceId:`)

**Definition.** A per-route instruction to mint an identifier at the edge, bind it into the request,
and surface it to the caller — so that a client can know the id of a thing it created even when the
write is asynchronous.

**Representation & storage.**

```yaml
resourceId:
  property: deliveryId
  generate: true
```
(`ntrada.yml:203-205`)

Routes using it (identical set in sync and async manifests):

| Route | `property` | `ntrada.yml` | `ntrada-async.yml` |
| --- | --- | --- | --- |
| `POST /deliveries/` | `deliveryId` | 203-205 | 238-240 |
| `POST /identity/sign-up` | `userId` | 259-261 | 304-306 |
| `POST /orders/` | `orderId` | 312-314 | 354-356 |
| `POST /parcels/` | `parcelId` | 382-384 | 441-443 |
| `POST /vehicles/` | `vehicleId` | 437-439 | 503-505 |

The generated value is exposed to the browser via the `Resource-ID` response header, which is in the
CORS `exposedHeaders` list (`ntrada.yml:39`), and is carried in the correlation envelope as
`ResourceId` (`Infrastructure/CorrelationContextBuilder.cs:48`).

**Lifecycle.** Generated per request by Ntrada; consumed by `CorrelationContextBuilder` and written
to the response header.

**Invariants & enforcement — the notable gap.** `POST /availability/resources` **does not use
`resourceId:`** in any manifest (`ntrada.yml:90-94`, `ntrada-async.yml:115-121`). It is the only
create-style route on the platform without edge id generation. The consequence is asymmetric by
mode:

- *Sync mode:* harmless. `AddResource`'s constructor substitutes a fresh `Guid` when the client
  sends `Guid.Empty` (`Pacco.Services.Availability/src/…Application/Commands/AddResource.cs:15`),
  and the service returns `Location: resources/{id}`
  (`Pacco.Services.Availability/src/…Api/Program.cs:43`). The caller learns the id from the header.
- *Async mode:* **the caller cannot learn the generated id at all.** There is no `Resource-ID`
  header, the 2xx carries no body, and `GET /operations/{operationId}` reports the operation, not
  the new resource. A client that wants a knowable id **must generate the GUID itself and send it in
  the body** — which is exactly what the service's own request sample does
  (`Pacco.Services.Availability/Pacco.Services.Availability.rest`, `POST {{url}}/resources` with an
  explicit `resourceId`). This is a real, undocumented client obligation.

**Extension procedure.** Add the `resourceId:` block with `property:` set to the downstream
command's id property name and `generate: true`. To close the gap above for
`POST /availability/resources`, add `resourceId: {property: resourceId, generate: true}` to that
route in all four manifests; the downstream command already accepts a caller-supplied
`ResourceId`, so this is backward-compatible.

**Failure modes.** `property:` not matching the downstream command's property → the generated id is
bound to nothing; the service mints its own, and the `Resource-ID` header the client trusts is
**wrong** — a silent, correlated-data-corruption failure, not an error.

---

### 3.12 Request ID

**Definition.** The per-request identifier the gateway manufactures and propagates; it is also the
platform's **operation id** in async mode.

**Representation & storage.** Enabled by `generateRequestId: true` (`ntrada.yml:16`). Surfaced to
callers as the `Request-ID` response header (CORS-exposed, `ntrada.yml:38`). Read in this
repository's own code at `Infrastructure/CorrelationContextBuilder.cs:40`
(`CorrelationId = executionData.RequestId`).

**Lifecycle.** Created by Ntrada per request; embedded in the correlation envelope; forwarded
downstream in the `Correlation-Context` header (§3.14); consumed by every Pacco service as
`IAppContext.RequestId` (for Availability:
`…Infrastructure/Contexts/AppContext.cs:15`, which sets `RequestId` from `context.CorrelationId`).

**Invariants & enforcement.** The gateway's `RequestId` becomes the *downstream* `RequestId`.
This is the join key that makes `operations-service` work: in async mode the client keeps the
`Request-ID` header from the 2xx and polls `GET /operations/{that id}`. If `generateRequestId` were
set to `false`, async mode would lose its only completion channel — the routes would still publish
and still return 2xx, and clients would simply never be able to observe an outcome. That is a
**silent, total** failure of the async contract driven by a single boolean.

**Extension procedure.** Nothing to extend; do not disable. If a client-supplied request id must be
honoured instead, that is an Ntrada-side capability not exercised anywhere in this repository —
**`Unverifiable — Missing Source Evidence`**.

**Failure modes.** Disabled → async completion tracking breaks silently. Not forwarded (i.e. if
`HttpRequestHook` were removed) → downstream services fall back to a locally-minted id
(`…Infrastructure/Contexts/AppContext.cs:11`), and correlation across the hop is lost with no error.

---

### 3.13 Trace ID

**Definition.** A distributed-tracing identifier generated at the edge, distinct from the OpenTracing
span context.

**Representation & storage.** `generateTraceId: true` (`ntrada.yml:17`); exposed via the `Trace-ID`
response header (`ntrada.yml:40`); copied into the envelope at
`Infrastructure/CorrelationContextBuilder.cs:49` (`TraceId = executionData.TraceId`).

**Lifecycle.** Per request; forwarded in the envelope; carried alongside — not instead of — the
Jaeger span context (§3.15).

**Invariants & enforcement.** None in this repository. Note that the downstream services'
`CorrelationContext` DTOs declare a `TraceId` property
(`Pacco.Services.Availability/src/…Infrastructure/Contexts/CorrelationContext.cs:12`) but no code in
`availability-service` reads it — it is carried and dropped. Treat `TraceId` as *available for*
correlation, not as *used for* it.

**Extension procedure / failure modes.** As §3.12; disabling it loses a header some client tooling
may key on, but breaks no in-platform mechanism that is visible in this workspace.

---

### 3.14 Correlation context envelope — the component's most load-bearing mechanism

**Definition.** A JSON document, built at the edge on every request, that carries the caller's
identity and request metadata to the downstream service. **This is how the entire Pacco platform
learns who the caller is.** No downstream service validates a JWT of its own — verified for
`availability-service`: a search for `AddJwt`/`AddAuth`/`Authorize` across its `src/` returns
nothing.

**Representation & storage.** Three files, all in this repository, all under 60 lines:

*Shape* — `src/Pacco.APIGateway/Infrastructure/CorrelationContext.cs`:

| Field | Type | Source |
| --- | --- | --- |
| `CorrelationId` | `string` | `executionData.RequestId` (`CorrelationContextBuilder.cs:40`) |
| `SpanContext` | `string` | active Jaeger span, or `""` (`:23-24`, `:53`) |
| `User.Id` | `string` | `executionData.UserId` (`:43`) |
| `User.Claims` | `IDictionary<string,string>` | `executionData.Claims` (`:44`) |
| `User.Role` | `string` | `Claims[ClaimTypes.Role]` (`:45`) |
| `User.IsAuthenticated` | `bool` | `!string.IsNullOrWhiteSpace(executionData.UserId)` (`:46`) |
| `ResourceId` | `string` | `executionData.ResourceId` (`:48`) |
| `TraceId` | `string` | `executionData.TraceId` (`:49`) |
| `ConnectionId` | `string` | `executionData.Context.Connection.Id` (`:50`) |
| `Name` | `string` | see below (`:26-36`, `:51`) |
| `CreatedAt` | `DateTime` | `DateTime.UtcNow` (`:52`) |

*Construction* — `CorrelationContextBuilder.Build(ExecutionData)` (`CorrelationContextBuilder.cs:20`),
registered as `IContextBuilder` at `Program.cs:41`.

*Transmission* — `HttpRequestHook.InvokeAsync` (`HttpRequestHook.cs:20-26`), registered as
`IHttpRequestHook` at `Program.cs:43`:

```csharp
var context = JsonConvert.SerializeObject(_contextBuilder.Build(data));
request.Headers.TryAddWithoutValidation("Correlation-Context", context);
```

The header name `Correlation-Context` is a **hard-coded string literal** in this file, and the
matching literal appears independently in each consuming service — for Availability at
`…Infrastructure/Extensions.cs:118`. There is no shared constant, no shared package and no test
binding the two. **This pair of string literals is the platform's authentication contract.**

*The `Name` field* is derived with a two-step fallback (`CorrelationContextBuilder.cs:26-36`):
if the route's `config` map contains a `routing_key`, `Name` is that routing key (so async requests
are named `add_resource`, `reserve_resource`, …); otherwise `Name` is
`"{Method} {Path}"`. In sync mode no route has a `config` block, so `Name` is always the
method+path form. This is the only field whose value differs by manifest mode.

**Lifecycle.**
- *Sync route:* `IHttpRequestHook` fires on the outbound `HttpRequestMessage`, so the envelope is
  attached to the proxied HTTP call.
- *Async route:* `IContextBuilder` is the RabbitMQ extension's hook
  (`using Ntrada.Extensions.RabbitMq;` at `CorrelationContextBuilder.cs:6`), so the same object is
  attached to the published message as the message context. The async manifests configure this at
  `ntrada-async.yml:76-78` — `messageContext: {enabled: true, header: message_context}`.
- Downstream, the envelope is rehydrated: `availability-service` reads either the broker message
  context (`ICorrelationContextAccessor`) or the HTTP header, at
  `…Infrastructure/Contexts/AppContextFactory.cs:19-33`.

**Invariants & enforcement — and the silent-bypass consequence.**
1. `IsAuthenticated` is computed at the edge as "the token yielded a non-blank subject". It is
   **asserted**, not proven, to the downstream — the downstream cannot re-verify it.
2. `Role` is read from the claims dictionary keyed by the full `ClaimTypes.Role` URI
   (`CorrelationContextBuilder.cs:45`). This is the same URI aliased at `ntrada.yml:5`. If the
   issuer changed the claim URI, `Role` would silently become `null`, `IsAdmin` downstream would
   become `false` (`…Infrastructure/Contexts/IdentityContext.cs:29-30` handles the null), the
   gateway's five admin gates would start rejecting, and every admin action on the platform would
   fail — **loudly at the gate, silently in the downstream authorization checks.**
3. **The bypass:** because there is no shared secret, signature or mutual TLS on the
   `Correlation-Context` header, *any* caller that can reach a downstream service's port directly
   can either omit the header (yielding `AppContext.Empty` → `IsAuthenticated: false`, which in
   `ReserveResourceHandler` **skips the ownership check entirely** — see
   `component-internals/availability-service.md` §3.13) or forge it with `IsAuthenticated: true` and
   any `Role`. The gateway is a perimeter, not a cryptographic boundary. `availability-service`
   partially mitigates this for *its own outbound* calls with Vault-issued client certificates
   (§3.18 of that document), but nothing protects its *inbound* surface.
4. Serialization uses `Newtonsoft.Json` defaults on both sides
   (`HttpRequestHook.cs:3`; `…Infrastructure/Extensions.cs:119`), and the consuming
   `CorrelationContext` class is a **hand-duplicated copy** of the gateway's, field for field.
   Adding a field here does not add it there.

**Extension procedure — add a field to the envelope (e.g. a tenant id).**
1. Add the property to `src/Pacco.APIGateway/Infrastructure/CorrelationContext.cs`.
2. Populate it in `CorrelationContextBuilder.Build` from `ExecutionData` (or from
   `executionData.Claims`).
3. **Add the identical property to every consuming service's own `CorrelationContext` class**
   (for Availability: `…Infrastructure/Contexts/CorrelationContext.cs`) — Newtonsoft **silently
   ignores** JSON members with no matching property, so a field added only at the gateway is
   transmitted and dropped with no error anywhere.
4. If the field must reach application code, also surface it on `IIdentityContext`/`IAppContext` and
   map it in that service's `IdentityContext`/`AppContext` constructors.
5. Note the extra hop on the async path: `AppContextFactory.Create()` serializes the broker's
   correlation context and **re-deserializes it into the local type**
   (`…Infrastructure/Contexts/AppContextFactory.cs:23-27`), so the field must round-trip through
   JSON twice.

**Failure modes.**
- Header absent → downstream `AppContext.Empty`, unauthenticated identity, **ownership checks
  skipped rather than denied**.
- Header present but malformed JSON → `JsonConvert.DeserializeObject` throws inside
  `Extensions.GetCorrelationContext`; the downstream request fails with the generic mapped error.
- Field renamed on one side only → silently `null`/default on the other.
- Header size: the envelope embeds the full claims dictionary, so a token with many claims produces
  a large header. Nothing truncates or caps it; a sufficiently large claim set would hit the
  downstream's header-size limit and fail the request at the HTTP layer.

---

### 3.15 Span context

**Definition.** The serialized OpenTracing/Jaeger span, propagated so that a downstream span becomes
a child of the gateway's span across both the HTTP and the RabbitMQ hop.

**Representation & storage.** `src/Pacco.APIGateway/Infrastructure/SpanContextBuilder.cs`,
registered as `ISpanContextBuilder` at `Program.cs:42`. It resolves `ITracer` from the service
provider **at call time** and defends against two nulls (`SpanContextBuilder.cs:21-22`):

```csharp
var tracer = _serviceProvider.GetService<ITracer>();
var spanContext = tracer is null ? string.Empty :
    tracer.ActiveSpan is null ? string.Empty : tracer.ActiveSpan.Context.ToString();
```

The identical three-line expression is duplicated in `CorrelationContextBuilder.cs:22-24` for the
`SpanContext` envelope field. Both are lazy `GetService` (nullable) rather than
`GetRequiredService`, so **tracing being switched off degrades to an empty string rather than
throwing** — a deliberate-looking, and correct, choice.

Tracing configuration lives at `extensions.tracing` (`ntrada.yml:58-64`): `serviceName: api-gateway`,
UDP to `localhost:6831` (`jaeger:6831` in the docker variants), `sampler: const`,
`useEmptyTracer: false`. On the async path the wire header is named by
`ntrada-async.yml:81` — `spanContextHeader: span_context`, which is exactly the value
`availability-service` defaults to and configures
(`…Infrastructure/Services/MessageBroker.cs:18`; `…Api/appsettings.json:151`).

**Lifecycle.** Built per request; embedded in the envelope; on async routes also emitted as the
`span_context` message header, which the consumer reads back at
`…Infrastructure/Extensions.cs:138-151` (`GetSpanContext`, which UTF-8 decodes a `byte[]` header).

**Invariants & enforcement.** `sampler: const` with the default parameter means **every** request is
sampled — there is no sampling rate to tune here, and no manifest sets one.
`useEmptyTracer: false` means a real tracer is constructed even when Jaeger is unreachable; UDP
emission is fire-and-forget, so an unreachable collector is silent. The consumer treats a missing or
non-`byte[]` header as `string.Empty` (`Extensions.cs:145-150`), so trace-context loss never fails a
request.

**Extension procedure.** Change `extensions.tracing.serviceName` to rename the gateway in Jaeger;
change `sampler` to reduce volume. To propagate over a *new* transport, implement the equivalent of
`SpanContextBuilder` for it and register it; the two existing builders are the template.

**Failure modes.** `spanContextHeader` mismatch between gateway (`span_context`) and a consumer →
traces silently break at the async hop, with no error and no gap indication other than orphan spans.
Jaeger unreachable → spans lost silently.

---

### 3.16 Async publish descriptor (`config.exchange` / `config.routing_key`)

**Definition.** For `use: rabbitmq` routes, the exchange to publish to and the routing key to
publish with — the async equivalent of `downstream:`.

**Representation & storage.** Per route in the async manifests:

```yaml
use: rabbitmq
config:
  exchange: availability
  routing_key: add_resource
```
(`ntrada-async.yml:118-121`)

The complete async write surface — **6 exchanges, 20 routing keys**:

| Exchange | Routing keys | Lines (`ntrada-async.yml`) |
| --- | --- | --- |
| `availability` | `add_resource`, `reserve_resource`, `release_resource`, `delete_resource` | 115-154 |
| `customers` | `complete_customer_registration`, `change_customer_state` | 195-218 |
| `deliveries` | `start_delivery`, `fail_delivery`, `complete_delivery`, `add_delivery_registration` | 235-274 |
| `orders` | `create_order`, `delete_order`, `add_parcel_to_order`, `delete_parcel_from_order`, `assign_vehicle_to_order` | 351-409 |
| `parcels` | `add_parcel`, `delete_parcel` | 438-460 |
| `vehicles` | `add_vehicle`, `update_vehicle`, `delete_vehicle` | 500-529 |

`identity`, `operations` and `pricing` have **no** async routes — sign-up/sign-in stay
`downstream:` even in async mode (`ntrada-async.yml:299-314`), which is necessary because the caller
needs the token back synchronously.

Broker connection settings are at `ntrada-async.yml:58-81`: `connectionName: api-gateway`,
port `5672`, vhost `/`, `guest`/`guest`, timeouts `3000` ms, heartbeat `60`, and
`exchange: {declareExchange: true, durable: true, autoDelete: false, type: topic}`.

**Lifecycle.** With `declareExchange: true` the gateway **declares** each exchange it publishes to
at connection/publish time, so the exchange exists even if no consumer has started. This is why a
mistyped exchange name does not fail: the gateway happily creates `availabilty`, publishes into it,
and the message dies unrouted.

**Invariants & enforcement.** Three separate name contracts must line up, none of them checked:

| Contract | Gateway side | Consumer side (Availability) |
| --- | --- | --- |
| Exchange name | `config.exchange: availability` | `rabbitMq.exchange.name: "availability"` (`…Api/appsettings.json:138`) |
| Exchange type | `type: topic` (`ntrada-async.yml:75`) | `type: "topic"` (`appsettings.json:137`) |
| Routing key | `routing_key: add_resource` | `SubscribeCommand<AddResource>()` + `conventionsCasing: "snakeCase"` (`…Infrastructure/Extensions.cs:107`; `appsettings.json:115`) |

A mismatch in any of the three produces the same symptom: **HTTP 2xx, message published, nothing
happens, no error anywhere.** The only observable is the RabbitMQ exchange's unroutable-message
counter.

**Extension procedure.** See §3.5. Additionally: if you rename a downstream command class you must
rename the `routing_key` in the async manifests in the same change, because the key is the
`snake_case` of the class name and nothing regenerates it.

**Failure modes.** As above. Also note the gateway publishes **directly at request time with no
outbox** — if the broker is down, `use: rabbitmq` routes fail at the edge. Unlike the downstream
services (which do have a transactional outbox), the gateway has no store-and-forward, so async mode
trades downstream durability for a *less* durable edge.

---

### 3.17 Response shaping

**Definition.** The two facilities for altering what the caller receives on a `downstream:` route.

**Representation & storage.**
- `onSuccess: {data: <path>}` — project a sub-document out of the downstream response. Used
  **once**: `GET /vehicles/` → `data: response.data.items` (`ntrada.yml:423-424`), which unwraps
  `vehicles-service`'s paged envelope so callers receive a bare array.
- `responseHeaders: {<name>: <value>}` — set literal response headers. Used **once**:
  `POST /identity/sign-in` → `content-type: application/json` (`ntrada.yml:268-269`).

**Lifecycle.** Applied per response, inside Ntrada.

**Invariants & enforcement.** The `onSuccess.data` path is a string; nothing validates it against
the downstream's actual response shape. If `vehicles-service` stopped returning `data.items`, the
projection would yield null/empty and the gateway would return an empty success — **silent**, and it
would look to the client exactly like "there are no vehicles".

The single `responseHeaders` use is a workaround for `sign-in` returning a body whose content type
the proxy does not otherwise set correctly; it is a symptom worth preserving in any rewrite.

**Extension procedure.** Add `onSuccess.data` with a dotted path rooted at `response`. Because the
projection is silent on mismatch, pair any such change with a consumer-driven contract test — see
`patterns/testing/consumer-driven-contract-test-pair.md`.

**Failure modes.** Path mismatch → empty body, 2xx. Not available for `use: rabbitmq` routes (there
is no downstream response to shape).

---

### 3.18 CORS policy

**Definition.** The browser-facing cross-origin policy for the whole gateway.

**Representation & storage.** `extensions.cors` (`ntrada.yml:27-41`), served by
`Ntrada.Extensions.Cors` (`Pacco.APIGateway.csproj:15`):

```yaml
allowCredentials: true
allowedOrigins: ['*']
allowedMethods: [post, put, delete]
allowedHeaders: ['*']
exposedHeaders: [Request-ID, Resource-ID, Trace-ID, Total-Count]
```

**Lifecycle.** Static, identical in all four manifests.

**Invariants & enforcement — two things a reader must know.**
1. `allowCredentials: true` combined with `allowedOrigins: ['*']` is the combination the CORS
   specification forbids: a compliant browser rejects a wildcard origin on a credentialed request.
   Depending on how `Ntrada.Extensions.Cors` materialises the wildcard (echoing the request origin
   versus emitting a literal `*`), this is either **an accidental allow-any-origin-with-credentials
   policy** or **a policy that silently fails for every credentialed cross-origin call**. Which one
   is **`Unverifiable — Missing Source Evidence`** (the extension's source is not in this
   workspace), but both outcomes are defects and the resolution is the same: name real origins.
2. `allowedMethods` lists only `post`, `put`, `delete` — **`get` is absent.** Every read route on the
   platform is a `GET`. Simple `GET` requests are not preflighted so they generally still work; a
   `GET` that becomes non-simple (e.g. because a client adds a custom header) would be preflighted
   and **rejected**. This is a latent, hard-to-diagnose browser-only failure.

`exposedHeaders` is the counterpart of §3.11/§3.12/§3.13 — without it a browser client cannot read
`Request-ID`/`Resource-ID` at all, which would break async completion tracking from the browser.
`Total-Count` is exposed for the paged services (`vehicles`, `parcels`) even though no route in this
repository sets it.

**Extension procedure.** Replace `allowedOrigins: ['*']` with the actual client origins (see ABQ-3 —
`Pacco.Web` is an empty repository, so the intended origin is unknown), and add `get` to
`allowedMethods`. Any new response header a client must read has to be added to `exposedHeaders` in
all four manifests.

**Failure modes.** As above; all browser-side and invisible in server logs.

---

### 3.19 JWT validation parameters

**Definition.** The material and rules used to validate the bearer token.

**Representation & storage.** `extensions.jwt` (`ntrada.yml:43-48`), served by
`Ntrada.Extensions.Jwt` (`Pacco.APIGateway.csproj:17`):

| Key | Value |
| --- | --- |
| `issuerSigningKey` | an 80-character symmetric key, **committed in plaintext in all four manifests** |
| `validIssuer` | `pacco` |
| `validateAudience` | `false` |
| `validateIssuer` | `true` |
| `validateLifetime` | `true` |

**Lifecycle.** Loaded with the manifest at startup. There is no JWKS endpoint, no key rotation
mechanism and no `kid` handling: **key rotation requires editing four files and restarting.**

**Invariants & enforcement.**
- Symmetric (HMAC) signing means the gateway holds the **signing** key, not just a verification key
  — it could mint tokens. The same literal key is also committed in `Pacco.Services.Operations`'
  `appsettings.json` (per `baselines/service-summaries.md` G13), so the secret is replicated across
  repositories.
- `validateAudience: false` — any audience is accepted.
- **The gateway and `availability-service` do not share validation material.** The service's
  `appsettings.json` declares `jwt.certificate.location: certs/localhost.cer` (an *asymmetric*
  certificate) — but see the finding in `component-internals/availability-service.md` §3.13: no
  `AddJwt()` call exists in that service, so that entire `jwt` block is **inert configuration**. The
  practical consequence is that JWT validation happens **only** at the gateway, exactly once, and
  everything downstream trusts §3.14's header.
- `certs/localhost.cer` is present in this repository too
  (`src/Pacco.APIGateway/certs/localhost.cer`, copied by `Pacco.APIGateway.csproj:41`) but **no
  manifest key references it** — it is a dead artifact on the gateway side.

**Extension procedure — move to asymmetric keys.** Change `issuerSigningKey` to the appropriate
certificate/JWKS configuration in all four manifests **and** change `identity-service`'s signing
configuration in the same change; the two are only coupled by this shared literal, so a one-sided
change rejects every token immediately (loudly, at least).

**Failure modes.** Key mismatch with the issuer → every authenticated route returns 401; loud and
immediate. Committed key compromise → an attacker can mint admin tokens; see
`baselines/service-summaries.md` B1, which is an open action item, not a resolved one.

---

### 3.20 HTTP retry policy

**Definition.** The gateway's retry behaviour on `downstream:` calls.

**Representation & storage.** Top-level `http:` block (`ntrada.yml:7-10`), identical in all four
manifests: `retries: 2`, `interval: 2.0`, `exponential: true`.

**Lifecycle.** Applied per downstream call by Ntrada.

**Invariants & enforcement.** This is a **global** policy — there is no per-route override anywhere
in the manifests. The critical property: **it applies to non-idempotent methods too.** Every
`POST`/`PUT`/`DELETE` in `ntrada.yml` is a `downstream:` route and is therefore subject to up to two
retries at ~2 s and ~4 s. Whether a retry fires on a connection failure only, or also on a 5xx
response, is **`Unverifiable — Missing Source Evidence`**; if it is the latter, a downstream that
processed a write and then failed to respond will be **called again**, producing duplicates.

The downstream mitigation is uneven and does **not** cover this path: `availability-service`'s
outbox/inbox de-duplication keys on the broker `MessageId`
(`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:26-29`), and on the HTTP path there is
no `MessageId`, so a fresh GUID is minted per invocation and **de-duplication does not apply**. A
retried `POST /availability/resources` is protected only by that service's own
`ExistsAsync` check (`…Application/Commands/Handlers/AddResourceHandler.cs:23`) — which works
because the id is client-supplied and stable. A retried
`POST /resources/{id}/reservations/{dateTime}` is *not* protected: the second attempt hits
`CannotExpropriateReservationException` (equal priority is treated as colliding —
`…Core/Entities/Resource.cs:63`) and returns a 400 for a reservation that in fact succeeded.

Worst case latency added: `2.0 + 4.0 = 6 s` on top of the downstream's own timeouts. No total request
timeout is configured anywhere.

**Extension procedure.** Adjust `retries`/`interval`/`exponential` in all four manifests. There is
no mechanism to exempt non-idempotent routes; if that is required it must be added upstream of
Ntrada or the value must go to `0`.

**Failure modes.** Duplicate writes on retried non-idempotent calls (above). Retry amplification
during a downstream brown-out: each client request becomes three downstream requests, exactly when
the downstream is least able to serve them. No circuit breaker exists to stop this.

---

### 3.21 Swagger surface

**Definition.** The aggregated API documentation the gateway publishes for the whole platform.

**Representation & storage.** `extensions.swagger` (`ntrada.yml:50-56`), served by
`Ntrada.Extensions.Swagger` (`Pacco.APIGateway.csproj:19`): `name: Pacco`, `title: Pacco API`,
`version: v1`, `routePrefix: docs`, `reDocEnabled: false`, `includeSecurity: true`.

**Lifecycle.** Generated **from the manifest**, at startup. It is therefore always consistent with
the routing table by construction — a genuine strength of the declarative design, and the reason
route documentation cannot drift here the way it can in the code-first downstream services.

**Invariants & enforcement.** `routePrefix: docs` means `GET /docs`. That path is **not** an
`auth: true` route — it is served by the extension, outside the module/route table, so the manifest's
auth model does not apply to it. The gateway's complete route topology, including every admin-gated
route and every downstream service name, is therefore **publicly readable** wherever the gateway is
reachable. `includeSecurity: true` adds the bearer-auth scheme to the document; it does not protect
the document.

**Extension procedure.** Change `routePrefix` to relocate it. To restrict access it must be handled
outside the manifest (reverse proxy / network policy), because no Ntrada key in this repository gates
it.

**Failure modes.** Publicly exposed topology (above). Because it is manifest-derived, request/response
*bodies* are not documented at all — only paths, methods and auth. A consumer cannot learn the shape
of `POST /resources` from `/docs`.

---

### 3.22 Host observability (logging, metrics, tracing)

**Definition.** The non-Ntrada cross-cutting services the ASP.NET Core host registers.

**Representation & storage.** `Program.cs:44-48`:

```csharp
.AddConvey()
.AddMetrics()
.AddSecurity())
.Configure(app => app.UseNtrada())
.UseLogging();
```

Configuration lives in `appsettings*.json` (**not** in the manifests) and is therefore selected by
`ASPNETCORE_ENVIRONMENT`, independently of `NTRADA_CONFIG`:

| Setting | `appsettings.json` (base) | `appsettings.local.json` | `appsettings.docker.json` |
| --- | --- | --- | --- |
| `logger.excludePaths` | `["/", "/ping", "/metrics"]` | inherited | inherited |
| `logger.console.enabled` | `true` | inherited | `true` |
| `logger.file.enabled` | `true` (`logs/logs.txt`, daily) | `false` | `false` |
| `logger.seq.enabled` | `true` (`http://localhost:5341`) | `false` | `true` (`http://seq:5341`) |
| `metrics.enabled` | `true` | `false` | `true` |
| `metrics.prometheusEnabled` | `true` | `false` | `true` |
| `metrics.influxEnabled` | `false` | — | `false` |
| `metrics.env` | `local` | — | `docker` |
| `metrics.interval` | `5` | — | `5` |

`ASPNETCORE_ENVIRONMENT` is `local` from both start scripts (`scripts/start.sh:2`,
`scripts/start-async.sh:2`) and `docker` from the image (`Dockerfile:10`).
`appsettings.development.json` is an empty object.

**Lifecycle.** Registered at startup; `UseLogging()` (Convey.Logging) wires Serilog sinks;
`AddMetrics()` (Convey.Metrics.AppMetrics) exposes `/metrics` for Prometheus.

**Invariants & enforcement.**
- **Two independent environment switches govern one process.** `ASPNETCORE_ENVIRONMENT` selects
  logging/metrics; `NTRADA_CONFIG` selects routing. They are not coupled, so
  `ASPNETCORE_ENVIRONMENT=docker` with `NTRADA_CONFIG=ntrada.yml` is a reachable, valid, and
  entirely wrong configuration: docker-style logging with `localhost` downstream URLs. Nothing warns.
- `logger.excludePaths` suppresses access logs for `/`, `/ping` and `/metrics` — note `/docs` is
  **not** excluded, so documentation scraping is logged.
- `metrics.enabled: false` in `local` means the `/metrics` endpoint is not useful during local
  development.
- Unlike `availability-service`, the gateway's logger config has **no `excludeProperties`
  redaction list** (contrast `Pacco.Services.Availability/src/…Api/appsettings.json:37-50`, which
  redacts `Password`, `Token`, `Secret`, `Email`, …). Since the gateway is the component that
  handles raw sign-in bodies and bearer tokens, this asymmetry is worth flagging: whether Convey
  redacts by default is **`Unverifiable — Missing Source Evidence`**, but the *configuration* to do
  so is present downstream and absent here.
- `logger.seq.apiKey: "secret"` is a committed literal in both `appsettings.json` and
  `appsettings.docker.json`.

**Extension procedure — add redaction.** Copy the `logger.excludeProperties` array from
`Pacco.Services.Availability/src/…Api/appsettings.json:37-50` into this repository's
`appsettings.json`. Add new sinks under `logger.*` per environment file. To add a metric, note the
gateway has **no custom metrics code at all** — unlike `availability-service`, which has
`CustomMetricsMiddleware`; anything beyond Convey's defaults would need new middleware here.

**Failure modes.** Seq/Prometheus unreachable → sink errors, no request impact. Mismatched
`ASPNETCORE_ENVIRONMENT`/`NTRADA_CONFIG` → wrong downstream addresses, discovered only by connection
failures at request time.

---

## 4. Primary control flows

Only two code paths exist in this component's own C# — `CorrelationContextBuilder.Build` and
`HttpRequestHook.InvokeAsync`. Everything else is Ntrada executing the manifest. The flows below
therefore mark each step as **[repo]** (verifiable here) or **[ntrada]** (behaviour inferred from the
manifest's shape and from what downstream services provably receive).

### 4.1 Startup

```
scripts/start.sh:1-3  or  Dockerfile:10-11  or  Pacco/compose/services.yml
  ├─ sets ASPNETCORE_ENVIRONMENT (local | docker)          → selects appsettings*.json
  └─ sets NTRADA_CONFIG (unset | ntrada-async | ntrada.docker | ntrada-async.docker)
Program.Main                                                [repo] Program.cs:20
  └─ CreateDefaultBuilder(args)
     └─ ConfigureAppConfiguration                           [repo] Program.cs:26-39
        ├─ resolve configPath: args[0] ?? NTRADA_CONFIG ?? "ntrada.yml"   (:30)
        ├─ append ".yml" if absent                                        (:32-35)
        └─ builder.AddYamlFile(configPath, optional:false)                (:38)
              ▲ throws FileNotFoundException here if missing — HARD FAIL
     └─ ConfigureServices                                   [repo] Program.cs:40-46
        ├─ AddNtrada()                                      [ntrada] binds the whole manifest
        ├─ AddSingleton<IContextBuilder, CorrelationContextBuilder>()     (:41)
        ├─ AddSingleton<ISpanContextBuilder, SpanContextBuilder>()        (:42)
        ├─ AddSingleton<IHttpRequestHook, HttpRequestHook>()              (:43)
        └─ AddConvey().AddMetrics().AddSecurity()                         (:44-46)
     └─ Configure(app => app.UseNtrada())                   [repo] Program.cs:47
     └─ UseLogging()                                        [repo] Program.cs:48
```

Three registrations at `Program.cs:41-43` are the entire extensibility contribution of this
repository: **one hook for the sync transport, one context builder shared by both transports, one
span builder.** Everything a Pacco service knows about its caller originates in those three
singletons.

### 4.2 Synchronous read — `GET /availability/resources/{resourceId}`

Manifest: `ntrada.yml:96-99` (`auth: true`, `downstream: availability-service/resources/{resourceId}`).

```
Kestrel
 └─ Ntrada pipeline                                              [ntrada]
    ├─ CORS (extensions.cors, ntrada.yml:27-41)
    ├─ route match: module "availability" (path: availability) + upstream "/resources/{resourceId}"
    ├─ auth: true → validate bearer against extensions.jwt (:43-48)
    │     ▲ 401 here; nothing below runs, no correlation context is built
    ├─ generateRequestId / generateTraceId  (:16-17)
    ├─ resolve downstream host: useLocalUrl:true → localhost:5001 (:124)
    └─ build HttpRequestMessage → invoke IHttpRequestHook
        └─ HttpRequestHook.InvokeAsync                        [repo] HttpRequestHook.cs:20
           ├─ _contextBuilder.Build(data)                     [repo] CorrelationContextBuilder.cs:20
           │    ├─ GetService<ITracer>() → ActiveSpan?.Context.ToString() ?? ""   (:22-24)
           │    ├─ Name: route.Config["routing_key"] ?? "GET /resources/{id}"     (:26-36)
           │    └─ new CorrelationContext{ CorrelationId=RequestId, User{Id,Claims,
           │         Role=Claims[ClaimTypes.Role], IsAuthenticated=UserId!=""},
           │         ResourceId, TraceId, ConnectionId, Name, CreatedAt=UtcNow,
           │         SpanContext }                                                (:38-54)
           ├─ JsonConvert.SerializeObject(...)                                    (:22)
           └─ request.Headers.TryAddWithoutValidation("Correlation-Context", json) (:24)
    ├─ send, with http retries: 2, interval 2.0, exponential (:7-10)
    └─ relay downstream status + body; add Request-ID / Trace-ID response headers
─────────────────────── network hop ───────────────────────
availability-service   [read-only clone, cross-referenced]
 └─ UseDispatcherEndpoints Get<GetResource, ResourceDto>       …Api/Program.cs:38
    └─ AppContextFactory.Create() reads the header             …Infrastructure/Contexts/AppContextFactory.cs
    └─ GetResourceHandler → ResourcesMongoRepository → mongo `resources`
    └─ 200 with body, or 404 when the handler returns null (dispatcher convention)
```

**Note the failure asymmetry at the last step.** `availability-service`'s
`ExceptionToResponseMapper` maps *every* mapped exception to **400** and every unmapped one to a
generic **400** — so the only non-400 error the gateway can ever relay from that service is the
dispatcher's own 404. The gateway does not translate status codes; whatever the service returns is
what the client sees.

### 4.3 Synchronous write — `POST /availability/resources/{resourceId}/reservations/{dateTime}`

Manifest: `ntrada.yml:100-105`.

```
… auth + request-id as above …
 ├─ bind:                                                       [ntrada] ntrada.yml:101-104
 │    resourceId:{resourceId}   ← from the upstream path
 │    customerId:@user_id       ← FROM THE VALIDATED TOKEN, overwriting any client value
 │    dateTime:{dateTime}       ← from the upstream path
 ├─ HttpRequestHook attaches Correlation-Context                 [repo]
 └─ POST http://localhost:5001/resources/{resourceId}/reservations  (retries ×2)
─────────────────────── network hop ───────────────────────
availability-service
 └─ Post<ReserveResource>  …Api/Program.cs:41
    └─ ReserveResourceHandler                …Application/Commands/Handlers/ReserveResourceHandler.cs
       ├─ identity guard (:29-33) — SKIPPED ENTIRELY when identity.IsAuthenticated is false
       ├─ customers-service GET /customers/{id}/state via CustomersServiceClient
       ├─ Resource.AddReservation → priority/expropriation rules  …Core/Entities/Resource.cs:57-72
       ├─ ResourcesMongoRepository.UpdateAsync — optimistic concurrency on Version
       └─ EventProcessor.ProcessAsync → EventMapper → RabbitMQ
```

The `customerId:@user_id` bind at `ntrada.yml:103` is the **only** thing preventing a caller from
reserving on another customer's behalf on this route, because the downstream guard compares
`identity.Id != command.CustomerId` — and that guard is itself conditional on `IsAuthenticated`.
Two independent, unchecked couplings (the bind name, and the header) hold this together.

### 4.4 Asynchronous write — the same route in async mode

Manifest: `ntrada-async.yml:128-136` (`use: rabbitmq`, `exchange: availability`,
`routing_key: reserve_resource`).

```
… auth + bind (identical) …
 └─ Ntrada.Extensions.RabbitMq                                   [ntrada]
    ├─ IContextBuilder.Build(...) → message context               [repo] CorrelationContextBuilder.cs:20
    │     (Name is now "reserve_resource", from route.Config["routing_key"])
    ├─ ISpanContextBuilder.Build(...) → span_context header       [repo] SpanContextBuilder.cs:18
    ├─ declare exchange "availability" (topic, durable)           ntrada-async.yml:72-75
    ├─ basic_publish(exchange="availability", routing_key="reserve_resource",
    │                headers={message_context: <json>, span_context: <string>})
    └─ return 2xx IMMEDIATELY — no domain result, no downstream status
─────────────────────── broker hop ───────────────────────
availability-service
 └─ SubscribeCommand<ReserveResource>()          …Infrastructure/Extensions.cs:108
    (queue "availability-service/availability.reserve_resource", snakeCase conventions)
 └─ inbox dedupe → handler → outbox → integration event or rejected event
```

The client's only completion channel is `GET /operations/{operationId}` keyed on the `Request-ID`
header — and that route is `auth: false` (`ntrada-async.yml:330`). If the message is unroutable
(bad exchange or routing key), the publish still succeeds, the client still gets 2xx, and
`operations-service` will never be told anything: the operation stays pending forever. **There is no
timeout, dead-letter or alert for this in the gateway.**

### 4.5 Constant route — `GET /`

`ntrada.yml:66-72`: `use: return_value`, `returnValue: "Welcome to Pacco API!"`
(async: `"Welcome to Pacco API [async]!"`, `ntrada-async.yml:72`). No auth, no hook, no downstream.
This string is the **only** runtime signal distinguishing sync from async mode from outside the
process.

---

## 5. Persistence & schema evolution

**This component owns no datastore.** There is no database client package in
`Pacco.APIGateway.csproj`, no repository type, and no `Data`/`Persistence` directory. Nothing it
computes survives the request. The `logs/logs.txt` rolling file (`appsettings.json`,
`logger.file`) is the only thing it writes to disk, and it is diagnostic only.

What it *does* have is **configuration schema**, and that schema evolves in four places at once:

| "Schema" | Where | How it changes | What breaks on drift |
| --- | --- | --- | --- |
| Route table | `ntrada*.yml` × 4 | manual edit + restart | endpoint 404s in one mode only |
| Correlation envelope | `Infrastructure/CorrelationContext.cs` **+ a hand-copied class in every service** | manual edit in N repositories | field silently `null` downstream |
| Async routing keys | `ntrada-async*.yml` `config.routing_key` | must equal `snake_case(CommandTypeName)` | message silently unrouted |
| JWT signing key | `extensions.jwt.issuerSigningKey` × 4 + `identity-service` | manual, no rotation mechanism | every request 401 |

**There is no versioning on any of these.** The correlation envelope in particular is an unversioned
wire contract duplicated across at least two repositories (verified: the gateway's
`Infrastructure/CorrelationContext.cs` and Availability's
`…Infrastructure/Contexts/CorrelationContext.cs` are field-for-field identical). The safe evolution
rule is therefore **additive only**:

1. Add a field at the gateway. Old services ignore it (Newtonsoft drops unknown members silently).
2. Deploy every consumer that needs it, adding the same property.
3. Never rename or retype an existing field — a rename is a *silent* break on both transports, with
   no error at either end.
4. Never remove a field until every consumer has stopped reading it, and note that "reading it" is
   not greppable from this repository.

For migrating a route between transports, see §3.5's four-step procedure — the manifest change alone
is insufficient, because it also changes the endpoint's observable contract.

---

## 6. Surface → internals map

`R` = read-only (no state change anywhere), `W` = mutating (causes a downstream write),
`—` = present in this manifest but resolved without leaving the process.

### 6.1 HTTP surface (`ntrada.yml`, sync mode) — 41 routes

| Method + upstream (effective path) | Auth | Claims | Mode | Internals reached | Line |
| --- | --- | --- | --- | --- | --- |
| `GET /` | public | — | — | `return_value` | 69-72 |
| `GET /availability/resources` | ✔ | — | R | availability-service `GET /resources` | 90-94 |
| `POST /availability/resources` | ✔ | — | W | `AddResource` — **no `resourceId:` generation** | 90-94 |
| `GET /availability/resources/{resourceId}` | ✔ | — | R | `GetResource` | 96-99 |
| `POST /availability/resources/{id}/reservations/{dateTime}` | ✔ | — | W | `ReserveResource`, binds `customerId:@user_id` | 100-105 |
| `DELETE /availability/resources/{id}/reservations/{dateTime}` | ✔ | — | W | `ReleaseResourceReservation` — **silent no-op if absent** | 106-110 |
| `DELETE /availability/resources/{resourceId}` | ✔ | — | W | `DeleteResource` — **no authz downstream** | 111-115 |
| `GET /customers/` | ✔ | `role:admin` | R | customers-service | 136-138 |
| `GET /customers/me` | ✔ | — | R | `customers/@user_id` | 140-144 |
| `GET /customers/{customerId}` | ✔ | `role:admin` | R | customers-service | 150-152 |
| `GET /customers/{customerId}/state` | ✔ | `role:admin` | R | customers-service | 158-160 |
| `POST /customers/` | ✔ | — | W | binds `customerId:@user_id`; **dangling `payload`/`schema`** | 165-171 |
| `PUT /customers/{customerId}/state/{state}` | ✔ | `role:admin` | W | customers-service | 176-181 |
| `GET /deliveries/…` (2) | ✔ | — | R | deliveries-service | 190-201 |
| `POST /deliveries/` | ✔ | — | W | `resourceId: deliveryId, generate:true` | 202-206 |
| `POST /deliveries/{id}/…` (2) | ✔ | — | W | deliveries-service | 208-224 |
| `POST /identity/sign-up` | **public** | — | W | `resourceId: userId, generate:true` | 256-261 |
| `POST /identity/sign-in` | **public** | — | W | sets `content-type` response header | 263-269 |
| `GET /identity/me` | ✔ | — | R | identity-service | 238-241 |
| `GET /identity/users/{userId}` | ✔ | `role:admin` | R | identity-service | 243-246 |
| `GET /operations/{operationId}` | **public** | — | R | **the async completion channel** | 280-286 |
| `GET /orders/…` (3) | ✔ | — | R | orders-service; `?customerId=@user_id` | 296-310 |
| `POST /orders/` + 2 sub-resources | ✔ | — | W | binds `customerId:@user_id` | 311-350 |
| `DELETE /orders/{orderId}` | ✔ | — | W | **binds nothing** — downstream authz only | 318-322 |
| `GET /parcels/…` (2) | ✔ | — | R | `?customerId=@user_id` | 360-378 |
| `POST /parcels/` | ✔ | — | W | `resourceId: parcelId`; binds `customerId` | 380-386 |
| `DELETE /parcels/{parcelId}` | ✔ | — | W | **binds nothing** | 388-392 |
| `GET /pricing/` | ✔ | — | R | `?customerId=@user_id` | 404-407 |
| `GET /vehicles/…` (2) | ✔ | — | R | `onSuccess: response.data.items` on the list | 418-434 |
| `POST /vehicles/` | ✔ | — | W | `resourceId: vehicleId` | 436-440 |
| `PUT /vehicles/{id}` · `DELETE /vehicles/{id}` | ✔ | — | W | vehicles-service | 442-452 |

### 6.2 Non-route surface

| Surface | Auth | Mode | Internals | Evidence |
| --- | --- | --- | --- | --- |
| `GET /docs` | **none** | R | Swagger generated from the manifest — exposes the full topology | `ntrada.yml:50-56` |
| `GET /metrics` | **none** | R | App.Metrics/Prometheus scrape | `Program.cs:45`; `appsettings.json` `metrics.prometheusEnabled` |
| `GET /ping` | **none** | R | excluded from access logs | `appsettings.json` `logger.excludePaths` |

### 6.3 Absent surfaces (asked for often; genuinely not here)

| Expected | Reality |
| --- | --- |
| Rate limiting / throttling | **Absent.** No key or package. |
| Circuit breaker / bulkhead | **Absent.** Only blind retries (§3.20). |
| Request/response caching | **Absent.** |
| Response aggregation / BFF composition | **Absent.** One upstream → one downstream, always. |
| Service discovery (Consul) | **Absent** from the gateway, though every downstream service registers. Addresses are hard-coded. |
| Load balancing (Fabio) | **Configured but `enabled: false`** in all four manifests. |
| Health check endpoint | **`Unverifiable — Missing Source Evidence`** — `/ping` is referenced only in `logger.excludePaths`; no route defines it, so it is Ntrada-provided or nonexistent. |
| Request body size limits | **Absent** from configuration; Kestrel defaults apply. |
| Outbound message durability (async mode) | **Absent.** Direct publish, no outbox — unlike every downstream service. |
| Audit log of admin actions | **Absent.** |

---

## 7. Change/extension guide

Ordered by how often it is actually needed.

**7.1 Add an endpoint.**
Edit `modules.<m>.routes` in **all four** manifests. Set `auth:` explicitly (§3.7 — omission means
public). For writes, add `bind:` entries matching the downstream command's parameter names exactly
(§3.9), and in the async manifests supply `config.exchange` + `config.routing_key` where the key is
`snake_case(CommandTypeName)` (§3.16). Restart. Verify in *both* modes — a route that works in sync
mode proves nothing about async mode.

**7.2 Add a downstream service.**
Add `modules.<new>.services.<key>` with `localUrl` (from that service's `launchSettings.json`) and
`url` (its compose `container_name`), in all four manifests (§3.6). Confirm the port is not already
taken in the 5001-5009 block.

**7.3 Change who may call something.**
Prefer `claims:` at the gateway for coarse role gates (§3.8); prefer the downstream handler for
ownership rules, because the gateway cannot see the resource. Remember the downstream ownership
check in Availability is skipped when `IsAuthenticated` is false (§3.14), so an `auth: false` route
into a write handler is a genuine hole — never combine them.

**7.4 Add a field to the correlation envelope.**
Follow §3.14's five steps. This is a **multi-repository change**; the gateway-only half compiles,
deploys and silently does nothing.

**7.5 Convert routes between sync and async.**
§3.5. Treat it as a **breaking API change** even though no path or method changes, because the
response body semantics change from "the result" to "nothing".

**7.6 Rotate the JWT signing key.**
Four manifests here plus `identity-service` plus any other holder of the literal, atomically. There
is no rotation window; every in-flight token is invalidated. See §3.19 and
`baselines/service-summaries.md` B1.

**7.7 Fix the known configuration defects.** In rough priority order:
1. Move `extensions.jwt.issuerSigningKey` and `logger.seq.apiKey` out of source into environment
   variables or Vault (the downstream services already use Vault; the gateway references none).
2. Replace `cors.allowedOrigins: ['*']` with real origins, and add `get` to `allowedMethods`
   (§3.18).
3. Add `resourceId: {property: resourceId, generate: true}` to `POST /availability/resources` so
   async callers can learn the id (§3.11).
4. Create the missing `create_customer`/`complete_customer_registration` payload+schema files, or
   remove the dangling references (§3.10).
5. Set `customErrors.includeExceptionMessage: false` for non-local manifests
   (`ntrada.yml:23-25` — it is `true` in all four, which leaks downstream exception text to
   callers).

**7.8 Things you cannot do without leaving this component.**
Add caching, rate limiting, circuit breaking, aggregation, or per-route retry policy. All of these
would require either a new Ntrada extension package (source not in this workspace) or middleware
registered around `app.UseNtrada()` at `Program.cs:47` — which is the only extension point the host
offers.

**7.9 Testing.** There is no test project in this repository. `.travis.yml` runs `scripts/build.sh`
and `scripts/test.sh`; `test.sh` executes `dotnet test` over a solution that contains no test
project, so **the pipeline is green by vacuity**. Any behavioural change here is verified only by
running the platform. That is the single largest risk in changing this component.

---

## 8. Assumptions, Blockers & Open Questions (ABQ)

### Assumptions

| # | Assumption | Basis | Falsifiable by |
| --- | --- | --- | --- |
| A-1 | Ntrada composes the effective path as `/{module.path}{route.upstream}` | Every `downstream:` value in the manifests is consistent with it, and the downstream services' own routes match | Reading `Ntrada` source |
| A-2 | `use: rabbitmq` publishes exactly one message per request, synchronously, with no local buffering | The manifest has no outbox/retry keys, and the gateway has no persistence | Reading `Ntrada.Extensions.RabbitMq` |
| A-3 | `@user_id` resolves from the validated token's subject, not from any request-supplied value | It is used as the sole ownership control on 6 routes and the platform's authorization depends on it | Reading Ntrada's `ExecutionData` population |
| A-4 | The retry policy at `ntrada.yml:7-10` applies to all `downstream:` calls including non-idempotent ones | It is a top-level block with no method filter and no per-route override | Reading Ntrada's HTTP client construction |
| A-5 | The deployed platform runs `ntrada-async.docker.yml` (per the sibling `Pacco/compose/services.yml`), not the image default `ntrada.docker.yml` | Composition file in a read-only sibling repo | Inspecting the actual deployment |

### Blockers

| # | Blocker | Impact on this model |
| --- | --- | --- |
| B-1 | **Ntrada source is not in this workspace.** `Ntrada 0.4.*` and its six extension packages are NuGet references (`Pacco.APIGateway.csproj:14-19`) with no source | Everything marked `[ntrada]` in §4 and every `Unverifiable — Missing Source Evidence` note in §3 stems from this. Route matching precedence, schema-file resolution, CORS wildcard handling, retry trigger conditions, and `/ping` cannot be settled from this repository. |
| B-2 | **No tests of any kind.** No test project, no fixtures, no contract tests | No executable specification of gateway behaviour exists; §7's procedures are verified only by manual platform runs. |
| B-3 | **Committed secrets.** JWT signing key in four manifests; `seq.apiKey` in two appsettings files | Any correction is a coordinated multi-repository change; see §7.6. Also `baselines/service-summaries.md` B1. |

### Open questions

| # | Question | Why it matters | How to answer |
| --- | --- | --- | --- |
| Q-1 | Do the `create_customer` / `complete_customer_registration` payload+schema files exist in a deployment artifact outside this repository, or is `POST /customers/` running unvalidated? | It is the only route with declared body validation | Inspect a built container's file system; read Ntrada's payload resolution |
| Q-2 | Does `Ntrada.Extensions.Cors` echo the request origin for `allowedOrigins: ['*']` with `allowCredentials: true`, or emit a literal `*`? | Determines whether §3.18 is an open-CORS vulnerability or a broken-CORS bug | Read the extension source; or observe the `Access-Control-Allow-Origin` response header |
| Q-3 | Does the retry policy fire on 5xx responses, or only on transport failures? | Determines whether retried non-idempotent writes can duplicate (§3.20) | Read Ntrada's HTTP client; or fault-inject a 500 downstream |
| Q-4 | Which manifest does the production/staging deployment actually use? | Sync and async are different APIs with different client contracts (§3.5) | Inspect the deployment's `NTRADA_CONFIG` |
| Q-5 | Is `GET /operations/{operationId}` intentionally public, and are operation ids treated as capability tokens? | It is the only async completion channel and it is unauthenticated (§3.7) | Product decision; also `baselines/service-summaries.md` Q6 |
| Q-6 | Why is `loadBalancer` present but disabled in all four manifests when Fabio is deployed and every downstream registers with Consul? | Determines whether the direct-addressing topology is intentional or vestigial | Deployment history; ask the platform owner |
| Q-7 | Is the absence of `resourceId:` on `POST /availability/resources` deliberate (client must supply the GUID) or an oversight? | Async callers currently cannot learn the created id (§3.11) | Compare with the `.rest` sample and the intended client |
| Q-8 | Is there a client application (`Pacco.Web`) whose origin should replace the CORS wildcard? | Blocks the §7.7.2 fix | The `Pacco.Web` repository is empty in this workspace |

---

## 9. Cross-references

| Topic | Document |
| --- | --- |
| The declarative gateway pattern in general | `patterns/integration/declarative-configuration-driven-api-gateway.md` |
| Edge auth + header-borne identity | `patterns/security/edge-enforced-authentication-with-identity-binding.md` |
| Sync/async dual manifests | `patterns/integration/dual-mode-edge-write.md` |
| Correlation & span propagation | `patterns/observability/correlation-and-span-propagation.md` |
| The consuming side of every contract above | `component-internals/availability-service.md` |
| Component inventory & platform-wide gaps | `baselines/service-summaries.md` §2.1 |
