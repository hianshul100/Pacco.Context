# Pacco — Architecture Baseline (Current State)

| Field | Value |
|-------|-------|
| Project | Pacco |
| Stage | `architecture_discovery` |
| Document role | Primary current-state architecture narrative; canonical reference for specification work, impact analysis, operational review, onboarding, AI-agent grounding, and architecture evolution |
| Date of analysis | 2026-09-04 |
| Branch | `arch-discovery-21758174-49b6-4af2-9774-025561defc90` |
| Repositories analysed | 13 clones fixed by backlog issue 12998 ("Pacco - Discovery - Attempt-2") |
| Scope | **Current state only.** No target state, no modernisation plan, no migration sequencing |
| Prior artifact at this path | None — this document is authored fresh |

## How to read this document

This baseline describes **what the Pacco platform is today**, as proven by the source code and
configuration in the thirteen cloned repositories. It is deliberately a narrative: it explains how
the pieces fit together, why the seams fall where they do, and what the code does *not* tell us.

Three companion artifacts already exist and are **referenced rather than duplicated**:

| Artifact | What it owns | Relationship to this document |
|----------|--------------|-------------------------------|
| [`../repo-inventory.md`](../repo-inventory.md) | Per-repository inventory across 14 dimensions; the raw cross-repo relationship tables | This document synthesises those tables into architecture; it does not restate them row by row |
| [`../architecture-views.md`](../architecture-views.md) | **All diagrams.** Context, dependency, runtime-flow, deployment, and data-model views | This document **consumes** those diagrams. It embeds a small subset verbatim (§12) and never modifies, reconstructs, or reinterprets them |
| [`capability-baseline.md`](capability-baseline.md) | Business capabilities CAP-01…CAP-16, ownership mapping, capability traceability | This document covers structure and runtime behaviour; capability semantics live there |
| [`service-summaries.md`](service-summaries.md) | Per-service narrative summaries | Referenced for service-level detail |

**Source of truth.** The source code repositories are authoritative. Where a document, a README, or
a prior artifact disagrees with the code, this baseline states the documented claim, states the code
reality with concrete file paths, and follows the code. Those conflicts are collected in §11 and
surfaced in the closing section — none of them is silently reconciled.

**Naming.** Every service is referred to by its exact deployable name (`orders-service`,
`api-gateway`, …) as declared in `compose/services.yml`. Descriptive aliases are bound on first
mention and never substituted for the deployable name afterwards. Technical identifiers — exchange
names, message names, API paths, package names, environment variables — are copied verbatim.

**Evidence taxonomy.** Every non-obvious statement carries one of:

| Marker | Meaning |
|--------|---------|
| *(unmarked)* | **Observed fact** — read directly from a file in a cloned repository, path cited |
| `[inferred]` | Derived from observed facts by reasoning that the code does not itself state |
| `[assumption]` | Taken as true to make the narrative coherent; recorded in the closing section |
| `[unknown]` | Not determinable from the available sources; not guessed |

Where a runtime relationship could not be proven, it is **omitted rather than guessed**. An honest
`[unknown]` is preferable to an incorrect architecture relationship.

## Table of contents

