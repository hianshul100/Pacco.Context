# ADR Candidates Backlog
**Total candidates: 20**

**Project:** Pacco — Common Architecture
**Stage:** `architecture_discovery` — ADR candidate identification
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`
**Date of analysis:** 2026-09-09

## Scope and adoption of existing artifacts

No Architecture Decision Record exists anywhere in scope, so **every candidate below is a new
decision record, not a re-proposal of a settled one**. Two independent checks establish this:

1. **File system.** There is no `docs/adr/`, `adr/`, or `decisions/` directory, and no file matching
   `*adr*`, `*decision-record*`, or `*rfc*`, in any of the thirteen cloned repositories or in this
   artifact repository. The complete Markdown set here is `README.md`, the
   `docs/architecture-inventory/` baselines and views, the twelve `repo-summary/` files, the twelve
   `component-internals/` models, and the twenty-seven `patterns/` files. None is an ADR.
2. **Governance catalog.** The knowledge graph for tenant `Q5SCXYFS` was queried for ADR, Decision,
   and Constraint nodes, and then for an unscoped node count. Both returned zero rows, and the corpus
   search returned no matching material. The catalog holds no governing decision for this platform.

This matches `baselines/architecture-baseline.md` §11.1 and `patterns/index.md`, which record the same
absence and which is why every catalogued pattern carries an empty **Related ADRs** entry. Because no
ids or prefixes are established, this backlog introduces the `ADR-CANDIDATE-NNN` id space; the
`adr_generation` stage owns the mapping to final `ADR-NNN` ids.

Source code is the source of truth throughout. Where a baseline document and the code disagreed, the
code was followed and the disagreement is named in the affected candidate's **Evidence** rather than
reconciled silently.

## Candidate Summary Table

| ID | Title | Category | Impact | Dependencies | Priority |
|----|-------|----------|--------|--------------|----------|
| ADR-CANDIDATE-001 | Event-driven microservices with service-owned topic exchanges | Integration | high | — | first-batch |
| ADR-CANDIDATE-002 | Convey as the platform standard in place of a shared internal library | Other | high | — | first-batch |
| ADR-CANDIDATE-003 | Message contracts by naming convention, with duplicated consumer DTOs | Integration | high | 001, 002 | second-batch |
| ADR-CANDIDATE-004 | A declarative configuration-driven gateway as the single north-south edge | Integration | high | 001 | first-batch |
| ADR-CANDIDATE-005 | Dual-mode edge writes selected by configuration rather than by code | Integration | high | 001, 004 | second-batch |
| ADR-CANDIDATE-006 | Authentication enforced at the edge, with fail-open in-service authorization | Security | high | 004 | first-batch |
| ADR-CANDIDATE-007 | Split JWT trust root between the gateway and the domain services | Security | high | 006 | second-batch |
| ADR-CANDIDATE-008 | Database per service on MongoDB, hand-mapped, with no migration tooling | Storage | high | 001, 002 | first-batch |
| ADR-CANDIDATE-009 | Event-carried customer replicas in place of synchronous customer reads | Storage | medium | 003, 008 | backlog |
| ADR-CANDIDATE-010 | Narrow synchronous point-reads as the bounded exception to messaging | Integration | medium | 001, 008 | backlog |
| ADR-CANDIDATE-011 | An orchestrated saga for order creation, and its state durability | Orchestration | high | 001, 003 | second-batch |
| ADR-CANDIDATE-012 | Transactional outbox and inbox applied by handler decorator | Storage | medium | 001, 008 | second-batch |
| ADR-CANDIDATE-013 | Contract-blind universal subscription by runtime type emission | Integration | medium | 003 | second-batch |
| ADR-CANDIDATE-014 | Ephemeral operation status and the real-time client notification channels | Orchestration | medium | 005, 013 | second-batch |
| ADR-CANDIDATE-015 | Registry-mediated service discovery and name-based routing | Networking | medium | 001, 002 | second-batch |
| ADR-CANDIDATE-016 | A secret store for dynamic database credentials and per-service PKI | Security | high | 015 | second-batch |
| ADR-CANDIDATE-017 | Container-compose and process-manager deployment, with no production orchestration | Deployment | high | 001, 015 | second-batch |
| ADR-CANDIDATE-018 | Repository per service, with independent per-repository release | Deployment | medium | 002, 017 | backlog |
| ADR-CANDIDATE-019 | Two internal service structures — layered domain split and single project | Other | medium | 002 | backlog |
| ADR-CANDIDATE-020 | .NET Core 3.1 as the platform runtime baseline | Other | high | 002 | backlog |

## Candidate Details

### ADR-CANDIDATE-001: Event-driven microservices with service-owned topic exchanges
**Category:** Integration
**Impact:** high
**Why this is an ADR:** The platform integrates ten deployables primarily by asynchronous messaging
rather than by synchronous calls, and it partitions that messaging so each service owns exactly one
topic exchange named after itself, publishes only its own messages, and binds consumer queues named
for the consuming service. The alternatives were live and are all visible as roads not taken: a
single shared exchange or bus for the whole platform, request/response RPC between services, or a
modular monolith with in-process calls. The trade-off taken is loose runtime coupling and independent
deployability, paid for with eventual consistency, no cross-service transaction, and failure modes
that only appear in the message topology. The lasting impact is that this choice fixes the shape of
every other integration candidate in this backlog: it is why contracts must be versioned by
convention (003), why the edge can publish instead of proxy (005), why data must be replicated (009),
and why a saga is needed at all (011).
**Evidence:** Eight topic exchanges carrying roughly 80 distinct messages, catalogued in
`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`;
per-service `rabbitMq` sections declaring the exchange, `snakeCase` conventions and a
`<service>/{{exchange}}.{{message}}` queue template in every service `appsettings.json`;
cross-service subscription handlers under `Events/External/Handlers/` in Availability, Customers,
Orders, Parcels and OrderMaker. Consolidated in `repo-inventory.md` §3.2,
`baselines/architecture-baseline.md` §1.2 and §4.2, and
`patterns/integration/service-owned-topic-exchange-messaging.md`. `pricing-service` is the single
non-participant — it references no broker package and has no `rabbitMq` section — which is itself
evidence that participation was a per-service choice rather than an unavoidable default.
**Dependencies:** None. This is the root decision of the integration branch of the backlog.
**Recommended priority:** first-batch

### ADR-CANDIDATE-002: Convey as the platform standard in place of a shared internal library
**Category:** Other
**Impact:** high
**Why this is an ADR:** Cross-cutting capability — command and query dispatch, message brokering,
persistence, discovery registration, secret loading, tracing, metrics — is supplied by one external
toolkit composed per service, and there is deliberately **no first-party shared library and no shared
code repository** anywhere in the workspace. The alternative, a `Pacco.Common` package owning these
concerns, is the obvious counterfactual and was not taken. The trade-off is real in both directions:
no shared package means no coordinated release and no single point of breakage, but it also means the
platform's uniformity is enforced by copying a composition root into each repository rather than by a
dependency, so drift has no build-time signal. The lasting impact is that the toolkit's version line
is the platform's real architectural standard, and that upgrading it is a per-repository act.
**Evidence:** Convey 0.4.\* package references in every service and gateway `.csproj`; a near-identical
`Infrastructure/Extensions.cs` composition root and identical `Decorators/`, `Contexts/`,
`Exceptions/`, `Logging/` and `Mongo/` folder structure across the layered services. The absence of a
shared package is recorded as an enumerated finding in `repo-inventory.md` §3.3 ("no first-party
shared library and no shared code repository"), `baselines/architecture-baseline.md` §1.2 and §3.4,
and `patterns/other/framework-supplied-platform-conventions.md`.
**Dependencies:** None. This is the root decision of the platform-convention branch of the backlog.
**Recommended priority:** first-batch

### ADR-CANDIDATE-003: Message contracts by naming convention, with duplicated consumer DTOs
**Category:** Integration
**Impact:** high
**Why this is an ADR:** Because there is no shared contract package (002), a message consumed across
a service boundary is redeclared as an independent class inside the consumer, and the only binding
contract between publisher and consumer is the `snake_case` message name derived from the type. The
alternatives were a shared contracts package, a schema registry with a compatibility policy, or
generated clients from a published schema. The trade-off is that publishers and consumers never block
each other at build time, at the cost that a rename or a field change in one service breaks the other
silently at runtime with no compile-time or CI signal. This is recorded in the baseline as constraint
C2, whose "what it prevents" column reads *nothing*. The lasting impact is that every future contract
change on this platform is an unguarded change, which is precisely the condition an ADR should record
and bound.
**Evidence:** `CustomerCreated` exists as four independent C# classes across Customers, Availability,
Orders and Parcels; `ResourceReserved` exists independently in Availability, Orders and OrderMaker
(`repo-inventory.md` §3.3). The platform-wide catalogue in `messages.json` is a hand-maintained copy
of names owned by eight other repositories with no generation or validation step. The only verification
mechanism found is a consumer-driven contract test pair between `Pacco.Services.Orders`
(`tests/Pacco.Services.Orders.PactConsumerTests`) and `Pacco.Services.Parcels`
(`tests/Pacco.Services.Parcels.PactProviderTests`), and no broker configuration exists in either
repository or either Travis file to carry the pact between them —
`patterns/testing/consumer-driven-contract-test-pair.md` rates this Weak for that reason. Constraint
C2 is stated in `baselines/architecture-baseline.md` §11.2.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-002
**Recommended priority:** second-batch

### ADR-CANDIDATE-004: A declarative configuration-driven gateway as the single north-south edge
**Category:** Integration
**Impact:** high
**Why this is an ADR:** The entire north-south edge — route table, authentication, claim gates,
payload transforms and downstream forwarding — is expressed as a YAML document interpreted by an
embedded gateway library, not as gateway code. The alternatives were a hand-written backend-for-frontend,
a code-configured gateway, or exposing services directly. The trade-off is that edge changes become
configuration changes deployable without a build, at the cost that the edge's behaviour is invisible to
the compiler, untestable by the service test suites, and expressed in a dialect specific to one library
whose version the platform is pinned to. The lasting impact is that the gateway configuration files
are the authoritative description of the platform's public surface, so the decision determines where
external contract review has to happen.
**Evidence:** `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Program.cs` is a host that embeds
Ntrada 0.4.\* and selects its configuration by the `NTRADA_CONFIG` environment variable; the modules
in `ntrada.yml` expose `/availability`, `/customers`, `/deliveries`, `/identity`, `/operations`,
`/orders`, `/parcels`, `/pricing` and `/vehicles`; the repository contains no controller and no route
code. Consolidated in `repo-inventory.md` §2.1 and §2.3, `baselines/api-inventory.md` §2, and
`patterns/integration/declarative-configuration-driven-api-gateway.md`.
**Dependencies:** ADR-CANDIDATE-001
**Recommended priority:** first-batch

### ADR-CANDIDATE-005: Dual-mode edge writes selected by configuration rather than by code
**Category:** Integration
**Impact:** high
**Why this is an ADR:** The same edge route can either proxy a write synchronously to a service or
publish it as a command onto that service's exchange, and which of the two happens is chosen by which
of four gateway configuration files is loaded. Twenty write routes across six services change transport
between the synchronous and asynchronous pairs. This is a decision about the platform's write
semantics disguised as a configuration switch: the synchronous mode gives callers a result and a
status code, the asynchronous mode gives them an acknowledgement and a correlation id and defers the
outcome to a separate channel (014). Those are different API contracts for the same URL. The
alternative — pick one mode and commit the edge to it — was available and not taken. The lasting
impact is that no client can be written against the edge without knowing which configuration is
deployed, and today **nothing in the workspace states which one production uses**.
**Evidence:** Four gateway configurations exist and differ architecturally rather than by hostname:
the asynchronous pair converts twenty write routes from downstream HTTP calls to broker publications
(`baselines/api-inventory.md` §2 versus §3). The Compose service stack in `hianshul100_Pacco` selects
the asynchronous Docker variant through `NTRADA_CONFIG`, and no document states production intent —
recorded as gap G6 in `repo-inventory.md` §6 and as the four-configuration problem in
`baselines/architecture-baseline.md` §4.3. Pattern:
`patterns/integration/dual-mode-edge-write.md`.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-004
**Recommended priority:** second-batch

### ADR-CANDIDATE-006: Authentication enforced at the edge, with fail-open in-service authorization
**Category:** Security
**Impact:** high
**Why this is an ADR:** The platform places its authentication boundary at the gateway: routes are
marked as requiring authentication, admin-only routes are gated on a role claim, and the caller's
identity claim is bound into the forwarded URL or message payload so services need not trust the body.
Behind that boundary, services re-check ownership in their handlers, but the guard is written so that
it only applies when the caller is already authenticated — an unauthenticated caller fails the first
term and the check is skipped, so the operation proceeds. The alternative placements were full
enforcement in each service, or a mesh/sidecar enforcing identity on every hop. The trade-off taken is
one place to reason about authentication and no duplicated auth code, paid for with a hard dependency
on every request having traversed the gateway. The lasting impact is that any path that reaches a
handler without passing the gateway bypasses authorization entirely, and two such paths exist today:
a direct publication onto a service exchange, and a direct call to a service's published container
port. This is a decision with a currently-unbounded consequence, which is exactly what an ADR is for.
**Evidence:** Per-route authentication flags and role-claim gates on five routes in the gateway
configurations, plus claim-to-URL binding (`baselines/api-inventory.md` §7.2). The fail-open guard
shape appears identically in `Orders.Application/Commands/Handlers/AddParcelToOrderHandler.cs`,
`AssignVehicleToOrderHandler.cs`, `ApproveOrderHandler.cs` and in `availability-service`'s
`ReserveResourceHandler.cs`; `CreateOrderHandler.cs` carries no identity check at all. Analysed in
`baselines/architecture-baseline.md` §8.2–§8.3. The saga in `ordermaker-service` constructs an empty
user context and so acts with no caller identity (`repo-inventory.md` §2.3), demonstrating the bypass
concretely. Pattern:
`patterns/security/edge-enforced-authentication-with-identity-binding.md`.
**Dependencies:** ADR-CANDIDATE-004
**Recommended priority:** first-batch

### ADR-CANDIDATE-007: Split JWT trust root between the gateway and the domain services
**Category:** Security
**Impact:** high
**Why this is an ADR:** Tokens are issued by one service, but the gateway and the domain services do
not validate them the same way: the gateway validates with a symmetric shared secret held in its
configuration, while the domain services validate against a certificate on disk. A token accepted by
one is therefore not automatically accepted by the other. The alternatives — one asymmetric key pair
with a published verification key, or a single shared symmetric secret everywhere — are the two
coherent designs this sits between. The trade-off is not visible as a benefit anywhere in the code,
which is what makes it worth recording: an ADR here either ratifies the split with a stated reason or
converges the two, and until one of those happens no service can be added without guessing which
trust root applies to it. The lasting impact also covers token revocation, which exists only in the
issuing service's Redis-backed access-token check and is not consulted at the gateway.
**Evidence:** The gateway's symmetric signing key is present in all four of its configuration files
and the same value appears in `operations-service`'s `appsettings.json`; the domain services instead
point at a certificate path with a development password. Recorded by path only, with no secret value
reproduced, in `baselines/architecture-baseline.md` §8.6 and as gap G9 in `repo-inventory.md` §6.
Token revocation lives in `Identity.Api/Program.cs` routing the revoke command to an access-token
service, with the cache and security registrations paired in `Identity.Infrastructure/Extensions.cs`
(`baselines/architecture-baseline.md` §11.3 conflict X6). Whether the committed material is live or
throwaway is `[unknown]` and is carried as blocker B1.
**Dependencies:** ADR-CANDIDATE-006
**Recommended priority:** second-batch

### ADR-CANDIDATE-008: Database per service on MongoDB, hand-mapped, with no migration tooling
**Category:** Storage
**Impact:** high
**Why this is an ADR:** Each persisting service owns exactly one logical MongoDB database named after
itself, maps its aggregates to documents by hand through a generic repository abstraction, and no
service holds another's connection string. Three consequences were chosen along with it: no ORM, no
shared schema, and **no migration tooling of any kind** — a workspace-wide search found no Alembic,
Flyway, Liquibase, EF Core migrations or equivalent, so schema evolution rests on document-model
tolerance and hand-written mappers. The alternatives were a shared relational database, a per-service
relational store with a migration framework, or event sourcing. The trade-off is service autonomy and
schema freedom against the loss of cross-service joins, cross-service transactions, and any
build-time or deploy-time check that a document shape change is backward compatible. The lasting
impact is that every data change on this platform is a compatibility decision made by an individual
developer, and it is the reason candidates 009 and 012 exist.
**Evidence:** Eight per-service databases named `availability-service`, `customers-service`,
`deliveries-service`, `identity-service`, `operations-service`, `orders-service`, `parcels-service`
and `vehicles-service`, each declared in that service's own `mongo` configuration section; document
classes under `*.Infrastructure/Mongo/Documents/` with explicit mapping methods. The single startup
index creation in `Identity.Infrastructure/Mongo/Extensions.cs` is the only schema management found
anywhere. Consolidated in `repo-inventory.md` §2.2, §3.5 and gap G11,
`baselines/architecture-baseline.md` §6.1–§6.2 and constraints C5 and C8, and
`patterns/data/database-per-service-with-document-mapping.md`.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-002
**Recommended priority:** first-batch

### ADR-CANDIDATE-009: Event-carried customer replicas in place of synchronous customer reads
**Category:** Storage
**Impact:** medium
**Why this is an ADR:** Two services hold their own local copy of customer data so they can decide
without calling the owning service, and the copies are populated from a creation event. The
alternative was the synchronous point read that two other services actually use for the same data
(010) — so the platform contains both answers to the same question, which is the strongest possible
signal that the choice deserves a recorded rationale. The trade-off is availability and latency
against staleness. The staleness is not hypothetical here: the owning service publishes both a state
change and a VIP-promotion event, **neither of which any replica consumes**, and no update or delete
event is consumed either, so a replica is written once at creation and never reconciled thereafter.
An ADR should record the replication decision together with the reconciliation obligation it creates.
**Evidence:** A `customers` collection owned by `customers-service` is replicated into
`orders-service` (`Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs`) and `parcels-service`
(`Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs`), kept in sync by the customer-created
event only (`repo-inventory.md` §3.5, which names this "the platform's principal data-coupling
risk"). The two unconsumed customer events are gap G8 in `repo-inventory.md` §6 and are analysed in
`baselines/architecture-baseline.md` §6.3. Pattern:
`patterns/data/event-carried-reference-replica.md`.
**Dependencies:** ADR-CANDIDATE-003, ADR-CANDIDATE-008
**Recommended priority:** backlog

### ADR-CANDIDATE-010: Narrow synchronous point-reads as the bounded exception to messaging
**Category:** Integration
**Impact:** medium
**Why this is an ADR:** Service-to-service HTTP exists but is deliberately confined: every one of the
eight synchronous edges is a single-entity read that gates a write, and there is no synchronous
composition, no fan-out and no synchronous write between services. That boundary is a rule, and a
rule with a stated scope is a decision — the alternatives were forbidding synchronous calls entirely
(forcing more replication, as in 009) or allowing them generally (producing a call graph with cascading
failure). The trade-off is a small, comprehensible synchronous surface against the need to replicate
whatever cannot be point-read. The lasting impact is that this rule is the test any future
cross-service call has to pass, and it is currently written nowhere — which is why two services read
customer data synchronously while two others replicate it.
**Evidence:** Eight synchronous edges, each proven by a typed client class and a logical service key
in the caller's `httpClient` configuration: Availability and Pricing both read from Customers, Orders
reads from Parcels, Pricing and Vehicles, and OrderMaker reads from Availability and Vehicles
(`repo-inventory.md` §3.1). Constraint C10 in `baselines/architecture-baseline.md` §11.2 states the
rule as observed. `ordermaker-service` is the one caller whose client is not routed through the
name-based router (`baselines/architecture-baseline.md` §4.1), so the exception already has an
exception. Pattern: `patterns/integration/narrow-synchronous-point-read.md`.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-008
**Recommended priority:** backlog

### ADR-CANDIDATE-011: An orchestrated saga for order creation, and its state durability
**Category:** Orchestration
**Impact:** high
**Why this is an ADR:** Almost all cross-service behaviour on this platform is choreographed — a
service publishes an event and interested services react — and exactly one process breaks that rule.
Order creation is driven by an explicit state machine in a dedicated service that reacts to events and
issues compensating actions on failure. Choosing orchestration for one process and choreography for
everything else is a decision with two halves, and both belong in the same record. The first half is
the exception itself, including the fact that this service **publishes commands onto other services'
exchanges**, breaking the ownership rule that constraint C1 otherwise holds platform-wide. The second
half is durability: the saga library is registered with no persistence configuration and no
persistence package, and its default is in-memory, which would lose in-flight saga state on restart.
The alternatives were extending choreography to cover order creation, or orchestrating with a durable
workflow engine. The lasting impact is a process that can strand orders mid-flight with no recorded
recovery path.
**Evidence:** The saga state machine and its data class live in `Sagas/AIOrderMakingSaga.cs` and
`Sagas/AIMakingOrderData.cs` in `hianshul100_Pacco.Services.OrderMaker`, with the saga library
referenced from that project's `.csproj`. The service publishes six commands belonging to Orders,
Parcels, Vehicles and Availability, and subscribes to five of their events (`repo-inventory.md` §2.2).
The persistence gap is G3 in `repo-inventory.md` §6 and §5.2 of
`baselines/architecture-baseline.md`, both of which record that the registration call carries no
persistence configuration. Two further loose ends are recorded as intended-but-not-implemented in
`baselines/architecture-baseline.md` §11.4: a declared approval command that is never published, and
an event forwarded to the coordinator with no saga action to receive it. The command that starts the
saga has no observable publisher anywhere in the thirteen repositories — labelled **Unverifiable —
Missing Source Evidence** — which is gap G2 and blocker B2 below. Pattern:
`patterns/orchestration/saga-process-manager.md`.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-003
**Recommended priority:** second-batch

### ADR-CANDIDATE-012: Transactional outbox and inbox applied by handler decorator
**Category:** Storage
**Impact:** medium
**Why this is an ADR:** Reliable publication and consumer deduplication are provided by decorating
every message handler, so no handler contains reliability code and the guarantee is a composition
concern rather than a business one. The alternatives were per-handler idempotency written by hand,
at-most-once delivery with no dedupe, or an external delivery guarantee from the broker. The trade-off
is uniform reliability with zero handler-level cost, against two dedicated collections per database
and a hidden control path that is easy to disable accidentally. That last point is the reason this
needs a record rather than a code comment: the outbox is enabled across the layered services, but it
is configured to run **without database transactions**, which is a coherent choice against a
single-node database and an incoherent one against a replica set. Whether that setting is deliberate
is an open question already raised at inventory time, and an ADR is where the answer belongs.
**Evidence:** Inbox and outbox collections alongside the domain collection in each of the layered
services' MongoDB databases, with the outbox enabled in each service's own configuration
(`repo-inventory.md` §2.2). The transaction-disabling setting and the single-node deployment it sits
against are recorded as the open question for `Pacco.Services.Availability` in `repo-inventory.md`
§2.3, and analysed in `baselines/architecture-baseline.md` §10.4 and §9.1 (every infrastructure
component runs as a single container, with no replication or quorum). Pattern:
`patterns/data/transactional-outbox-handler-decorator.md`.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-008
**Recommended priority:** second-batch

### ADR-CANDIDATE-013: Contract-blind universal subscription by runtime type emission
**Category:** Integration
**Impact:** medium
**Why this is an ADR:** One service subscribes to every message on the platform without holding a
compile-time type for any of them: the message names are listed in a configuration manifest and the
subscription types are generated at startup by runtime reflection emission. The alternatives were
referencing eight services' contract types, generating types at build time from the manifest, or
consuming the raw broker payload. The trade-off is that this service never needs recompiling when
another service adds a message, paid for with a total loss of payload visibility — the emitted types
have no fields, so what this service actually receives on the wire is unknown from static reading.
The manifest is also a hand-maintained copy of names owned by eight other repositories with no
generation or validation step, so it can silently fall out of date. The lasting impact is that the
platform's one cross-cutting observer is structurally incapable of validating what it observes.
**Evidence:** `Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` emits subscription
types at runtime from the names in `messages.json`, which enumerates eight exchanges, 26 commands, 30
events and 31 rejected events; the generic command, event and rejected-event handlers under
`Handlers/` receive them. The field-less consequence is gap G5 in `repo-inventory.md` §6, marked
**Unknown — requires runtime capture**, and is analysed in `baselines/architecture-baseline.md` §3.3
and `baselines/api-inventory.md` §9.5. Pattern:
`patterns/integration/declarative-message-manifest-subscription.md`.
**Dependencies:** ADR-CANDIDATE-003
**Recommended priority:** second-batch

### ADR-CANDIDATE-014: Ephemeral operation status and the real-time client notification channels
**Category:** Orchestration
**Impact:** medium
**Why this is an ADR:** When the edge accepts a write asynchronously (005), the caller gets an
acknowledgement and a correlation id, and the outcome has to arrive some other way. The platform's
answer is a dedicated service that projects every observed message into an operation status and pushes
it to clients over a hub with a shared cache backplane, and in parallel over a streaming RPC surface.
Three sub-decisions ride together and should be recorded as one: that completion is reported on a
separate channel rather than by polling or callback; that two client protocols are offered for the
same projection rather than one; and that the status itself is held in a cache with a five-minute
expiry rather than durably stored. The last is the sharpest trade-off — a status that expires before a
long-running saga finishes cannot report that saga's outcome. The alternatives were durable operation
records, client polling of a status endpoint, or webhooks.
**Evidence:** A hub endpoint and a streaming RPC service are both exposed by
`hianshul100_Pacco.Services.Operations`, with the proto contract in
`Pacco.Services.Operations.Api/Operations.proto` and the hub in `Hubs/PaccoHub.cs`; the cache
backplane package is referenced by the same project (`repo-inventory.md` §2.2–§2.3,
`baselines/api-inventory.md` §5). The service configures a MongoDB database but registers no
repository for it, while the only observed state path is the distributed cache with a 300-second
expiry — a documented conflict resolved in favour of the code as X5 in
`baselines/architecture-baseline.md` §11.3, and recorded as gap G4 in `repo-inventory.md` §6.
Patterns: `patterns/orchestration/acknowledge-then-notify-completion.md` and
`patterns/orchestration/real-time-push-with-shared-backplane.md`.
**Dependencies:** ADR-CANDIDATE-005, ADR-CANDIDATE-013
**Recommended priority:** second-batch

### ADR-CANDIDATE-015: Registry-mediated service discovery and name-based routing
**Category:** Networking
**Impact:** medium
**Why this is an ADR:** Services self-register with a discovery registry at startup and address each
other by logical name through a registry-driven router, rather than by DNS, static configuration, or a
service mesh. The alternatives are all live options for a platform of this size, and the trade-off is
dynamic membership and health-based deregistration against two more infrastructure components to run
and a routing layer with no in-code test coverage. What makes this ADR-worthy rather than a
configuration note is the gap between how widely it is provisioned and how narrowly it is used: every
service registers, but the router's load balancing is switched off in all four gateway configurations,
so the registry mediates east-west traffic only, and one service's client bypasses it entirely.
Recording the decision means recording which traffic it is actually responsible for.
**Evidence:** Every service declares registry registration with its own service name, port and health
ping settings, and every service declares the router with its own name, in that service's
`appsettings.json` (`baselines/architecture-baseline.md` §9.3). The gateway's load balancing is
disabled in all four of its configurations —
`patterns/deployment/registry-mediated-discovery-and-routing.md` lists this as one of three
documentation-versus-code conflicts in the catalog. The registry address committed in configuration is
a Windows Docker Desktop host alias, further evidence these files describe a developer machine
(`baselines/architecture-baseline.md` §9.3).
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-002
**Recommended priority:** second-batch

### ADR-CANDIDATE-016: A secret store for dynamic database credentials and per-service PKI
**Category:** Security
**Impact:** high
**Why this is an ADR:** The platform takes its database credentials from a secret store as short-lived
per-service leases with automatic renewal, rather than holding static passwords, and it uses the same
store as a certificate authority issuing each service an identity certificate. The alternatives were
static credentials in configuration, credentials injected by the deployment platform, or no
service-identity layer at all. The trade-off is the strongest security control the platform has — no
service holds a static database password and each service's database access is scoped to its own role
— bought with a hard runtime dependency on the store being available at startup. Two things make this
record necessary rather than optional. First, the certificate half is **provisioned platform-wide but
verified in exactly one place**: one service accepts and checks a caller certificate against an access
list, the other nine do not enable certificate checking at all, so "this platform uses mutual TLS
between services" would be a false statement. Second, one service that calls the checking service over
HTTP is **not on its access list**, so either the enforcement is not actually active or that call has
no grant — an unresolved question carried as Q3 below.
**Evidence:** Every service declares the secret store with key-value settings loading, a per-service
PKI role and common name, and a dynamic database lease with automatic renewal
(`baselines/architecture-baseline.md` §8.5). The single enabled certificate access list belongs to
`customers-service` and grants `availability-service` one read permission scoped to a domain; the
remaining sections are empty (`baselines/architecture-baseline.md` §8.4, `repo-inventory.md` §2.3).
`pricing-service` calls `customers-service` over HTTP (`repo-inventory.md` §3.1) and appears in no
access list. No TLS termination is configured anywhere in the workspace
(`baselines/architecture-baseline.md` §8.7). Pattern:
`patterns/security/vault-issued-dynamic-credentials-and-service-pki.md`. Bootstrap credential
material is committed in the platform repository and is referenced by path only here; whether it is
live is `[unknown]` and is carried as blocker B1.
**Dependencies:** ADR-CANDIDATE-015
**Recommended priority:** second-batch

### ADR-CANDIDATE-017: Container-compose and process-manager deployment, with no production orchestration
**Category:** Deployment
**Impact:** high
**Why this is an ADR:** The platform has two parallel deployment paths and no statement of which is
authoritative. One assembles the environment from small per-concern container-compose stacks; the
other runs the services as bare processes under a process manager, in a development variant and a
variant named for production. Neither is production-grade: every infrastructure component runs as a
single instance with no replication, quorum or persistent volume, and the production-named manifest is
a single-host manifest with a restart cap, no health gating, no rolling update and no multi-instance
configuration. There is **no container orchestration, no packaging chart, no infrastructure-as-code and
no service mesh in any of the thirteen repositories**. The alternatives were orchestrated deployment or
a managed platform. The lasting impact is that replica counts, autoscaling, resource limits, probes,
network policy, ingress and TLS termination, rollout and rollback, environment promotion, and backup
and recovery are all undefined and cannot be derived from these sources at any confidence — an ADR is
where that boundary should be stated rather than discovered.
**Evidence:** The per-concern compose stacks and the two process manifests are enumerated in
`baselines/architecture-baseline.md` §9.1, with the absence of orchestration stated in §9.2 and the
list of consequently-undefined operational properties given there. The two paths also disagree on
membership: `ordermaker-service` is defined in both compose service stacks, listed as a gateway
dependency and scraped for metrics, yet is absent from both process manifests and from all four
gateway configurations — gap G2 in `repo-inventory.md` §6 and §2.3 of the architecture baseline.
Pattern: `patterns/deployment/composable-per-concern-environment-stacks.md`.
**Dependencies:** ADR-CANDIDATE-001, ADR-CANDIDATE-015
**Recommended priority:** second-batch

### ADR-CANDIDATE-018: Repository per service, with independent per-repository release
**Category:** Deployment
**Impact:** medium
**Why this is an ADR:** Every deployable lives in its own repository with its own identical
three-script pipeline, builds its own container image, and releases without reference to the others.
The alternatives were a monorepo with a shared pipeline, or a coordinated multi-service release train.
The trade-off is that no team blocks another and no release requires coordination, against the fact
that there is **no mechanism to perform a coordinated release even when a contract change requires
one** — which is the same exposure candidate 003 records from the contract side. Two secondary
consequences are visible and belong in the record: the identical pipeline is duplicated eleven times
with no template, so a pipeline improvement is an eleven-repository change; and the aggregate solution
file in the platform repository references other repositories by relative path, which only resolves if
every clone sits in the same parent directory.
**Evidence:** Eleven repositories each carrying a Travis configuration that runs the same build, test
and dockerize scripts on the main branches, publishing a per-service image
(`repo-inventory.md` §2.3, `baselines/architecture-baseline.md` §9.5). Constraint C3 in
`baselines/architecture-baseline.md` §11.2 states that coordinated multi-service releases are
prevented because no mechanism exists to perform one. The aggregate solution file's relative-path
references are recorded in `repo-inventory.md` §7. Pattern:
`patterns/deployment/independent-per-repository-release.md`. Note the documentation-versus-code
conflict X1 in `baselines/architecture-baseline.md` §11.3: `architecture-views.md` §4.5 claims no CI
pipeline definition exists; the eleven Travis files show otherwise and the code is followed here. The
CD half of that claim stands — no deployment automation was observed.
**Dependencies:** ADR-CANDIDATE-002, ADR-CANDIDATE-017
**Recommended priority:** backlog

### ADR-CANDIDATE-019: Two internal service structures — layered domain split and single project
**Category:** Other
**Impact:** medium
**Why this is an ADR:** Seven services use a four-project layout in which every dependency points
inward and the innermost project references nothing at all, while three deployables are deliberately
single-project. This is not accidental drift: the platform's own README frames the choice as clean
architecture and domain-driven design "or another style that is the best fit", so the divergence is
sanctioned but unbounded. The alternatives were mandating the layered split everywhere, or leaving
internal structure entirely to each team. The trade-off is testable domain rules and a clear
dependency direction against four projects and a composition root for a service whose logic may be a
single pure function. What is missing, and what an ADR would supply, is the rule for choosing: today
nothing states when a service is simple enough to stay single-project, so the next service's structure
is a coin toss.
**Evidence:** The four-project layout is visible in the solution files of Availability, Customers,
Deliveries, Identity, Operations, Orders, Parcels and Vehicles, each with an inner domain project that
carries no infrastructure reference; `pricing-service` and `ordermaker-service` are single-project
(`repo-inventory.md` §2.1). The README framing is quoted in `repo-inventory.md` §2.3 as the platform
repository's recorded architectural intent. Constraint C4 in `baselines/architecture-baseline.md`
§11.2 states the dependency direction as an enforced constraint, and §3.1 analyses the two structures.
Pattern: `patterns/domain/inward-dependency-service-skeleton.md`.
**Dependencies:** ADR-CANDIDATE-002
**Recommended priority:** backlog

### ADR-CANDIDATE-020: .NET Core 3.1 as the platform runtime baseline
**Category:** Other
**Impact:** high
**Why this is an ADR:** Every deployable targets one runtime version, pinned identically in every
build configuration and every container image, so no service can drift to a different version. Pinning
a single runtime across a microservice platform is a real decision — the alternative, per-service
runtime choice, is one of the things microservices are usually adopted to allow, and it was given up
here in exchange for uniformity. The trade-off has now inverted: **this runtime reached end of support
in December 2022**, so the pin that once guaranteed consistency now guarantees that the whole platform
is on an unsupported runtime simultaneously, and that any upgrade is an eleven-repository change
coordinated through a release mechanism the platform does not have (018). The impact is rated high not
because the original choice was contentious but because the record needs to carry the supersession
path, and because the toolkit version line (002) is coupled to it.
**Evidence:** The runtime is pinned in all eleven Travis configurations, targeted by every project
file, and used by every container image's build and runtime stages, with publish output under the
matching target framework path (`repo-inventory.md` §2.1 and §2.3,
`baselines/architecture-baseline.md` §1.3). Constraint C9 in
`baselines/architecture-baseline.md` §11.2 states both the pin and the end-of-support date as a
current-state fact.
**Dependencies:** ADR-CANDIDATE-002
**Recommended priority:** backlog

## Recommended First Batch (3-5 ADRs)

Five candidates, ordered so that each is written after everything it depends on. These five are the
records that every other candidate cites: nothing else in the backlog can be written without assuming
their answers, and all five are high impact with strong, uncontested code evidence.

1. **Convey as the platform standard in place of a shared internal library** (ADR-CANDIDATE-002) —
   first because it has no dependencies and because it is what makes candidates 003, 019 and 020
   consequences rather than independent choices.
2. **Event-driven microservices with service-owned topic exchanges** (ADR-CANDIDATE-001) — also
   dependency-free, and the root of the integration branch. Writing it second lets it reference the
   framework record for how the exchange conventions are actually supplied.
3. **Database per service on MongoDB, hand-mapped, with no migration tooling** (ADR-CANDIDATE-008) —
   depends on 001 and 002. Written third because the data-ownership rule it fixes is the premise of
   candidates 009, 010 and 012.
4. **A declarative configuration-driven gateway as the single north-south edge**
   (ADR-CANDIDATE-004) — depends on 001. Written fourth because it establishes the edge that the
   authentication record then places a trust boundary on.
5. **Authentication enforced at the edge, with fail-open in-service authorization**
   (ADR-CANDIDATE-006) — depends on 004. Written last in the batch and included despite the
   dependency chain because the fail-open guard and the two gateway-bypassing paths are a live
   exposure, not a stylistic choice, and the batch should not close without it on record.

## Merge Recommendations

1. **ADR-CANDIDATE-011 absorbs saga state durability rather than splitting it out.** "Adopt an
   orchestrated saga" and "choose the saga's persistence backend" were assessed as two candidates and
   merged into one. They are inseparable in practice: the orchestration decision is not decidable
   without stating what happens to in-flight state on restart, and a durability record with no
   orchestration record has nothing to attach to. The merged candidate carries both halves and the
   unresolved backend as blocker B2.
2. **ADR-CANDIDATE-014 absorbs both the operation-status store and the client push protocols.**
   "Hold operation status in a cache with an expiry", "push completion over a hub with a shared
   backplane" and "expose a parallel streaming RPC surface" were assessed as three candidates and
   merged into one. All three exist only to answer the same question — how does a caller learn the
   outcome of a write the edge accepted asynchronously — and separating them would produce three
   records that each have to restate the other two to make sense.
3. **ADR-CANDIDATE-003 absorbs the consumer-driven contract testing decision.** Adopting a pact-based
   consumer/provider test pair was assessed as its own candidate and merged in, because it is the
   only verification mechanism the no-shared-contract-package decision has, and because it is
   currently incomplete — the pact file has no configured route between the two repositories. A
   separate record would either duplicate 003's context or understate that incompleteness.
4. **ADR-CANDIDATE-016 absorbs the certificate-ACL decision alongside dynamic credentials and PKI.**
   Service-to-service certificate authentication was assessed separately and merged, because the
   certificates it depends on are issued by the same secret store under the same per-service roles;
   splitting them would put the issuance and the verification of one credential in two records that
   disagree about how widely it is enforced.
5. **Candidates 009 and 010 should be reviewed together even though they stay separate.** They are
   two different answers to one question — how a service obtains another service's data — and the
   platform currently uses both for the same customer data. They are kept apart because each has its
   own trade-off profile and its own evidence, but whoever writes one should write the other, and if
   the `adr_generation` stage prefers a single "cross-service data access" record, merging them is
   defensible. Carried as Q4 below.

## Excluded as Implementation Details

Each item below was considered as a candidate and rejected against the stated test — a decision needs
alternatives that were genuinely available, a trade-off, and lasting impact. Implementation mechanics,
configuration values, and naming conventions are not decisions.

1. **The `snakeCase` message-name casing and the `<service>/{{exchange}}.{{message}}` queue naming
   template.** A naming convention. Its *consequence* — that the message name is the only binding
   contract — is a decision and is recorded as ADR-CANDIDATE-003; the casing itself is not.
2. **The per-service container port assignments (5000–5009 and 5015).** Configuration values with no
   alternative worth weighing.
3. **How runtime type emission works internally.** The reflection-emit mechanism is implementation
   detail. The decision to subscribe without compile-time types, and its consequence for payload
   visibility, is ADR-CANDIDATE-013.
4. **The route-table-to-handler dispatch style used in place of controllers.** A code-organisation
   idiom supplied by the framework and adopted wherever it is used; it follows from
   ADR-CANDIDATE-002 and carries no independent trade-off. Catalogued as
   `patterns/domain/dispatcher-bound-cqrs-endpoints.md`.
5. **The rejected-event failure contract and the aggregate event-buffering mechanism.** Both are
   consistent framework-shaped implementation patterns rather than decisions with contested
   alternatives. Catalogued as `patterns/integration/rejected-event-failure-contract.md` and
   `patterns/domain/aggregate-buffered-domain-events.md`.
6. **The redacted-property list and the per-message log templates.** Operational configuration.
7. **The shared cache partitioned by per-service key prefix.** Nine services register the cache and
   two have an observable use for it. There is no evidence a partitioning alternative was weighed,
   and the pattern file rates the evidence Weak for that reason. Revisit if a third consumer appears.
8. **The platform's client or frontend strategy.** Assessed and excluded because **no decision was
   made and none can be reconstructed**: `Pacco.Web` tracks a single one-line README on one commit,
   no frontend asset exists anywhere in the workspace, and the only browser-facing surface is a
   static developer test page inside one service. There is nothing here with alternatives, trade-offs
   or lasting impact to record — only an absence. This is not an implementation detail and it should
   become a candidate the moment a client is planned; it is carried as Q2 below rather than written
   as an empty record.
9. **The unresolved gaps that are questions rather than decisions** — the wire payloads received by
   the contract-blind subscriber, the trigger for the delivery lifecycle, and the ownership of every
   repository. These are gaps G5, G7 and G10 in `repo-inventory.md` §6. They need investigation, not
   a decision record; where one of them blocks a candidate, it is carried in the section below.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The thirteen cloned repositories at base ref `feature/12998/aidlc` are the whole platform, so no candidate here duplicates a decision recorded somewhere outside this workspace | The gateway route table, the compose stacks and the process manifests all resolve to services inside this set, and nothing in any repository points outside it | A decision already recorded elsewhere would be re-proposed as new, and the `adr_generation` stage would write a record that conflicts with an existing one | Ask the platform owner whether any deployable, or any decision log, exists outside these repositories |
| A2 | The absence of ADRs is genuine and not a search artifact | Two independent checks agree: the file system holds no ADR, decision-record or RFC file in any of the fourteen clones, and the governance catalog for tenant `Q5SCXYFS` returned zero rows for an unscoped node count as well as for ADR, Decision and Constraint queries | Every candidate would need re-checking against the missed records before any is written, and ids assigned here could collide with an existing id space | Confirm with the platform owner that no decision log is kept in a wiki, ticket system or document store outside the repositories |
| A3 | The message manifest in `operations-service` is a faithful list of the platform's messages | It enumerates eight exchanges and roughly 87 message names that match the publisher and subscriber classes found independently in the other repositories | Candidates 003 and 013 would understate or overstate the contract surface, and the "no validation step" finding could be wrong | Generate the message list from the publisher and subscriber types in each service and compare it to the manifest |
| A4 | The compose stacks describe developer environments rather than any deployed environment | Every infrastructure component is a single container with no replication or quorum, and the committed registry address is a Windows Docker Desktop host alias | ADR-CANDIDATE-017 would misdescribe the platform's actual runtime, and its list of undefined operational properties would be wrong | Ask whoever operates the platform which manifest, if either, produced the environment currently running |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Nobody has confirmed whether the credential material committed in the repositories is real or throwaway — the gateway's shared token-signing key, which appears in five files, and the secret-store bootstrap material in the platform repository. Anyone holding that signing key can mint a token the gateway accepts, including an administrator one | ADR-CANDIDATE-007 and ADR-CANDIDATE-016 cannot be written as decisions until this is answered — if the material is live these are incident records, not architecture records | Platform security owner | Check whether the committed values match anything in a running environment; if they do, rotate first and write the ADRs against the rotated design | TBD |
| B2 | **[ACTION NOW]** The order-creation saga's state is held by whatever the saga library defaults to, because no persistence is configured and no persistence package is referenced. The default is in-memory, which loses in-flight orders on restart, but nobody has confirmed the running behaviour | ADR-CANDIDATE-011 cannot state the current durability position, so its consequences section would be speculative | Owner of `Pacco.Services.OrderMaker` (unassigned — see B4) | Start the service, begin a saga, restart the process, and observe whether the saga resumes. Record the result in the ADR as the current state | TBD |
| B3 | **[ACTION NOW]** No one has said which gateway configuration production uses. The synchronous and asynchronous pairs give the same twenty write URLs different contracts — one returns a result, the other returns an acknowledgement and defers the outcome to a separate channel | ADR-CANDIDATE-005 and ADR-CANDIDATE-014 both depend on this. Until it is answered neither can record what callers of the platform's write API should expect | Platform owner | Confirm which configuration is loaded in each environment, then record the intended mode in the ADR and delete or clearly mark the configurations that are not used | TBD |
| B4 | **[ACTION NOW]** No repository has an owner. There is no `CODEOWNERS`, no contributing guide and no team metadata in any of the fourteen clones, so no candidate in this backlog can be assigned a decision owner or a reviewer | Every candidate's approval path. An ADR nobody owns cannot move past draft, so the whole backlog stalls at generation | Platform owner | Name an owner per subsystem using the six groupings in `repo-inventory.md` §4, and record them in the artifact repository before `adr_generation` runs | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should a candidate that describes a current state nobody is happy with be written as an ADR recording that state, or as an ADR proposing the fix? | Several candidates describe something that works but has a known defect — the authorization guard that is skipped for unauthenticated callers (006), the split token trust root (007), replicas that are never reconciled (009). Recording them as-is blesses the defect; recording the fix means the ADR does not describe the running system | Record the current state and its consequences, and add an explicit "known defect / supersession expected" note naming what must change. That keeps the ADR true about today without endorsing it | Platform owner |
| Q2 | **[ACTION NOW]** Is a client application planned for this platform? | `Pacco.Web` is an empty repository and no frontend exists anywhere, so no client decision was excluded as an implementation detail — there simply is none to record. If a client is planned, a twenty-first candidate is needed, and the edge decisions (004, 005, 006, 014) all currently assume a machine caller rather than a browser | If a client is planned, add the candidate before any client code is written, and derive its assumptions from ADR-CANDIDATE-005 and ADR-CANDIDATE-014, since both change what a caller has to handle | Platform owner |
| Q3 | **[ACTION NOW]** Is the certificate check on `customers-service` actually enforced at runtime? | `pricing-service` calls it over HTTP and is not on its access list. Either the call is failing and nobody has noticed, or the enforcement is not really active — and ADR-CANDIDATE-016 says something different in each case | Call `customers-service` from `pricing-service` in a running environment without a certificate and observe the response before writing the ADR | Platform security owner |
| Q4 | **[handled later by adr_generation]** Should candidates 009 and 010 be written as two records or merged into one "cross-service data access" record? | They are two answers to the same question and the platform uses both for the same customer data. Two records risk contradicting each other; one record risks burying the trade-off that distinguishes them | Keep them separate, and have each state explicitly when the other applies. Merge only if the first draft of one cannot be written without restating the other | `adr_generation` stage |
| Q5 | **[handled later by adr_generation]** What id scheme should the generated ADRs use, and how should this backlog's `ADR-CANDIDATE-NNN` ids map onto it? | No ADR ids exist anywhere in scope, so there is no prefix to preserve and no collision to avoid. Downstream artifacts and the pattern catalog's empty **Related ADRs** columns will cite whatever is chosen, so the mapping has to be recorded once rather than inferred per file | Use sequential `ADR-001`…`ADR-020` matching this backlog's numbering, and record the mapping in the generated index so a reader can trace any ADR back to its candidate | `adr_generation` stage |
