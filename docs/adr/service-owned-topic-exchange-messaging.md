# ADR-001: Event-driven microservices with service-owned topic exchanges

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-001` |
| **Backlog candidate** | `ADR-CANDIDATE-001` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Integration / high |
| **Supersedes / Superseded by** | — (first ADR corpus on this platform; nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-002` (the toolkit that supplies the broker client and the conventions), `ADR-008` (why data cannot be joined across services), `ADR-004` (the edge that can publish onto these exchanges) |

## Notation

| Symbol | Meaning |
| --- | --- |
| ✅ | Confirmed current behaviour — observed in source at the cited path and line range |
| 🎯 | Target state — intended design, not in production today |
| ❓ | Needs validation — assumed or inferred, not observed in source |
| `[INFERRED]` | Conclusion drawn from observed evidence rather than stated by it |

## Contents

1. [Context](#1-context)
2. [Decision](#2-decision)
3. [Alternatives Considered](#3-alternatives-considered)
4. [Consequences](#4-consequences)
5. [Relationship to the implementation pattern catalog](#5-relationship-to-the-implementation-pattern-catalog)
6. [Evidence](#6-evidence)
7. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Context

Pacco is a parcel-delivery platform built as ten messaging deployables that have to collaborate on
processes no single one of them owns: a customer signs up, reserves a limited resource, adds parcels to
an order, has a vehicle assigned, and has that order approved and delivered. Every step in that
sequence crosses a service boundary.

The platform answers that with asynchronous messaging as its primary integration mechanism, and it
partitions the messaging in a specific way:

1. ✅ **One topic exchange per service, named after the service.** Eight exchanges exist —
   `availability`, `customers`, `deliveries`, `identity`, `operations`, `orders`, `parcels`,
   `vehicles` — each declared in the owning service's own configuration
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:133-139`).
2. ✅ **A service publishes only its own messages** onto its own exchange, and consumers bind queues
   named for the consuming service through a queue template that embeds both the source exchange and
   the message name
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:140-146`).
3. ✅ **Roughly 87 message names across those eight exchanges**, catalogued as commands, events and
   rejected events in a platform-wide manifest
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`).
4. ✅ **Cross-service consumption is explicit and local.** A service that consumes another's event
   declares that event as its own class under `Events/External/` and a handler under
   `Events/External/Handlers/` — thirty-seven such files exist across Availability, Customers, Orders,
   Parcels and OrderMaker.
5. ✅ **One deployable does not participate at all.** `pricing-service` references no broker package
   and declares no `rabbitMq` section, which shows that participation was a per-service choice rather
   than an unavoidable default.

The alternatives were all live at the time: one shared bus for the platform, synchronous RPC between
services, or a modular monolith. This record states which was taken and what it costs.

### 1.1 What this record does *not* cover

This record fixes the *topology* — who owns an exchange, who may publish to it, how queues are named.
It does not fix the *contract* carried on it (`ADR-CANDIDATE-003`), the saga that orchestrates order
creation (`ADR-CANDIDATE-011`), the outbox that makes publication reliable (`ADR-CANDIDATE-012`), or
the bounded synchronous exception (`ADR-CANDIDATE-010`). Those are separate records that all assume
this one.

## 2. Decision

**Pacco integrates its services primarily by asynchronous messaging over a broker, and partitions that
messaging so that each service owns exactly one topic exchange named after itself, publishes only its
own messages onto it, and consumes other services' messages through queues named for itself.**
Synchronous service-to-service HTTP is the bounded exception, not the default.

Three rules follow from the decision and are part of it:

1. ✅ **A service publishes to its own exchange only.** One documented exception exists today —
   `ordermaker-service` publishes commands onto four other services' exchanges — and it is not to be
   extended to another service without a recorded decision.
2. 🎯 **Every new message must carry a declared payload contract.** The platform has none today: the
   message name is the only binding contract, which is why `ADR-CANDIDATE-003` exists.
3. 🎯 **Every new queue must have an explicit dead-letter destination.** None is configured anywhere in
   the workspace today.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **A single shared exchange (or one enterprise bus) for the whole platform,** with routing keys distinguishing sources. | Rejected because it makes the broker topology a shared, centrally-owned artifact. Every new message type becomes a change to something all ten services depend on, and no service can reason about its own publication surface in isolation — the opposite of what the repository-per-service release model (`ADR-002`) is built for. A per-service exchange keeps the blast radius of a topology change inside one repository. |
| 2 | **Synchronous request/response RPC between services** for the whole collaboration. | Rejected because the platform's core processes fan out across four or five services, and a synchronous call graph over them produces cascading failure and latency addition: a slow `vehicles-service` would make order approval slow, and an unavailable one would make it fail. Messaging decouples availability, and the platform accepted eventual consistency to get it. The narrow exception that survives — single-entity reads that gate a write — is scoped separately in `ADR-CANDIDATE-010`. |
| 3 | **A modular monolith with in-process calls** and one database. | Rejected because it removes independent deployability, which is the platform's organising principle: eleven repositories, eleven pipelines, eleven images, no coordinated release. It would also have made the data-ownership decision in `ADR-008` unnecessary and the whole platform one unit of failure and of scaling. It remains the cheapest design for a system this size, which is exactly why the trade-off is worth recording rather than assuming. |
| 4 | **Consumer-owned exchanges** — each consumer declares an exchange and publishers write into it. | Rejected implicitly by the naming convention in every service's configuration: exchanges are named for the *publisher*. Consumer-owned exchanges would force a publisher to know its audience, so adding a consumer would change the publisher — precisely the coupling this topology avoids. |

## 4. Consequences

### 4.1 Positive

1. ✅ Adding a consumer changes only the consumer. A new subscriber declares the message locally and
   binds its own queue; the publishing service is untouched.
2. ✅ A service's whole publication surface is one exchange, readable from its own configuration and
   its own `Events/` folder.
3. ✅ Services are decoupled in availability: a consumer being down does not fail the publisher's
   write, because the broker holds the message.
4. ✅ Queue names embed both the consuming service and the source exchange and message, so the
   binding topology is legible from the broker without reading code.

### 4.2 Negative

1. ✅ **No cross-service transaction exists,** and none can. Every multi-service process is eventually
   consistent, which is why order creation needs a saga with compensating actions at all.
2. ✅ **Failure modes live in the topology, not in the code.** A misnamed message, a missing binding
   or an unconsumed event is invisible to every compiler and every test in the workspace — the
   platform has two unconsumed customer events today and nothing reports that fact.
3. ✅ **No dead-letter destination is configured on any queue.** A message a consumer cannot process
   has no recorded destination. `[INFERRED]` its fate is whatever the toolkit's default is, which is
   not visible in this workspace.
4. `[INFERRED]` **The message topology is now a platform-wide contract with no owner.** The manifest
   that describes it is a hand-maintained file inside one service, copied from names owned by eight
   others.

### 4.3 Neutral / follow-on

1. ✅ `pricing-service` sits outside the topology entirely and is reachable only synchronously. Any
   future event-driven pricing behaviour is a new decision, not an extension of this one.
2. ✅ `ordermaker-service` publishes onto other services' exchanges, which is the one live violation of
   rule 1 in §2. It is recorded here rather than silently tolerated, and is analysed as a decision in
   `ADR-CANDIDATE-011`.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/integration/service-owned-topic-exchange-messaging.md` (Status:
   `Candidate`, evidence Strong). The two obligations in §2 — a declared payload contract per message
   and an explicit dead-letter destination per queue — are that pattern's recommendation adopted as
   binding rather than advisory.
2. **Constrains** `patterns/integration/rejected-event-failure-contract.md`,
   `patterns/domain/aggregate-buffered-domain-events.md` and
   `patterns/data/transactional-outbox-handler-decorator.md`: all three exist only because integration
   is asynchronous, and all three would be unnecessary under alternative 3.
3. **Deliberately diverges** from its own rule in exactly one place, `ordermaker-service`'s
   cross-exchange publication, and the divergence is bounded by rule 1 in §2 rather than generalised.
4. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*). Once the messaging pattern is approved,
   `ordermaker-service`'s cross-exchange publication becomes drift and needs the recorded decision that
   rule 1 demands.
5. **Pattern Update Proposal:** none from this record. Both obligations in §2 are already the
   pattern's own recommendation.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | A service declares one durable topic exchange named after itself | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:133-139` |
| E2 | Consumer queues are named for the consuming service and templated on the source exchange and message name; message names use snake-case conventions | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:115, 140-146` |
| E3 | Eight exchanges exist, one per participating service, each declared in that service's own configuration | ✅ | `appsettings.json` of Availability, Customers, Deliveries, Identity, Operations, Orders, Parcels and Vehicles (`…Api/appsettings.json`, `rabbitMq.exchange.name`) |
| E4 | The platform's message catalogue enumerates eight exchanges with their commands, events and rejected events | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:1-151` |
| E5 | Subscriptions are declared explicitly per service at startup, for both commands and other services' events | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Extensions.cs:106-112` |
| E6 | The broker client, its CQRS bindings, its outbox and its tracing plugin are all toolkit registrations composed per service | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Extensions.cs:81-83` |
| E7 | Cross-service events are redeclared locally by each consumer, with a handler alongside | ✅ | 37 files under `*/Events/External/` and `*/Events/External/Handlers/` across Availability, Customers, Orders, Parcels and OrderMaker — e.g. `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Events/External/ResourceReserved.cs` |
| E8 | `pricing-service` participates in no messaging: no broker configuration section exists in its settings | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json` — no `rabbitMq` section |
| E9 | The edge can publish directly onto six services' exchanges in the asynchronous gateway configuration, which is only possible because exchanges are per service and publicly named | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada-async.yml:118-528` (20 routes routed to the `availability`, `customers`, `deliveries`, `orders`, `parcels` and `vehicles` exchanges) |
| E10 | No dead-letter queue or dead-letter exchange is configured in any service's broker settings | ✅ | Workspace-wide search of all `rabbitMq` configuration sections — no dead-letter key present |

### 6.1 Documentation-versus-code conflicts

None found for this decision. `docs/architecture-inventory/repo-inventory.md` §3.2 and
`docs/architecture-inventory/baselines/architecture-baseline.md` §1.2 and §4.2 describe the same
topology the source shows.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The message manifest in `operations-service` is a faithful list of the platform's messages | Its eight exchanges and their message names match the publisher and subscriber classes found independently in the other repositories | The message surface described here would be incomplete, and the "no validation step" consequence could be understated or overstated | Generate the message list from the publisher and subscriber types in each repository and compare it to the manifest |
| A2 | The broker declares exchanges and queues from these settings at startup, as the configuration asks it to, because the toolkit's source is not in this workspace | Each service's settings request exchange and queue declaration, but only the toolkit performs it, and the toolkit is a package reference with no source here | The runtime topology could differ from the declared one — for example if a queue is pre-created differently by an operator | Inspect the broker's live exchange and queue list in a running environment and compare it to the declared topology |
| A3 | A message with no consumer is silently discarded by the broker rather than retained | This follows from topic-exchange routing with no bound queue, but nothing in the workspace states it and no dead-letter destination is configured | The two unconsumed customer events could be accumulating somewhere rather than vanishing, which changes the operational picture | Publish one of the unconsumed events in a test environment and inspect the broker for any retained copy |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no one can approve this record or own the message topology it describes | This ADR leaving `Proposed`, and rule 1 in §2 — which requires "a recorded decision" from someone before another service publishes cross-exchange | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Where should a message that a consumer repeatedly fails to process go? | No queue on this platform has a dead-letter destination. Today a poison message either blocks its queue or disappears, and nobody has decided which is acceptable | Give every queue a dead-letter exchange named for the consuming service, and add an alert on non-zero depth. Decide this before any new queue is created, so the convention lands once | Platform owner |
| Q2 | **[ACTION NOW]** Who owns the platform's message topology as a whole? | The exchanges are per service, but the *catalogue* of what exists is a hand-maintained file inside `operations-service` describing names owned by eight other repositories. It has no owner and no validation | Give the catalogue an owner, or generate it from the publishing services so it cannot drift. Until then treat it as a description, not a contract | Platform owner |
| Q3 | **[ACTION NOW]** Should `pricing-service` be brought into the messaging topology, or is its exclusion deliberate? | It is the only deployable outside the topology, reachable only synchronously. If the exclusion is deliberate it should be stated; if it is an oversight, the platform has a service that cannot react to anything | State it as deliberate — pricing is a pure calculation over data it is given — and record that any pricing behaviour that must react to platform state is a new decision, not a configuration change | Platform owner |
| Q4 | **[handled later by adr_generation]** Should `ordermaker-service`'s cross-exchange publication be ratified as a bounded exception or removed? | It is the one live violation of the ownership rule this record states. Left unrecorded, it becomes the precedent that dissolves the rule | Ratify it as a single named exception inside `ADR-CANDIDATE-011`, with the explicit condition that no second service may do the same without its own record | `adr_generation` stage, in `ADR-CANDIDATE-011` |