1. [Platform overview](#1-platform-overview)
2. [System boundaries and deployable inventory](#2-system-boundaries-and-deployable-inventory)
3. [Service and component architecture](#3-service-and-component-architecture)
4. [Communication and integration patterns](#4-communication-and-integration-patterns)
5. [Runtime coordination and process flow](#5-runtime-coordination-and-process-flow)
6. [Data and persistence architecture](#6-data-and-persistence-architecture)
7. [Frontend and UI layer](#7-frontend-and-ui-layer)
8. [Authentication, authorization, and security patterns](#8-authentication-authorization-and-security-patterns)
9. [Deployment and operational topology](#9-deployment-and-operational-topology)
10. [Cross-cutting operational concerns](#10-cross-cutting-operational-concerns)
11. [Architectural constraints and recorded decisions](#11-architectural-constraints-and-recorded-decisions)
12. [Architecture Views Summary](#12-architecture-views-summary)
13. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Platform overview

### 1.1 What Pacco is

Pacco is a **parcel delivery platform** built as a set of .NET Core 3.1 microservices. Its
distinguishing domain idea, stated in the platform README, is *exclusive* delivery built around
**limited resource availability**: a customer does not simply request a delivery, they reserve a
scarce, time-slotted delivery resource, and the platform's rules decide whether that customer is
eligible to hold it.

That single idea explains most of the architecture. Because availability is scarce and contended,
reservation is its own service with its own consistency boundary (`availability-service`); because
eligibility depends on customer standing, customer state is replicated to the services that must
decide quickly; and because a "make an order" request touches reservation, parcels, vehicles, and
pricing, there is a dedicated process-coordination service (`ordermaker-service`) rather than a
chain of synchronous calls.

### 1.2 Architectural style

Pacco is an **event-driven microservice platform with a mediating API gateway**. Four properties
characterise it, all of them visible in code:

**Framework-defined uniformity.** Every service is built on **Convey** 0.4.\* (`convey-stack.github.io`),
a .NET microservice toolkit that supplies command/query dispatching, RabbitMQ brokering, MongoDB
persistence, Consul registration, Fabio-aware HTTP clients, Vault secret loading, Jaeger tracing,
and App.Metrics/Prometheus instrumentation as opt-in packages. The consequence is that Pacco's
services are strikingly homogeneous: the same `Extensions.cs` shape, the same `appsettings.json`
section names, the same `IMessageBroker`/`IHttpClient`/`IMongoRepository` abstractions. Convey is
the platform's real architectural standard — there is no shared Pacco library of any kind
(`repo-inventory.md` §3.3 records **no shared internal package** across the thirteen repositories).

**Messaging-first integration.** RabbitMQ carries the platform's integration traffic: roughly 80
distinct messages across **eight topic exchanges**, one owned per service. Synchronous HTTP between
services exists but is deliberately narrow — a handful of point reads (`GET` a parcel, a vehicle, a
price, a customer's state) where a caller needs an answer inside a single request.

**CQRS at the handler level.** Every service separates `Commands`/`Handlers` from `Queries`/`Handlers`
via Convey's `ICommandHandler<T>` / `IQueryHandler<TQuery,TResult>`. This is CQRS as a code
organisation and dispatch pattern; it is **not** event sourcing and **not** read/write store
separation — commands and queries hit the same MongoDB collections through the same
`IMongoRepository<TDocument, Guid>`.

**Choreography with one orchestrated exception.** Most cross-service behaviour is choreographed:
a service publishes a domain event onto its own exchange, and interested services subscribe. The
exception is `ordermaker-service`, which runs a **saga** (the `Chronicle_` 3.2.1 library) to drive
the multi-step "make an order" process explicitly.

### 1.3 Technology baseline

| Concern | Technology | Evidence |
|---------|-----------|----------|
| Runtime | .NET Core 3.1 | `dotnet: 3.1.100` in every `.travis.yml`; `netcoreapp3.1` target frameworks |
| Service framework | Convey 0.4.\* | `PackageReference` blocks in every service `.csproj` |
| API gateway | Ntrada 0.4.\* (declarative YAML gateway) | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/` |
| Messaging | RabbitMQ, topic exchanges, `snakeCase` conventions | `rabbitMq` section in every service `appsettings.json` |
| Persistence | MongoDB, one logical database per service | `mongo` section in every service `appsettings.json` |
| Cache / backplane | Redis (single server, key-prefix partitioned) | `redis` section; `AddRedis()` registrations |
| Saga | `Chronicle_` 3.2.1 | `Pacco.Services.OrderMaker.csproj:9` |
| Service discovery | Consul | `consul` section in service configs |
| Load balancing | Fabio (`fabio:9999`) | `fabio` section; `httpClient.type: "fabio"` |
| Secrets | Vault (PKI + KV) | `vault` section; `compose/consul-fabio-vault.yml` |
| Tracing | Jaeger | `jaeger` section in service configs |
| Metrics | App.Metrics → Prometheus → Grafana | `metrics` section; `compose/grafana-seq-jaeger-prometheus.yml` |
| Logging sink | Seq | `serilog` section; `compose/grafana-seq-jaeger-prometheus.yml` |
| Real-time push | SignalR (hub `/pacco`), Redis backplane | `operations-service` |
| Streaming RPC | gRPC (`Operations.proto`) | `operations-service` |
| Packaging | Docker images `devmentors/pacco.*` | `compose/services.yml` |
| Local orchestration | Docker Compose; PM2 process manifests | `hianshul100_Pacco/compose/`, `services.yml`, `prod-services.yml` |
| CI | Travis CI | `.travis.yml` in 11 of 13 repositories |

**Not present anywhere in the workspace:** Kubernetes manifests, Helm charts, Terraform, Kustomize,
any database migration tooling, any service mesh configuration, any shared internal NuGet package.
These absences are load-bearing for §9 and are recorded as gaps, not as design choices.

---

## 2. System boundaries and deployable inventory

### 2.1 The outer boundary

Pacco presents **one intended front door**: `api-gateway`, published on host port `5000`
(`compose/services.yml:11`, `5000:80`). Every service behind it is also published on a host port in
the Compose stacks (`5001`–`5009`), which means that in the Compose topology the gateway is the
*conventional* entry point rather than an *enforced* one — nothing in the Compose network
definition prevents a client from calling `orders-service` on `5006` directly. Whether a real
environment places a firewall or ingress in front of those ports is `[unknown]`; no manifest in the
workspace describes network policy.

Traffic crossing the boundary arrives over three protocols, all terminating on `operations-service`
or `api-gateway`:

| Ingress | Terminates at | Purpose |
|---------|---------------|---------|
| HTTP/REST | `api-gateway` (`:5000`) | All business operations — reads proxied, writes either proxied or converted to messages depending on configuration (§4.3) |
| WebSocket (SignalR, hub path `/pacco`) | `operations-service` (`:5005`) | Server-to-client push of operation lifecycle events |
| gRPC (`Operations.proto`) | `operations-service` | `GetOperation` unary + `SubscribeOperations` server-streaming |

There are **no outbound integrations to third-party systems** observed anywhere in the thirteen
repositories — no payment provider, no carrier API, no email or SMS gateway, no external identity
provider. Pacco as cloned is a closed system. `[inferred]` from the absence of any external base
URL, API key configuration, or outbound HTTP client target outside the `httpClient.services` maps.

### 2.2 Deployable inventory

Eleven deployables are defined in `hianshul100_Pacco/compose/services.yml`. Names below are the
**exact** `container_name` values and are used consistently throughout this document.

| Deployable name | Image | Host port | Repository | Role (alias bound on first mention) |
|-----------------|-------|-----------|------------|--------------------------------------|
| `api-gateway` | `devmentors/pacco.apigateway` | 5000 | `Pacco.APIGateway` | *the gateway* — Ntrada-based declarative edge router |
| `availability-service` | `devmentors/pacco.services.availability` | 5001 | `Pacco.Services.Availability` | *the reservation service* — resources and time-slot reservations |
| `customers-service` | `devmentors/pacco.services.customers` | 5002 | `Pacco.Services.Customers` | *the customer service* — customer profiles, state, VIP standing |
| `deliveries-service` | `devmentors/pacco.services.deliveries` | 5003 | `Pacco.Services.Deliveries` | *the delivery service* — delivery lifecycle and registrations |
| `identity-service` | `devmentors/pacco.services.identity` | 5004 | `Pacco.Services.Identity` | *the identity service* — sign-up, sign-in, token issuance |
| `operations-service` | `devmentors/pacco.services.operations` | 5005 | `Pacco.Services.Operations` | *the operations service* — operation status tracking and real-time push |
| `orders-service` | `devmentors/pacco.services.orders` | 5006 | `Pacco.Services.Orders` | *the order service* — order aggregate and its lifecycle |
| `ordermaker-service` | `devmentors/pacco.services.ordermaker` | (see §2.3) | `Pacco.Services.OrderMaker` | *the order saga* — multi-step order creation process |
| `parcels-service` | `devmentors/pacco.services.parcels` | 5007 | `Pacco.Services.Parcels` | *the parcel service* — parcel records |
| `pricing-service` | `devmentors/pacco.services.pricing` | 5008 | `Pacco.Services.Pricing` | *the pricing service* — order pricing and VIP discounting |
| `vehicles-service` | `devmentors/pacco.services.vehicles` | 5009 | `Pacco.Services.Vehicles` | *the vehicle service* — vehicle catalogue |

Two further repositories are in the clone set but contribute no deployable:

| Repository | Contents | Status |
|------------|----------|--------|
| `Pacco` | Orchestration only: `Pacco.sln`, `compose/`, `services.yml`, `prod-services.yml`, `scripts/`, `docker-images.txt`, `assets` | Not a deployable; it is the platform's assembly point |
| `Pacco.Web` | **One tracked file**: `README.md` containing the single line `# Pacco.Web` (verified via `git ls-files`) | **Empty placeholder.** No source, no build, no package manifest. See §7 |

### 2.3 A deployable-set discrepancy that matters

There are **two different deployable sets** in the orchestration repository, and they disagree:

- `compose/services.yml` and `compose/services-local.yml` each declare **11** services
  (`api-gateway` plus the ten `*-service` containers).
- `services.yml` and `prod-services.yml` — PM2 process manifests at the repository root — each
  declare **10** apps: `api`, `availability`, `customers`, `deliveries`, `identity`, `operations`,
  `orders`, `parcels`, `pricing`, `vehicles`.

**`ordermaker` is absent from both PM2 manifests.** Combined with the fact that `ordermaker-service`
has no route in any `ntrada*.yml` gateway configuration, this means the saga service is reachable
**only** by publishing directly onto the RabbitMQ `ordermaker` exchange, and is not started at all
by the PM2 path. Whether it runs in any real environment is `[unknown]` and is carried as a blocker
(see the closing section, and B1 in `capability-baseline.md`). The two sets are also *not* simply
"local vs production": `prod-services.yml` is a PM2 manifest, so the naming implies a production
process-manager deployment path that the Compose stacks do not describe.

---

## 3. Service and component architecture

### 3.1 Two internal structures, not one

Pacco's services are homogeneous at the framework level but split into **two internal layouts**:

**Four-project clean architecture** — used by `availability-service`, `customers-service`,
`deliveries-service`, `orders-service`, `parcels-service`, `pricing-service`, `vehicles-service`:

```
Pacco.Services.<Name>.Api             ← host, HTTP endpoint definitions, DI composition root
Pacco.Services.<Name>.Application     ← commands, queries, handlers, events, DTOs, service clients
Pacco.Services.<Name>.Core            ← entities, aggregates, domain events, domain exceptions
Pacco.Services.<Name>.Infrastructure  ← Mongo documents + mappers, HTTP clients, contexts, decorators
```

The dependency direction is inward: `Api → Infrastructure → Application → Core`. `Core` references
nothing but the framework primitives, which is why domain rules (order state transitions, resource
reservation invariants) are unit-testable without infrastructure.

**Single-project layout** — used by `identity-service`, `operations-service`, `ordermaker-service`:
one project holding entities, handlers, and infrastructure together. For `identity-service` and
`ordermaker-service` this reflects a small surface. For `operations-service` it reflects that the
service has almost no domain at all — it is a projection and fan-out component (§5.4).

`api-gateway` has a third shape entirely: it is **configuration, not code**. `Program.cs` starts
Ntrada; all behaviour lives in YAML (§4.3).

### 3.2 Component roles

Grouping the eleven deployables by architectural role — this is a **logical** grouping, not a
runtime or deployment relationship:

| Role | Deployables | What defines the role |
|------|-------------|-----------------------|
| Edge | `api-gateway` | Sole declarative routing surface; owns authentication enforcement and route-level authorization |
| Domain services | `availability-service`, `customers-service`, `deliveries-service`, `orders-service`, `parcels-service`, `pricing-service`, `vehicles-service` | Own an aggregate, a MongoDB database, and a RabbitMQ exchange; publish domain events |
| Identity | `identity-service` | Owns accounts, refresh tokens, and JWT issuance; publishes `signed_up` / `signed_in` |
| Process coordination | `ordermaker-service` | Owns no aggregate; drives a saga across four other services' exchanges |
| Observation / projection | `operations-service` | Owns no aggregate; subscribes to all eight exchanges and projects operation status; pushes to clients |

The last two rows are the platform's only components that are **not** aggregate owners, and both are
architecturally interesting for the same reason: they read the whole message topology. `[inferred]`
that this is why `operations-service` is the only service that subscribes to `identity`,
`ordermaker`, and `orders` exchanges simultaneously.

### 3.3 Runtime type emission — a notable structural mechanism

`api-gateway` does not have compiled C# types for the 80-odd messages it publishes. Ntrada reads a
`messages.json` payload contract and uses `System.Reflection.Emit` to **generate the message types
at runtime** from the incoming request body. This has three architectural consequences worth stating
plainly:

1. There is no compile-time contract between `api-gateway` and the services that consume its
   messages. A field renamed in a service's command class will not break the gateway build.
2. Message shape validation at the edge is limited to whatever `messages.json` declares.
3. The wire payload of a message is discoverable only by reading `messages.json` and the consuming
   service's command class side by side — there is no single generated schema. This is why the wire
   payloads of the ~80 messages remain an open item (see the closing section).

### 3.4 No shared contract package

`repo-inventory.md` §3.3 records that **no shared internal library exists** across the thirteen
repositories. Every service declares its own copy of the event and command classes it needs. Two
services that exchange `order_created` each hold an independently-declared `OrderCreated` type; they
agree only because both are serialised through Convey's `snakeCase` RabbitMQ conventions and the
routing key is derived from the type name.

This is a real architectural constraint: **message compatibility is maintained by convention and
naming, not by a shared artifact or a schema registry.** No schema registry, no Avro/Protobuf
contract for RabbitMQ messages, and no contract-version field were observed in any repository. The
only contract-verification mechanism present anywhere is Pact (`Pactify` 1.1.0) covering exactly one
pair — `orders-service` as consumer, `parcels-service` as provider — and it runs without a Pact
broker, so the pact file transport between the two repositories is `[unknown]`.

---

## 4. Communication and integration patterns

Pacco uses **four distinct integration mechanisms**, and the single most important thing to
understand about the platform is that they are not interchangeable — each carries a different class
of traffic, and mixing them up produces incorrect impact analysis.

### 4.1 Synchronous HTTP between services — narrow and Fabio-mediated

Service-to-service HTTP is deliberately sparse. It exists only where a caller needs an answer inside
the current request, and every call is a **point read**, never a write.

Each caller declares its targets in an `httpClient.services` map in `appsettings.json`, and Convey's
`IHttpClient` resolves the logical name at call time. The observed callers:

| Caller | Targets declared | `httpClient.type` | Mediated by Fabio? |
|--------|------------------|-------------------|--------------------|
| `orders-service` | `parcels` → `parcels-service`, `pricing` → `pricing-service`, `vehicles` → `vehicles-service` | `"fabio"` | **Yes** |
| `availability-service` | `customers` → `customers-service` | `"fabio"` | **Yes** |
| `pricing-service` | `customers` → `customers-service` | `"fabio"` | **Yes** |
| `ordermaker-service` | `availability` → `availability-service`, `vehicles` → `vehicles-service` | `""` (empty string) | **No — direct** |

`ordermaker-service` is the **only** service whose outbound HTTP bypasses the load balancer.
`hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json` declares:

```json
"httpClient": { "type": "", "retries": 3,
  "services": { "availability": "availability-service", "vehicles": "vehicles-service" },
  "requestMasking": { "enabled": true, "maskTemplate": "*****" } }
```

whereas `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json` declares
`"type": "fabio"`. This is a genuine behavioural difference — `ordermaker-service` resolves
`availability-service` and `vehicles-service` by container DNS name and gets no load balancing,
whereas every other caller goes through `fabio:9999`. It is precisely the kind of **subset**
behaviour that must not be generalised: "Pacco services call each other through Fabio" is true for
three of four callers and false for the fourth.

`ordermaker-service` is also the only service with `requestMasking` enabled, which masks outbound
request payloads in its own logs with `*****`.

**`customers-service` is the platform's synchronous leaf.** It is called by both
`availability-service` and `pricing-service` and calls nothing itself. Every reservation eligibility
check and every price calculation therefore has a hard synchronous dependency on it being up.
`orders-service` is the heaviest caller, with three outbound dependencies.

### 4.2 Asynchronous messaging — the platform's primary integration fabric

RabbitMQ carries roughly **80 distinct messages** across **eight topic exchanges**, one owned by
each publishing service: `identity`, `customers`, `availability`, `vehicles`, `orders`, `parcels`,
`deliveries`, `ordermaker`. The full message-to-exchange and subscription tables are in
`repo-inventory.md` §3.2 and are not restated here.

Structurally, the pattern is uniform across every service:

**Exchange topology.** One topic exchange per service, named after the service, declared from the
`rabbitMq.exchange` section of that service's `appsettings.json`. A service publishes **only** to its
own exchange — with one deliberate exception (below).

**Queue naming.** The queue template `<service>/{{exchange}}.{{message}}` gives every
(subscriber, exchange, message) triple its own queue. The effect is that each subscriber gets an
independent, individually-inspectable queue per message type, and a slow consumer of one message
cannot head-of-line-block that service's consumption of a different message.

**Naming conventions.** `snakeCase` conventions throughout: routing keys and message names are
snake_case (`order_created`, `resource_reserved`, `customer_state_changed`), and payload properties
are snake_cased on the wire.

**No service calls another service asynchronously.** Every async relationship is
publisher → exchange → subscriber. The mediation hop is real and must never be collapsed: when
`orders-service` reacts to `resource_reserved`, it is not being called by `availability-service`; it
is consuming from the `availability` exchange, and `availability-service` has no knowledge of it.

**The one cross-exchange publisher.** `ordermaker-service` publishes **commands onto other services'
exchanges** — five commands onto the `orders` exchange and `reserve_resource` onto the `availability`
exchange — in addition to publishing its own `make_order_completed` / `make_order_rejected` events
onto the `ordermaker` exchange. This is the platform's only violation of "publish only to your own
exchange", and it is the mechanism by which the saga drives other services.

**Transactional inbox/outbox.** Convey's inbox/outbox is configured in the service `appsettings.json`
files, giving at-least-once delivery with deduplication on the consuming side. Whether it is enabled
in every environment depends on the `outbox.enabled` flag per service.

**Message context propagation.** Two headers travel with every message: `message_context` (carrying
correlation identity, including the originating user) and `span_context` (carrying the Jaeger trace
context). This is what allows a single Jaeger trace to span an HTTP request at the gateway, a
RabbitMQ hop, and a handler in a downstream service.

### 4.3 The gateway — and the four-configuration problem

`api-gateway` runs **Ntrada**, a declarative YAML gateway. It has no compiled routing logic; a route
is a YAML block declaring an upstream path, a downstream target, an HTTP method, and optional auth
and claim requirements.

Which YAML file is loaded is chosen at runtime by the **`NTRADA_CONFIG`** environment variable.
`compose/services.yml:9` sets `NTRADA_CONFIG=ntrada-async.docker.yml`. Four configurations exist in
`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/`, and they are **not environment variants of one
behaviour — they are two architecturally different systems**:

| Configuration family | Write-route behaviour | Consequence |
|----------------------|-----------------------|-------------|
| `ntrada.yml` / `ntrada.docker.yml` | `use: downstream` — the gateway proxies the write over HTTP to the owning service and returns its response | Writes are synchronous. A caller gets a real result code. `operations-service` is observational |
| `ntrada-async.yml` / `ntrada-async.docker.yml` | `use: rabbitmq` — the gateway converts the write into a message and publishes it to the owning service's exchange, returning `202 Accepted` with an operation id | Writes are asynchronous and fire-and-forget at the edge. `operations-service` is **on the critical path** — it is the only way a caller learns the outcome |

`ntrada-async.yml` contains exactly **20 `use: rabbitmq`** route entries and **20 `use: downstream`**
entries — twenty write operations converted to messages, twenty reads still proxied. The exchanges
it publishes to are six: `availability` (line 120), `customers` (200), `deliveries` (243), `orders`
(359), `parcels` (446), `vehicles` (508). **`identity` is not among them** — sign-up and sign-in
remain synchronous downstream proxies even in async mode, which is the correct choice given the
caller needs the token back.

The `.docker.yml` and non-`.docker.yml` variants differ in **how the gateway reaches services**:

- `ntrada.yml` sets `useLocalUrl: true` (line 18) and `loadBalancer: { enabled: false, url: localhost:9999 }`
  (lines 19–21), and each of the nine service route groups carries a `localUrl: localhost:500X` pair
  alongside its `url: <service>-service` — availability `5001` (line 125), customers `5002` (185),
  deliveries `5003` (233), identity `5004` (273), operations `5005` (288), orders `5006` (352),
  parcels `5007` (396), pricing `5008` (411), vehicles `5009` (455). In this mode the gateway calls
  each service **directly on localhost**, bypassing Fabio entirely.
- The `.docker.yml` variants set `useLocalUrl: false` and route through the Fabio load balancer.

Nothing in any repository states which configuration a real environment uses. Since the two families
differ in whether twenty business writes are synchronous or asynchronous, **this is the single
largest open architectural question in the platform** and is carried as a blocker.

### 4.4 Real-time and streaming — `operations-service`

`operations-service` exposes two client-facing push mechanisms in addition to its HTTP routes:

**SignalR hub at `/pacco`**, backed by a **Redis backplane** so that push works across multiple
instances. A client connects, then calls the hub method `initializeAsync` passing a JWT, and
subsequently receives `connected`, `disconnected`, `operation_pending`, `operation_completed`, and
`operation_rejected` messages.

**gRPC service defined by `Operations.proto`**, exposing `GetOperation` (unary) and
`SubscribeOperations` (**server-streaming**). This is a second, parallel path to the same operation
status data.

Whether either path scopes its delivery **per caller** — that is, whether a connected client
receives only its own operations or all operations — was not established from the code available and
is carried as an open question. This matters directly: an unscoped broadcast would leak other
customers' operation activity to every connected client.

### 4.5 Integration pattern summary

| Pattern | Where used | Coupling |
|---------|-----------|----------|
| API Gateway (declarative) | `api-gateway` fronting all nine routed services | Configuration-time; no compiled contract |
| Request/response over HTTP | 4 caller services, point reads only | Runtime, synchronous, availability-coupled |
| Publish/subscribe (topic exchange) | All 10 services | Runtime, asynchronous, schema-coupled by convention |
| Command-over-message | `api-gateway` (async mode) and `ordermaker-service` → other services' exchanges | Runtime, asynchronous, fire-and-forget |
| Saga / process manager | `ordermaker-service` only | Explicit orchestration over 4 services |
| Server push (WebSocket) | `operations-service` SignalR `/pacco` | Runtime, stateful connection |
| Server-streaming RPC | `operations-service` gRPC | Runtime, stateful stream |
| Data replication | `customers-service` → 3 subscriber replicas (§6.3) | Eventual consistency |
| Client-side load balancing via registry | Convey `IHttpClient` + Consul + Fabio | Infrastructure-mediated |
| Consumer-driven contract test | `orders-service` ↔ `parcels-service` only | Build-time, one pair |

---

## 5. Runtime coordination and process flow

Diagrams for the platform's runtime flows live in `architecture-views.md` §3 (five flows, all
marked `[Confidence: partial]`). This section explains **how coordination works**, with every hop
preserved.

### 5.1 The order-making saga — the platform's only orchestrated process

`ordermaker-service` runs `AIOrderMakingSaga` (`Sagas/AIOrderMakingSaga.cs`), declared as
`Saga<AIMakingOrderData>` implementing `ISagaStartAction<MakeOrder>` plus `ISagaAction<OrderCreated>`,
`ISagaAction<ParcelAddedToOrder>`, `ISagaAction<VehicleAssignedToOrder>`, and `ISagaAction<OrderApproved>`.

The complete observed sequence, with **no hops collapsed**:

| # | Trigger consumed | What the saga does | What it emits, and where |
|---|------------------|--------------------|--------------------------|
| 1 | `MakeOrder` (start) | Assigns `Data.OrderId` | Publishes `CreateOrder` → **`orders` exchange** (`AIOrderMakingSaga.cs:63`) |
| 2 | `OrderCreated` from **`orders` exchange** | Iterates the parcel ids in saga data | Publishes one `AddParcelToOrder` per parcel → **`orders` exchange** (`:74`) |
| 3 | `ParcelAddedToOrder` from **`orders` exchange** | Calls `_vehiclesServiceClient.GetBestAsync()` (`:93`) — **direct HTTP to `vehicles-service`, no Fabio**; then `_resourceReservationsService.GetBestAsync(Data.VehicleId)` (`:98`) — **direct HTTP to `availability-service`, no Fabio** | Publishes `AssignVehicleToOrder` → **`orders` exchange** (`:103`) |
| 4 | `VehicleAssignedToOrder` from **`orders` exchange** | — | Publishes `ReserveResource` → **`availability` exchange** (`:113`) |
| 5 | *(outside `ordermaker-service`)* | `availability-service` handles `ReserveResource`, reserves the resource | Publishes `resource_reserved` → **`availability` exchange** |
| 6 | *(outside `ordermaker-service`)* | `orders-service` consumes `resource_reserved` from the **`availability` exchange**; `ResourceReservedHandler.cs:26` looks the order up by `(ResourceId, DateTime)`, `:32` calls `order.Approve()`, `:33` persists | Publishes `order_approved` → **`orders` exchange** (`:35`) |
| 7 | `OrderApproved` from **`orders` exchange** | Marks the saga done | Publishes `MakeOrderCompleted` → **`ordermaker` exchange** (`:124`), then calls `CompleteAsync()` (`:131`) |

Step 6 is the hop most easily lost. The saga does **not** approve the order and does **not** call
`orders-service`; the approval is a side effect of `orders-service` independently subscribing to
`availability`'s `resource_reserved` event. The saga only learns about it because it also subscribes
to `order_approved`. **Two independent services each consuming the same event is what closes this
loop** — there is no direct edge between `ordermaker-service` and `orders-service` at step 6.

**Compensation.** `CompensateAsync` is implemented for all five actions, but only one does real
work: `CompensateAsync(ParcelAddedToOrder, …)` publishes `CancelOrder` onto the `orders` exchange
(`:141`) with the literal reason string `"Because I'm saga"`. The other four
(`MakeOrder`, `OrderCreated`, `VehicleAssignedToOrder`, `OrderApproved`) are no-ops. Compensation
coverage is therefore **partial**: a failure after vehicle assignment or after resource reservation
has no compensating action, so a reserved resource is not released.

**Two dead-code observations at the saga boundary**, both verified:

- `Commands/External/ApproveOrder.cs` declares an `ApproveOrder : ICommand` type that **nothing in
  `ordermaker-service` ever publishes** (`grep -rn "ApproveOrder"` returns only the class
  declaration and its constructor). Approval arrives via step 6 instead.
- `AIOrderMakingHandler.cs` implements `IEventHandler<ResourceReserved>` and forwards it to
  `_coordinator.ProcessAsync(@event, SagaContext.Empty)`, but `AIOrderMakingSaga` does **not**
  implement `ISagaAction<ResourceReserved>`. The subscription exists and the message is consumed;
  the saga has no action bound to it. The `availability` → `ordermaker` `resource_reserved` edge
  drawn in `architecture-views.md` §2.2 is therefore correct as a *subscription* but carries no
  saga behaviour.

### 5.2 Saga durability — an unresolved risk

`Pacco.Services.OrderMaker.csproj:9` references `Chronicle_` version `3.2.1`, and `Extensions.cs:43`
registers it with a bare `builder.Services.AddChronicle();`. **No `Chronicle.Persistence.*` package
is referenced in the project.** `[inferred]` from this that saga state is held in the library's
default store, which for Chronicle is in-memory — meaning an in-flight order-making saga would not
survive a restart of `ordermaker-service`, and would be lost rather than resumed or failed. This is
an inference from a missing package reference, not an observed behaviour, and remains open.

### 5.3 Correlation and user context through the saga

Every publish in the saga passes `messageContext: _accessor.CorrelationContext`, so correlation
identity travels with the emitted commands. But `AIOrderMakingHandler` invokes the coordinator with
**`SagaContext.Empty`** on every one of its six handler methods, and `ordermaker-service` has no
gateway route — so the `MakeOrder` command can only arrive by a direct publish onto the `ordermaker`
exchange, from a publisher that is unidentified.

The practical consequence, verified in `orders-service`: the authorization guards on the order
handlers are written as

```csharp
var identity = _appContext.Identity;
if (identity.IsAuthenticated && identity.Id != order.CustomerId && !identity.IsAdmin)
{
    throw new UnauthorizedOrderAccessException(order.Id, identity.Id);
}
```

(`AddParcelToOrderHandler.cs:53-56`, `AssignVehicleToOrderHandler.cs:38-41`,
`ApproveOrderHandler.cs:34-37`; the same shape appears in `availability-service`'s
`ReserveResourceHandler.cs` guarding `UnauthorizedResourceAccessException`.)

The guard **short-circuits on `identity.IsAuthenticated`**. A message arriving with an empty user
context is not authenticated, so the condition is false and the guard never fires — the operation is
allowed. `CreateOrderHandler.cs` has no identity check at all; it takes
`IOrderRepository, ICustomerRepository, IMessageBroker, IEventMapper, IDateTimeProvider` and
validates only that the customer exists:

```csharp
if (!(await _customerRepository.ExistsAsync(command.CustomerId)))
    throw new CustomerNotFoundException(command.CustomerId);
```

So the saga's commands pass every guard in the platform, regardless of who initiated them. Whether
this is a deliberate "internal caller" design or an oversight is `[unknown]` — no code comment or
document states an intent — and it is carried as an open question. `CorrelationContext.UserContext`
(`ordermaker-service/CorrelationContext.cs`) carries `Id`, `IsAuthenticated`, `Role`, and `Claims`,
so the mechanism to propagate identity exists; it simply is not populated on this path.

### 5.4 Operation status tracking — the async write-path completion mechanism

When `api-gateway` runs an `ntrada-async*` configuration, a write returns `202 Accepted` with an
operation id and the caller must learn the outcome elsewhere. That elsewhere is `operations-service`.

`operations-service` subscribes to **all eight exchanges** and projects each observed message into an
operation record keyed by correlation id, then pushes `operation_pending`, `operation_completed`, or
`operation_rejected` to connected clients over the SignalR hub `/pacco` (and serves the same data
over gRPC `GetOperation` / `SubscribeOperations`).

This makes `operations-service` structurally unusual: it is the only service that depends on the
**entire** message topology, so any new message type added anywhere either flows through it or is
invisible to async callers. It owns no aggregate and no MongoDB database of its own for operation
state; that state lives in **Redis** (§6.4).

The `depends_on` list in `compose/services.yml` for `operations-service` names eight other services.
That is a **Compose start-ordering directive, not a runtime dependency** — `operations-service`
reaches none of those services directly at runtime; it only consumes from their exchanges. The two
concerns must not be conflated.

---

## 6. Data and persistence architecture

Entity-relationship diagrams for the seven services with a domain model are in
`architecture-views.md` §5 and are not reproduced here.

### 6.1 Database-per-service, with three exceptions

The platform follows database-per-service strictly. Each service's `appsettings.json` declares a
`mongo.database` value equal to its own deployable name:

| Deployable | `mongo.database` | Redis key prefix (`redis.instance`) |
|------------|------------------|--------------------------------------|
| `availability-service` | `availability-service` | `availability:` |
| `customers-service` | `customers-service` | `customers:` |
| `deliveries-service` | `deliveries-service` | `deliveries:` |
| `identity-service` | `identity-service` | `identity:` |
| `operations-service` | `operations-service` | `operations:` |
| `orders-service` | `orders-service` | `orders:` |
| `parcels-service` | `parcels-service` | `parcels:` |
| `vehicles-service` | `vehicles-service` | `vehicles:` |
| `ordermaker-service` | **none configured** | `ordermaker:` |
| `pricing-service` | **none configured** | **none configured** |

Two services have no MongoDB database at all, and both absences are meaningful:

- **`pricing-service` registers no persistence whatsoever** — neither `AddMongo()` nor `AddRedis()`
  appears anywhere in `hianshul100_Pacco.Services.Pricing/src`. It is genuinely stateless: it reads
  the customer over HTTP (via Fabio) and computes a price. Its only durable dependency is
  `customers-service`.
- **`ordermaker-service` registers only `AddRedis()`** (`Extensions.cs:37`). Combined with the
  missing Chronicle persistence package (§5.2), the saga service has no observed durable store for
  saga state.

### 6.2 Persistence style — document mapping, not ORM

Every persisting service uses Convey's `IMongoRepository<TDocument, Guid>` with a hand-written
document type and an explicit mapper between the document and the domain entity — for example a
`Pacco.Services.Orders.Infrastructure.Mongo.Documents.OrderDocument` alongside an `Order` aggregate
in `.Core`. The document type, not the domain entity, is what MongoDB serialises. Three consequences:

1. The domain model can change shape without changing the stored schema, and vice versa — the mapper
   absorbs it. This is why `.Core` has no persistence dependency.
2. There is **no schema enforcement** in the database. MongoDB accepts whatever the document type
   serialises to.
3. **There is no migration tooling in any repository.** No EF Core migrations, no Mongock, no
   versioned migration scripts, no `schema_version` field. Schema evolution is therefore performed
   by deploying new document types against existing data, with backward compatibility maintained by
   the mapper. How historical documents written by an older mapper are handled is `[unknown]`.

### 6.3 Cross-service data replication — the customer replica

The one deliberate data-duplication pattern in the platform: three services keep a **local replica of
customer identity** so that they can validate a `CustomerId` without a synchronous call.

`customers-service` publishes `customer_created` onto the `customers` exchange, and
`availability-service`, `orders-service`, and `parcels-service` each subscribe to it and write a
minimal local customer record. `orders-service`'s `CreateOrderHandler` then validates against its own
replica — `await _customerRepository.ExistsAsync(command.CustomerId)` — rather than calling
`customers-service`.

This is **eventual consistency by design**: an order can be rejected as "customer not found" purely
because the `customer_created` event has not yet been consumed. `customers-service` also publishes
`customer_state_changed` and `became_vip`, but **no subscriber for either was found in any of the
thirteen repositories** — so the replicas capture creation only and never learn about state or VIP
changes. Services that need current customer state fetch it synchronously instead:
`availability-service` calls `GetStateAsync` before permitting a reservation
(`ReserveResourceHandler.cs`), and `pricing-service` calls `customers-service` for VIP discounting.

**No reconciliation mechanism was observed** — no periodic resync, no snapshot replay, no
version/sequence check on the replicated records. A missed `customer_created` event leaves a
permanently inconsistent replica. This is recorded as an open question rather than a defect, because
whether the outbox guarantees make the miss impossible in practice is `[unknown]`.

### 6.4 Redis — registered widely, actively used narrowly

**Nine services register `AddRedis()`**: `availability-service`, `customers-service`,
`deliveries-service`, `identity-service`, `operations-service`, `ordermaker-service`,
`orders-service`, `parcels-service`, `vehicles-service`. Only `pricing-service` does not.

Applying the two-proof rule for infrastructure — the component must exist in deployment
configuration **and** be an active runtime dependency evidenced in code — Redis passes for the
deployment half everywhere (`compose/infrastructure.yml:91` declares the `redis` container, and all
nine services carry a `redis` config section with a distinct key prefix). The runtime half is where
it narrows:

| Observed active use | Evidence | Verdict |
|---------------------|----------|---------|
| Operation status storage in `operations-service` | `Operations.Api/Services/OperationsService.cs:13,16` injects `IDistributedCache` — **the only `IDistributedCache` injection site in the entire workspace** | Active runtime dependency, confirmed |
| SignalR backplane in `operations-service` | `Operations.Api/Infrastructure/Extensions.cs:106` — `signalR.AddRedis(redisOptions.ConnectionString)` | Active runtime dependency, confirmed |
| The other eight registering services | Registration only; no `IDistributedCache` injection, no cache read or write found | **Registered but no active use observed.** Convey may use it internally for the message inbox/outbox; that was not verified |

So the honest statement is: **Redis is a confirmed runtime dependency of `operations-service` for two
distinct purposes, and a registered-but-unexercised dependency of eight other services.** Drawing a
runtime edge from all nine services to Redis would over-generalise a subset behaviour.

`operations-service` also registers `AddMongo()` (`Extensions.cs:71`) and has a `mongo.database`
configured, yet the only operation-state read/write path observed is through `IDistributedCache`.
Operation status is therefore **cached, not durably stored** `[inferred]` — an operation record would
be lost on Redis eviction or restart, and in async gateway mode that is the caller's only channel for
learning an outcome. `capability-baseline.md` records this as CONFLICT-01.

### 6.5 Redis deployment shape

One Redis server serves the whole platform, partitioned only by the `redis.instance` key prefix.
This is a **shared-infrastructure** relationship, not a service-to-service one: two services sharing
a Redis instance have no runtime relationship with each other. The prefixes give logical isolation;
they give no isolation of memory pressure, eviction, or failure. A single Redis outage removes
operation status tracking and the SignalR backplane simultaneously.

### 6.6 Data model ownership summary

| Aggregate / entity | Owner | Replicated to |
|--------------------|-------|---------------|
| `Order` | `orders-service` | — |
| `Resource`, `Reservation` | `availability-service` | — |
| `Customer` | `customers-service` | `availability-service`, `orders-service`, `parcels-service` (identity only, creation-time) |
| `Parcel` | `parcels-service` | — |
| `Vehicle` | `vehicles-service` | — |
| `Delivery`, `DeliveryRegistration` | `deliveries-service` | — |
| `User`, `RefreshToken` | `identity-service` | — |
| Operation status record | `operations-service` (Redis) | — |
| `AIMakingOrderData` (saga state) | `ordermaker-service` (no durable store observed) | — |
| *(none — stateless)* | `pricing-service` | — |

---

## 7. Frontend and UI layer

> **Input gap.** `docs/architecture-inventory/baselines/ui-inventory.md` **does not exist** in this
> repository. No dedicated UI inventory was produced by any earlier stage, so this section is
> derived directly from source code rather than summarised from an existing artifact. If a
> `ui-inventory.md` is authored later, it supersedes the detail below and this section should
> reference it instead.

### 7.1 There is no application frontend

The finding that matters most: **Pacco as cloned has no web application.**

`hianshul100_Pacco.Web` — the repository whose name implies the platform's web client — tracks
**exactly one file**. `git -C hianshul100_Pacco.Web ls-files` returns `README.md`, and that file
contains the single line `# Pacco.Web`. There is no source, no build configuration, no package
manifest, no framework dependency. It is an empty placeholder.

Corroborating searches across all thirteen repositories:

| Searched for | Result |
|--------------|--------|
| `package.json`, `angular.json`, `vite.config.*` | **None found anywhere** |
| `.cshtml` (Razor views) or `.razor` (Blazor components) | **None found anywhere** |
| `wwwroot` directories | **Exactly one**: `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot` |

So there is no Node toolchain, no server-side view engine, and no client application project in the
workspace. Whether a Pacco frontend exists in a repository outside this thirteen-clone scope is
`[unknown]`.

### 7.2 The only UI: a static SignalR test page in `operations-service`

The single piece of user-facing HTML in the platform lives at
`Pacco.Services.Operations.Api/wwwroot/ui/`, served because `Extensions.cs:88` calls
`.UseStaticFiles()` on the `operations-service` host. Answering the standard frontend questions
directly:

**How is the UI served?** As plain static files from `operations-service` on port `5005`, alongside
that service's HTTP API, SignalR hub, and gRPC endpoint. There is no separate web host, no CDN, no
reverse-proxy rule for UI assets, and no route for it in any `ntrada*.yml` gateway configuration —
it is reachable only by hitting `operations-service` directly.

**What renders the initial HTML?** Nothing at runtime. `wwwroot/ui/index.html` is a **static file
shipped verbatim**. There is no server-side rendering, no templating, no view engine, no
pre-rendering step, and no build. The document is exactly what the browser receives.

**JS framework and version.** There is **no JavaScript framework**. `wwwroot/ui/js/app.js` is a
vanilla-JavaScript IIFE opening with `'use strict'`, using direct DOM handles and no library. The
only third-party assets are:

| Asset | What it is | Version | How it is delivered |
|-------|-----------|---------|---------------------|
| Bootstrap CSS | Styling only, no JS component usage | **4.0.0** | Loaded from the CDN `https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css` — a live external dependency of the page |
| `js/signalr.js` | The ASP.NET Core SignalR JavaScript client | **`[unknown]`** — the file is a vendored, pre-built webpack UMD bundle (`webpackUniversalModuleDefinition`) with no version marker and no `package.json` recording it | Committed into the repository and served locally |

There is no bundler, no transpiler, no minification step, and no dependency lockfile. The SignalR
client was vendored by hand, which is why its version cannot be established.

**How does user/session context pass from server to client?** **It does not — the flow is the
reverse.** There is no server-rendered session, no cookie handshake, and no token embedded in the
markup. Instead:

1. `index.html` presents a bare `<input id="jwt">` text box and a `<button id="connect">`.
2. The user **pastes a JWT into the input by hand**.
3. `app.js` builds the connection with
   `new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco').configureLogging(signalR.LogLevel.Information).build()`.
4. `connection.start()` completes — the WebSocket is established **before** any credential is sent.
5. Only then does the page call `connection.invoke('initializeAsync', $jwt.value)`, passing the
   pasted token as an ordinary hub-method argument.
6. The page then listens for `connected`, `disconnected`, `operation_pending`, `operation_completed`,
   and `operation_rejected`, appending each into `<ul id="messages">`.

Two architectural facts follow from step 3 and step 4. The hub URL is **hard-coded to
`http://localhost:5005/pacco`** — plaintext HTTP, a fixed host, and a direct address that bypasses
`api-gateway` entirely. And the connection is **established unauthenticated**, with the token
supplied afterwards over the open channel, rather than presented during the connection handshake.

### 7.3 What this page is, architecturally

This is a **developer test harness, not a product UI**. Its `<title>` is `Pacoo SignalR` (spelled
with the double-o), it has no navigation, no routing, no state management, no error handling beyond
appending strings to a list, and it exercises exactly one capability — watching operation status
events arrive. It should be read as evidence of *how the SignalR contract works*, which it documents
precisely, and not as evidence of a frontend architecture, which the platform does not have.

**Consequence for the platform's shape.** Because there is no frontend, every design question that a
frontend would normally answer is unanswered here: how a user actually authenticates in practice, how
a JWT is obtained and refreshed on the client, how the async `202 Accepted` operation id is correlated
back to a user action in a real UI, and whether the SignalR or the gRPC path is the intended
production consumer. These are `[unknown]`, and they are the reason several of the platform's
edge-facing behaviours (§4.3, §4.4, §8) cannot be validated end to end from this workspace.

---

## 8. Authentication, authorization, and security patterns

Pacco layers **four** distinct security mechanisms. They are independent of each other, and three of
them are applied unevenly across the platform.

### 8.1 Token issuance — `identity-service`

`identity-service` owns accounts and mints JWTs. Its `jwt` configuration declares
`issuer: pacco`, `expiryMinutes: 60`, and `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`. It
publishes `signed_up` and `signed_in` onto the `identity` exchange; `customers-service` consumes
`signed_up` to create the corresponding customer record.

Note that `identity-service` sets **`validateIssuer: false`** and `validateAudience: false` on its own
token validation, whereas the domain services set `validateIssuer: true` with `validIssuer: "pacco"`.
`validateLifetime: true` is set everywhere.

### 8.2 Edge enforcement — `api-gateway` route-level auth and claim gates

Authentication is enforced **at the gateway**, declaratively per route. `ntrada.yml` opens with an
`auth:` block mapping the logical `role` claim to the full Microsoft claim URI
`http://schemas.microsoft.com/ws/2008/06/identity/claims/role` (lines 4–5). Individual routes then
declare:

- `auth: true` — a valid bearer token is required. Applied to the large majority of routes.
- `claims: { role: admin }` — the token must additionally carry the `admin` role. Observed on
  administrative routes at lines 137–138, 151–152, 159–160, 180–181 among others.
- **`@user_id` binding** — the gateway substitutes the caller's identity into the request. Two
  distinct forms appear: `bind: - customerId:@user_id` (lines 103, 167), which forces a request-body
  field to the caller's own id, and `downstream: customers-service/customers/@user_id` (line 143),
  which rewrites the downstream path so a caller can only ever fetch their own record.

The `@user_id` binding is the platform's most effective authorization control, because it removes the
possibility of a caller naming someone else's id in the first place, rather than checking it
afterwards.

### 8.3 In-service authorization — a guard that fails open

Domain services re-check ownership in their handlers via an injected `IAppContext`. `IdentityContext`
(`Orders.Infrastructure/Contexts/IdentityContext.cs`) exposes `IsAuthenticated`, `Id`, `Role`,
`Claims`, and derives `IsAdmin = Role.Equals("admin", StringComparison.InvariantCultureIgnoreCase)`
(`:29`).

The guard is written identically in every place it appears:

```csharp
if (identity.IsAuthenticated && identity.Id != order.CustomerId && !identity.IsAdmin)
    throw new UnauthorizedOrderAccessException(order.Id, identity.Id);
```

Observed in `AddParcelToOrderHandler.cs:53-56`, `AssignVehicleToOrderHandler.cs:38-41`,
`ApproveOrderHandler.cs:34-37`, and — with `UnauthorizedResourceAccessException` — in
`availability-service`'s `ReserveResourceHandler.cs`.

**The `IsAuthenticated &&` prefix means the guard fails open.** An unauthenticated caller does not
satisfy the first term, so the check is skipped and the operation proceeds. This is safe *only*
because the gateway is expected to reject unauthenticated requests before they arrive — the in-service
check is a defence-in-depth layer for authenticated-but-wrong-user, not an independent gate. Any path
that reaches a handler without passing the gateway (a direct publish onto a service's exchange, or a
direct call to a service's published Compose port) bypasses authorization entirely. §5.3 shows this is
exactly what the `ordermaker-service` saga path does.

`CreateOrderHandler.cs` has **no identity check at all** — it validates only that the customer exists
in the local replica. So order creation is gated solely by the gateway.

### 8.4 Service-to-service certificate authentication — enabled on exactly one service

Every service carries a `security.certificate` configuration section, but the sections are almost all
empty. Reading them across all ten services:

| Service | `security.certificate` contents |
|---------|--------------------------------|
| `customers-service` | **Fully configured and `enabled: true`** |
| `availability-service` | Header name only (`{"header": "Certificate"}`) — not enabled |
| All eight others | **`{}` — empty** |

`customers-service`'s block is the only real one:

```json
"certificate": {
  "enabled": true, "header": "Certificate", "skipRevocationCheck": false,
  "allowedDomains": ["pacco.io"], "allowSubdomains": true, "allowedHosts": ["localhost"],
  "acl": { "availability-service": { "validIssuer": "localhost", "permissions": ["customers:read"] } }
}
```

This is a **client-certificate ACL**: `customers-service` accepts a certificate presented in a
`Certificate` header, requires it to chain to the `pacco.io` domain, and grants the named caller
`availability-service` exactly the `customers:read` permission.

Two things follow, and neither should be generalised. First, **this is not platform-wide mTLS** —
nine of ten services do not enable certificate checking at all, so "Pacco uses mTLS between services"
would be a false statement. Second, **`pricing-service` is not in the ACL** even though it calls
`customers-service` over HTTP (§4.1). If certificate enforcement is genuinely active, that call has no
grant; if it succeeds anyway, the enforcement is not actually in effect. Which of the two is true is
`[unknown]` and is carried as an open question.

The certificates themselves come from **Vault PKI**: each service's `vault.pki` block declares
`enabled: true`, a per-service `roleName`, and a `commonName` of `<service>.pacco.io` — so
`orders-service` requests a certificate for `orders-service.pacco.io`. The PKI machinery is therefore
provisioned platform-wide even though the verification side is switched on in one place.

### 8.5 Secret management — Vault, with real dynamic credentials

Every service declares a `vault` section with `enabled: true`, pointing at `http://localhost:8200`,
and uses three Vault features:

| Feature | Configuration | What it does |
|---------|--------------|--------------|
| KV v2 secrets | `kv.mountPoint: "kv"`, `kv.path: "<service>/settings"` | Per-service configuration overlay loaded at startup |
| PKI | `pki.roleName: "<service>"`, `pki.commonName: "<service>.pacco.io"` | Issues the service's client certificate (§8.4) |
| **Dynamic database credentials** | `lease.mongo`: `type: "database"`, `roleName: "<service>"`, `autoRenewal: true`, connection template `mongodb://{{username}}:{{password}}@localhost:27017` | MongoDB credentials are **issued per-service by Vault with automatic lease renewal**, not stored in configuration |

The dynamic Mongo lease is the strongest security control in the platform: no service holds a static
database password, and each service's database access is scoped to its own Vault role.

### 8.6 Committed credentials — a live exposure

Against that, the repositories **commit real credential material in plaintext**:

| What | Where |
|------|-------|
| Vault unseal keys and a Vault root token | Vault bootstrap material in `hianshul100_Pacco` |
| The gateway's symmetric JWT `issuerSigningKey` | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:44` |
| A development certificate password (`"test"`) | `identity-service` `jwt.certificate.password` |
| Static Vault `token` / `password` values | The `vault` section of every service `appsettings.json` |

This document records these **by path only and reproduces no secret values**, matching the handling in
the other baseline artifacts. Anyone holding the `issuerSigningKey` can mint a token that
`api-gateway` will accept, including one carrying `role: admin`. Whether these are throwaway
local-development values or real ones is `[unknown]`; it is carried as an `[ACTION NOW]` blocker.

Note the signing asymmetry this exposes: `api-gateway` validates tokens with a **symmetric** shared
secret (`issuerSigningKey`), while the domain services validate with a **certificate**
(`jwt.certificate.location: "certs/localhost.cer"`). The gateway and the services are therefore not
validating the same way, and a token accepted by one is not automatically accepted by the other.

### 8.7 Transport security

No TLS termination is configured anywhere in the workspace. Every service publishes plain HTTP on its
Compose port, the gateway's Fabio target is `localhost:9999` over HTTP, Vault is reached at
`http://localhost:8200`, and the SignalR test page connects to `http://localhost:5005/pacco`. Whether
a real environment terminates TLS at an ingress in front of `api-gateway` is `[unknown]` — no manifest
in the workspace describes one.

---

## 9. Deployment and operational topology

The Compose topology diagram is in `architecture-views.md` §4.1, and the Compose-vs-PM2 comparison
table is §4.2. This section explains what those artifacts mean operationally.

### 9.1 Two parallel deployment paths, neither production-grade

**Path A — Docker Compose.** `hianshul100_Pacco/compose/` holds a layered set of stacks that are
composed rather than used individually:

| File | Declares |
|------|----------|
| `infrastructure.yml` | The full backing stack: `consul`, `fabio`, `grafana`, `jaeger`, `mongo`, `prometheus`, `rabbitmq`, `redis`, `seq`, `vault` |
| `consul-fabio-vault.yml` | Discovery, routing, and secrets only |
| `mongo-rabbit-redis.yml` | Data and messaging only |
| `grafana-seq-jaeger-prometheus.yml` | Observability only |
| `host-infrastructure.yml` | Host-networking variant of the same infrastructure |
| `services.yml` / `services-local.yml` | The eleven application containers |

All of them join a single shared bridge network named `pacco`. Every infrastructure component is a
**single container** — one Mongo, one RabbitMQ, one Redis, one Consul, one Vault. There is no
replication, no clustering, no quorum, and no persistent-volume declaration for most components. This
is a developer environment.

**Path B — PM2.** `services.yml` and `prod-services.yml` at the orchestration repository root are
**PM2 process manifests**, and they run the services as bare .NET processes on a host rather than as
containers:

- `services.yml` runs `dotnet run` from each project's source directory (`cwd: ../Pacco.Services.X/src/...`)
  with `max_restarts: 3` — a development-mode manifest.
- `prod-services.yml` runs the published assembly (`dotnet Pacco.Services.X.Api.dll` from
  `bin/release/netcoreapp3.1/publish`) with `ASPNETCORE_URLS: http://*:500X` per app.

The name `prod-services.yml` implies a production path, but it is still a single-host PM2 manifest
with `max_restarts: 3` and no health gating, no rolling update, and no multi-instance configuration.
It also **omits `ordermaker`** (§2.3).

**Nothing in the workspace reconciles these two paths.** No document says which one is authoritative,
whether they are alternatives or successive stages, or which environment uses which. `[assumption]`
that the Compose stacks describe local development; carried in the closing section.

### 9.2 No production orchestration exists

Stated plainly because so many downstream questions depend on it: **there are no Kubernetes
manifests, no Helm charts, no Terraform, no Kustomize overlays, and no service-mesh configuration in
any of the thirteen repositories.** Consequently, the following are `[unknown]` and cannot be derived
from these sources at any confidence:

replica counts · autoscaling policy · resource requests and limits · pod disruption budgets ·
liveness and readiness probe definitions beyond the Consul `ping` endpoint · network policy ·
ingress and TLS termination · secrets injection at deploy time · rollout and rollback strategy ·
multi-environment promotion · disaster recovery and backup

### 9.3 Service discovery and routing

Two components mediate east-west traffic, and both satisfy the two-proof test (declared in
`compose/infrastructure.yml` **and** an active runtime dependency in service code):

**Consul.** Every service declares `consul.enabled: true` with `url: http://localhost:8500`,
`service: "<service-name>"`, its own `port`, and health-check settings `pingEnabled: true`,
`pingEndpoint: "ping"`, `pingInterval: 3`, `removeAfterInterval: 3`. Each service therefore
self-registers on startup and is deregistered after three failed pings. Note the `address` value:
`docker.for.win.localhost` — a Windows-Docker-Desktop host alias hard-coded into the committed
configuration, which is further evidence that these files describe a developer machine.

**Fabio.** Every service declares `fabio.enabled: true`, `url: http://localhost:9999`, and its own
service name. Fabio reads the Consul registry and routes by service name. Callers reach it only when
their `httpClient.type` is `"fabio"` — which, as §4.1 records, excludes `ordermaker-service`.

### 9.4 The datastore inventory conflict

`hianshul100_Pacco/docker-images.txt` is a **setup cookbook**, not a deployment manifest: it is a
plain-text document of `docker run` commands and Vault CLI transcripts. It lists **SQL Server 2017,
PostgreSQL, InfluxDB, Elasticsearch, Kibana, and Logstash** among the platform's datastores and
logging components.

**None of those six is used by any service.** No service references a relational connection string,
no `AddPostgres`/`AddSqlServer` registration exists, `metrics.influxEnabled` is `false` everywhere,
and `logger.elk.enabled` is `false` everywhere. The code is authoritative: the platform's actual
datastores are **MongoDB and Redis only**, its actual metrics sink is Prometheus, and its actual log
sink is Seq plus rolling files. Whether the six extras are aspirational, vestigial, or simply a
generic developer cheat-sheet that was copied in is `[unknown]`.

`docker-images.txt` is also where the Vault unseal keys and root token are committed in plaintext
(§8.6).

### 9.5 Build and release

Eleven of the thirteen repositories carry a `.travis.yml`: `Pacco.APIGateway` and all ten
`Pacco.Services.*` repositories. The two without one are `Pacco` (orchestration) and `Pacco.Web`
(empty). A representative pipeline — `hianshul100_Pacco.Services.Orders/.travis.yml`:

```yaml
language: csharp
mono: none
dist: xenial
dotnet: 3.1.100
branches:
  only: [master, develop]
script:
  - ./scripts/build.sh
  - ./scripts/test.sh
after_success:
  - ./scripts/dockerize.sh
```

So the release path is: **Travis CI, per repository, on `master` and `develop` only, running
build → test → dockerize.** Each repository ships its own image under the `devmentors/pacco.*`
namespace; there is no shared pipeline template and no cross-service release coordination — services
version and ship independently.

There is **no CD stage anywhere**. `dockerize.sh` builds and pushes an image; nothing observed
deploys it. How an image reaches a running environment is `[unknown]`.

> **Conflict with an existing artifact.** `architecture-views.md` §4.5 states that "No CI or CD
> pipeline definition exists in the orchestration repository… Individual service repositories were
> not observed to carry a shared pipeline template either… Whether images are built by hand or by a
> pipeline outside the workspace is `[unknown]`." The code contradicts the CI half of that: eleven
> `.travis.yml` files exist and define a build-test-dockerize pipeline. `repo-inventory.md` §2.3
> records the Travis pipelines correctly. **This document follows the code.** The claim in
> `architecture-views.md` §4.5 is stale and is surfaced in the closing section rather than corrected
> here — this stage does not modify that artifact. The CD half of its statement stands: no deployment
> automation was observed.

---

## 10. Cross-cutting operational concerns

Every one of these is supplied by Convey as an opt-in package and configured identically across
services, which is why they are described once here rather than per service.

### 10.1 Distributed tracing

`jaeger.enabled: true` in every service, reporting over UDP to `localhost:6831` with
`sampler: "const"` — **100% sampling, no rate limiting.** Each service sets a short `serviceName`
(`orders`, `availability`, …) rather than its full deployable name, so trace-service names and
deployable names differ and must be mapped when correlating. `excludePaths` drops `/`, `/ping`, and
`/metrics` from tracing.

The `span_context` RabbitMQ header (§4.2) is what makes traces continuous across broker hops — a
trace can follow a request from the gateway, through a published message, into a consuming handler in
another service. **This is the platform's only end-to-end observability across the async path**, and
it is the mechanism to use when investigating anything that crosses an exchange.

### 10.2 Metrics

`metrics.enabled: true`, `prometheusEnabled: true`, `influxEnabled: false`, `interval: 5`,
`database: "pacco"`, `env: "local"` — note the environment label is hard-coded to `local` in the
committed configuration. App.Metrics exposes a scrape endpoint; the sample Prometheus scrape config
in `docker-images.txt` targets `metrics_path: '/metrics-text'` on port `5000`, i.e. `api-gateway`
only. Whether a real Prometheus scrapes all eleven deployables or just the gateway is `[unknown]` —
`compose/prometheus/` holds the actual scrape configuration used by the Compose stack.

### 10.3 Logging

Serilog with four sinks configured per service: `console` (enabled), `file` (enabled, rolling daily
to `logs/logs.txt`), `seq` (enabled), and `elk` (**disabled**, pointing at `http://localhost:9200`).
Level `information`.

The `excludeProperties` list is a genuine security control and worth naming in full, because it is the
platform's log-redaction policy: `api_key`, `access_key`, `ApiKey`, `ApiSecret`, `ClientId`,
`ClientSecret`, `ConnectionString`, `Password`, `Email`, `Login`, `Secret`, `Token`. Note that
`Email` and `Login` are redacted — this is PII handling, not just credential handling.

`ordermaker-service` adds a second redaction layer with `httpClient.requestMasking.enabled: true` and
`maskTemplate: "*****"`, masking outbound HTTP request payloads. It is the only service that does so.

### 10.4 Reliability — inbox/outbox, and one important caveat

Every persisting service configures Convey's transactional inbox/outbox:

```json
"outbox": { "enabled": true, "type": "sequential", "expiry": 3600,
  "intervalMilliseconds": 2000, "inboxCollection": "inbox",
  "outboxCollection": "outbox", "disableTransactions": true }
```

The `inbox` collection deduplicates incoming messages (at-least-once delivery made effectively
once-only per consumer), and the `outbox` collection stages outgoing messages, drained sequentially
every 2 seconds with a 3600-second expiry.

**`disableTransactions: true` is the caveat.** With MongoDB transactions disabled, the domain write
and the outbox write are not atomic. A crash between the two loses the event while keeping the state
change — the classic dual-write failure the outbox pattern exists to prevent. The pattern's
*structure* is present; its core guarantee is switched off. `[inferred]` that this is because the
Compose stack runs a **single-node** MongoDB, and MongoDB multi-document transactions require a
replica set. That inference is consistent with §9.1 but is not stated anywhere in the code.

Beyond this, resilience is thin: `httpClient.retries: 3` on outbound HTTP calls, `max_restarts: 3` in
the PM2 manifests, and `restart: unless-stopped` in Compose. **No circuit breaker, no bulkhead, no
timeout policy, no rate limiting, and no dead-letter queue configuration was observed anywhere.** A
handler that throws returns the message to RabbitMQ with no configured failure destination.

### 10.5 Health checking

The only health mechanism in the platform is the Consul `ping` endpoint (§9.3), polled every 3
seconds with deregistration after 3 failures. There are no ASP.NET Core `/health` endpoints, no
readiness-versus-liveness distinction, and no dependency health checks — a service whose MongoDB is
unreachable still answers `ping` and stays in the Consul registry.

---

## 11. Architectural constraints and recorded decisions

### 11.1 There are no Architecture Decision Records

**No ADR files exist.** A search across all thirteen cloned repositories and this artifact repository
found no `docs/adr/` directory, no `adr/` directory, no `decisions/` directory, and no file matching
`*adr*`, `*decision-record*`, or `*rfc*`. The complete Markdown documentation set in this artifact
repository is `README.md` plus the four `docs/architecture-inventory/` artifacts and the thirteen
`repo-summary/*.md` files; none of them is an ADR.

This section therefore records **no ADR-derived constraints**, because there are none to derive. No
architecture decision has been invented, inferred, or back-filled from code to fill the gap. If ADRs
exist outside this workspace, they are `[unknown]`.

### 11.2 Constraints enforced by the code

What the code *does* impose — these are observed constraints on how the platform can change, not
decisions anyone recorded:

| # | Constraint | Enforced by | What it prevents |
|---|-----------|-------------|------------------|
| C1 | A service may publish only to its own exchange | `rabbitMq.exchange.name` per service | Cross-ownership publishing — **broken deliberately by `ordermaker-service`** (§4.2), the platform's only exception |
| C2 | Message compatibility is maintained by naming convention, not by a shared artifact | No shared library (§3.4); `snakeCase` conventions; routing key derived from type name | Nothing. A rename in one service silently breaks the other at runtime, with no build-time signal |
| C3 | Services must be independently deployable | Per-repository Travis pipeline; per-repository Docker image; no shared package | Coordinated multi-service releases — there is no mechanism to perform one |
| C4 | Domain logic must not depend on infrastructure | The four-project layout's inward dependency direction (§3.1) | Persistence concerns leaking into `.Core`; makes domain rules unit-testable |
| C5 | Every service owns exactly one MongoDB database, named after itself | `mongo.database` per service | Cross-service database access. No service holds another's connection string |
| C6 | Database credentials must be dynamic, never static | Vault `lease.mongo` with `autoRenewal: true` (§8.5) | Long-lived shared database passwords |
| C7 | Authentication is enforced at the edge | `auth: true` per route in `ntrada*.yml`; in-service guards fail open (§8.3) | Nothing, for any path that does not traverse the gateway |
| C8 | Schema evolution has no tooling | No migration framework in any repository (§6.2) | Nothing. Compatibility rests entirely on hand-written document mappers |
| C9 | Only `.NET Core 3.1` is buildable | `dotnet: 3.1.100` pinned in all eleven Travis files | Per-service runtime version drift — but also pins the whole platform to a runtime that is long out of support |
| C10 | Cross-service reads are point reads only | Every `httpClient.services` target is a single-entity `GET` (§4.1) | Chatty synchronous composition; keeps the sync surface small |

C9 deserves a plain statement: **.NET Core 3.1 reached end of support in December 2022.** That is a
current-state fact about the platform's runtime, recorded here as a constraint, not as a
recommendation.

### 11.3 Doc-versus-code conflicts

Each of the following is a place where a document in the workspace disagrees with the code. In every
case **the code is followed**, the documented claim is stated, and the disagreement is surfaced in the
closing section rather than silently reconciled.

| # | Documented claim | Code reality | Resolution |
|---|-----------------|--------------|------------|
| X1 | `architecture-views.md` §4.5: "No CI or CD pipeline definition exists… Individual service repositories were not observed to carry a shared pipeline template either" | Eleven `.travis.yml` files define `build.sh` → `test.sh` → `dockerize.sh` on `master`/`develop` (§9.5). `repo-inventory.md` §2.3 records them correctly | **Follow the code.** CI exists. The CD half of the claim stands — no deployment automation was observed |
| X2 | `architecture-views.md` §6 GAP-9: `AddRedis()` is registered in "eight services" | `AddRedis()` is registered in **nine**: `availability-service`, `customers-service`, `deliveries-service`, `identity-service`, `operations-service`, `ordermaker-service`, `orders-service`, `parcels-service`, `vehicles-service`. Only `pricing-service` omits it | **Follow the code.** Nine. The substance of GAP-9 — registered but unexercised — is unaffected |
| X3 | `docker-images.txt` lists SQL Server 2017, PostgreSQL, InfluxDB, Elasticsearch, Kibana, and Logstash as platform components | None is referenced by any service. `influxEnabled: false`, `elk.enabled: false` everywhere (§9.4) | **Follow the code.** Actual datastores are MongoDB and Redis; actual sinks are Prometheus and Seq. The file is a setup cookbook, not a manifest |
| X4 | The repository name `Pacco.Web` implies a web client is part of the platform | The repository tracks one file, `README.md`, containing `# Pacco.Web`. No frontend exists anywhere in the workspace (§7.1) | **Follow the code.** There is no frontend. Whether one exists outside this scope is `[unknown]` |
| X5 | `capability-baseline.md` CONFLICT-01: operation status durability | `operations-service` registers `AddMongo()` and configures a `mongo.database`, yet the only observed operation-state path is `IDistributedCache` (Redis) (§6.4) | **Follow the code.** Operation status is cached in Redis, not durably stored. Consistent with CONFLICT-01 |

### 11.4 Future or intended state, not implemented

Two things in the workspace describe intent rather than reality and are labelled as such:

**Future/Intended State (Not Implemented) — the extra datastores.** The SQL Server, PostgreSQL,
InfluxDB, and ELK entries in `docker-images.txt`, and the `influxEnabled` / `elk.enabled` flags that
exist in every service's configuration set to `false`. The wiring is present; nothing uses it.

**Future/Intended State (Not Implemented) — `ApproveOrder` in `ordermaker-service`.** The command
type is declared (`Commands/External/ApproveOrder.cs`) and never published (§5.1). Likewise
`AIOrderMakingHandler` consumes `ResourceReserved` and forwards it to the saga coordinator, but no
`ISagaAction<ResourceReserved>` exists to receive it.

**Unverifiable — Missing Source Evidence.** The `MakeOrder` command that starts the saga has no
observable publisher: `ordermaker-service` has no gateway route in any `ntrada*.yml`, and no other
service in the thirteen repositories publishes `MakeOrder`. The saga's entry point cannot be traced
from the available sources.

---

## 12. Architecture Views Summary

All architectural diagrams for the Pacco platform live in
**[`../architecture-views.md`](../architecture-views.md)**. That artifact is the single owner of the
platform's visual model; this baseline consumes it and does not modify, reconstruct, or reinterpret
it. No new diagrams have been created here.

### 12.1 What `architecture-views.md` contains

| Section | Diagram type | What it shows |
|---------|-------------|---------------|
| §1 | Context (C1) — six diagrams | The platform boundary and its actors, drawn from several perspectives |
| §2 | Service dependency graphs — two diagrams plus a replication graph | §2.1 synchronous HTTP dependencies; §2.2 asynchronous exchange topology; plus the `customers-service` replication fan-out |
| §3 | Runtime flow diagrams — five | End-to-end sequences (order making, reservation, delivery, identity, operation tracking). All five carry `[Confidence: partial]` |
| §4 | Deployment views | §4.1 the Docker Compose topology diagram; §4.2 Compose-vs-PM2 comparison; §4.3 the four gateway configurations; §4.4 a doc-vs-code conflict; §4.5 build and release |
| §5 | Data model — seven ER diagrams | One per service that owns a domain model |
| §6 | Gap register GAP-1 … GAP-25 | Everything the views could not establish |

Diagram edges there carry an explicit confidence marker — `[confirmed]`, `[inferred]`, or
`[unknown]` — so a reader can tell a proven edge from a probable one without leaving the diagram.

### 12.2 Embedded subset

Two diagrams are embedded below because §4 of this document depends on them directly. Both were
verified edge by edge against source configuration before embedding; both are reproduced **verbatim**
from `architecture-views.md`.

**Synchronous dependency graph** (`architecture-views.md` §2.1) — every HTTP call between services,
showing which callers go through Fabio and which do not.

```mermaid
graph LR
    gw["api-gateway<br/>repo Pacco.APIGateway"]
    fabio["Fabio 9999<br/>with Consul registry"]
    orders["orders-service"]
    avail["availability-service"]
    pricing["pricing-service"]
    om["ordermaker-service"]
    customers["customers-service"]
    parcels["parcels-service"]
    vehicles["vehicles-service"]
    deliveries["deliveries-service"]
    identity["identity-service"]
    ops["operations-service"]

    gw -->|"docker configs useLocalUrl false [confirmed]"| fabio
    gw -->|"non docker configs direct localUrl 5001-5009 [confirmed]"| orders
    fabio -->|"downstream routing to all 9 services [confirmed]"| orders
    orders -->|"httpClient.type fabio [confirmed]"| fabio
    avail -->|"httpClient.type fabio [confirmed]"| fabio
    pricing -->|"httpClient.type fabio [confirmed]"| fabio
    om -->|"httpClient.type empty so not load balanced [confirmed]"| vehicles
    om -->|"httpClient.type empty so not load balanced [confirmed]"| avail
    fabio -->|"GET parcel by id from orders [confirmed]"| parcels
    fabio -->|"GET vehicle by id from orders [confirmed]"| vehicles
    fabio -->|"GET pricing from orders [confirmed]"| pricing
    fabio -->|"GET customer state from availability [confirmed]"| customers
    fabio -->|"GET customer from pricing [confirmed]"| customers
    fabio -->|"gateway downstream only [confirmed]"| deliveries
    fabio -->|"gateway downstream only [confirmed]"| identity
    fabio -->|"gateway downstream only [confirmed]"| ops
```

**Asynchronous dependency graph** (`architecture-views.md` §2.2) — the eight topic exchanges with
every publisher and subscriber, preserving the broker as an explicit hop so that no service appears
to call another asynchronously.

```mermaid
graph LR
    identity["identity-service"]
    customers["customers-service"]
    avail["availability-service"]
    vehicles["vehicles-service"]
    orders["orders-service"]
    parcels["parcels-service"]
    deliveries["deliveries-service"]
    om["ordermaker-service"]
    gw["api-gateway async mode only"]
    ops["operations-service"]

    xid["identity exchange"]
    xcu["customers exchange"]
    xav["availability exchange"]
    xve["vehicles exchange"]
    xor["orders exchange"]
    xpa["parcels exchange"]
    xde["deliveries exchange"]
    xom["ordermaker exchange"]

    gw -->|"20 write routes as commands [confirmed]"| xav
    gw -->|"commands [confirmed]"| xcu
    gw -->|"commands [confirmed]"| xde
    gw -->|"commands [confirmed]"| xor
    gw -->|"commands [confirmed]"| xpa
    gw -->|"commands [confirmed]"| xve
    identity -->|"signed_up signed_in [confirmed]"| xid
    customers -->|"customer_created became_vip state_changed [confirmed]"| xcu
    avail -->|"resource_reserved and 4 more [confirmed]"| xav
    vehicles -->|"vehicle_added updated deleted [confirmed]"| xve
    orders -->|"9 events and 11 rejected events [confirmed]"| xor
    parcels -->|"parcel_added parcel_deleted [confirmed]"| xpa
    deliveries -->|"delivery_started completed failed registration_added [confirmed]"| xde
    om -->|"make_order_completed make_order_rejected [confirmed]"| xom
    om -->|"5 commands onto another service exchange [confirmed]"| xor
    om -->|"reserve_resource onto another service exchange [confirmed]"| xav

    xid -->|"signed_up [confirmed]"| customers
    xcu -->|"customer_created [confirmed]"| avail
    xcu -->|"customer_created [confirmed]"| orders
    xcu -->|"customer_created [confirmed]"| parcels
    xve -->|"vehicle_deleted [confirmed]"| avail
    xav -->|"resource_reserved and reservation_canceled [confirmed]"| orders
    xav -->|"resource_reserved [confirmed]"| om
    xpa -->|"parcel_deleted [confirmed]"| orders
    xor -->|"order_completed [confirmed]"| customers
    xor -->|"order_canceled deleted parcel_added_to_order parcel_deleted_from_order [confirmed]"| parcels
    xor -->|"order_created approved parcel_added_to_order vehicle_assigned_to_order [confirmed]"| om
    xde -->|"delivery_started completed failed [confirmed]"| orders
    xom -->|"all 8 exchanges observed [confirmed]"| ops
    xid -->|"observed [confirmed]"| ops
    xor -->|"observed [confirmed]"| ops
```

For the context views, the five runtime flows, the Compose deployment topology, and the seven ER
diagrams, see **[`../architecture-views.md`](../architecture-views.md)** directly — they are not
duplicated here.

### 12.3 Consistency check against the other artifacts

Runtime relationships were compared across this narrative, `architecture-views.md`,
`capability-baseline.md`, and the two embedded diagrams. Agreement was found on: the four
synchronous callers and their targets; `ordermaker-service` as the sole non-Fabio caller; the eight
exchanges and their owners; `ordermaker-service` as the sole cross-exchange publisher;
`operations-service` subscribing to all eight exchanges; and the `customer_created` replication
fan-out to three services.

Three inconsistencies were found and are **surfaced, not reconciled**: the Travis CI claim in
`architecture-views.md` §4.5 (X1), the eight-versus-nine `AddRedis()` count in its GAP-9 (X2), and
the `availability` → `ordermaker` `resource_reserved` edge, which is a correct subscription but
carries no saga behaviour (§5.1). All three appear in the closing section.

One caveat on the embedded synchronous graph, stated so a reader does not over-read it: it draws
`gw --> orders` as the illustrative direct-`localUrl` edge, but `ntrada.yml` declares `localUrl`
pairs for **all nine** routed services (`5001`–`5009`, §4.3). The diagram under-draws that fan-out.
That is incompleteness rather than an incorrect edge, so it does not disqualify the diagram from
being embedded — but "only `orders-service` is reached directly" would be a wrong reading of it.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything in this baseline that is not directly proven by code or configuration is collected
> here. Each assumption states what was taken as true and what breaks if it is wrong. Each blocker
> and open question is tagged either **[ACTION NOW]** — someone must answer it before the next
> stage can rely on this document — or **[handled later by \<stage\>]**, meaning a named later
> stage will resolve it and no action is needed now.

### Assumptions

| ID | Assumption | Why it was needed | Impact if wrong | Validation Path |
|----|------------|-------------------|-----------------|-----------------|
| A1 | Convey's `AddRedis()` makes `IDistributedCache` resolve to Redis rather than to an in-process memory cache | Underpins §6.4's finding that operation status lives in Redis, and therefore the whole description of the async completion path in §5.4 | Operation status would be per-instance and lost on every restart, making the async gateway mode far less reliable than §4.3 describes | Read the `Convey.Persistence.Redis` 0.4.x registration code, or run `operations-service`, drive one operation, and check for an `operations:` prefixed key in Redis |
| A2 | `Chronicle_` 3.2.1 registered with a bare `AddChronicle()` and no `Chronicle.Persistence.*` package keeps saga state in memory | §5.2's statement that an in-flight order-making saga does not survive a restart of `ordermaker-service` | Saga state would in fact be durable, and the reliability concern in §5.2 disappears | Read the `Chronicle_` 3.2.1 package source for its default `ISagaStateRepository` registration, or restart `ordermaker-service` mid-saga and observe whether the saga resumes |
| A3 | The Docker Compose stacks and the two PM2 manifests describe developer environments, not production | Needed because no manifest in the workspace declares its target environment, and §9 rests on knowing what it is looking at | Production would be running single-node MongoDB, RabbitMQ, Redis, Consul, and Vault with a `docker.for.win.localhost` Consul address and an `env: local` metrics label — a materially different risk picture | Ask a platform operator which manifest deploys production, or obtain the production deployment definition. Resolves together with B4 |
| A4 | A message with no subscriber anywhere in the thirteen cloned repositories genuinely has no consumer | Basis for §6.3's finding that `customer_state_changed` and `became_vip` are unconsumed, and for the unconsumed-event gaps carried from `architecture-views.md` | Those findings would be false, and consumers would exist in code outside this workspace | Inspect queue bindings for the `customers`, `vehicles`, and `deliveries` exchanges on the running broker; a binding with no repository behind it proves a consumer outside this scope |
| A5 | `outbox.disableTransactions: true` is set because the Compose stack runs a single-node MongoDB, which cannot serve multi-document transactions | Explains §10.4's finding that the outbox loses its atomicity guarantee, rather than leaving it unexplained | The flag would be a deliberate performance choice rather than an environment limitation, and would not automatically become safe to flip in a replica-set environment | Confirm whether any environment runs MongoDB as a replica set, then test whether `disableTransactions: false` works there |
| A6 | The static page at `Pacco.Services.Operations.Api/wwwroot/ui/` is a developer test harness, not the platform's intended user interface | §7.3 characterises it as a harness rather than treating it as Pacco's frontend architecture | Pacco's real UI would be a hand-pasted-JWT page hard-coded to `http://localhost:5005`, which would change the entire security and usability picture of §8 | Ask the original author or a platform owner what the page was built for; check whether any deployment serves it to end users |
| A7 | `Pacco.Web` appearing in the clone set means a frontend is in scope for this platform even though the repository holds no source | Kept the frontend visible as an explicit gap in §7 rather than dropping it silently | A frontend exists that this baseline does not describe at all, and §7's central finding would be wrong | Check `Pacco.Web`'s commit history and default branch upstream, and ask the scope owner for issue 12998 whether a web client exists elsewhere. See Q14 |
| A8 | The absence of any external base URL, API key, or outbound client target outside the `httpClient.services` maps means Pacco integrates with no third-party system | §2.1's statement that Pacco as cloned is a closed system | Undocumented external integrations exist, and the system boundary in §2.1 is drawn too tightly | Search for outbound network calls at runtime, or ask a platform owner whether any payment, carrier, notification, or external identity integration exists |
| A9 | The `role` claim that `identity-service` issues is the same claim the gateway checks in its `claims: { role: admin }` route gates | Needed to describe the admin gates in §8.2 as effective controls | Admin-only routes would either reject everyone or admit everyone, and §8.2's description of edge authorization would not hold | Decode a token minted by `identity-service` for an admin user and confirm the claim name and value match the `auth.claims.role` mapping at `ntrada.yml:4-5` |

### Blockers

**On the Owner column.** No repository in this workspace records a named owner for anything — there
is no `CODEOWNERS`, `CONTRIBUTING.md`, or team manifest in any of the thirteen clones (see Q18). The
Owner column therefore names the **role that must supply the answer**, and assigning a person to that
role is itself part of resolving the blocker. Target dates are **proposed** by this stage relative to
the analysis date of 2026-09-04; none is an agreed commitment.

| ID | Blocker | Blocks | Why it blocks | Owner | Resolution Path | Target Date |
|----|---------|--------|---------------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Four gateway routing configurations exist (`ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml`) and nothing records which one a real environment loads via `NTRADA_CONFIG`. The sync and async families are behaviourally different systems, not environment variants | §4.3 and §4.4 in full; whether `operations-service` is on the critical path or merely observational; whether twenty business writes are synchronous or fire-and-forget; the runtime flows in `architecture-views.md` §3 | The platform's edge behaviour has two mutually exclusive readings and this document cannot choose between them. Every impact analysis on a write path depends on the answer | Platform owner / operator for the Pacco runtime (no named individual recorded — see Q18) | Read the effective `NTRADA_CONFIG` value in each running environment and record which `ntrada*.yml` is authoritative per environment | 2026-09-11 (proposed) |
| B2 | **[ACTION NOW]** Five Vault unseal keys and an initial Vault root token are committed in plaintext in `hianshul100_Pacco/docker-images.txt`, and the gateway's symmetric JWT `issuerSigningKey` is committed at `ntrada.yml:44` | Any security review of §8; whether the Vault controls in §8.5 are real or nominal; whether the edge authentication in §8.2 can be trusted at all | Anyone holding the `issuerSigningKey` can mint a token `api-gateway` accepts, including one carrying `role: admin`. This is a live exposure independent of any architecture work | Security owner for the Pacco platform (no named individual recorded — see Q18) | Confirm whether the committed values are throwaway local-development credentials; if they are not, rotate the Vault root token and unseal keys and the JWT signing key, and purge them from history | 2026-09-08 (proposed) — treat as immediate if the values are live |
| B3 | **[ACTION NOW]** Nobody in the available sources can say whether `ordermaker-service` runs in any real environment. It has no route in any `ntrada*.yml`, is absent from both PM2 manifests, has no observable publisher for its `MakeOrder` entry command, and can only be reached by publishing directly onto the `ordermaker` exchange | §5.1 and §5.2 in full; the saga description that four other services participate in; the `ordermaker-service` edges in both embedded diagrams | The platform's only orchestrated process is described as current behaviour. If it is dead code, a substantial part of this baseline describes something that never runs | Platform owner / operator for the Pacco runtime (no named individual recorded — see Q18) | Confirm whether `ordermaker-service` is deployed in any environment, and inspect the `ordermaker` exchange on the running broker for a publisher; if it is deployed but unreachable, record it as dead code | 2026-09-11 (proposed) |
| B4 | **[handled later by runtime and deployment validation]** No production runtime manifest exists in any repository — no Kubernetes, Helm, Terraform, Kustomize, or service-mesh configuration | Every scaling, availability, replica-count, probe, and network-policy statement in §9.2; assumption A3; any operational review that this document is meant to support | Runtime shape cannot be established from source repositories at any confidence, because the sources simply do not contain it | The runtime and deployment validation stage (owner assigned when that stage is scheduled) | Obtain read access to the deployment platform itself and re-derive the topology there, rather than from source repositories | n/a — resolves when the runtime and deployment validation stage runs; no action required from this stage |
| B5 | **[handled later by architecture views generation]** `architecture-views.md` §4.5 states no CI pipeline definition exists in any repository, which the code contradicts — eleven `.travis.yml` files define a build-test-dockerize pipeline (§9.5, X1). Its §6 GAP-9 also states `AddRedis()` is registered in eight services where the code shows nine (X2) | Trust in `architecture-views.md` as a reference alongside this document; any consumer that reads both and finds them disagreeing | Two artifacts in the same inventory make contradictory factual claims. This stage consumes `architecture-views.md` and must not modify it, so the contradiction can only be surfaced here, not fixed here | The architecture views generation stage, which owns `architecture-views.md` | Re-run or amend the architecture views generation stage so that §4.5 records the eleven Travis pipelines and GAP-9 records nine `AddRedis()` registrations, matching `repo-inventory.md` §2.3 and §9.5 of this document | n/a — resolves when the architecture views generation stage next runs; no action required from this stage |

### Open Questions

**On the Decision Owner column.** As with the Blockers table, no named individual is recorded
anywhere in the workspace (Q18). Decision Owner names the **role empowered to settle the question**;
naming a person to it is part of answering. A *Proposed Answer* is this stage's best reading of the
evidence, offered so the owner can confirm or reject rather than start from nothing — it is **not** a
finding, and nothing elsewhere in this baseline rests on it.

| ID | Question | Why it matters | Proposed Answer (if any) | Decision Owner | Where the answer would come from |
|----|----------|----------------|--------------------------|----------------|----------------------------------|
| Q1 | **[ACTION NOW]** Does order-making saga state survive a restart of `ordermaker-service`? `Chronicle_` 3.2.1 is registered with a bare `AddChronicle()` and no persistence package | An order half-way through the seven-step sequence in §5.1 would be silently abandoned — the customer's order exists, a vehicle is assigned, and nothing ever completes or compensates it | Likely not durable. The absence of a `Chronicle.Persistence.*` reference is the whole basis for this; no observed behaviour confirms it | Platform owner, or the original author of `ordermaker-service` | Reading the `Chronicle_` 3.2.1 default registration, or a restart test against a running saga |
| Q2 | **[ACTION NOW]** Is losing operation status acceptable? It is held only in Redis with `requests.expirySeconds: 300`, and in async gateway mode it is the caller's only channel for learning an outcome | A caller who gets `202 Accepted` and waits longer than five minutes, or who is connected when Redis restarts, never learns whether their order succeeded | Probably an accepted design choice for a status projection — the expiry is explicitly configured rather than defaulted, and `operations-service` writes no domain state. Whether five minutes suits the real order lifecycle is the actual question | Platform owner, or the original author of `operations-service` | A platform owner or the original author |
| Q3 | **[handled later by data and integration analysis]** Why do eight services register `AddRedis()` and never inject `IDistributedCache`? Only `operations-service` uses it | Determines whether Redis is a genuine runtime dependency of nine services or of one. §6.4 draws the narrow reading; a wrong call here mis-scopes every Redis outage impact assessment | Likely vestigial copy-paste across service templates, possibly with Convey using the connection internally for inbox/outbox. Not verified either way | Data and integration analysis stage, with the platform owner | Reading the Convey 0.4.x inbox/outbox implementation, or observing Redis key traffic per service at runtime |
| Q4 | **[ACTION NOW]** Is it deliberate that commands issued by the saga carry an empty user context, so every ownership guard in the platform is skipped? The guards are written `if (identity.IsAuthenticated && …)` and therefore fail open | An unauthenticated message reaching a handler bypasses authorization entirely. §5.3 shows the saga path does exactly this, and §8.3 shows the same guard shape in four handlers | Probably an intentional "trusted internal caller" pattern — the mechanism to carry identity exists (`CorrelationContext.UserContext`) and is simply not populated on this path. But nothing states the intent, and the same failure mode applies to any direct exchange publish | Security owner, with the platform owner | A security owner's ruling on whether internal messages are trusted, plus a decision on whether direct broker access is possible in a real environment |
| Q5 | **[handled later by data and integration analysis]** Nothing consumes `customer_state_changed` or `became_vip`. The three customer replicas are built from `customer_created` and never updated. There is no reconciliation mechanism at all | `orders-service`, `availability-service`, and `parcels-service` can act on permanently stale customer data, and a single missed `customer_created` leaves a replica inconsistent forever | Likely an oversight rather than a design: both events are published and declared, yet no handler exists in any of the thirteen repositories. Services that need current state fetch it synchronously instead, which suggests the replica was only ever meant for existence checks | Data and integration analysis stage, with the owners of the three subscribing services | A stage that examines data flow and consistency requirements across services |
| Q6 | **[ACTION NOW]** Should `pricing-service` be in `customers-service`'s certificate ACL? It calls `customers-service` over HTTP but the ACL grants `customers:read` only to `availability-service` | Either a legitimate caller is missing a grant, or certificate enforcement is not actually in effect — and §8.4 cannot tell which | No confident reading. The two possibilities are indistinguishable from configuration alone | Security owner for the Pacco platform | Calling `customers-service` from `pricing-service` in a running environment and observing whether the certificate check rejects it |
| Q7 | **[ACTION NOW]** Is client-certificate authentication actually enforced anywhere? Exactly one of ten services sets `security.certificate.enabled: true`; `availability-service` has a header name and nothing else; the other eight sections are empty | Determines whether §8.4 describes a working control or a single-service experiment. "Pacco uses mTLS between services" would be a false statement either way, but the degree matters | Likely a partial, experimental rollout — Vault PKI is provisioned for every service (each has a `pki.roleName` and a `<service>.pacco.io` common name) while verification is switched on in one place | Security owner for the Pacco platform | A security owner, or a runtime test presenting no certificate to `customers-service` |
| Q8 | **[ACTION NOW]** What publishes `MakeOrder`, the command that starts the order-making saga? No gateway route targets `ordermaker-service`, and no service in the thirteen repositories publishes it | Without an entry point, the platform's only orchestrated process has no traceable trigger, and §5.1 describes a sequence nothing is known to start. Directly compounds B3 | None. This is an *Unverifiable — Missing Source Evidence* case, not an inference gap | Platform owner / operator for the Pacco runtime | Inspecting publishers on the `ordermaker` exchange in a running broker, or a client outside these thirteen repositories |
| Q9 | **[handled later by data and integration analysis]** What are the wire payloads of the roughly 80 messages? `api-gateway` emits them via `System.Reflection.Emit` from `messages.json`, so no compiled contract exists | Any consumer or producer change is unguarded by a build-time contract (§3.4, C2). Without payload shapes, message-level impact analysis is guesswork | None. The shapes are recoverable only by reading `messages.json` against each consuming service's command class | Data and integration analysis stage | Cross-reading `messages.json` with each service's command and event classes, or capturing live messages from the broker |
| Q10 | **[ACTION NOW]** Do the SignalR hub and the gRPC `SubscribeOperations` stream scope delivery per caller, or broadcast all operations to every connected client? | An unscoped broadcast would leak every customer's operation activity to every connected client — a data exposure, not just a design question | None. The static test page in §7.2 passes a JWT to `initializeAsync` after connecting, which suggests per-caller scoping is intended, but the hub's delivery logic was not established | Security owner, with the owner of `operations-service` | Reading `PaccoHub` and the gRPC service implementation in full, or connecting two clients and observing cross-delivery |
| Q11 | **[handled later by test and quality analysis]** How does the Pact file get from `orders-service` (consumer) to `parcels-service` (provider)? `Pactify` 1.1.0 is used with no Pact broker | The platform's only cross-service contract test may not actually verify anything if the provider never receives the consumer's pact | Likely a manually copied file or a test that only runs on the consumer side. Not established | Test and quality analysis stage | Reading the two repositories' test projects and CI scripts together, or asking the original author |
| Q12 | **[ACTION NOW]** Which deployment path is authoritative — Docker Compose or PM2 — and why does `prod-services.yml` omit `ordermaker` while `compose/services.yml` includes it? | §9.1 describes two parallel paths and cannot say which one describes reality. The omission is also the strongest single piece of evidence behind B3 | Possibly successive stages rather than alternatives, with the PM2 manifests predating the Compose stacks. The `ordermaker` omission would then simply be an un-updated file | Platform owner / operator for the Pacco runtime | A platform owner, or the deployment history of any real environment |
| Q13 | **[ACTION NOW]** Is `outbox.disableTransactions: true` intentional? With it set, the domain write and the outbox write are not atomic, which removes the guarantee the outbox pattern exists to provide | A crash between the two writes loses the event while keeping the state change. Every eventual-consistency claim in §6.3 rests on the outbox actually working | Likely forced by the single-node MongoDB in the Compose stack (A5), since MongoDB transactions require a replica set. If so, flipping it in a replica-set environment would be safe — but that has not been tested | Platform owner, with the data and integration analysis stage | Confirming whether any environment runs a MongoDB replica set, then testing `disableTransactions: false` there |
| Q14 | **[handled later by frontend and UI analysis]** Does a real Pacco frontend exist outside these thirteen repositories? `Pacco.Web` tracks only a one-line `README.md`, and no framework, bundler, or view engine exists anywhere in the workspace | §7's central finding — that the platform has no frontend — is scope-bound. If a client exists elsewhere, the entire UI section is describing an absence that is not real | Likely no frontend exists at all: the platform's only HTML is a test harness, and every gateway route returns JSON. But an empty repository is equally consistent with a client that lives elsewhere | Frontend and UI analysis stage, with the scope owner for issue 12998 | The `Pacco.Web` upstream commit history and default branch, plus the scope owner's confirmation. Resolves together with A7 |
| Q15 | **[ACTION NOW]** Is the unpublished `ApproveOrder` command and the unbound `ISagaAction<ResourceReserved>` dead code, or an incomplete implementation? | If incomplete, the saga's approval step is meant to work differently from the step-6 side effect described in §5.1, and the sequence in this document describes an accident rather than a design | Likely superseded: `orders-service` already approves the order on `resource_reserved` (`ResourceReservedHandler.cs:32`), so the saga's own approval path became redundant and was left behind | Original author of `ordermaker-service`, or the platform owner | Reading the commit history of `AIOrderMakingSaga.cs` and `ResourceReservedHandler.cs` together |
| Q16 | **[handled later by data and integration analysis]** What happens to a message whose handler throws? No dead-letter exchange, retry policy, or poison-message handling was observed anywhere | A permanently failing message would be redelivered indefinitely or dropped, and neither behaviour is documented. Affects every one of the roughly 80 messages | None. RabbitMQ defaults would apply, but no `deadLetter` configuration exists in any service to confirm what those are here | Data and integration analysis stage, with the platform owner | Reading the Convey 0.4.x RabbitMQ subscriber implementation, or inspecting queue arguments on a running broker |
| Q17 | **[ACTION NOW]** Why does `identity-service` set `validateIssuer: false` and `validateAudience: false` while every domain service sets `validateIssuer: true` with `validIssuer: "pacco"`? | The token issuer applies weaker validation than the token consumers. Combined with the gateway validating by symmetric key while services validate by certificate (§8.6), the platform has three different validation postures | Possibly because `identity-service` primarily issues rather than validates, and its own protected endpoints are few. Not stated anywhere | Security owner for the Pacco platform | A security owner's review of the three `jwt` configurations side by side |
| Q18 | **[ACTION NOW]** Who owns each service? No `CODEOWNERS`, `CONTRIBUTING.md`, team manifest, or maintainer record exists in any of the thirteen repositories | Every blocker and question above names a *role* rather than a person, because no person can be named from these sources. Nothing here can actually be assigned until this is answered | None. The information is genuinely absent from the workspace, not merely unretrieved | Engineering leadership for the Pacco platform | An engineering directory, a team-to-service mapping, or the backlog owner for issue 12998 |
