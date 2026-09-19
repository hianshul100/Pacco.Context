# ADR-012: Transactional outbox and inbox applied by handler decorator

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-012` |
| **Backlog candidate** | `ADR-CANDIDATE-012` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Storage / medium |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-001` (the messaging this makes reliable), `ADR-008` (the per-service database the collections live in), `ADR-002` (the toolkit that supplies the outbox), `ADR-003` (the contract layer this does *not* verify) |

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

`ADR-001` made asynchronous messaging the platform's primary integration mechanism and `ADR-008` gave
each service its own document database. Together those create the classic dual-write problem: a handler
that changes its own state and publishes a message can succeed at one and fail at the other, and no
cross-service transaction exists to fix it (`ADR-001` §4.2).

1. ✅ **Reliability is applied by decorating every handler, not by writing it into any handler.** Each
   layered service registers one decorator over its command-handler interface and one over its
   event-handler interface
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:62-63`),
   so no business handler in the platform contains reliability code.
2. ✅ **The decorator is thin and identical everywhere.** It captures the inbound message identifier —
   or generates one when the handler was reached over HTTP rather than over the broker — and either
   wraps the inner handler in the outbox or calls it directly, depending on a single enabled flag
   (`…Orders.Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:19-35`). The event variant is
   the same file with the event interface substituted.
3. ✅ **The outbox is backed by the service's own database**, registered as one toolkit call alongside
   the broker registration (`…Orders.Infrastructure/Extensions.cs:73`), writing to an inbox collection
   and an outbox collection beside the domain collections in the same per-service database
   (`…Orders.Api/appsettings.json:104-112`).
4. ✅ **Seven of ten deployables use it.** Availability, Customers, Deliveries, Identity, Orders,
   Parcels and Vehicles each carry both decorator files and the configuration section.
   `operations-service`, `ordermaker-service` and `pricing-service` carry neither.
5. ✅ **It is configured to run without database transactions.** Every one of the seven sets the
   transaction-disabling flag, uses sequential processing, a one-hour message expiry and a two-second
   dispatch interval — the same seven values in all seven services.
6. ✅ **The database runs as a single container with no replica set anywhere in the workspace**
   (`hianshul100_Pacco/compose/infrastructure.yml:53-65`; workspace-wide search finds no replica-set
   configuration in any settings file, Compose file or connection string).

Point 5 read against point 6 is the reason this needs a record rather than a code comment: disabling
transactions is a *coherent* choice against a single-node document database, because that deployment
cannot offer multi-document transactions at all — and an *incoherent* one the moment the database
becomes a replica set, because the guarantee would then be available and switched off.

### 1.1 What this record does *not* cover

This record fixes how reliable publication and consumer deduplication are applied. It does not fix the
message topology (`ADR-001`), the payload contract (`ADR-003`), the database-per-service rule or the
absence of migration tooling (`ADR-008`), or the saga's separate state-durability problem
(`ADR-CANDIDATE-011`), which the outbox does not address.

## 2. Decision

**Pacco obtains reliable message publication and consumer deduplication by decorating every command and
event handler with a transactional outbox, backed by an inbox and an outbox collection inside the
service's own database. Handlers contain no reliability code, and the guarantee is a composition
concern applied uniformly in each service's composition root.**

Four rules follow from the decision and are part of it:

1. ✅ **A service that both persists state and publishes messages applies both decorators.** The seven
   layered services do; the three that do not are the three that do not fit the shape (§4.3).
2. ✅ **A handler must not implement its own idempotency or retry logic.** The decorator owns
   deduplication by message identifier, and a handler that also deduplicates would hide which mechanism
   is actually protecting it.
3. 🎯 **The transaction-disabling flag must match the database deployment.** It is correct against a
   single node and wrong against a replica set. Today it is set to disabled in all seven services and no
   replica set exists — so the setting is currently right for the wrong reason, because nothing ties the
   two together.
4. 🎯 **Outbox depth and age must be observable.** A message stuck in an outbox is invisible today: the
   platform scrapes service metrics but nothing reports outbox backlog, and the one-hour expiry means a
   stuck message is discarded rather than escalated.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Per-handler idempotency written by hand** — each handler checks whether it has already processed a message and makes its own publication safe. | Rejected because it puts the same non-business logic into every handler across seven repositories, and because a handler that forgets it is indistinguishable from one that does not need it. The decorator makes the guarantee a property of the composition root, where it can be verified by reading one file per service rather than every handler. |
| 2 | **At-most-once delivery with no deduplication** — accept that a redelivered message is processed twice and design handlers to tolerate it. | Rejected because most of the platform's handlers are not naturally idempotent: reserving a resource, adding a parcel to an order and assigning a vehicle all change state in ways that repeat badly. Making them all idempotent by hand is alternative 1 with extra steps. |
| 3 | **Rely on the broker's own delivery guarantee** and publish directly from the handler. | Rejected because the broker cannot solve the dual-write problem: it can guarantee delivery of a message it received, but not that the message was published if and only if the database change committed. That gap is exactly what the outbox closes, and it is the reason the outbox lives in the service's database rather than beside the broker. |
| 4 | **Event sourcing** — make the event log the state, so there is no second write to keep consistent. | Rejected because it would replace the whole persistence model established in `ADR-008`, including the hand-mapped documents and the per-service schema freedom, for a problem the outbox solves at the cost of two collections. It is the design that removes the problem rather than managing it, which is why it is recorded rather than dismissed. |

## 4. Consequences

### 4.1 Positive

1. ✅ No handler in the platform contains reliability code. The seven services' handlers are business
   logic only, and the guarantee is added in one place per service.
2. ✅ The mechanism is uniform: the same two decorator files, the same registration lines and the same
   seven configuration values in all seven services, so a reviewer can confirm a service is protected by
   reading its composition root.
3. ✅ Deduplication covers both transports. The decorator falls back to a generated identifier when
   there is no inbound message identifier, so a handler reached over HTTP behaves consistently with one
   reached over the broker.
4. ✅ The outbox lives inside the service's own database, so it inherits the ownership rule from
   `ADR-008` and adds no shared infrastructure.
5. ✅ Turning the mechanism off is one flag and does not change any handler — useful for testing, and
   the reason the decorator branches rather than being conditionally registered.

### 4.2 Negative

1. ✅ **Transactions are disabled, so the dual-write guarantee is weaker than the pattern's name
   implies.** `[INFERRED]` without a transaction the state change and the outbox insert are two separate
   writes, so a crash between them can still lose a message — the outbox reduces the window rather than
   closing it.
2. ✅ **The setting is not tied to the deployment.** Nothing in the workspace connects the
   transaction-disabling flag to the fact that the database is a single node. If the database becomes a
   replica set, seven services silently continue without transactions they could now have.
3. ✅ **A hidden control path that is easy to disable accidentally.** The guarantee depends on two
   registration lines and one flag per service. Removing either registration line compiles, passes every
   test in the repository, and silently removes the guarantee.
4. ✅ **Two extra collections per database**, plus a dispatch loop running every two seconds in each of
   seven services, whether or not there is anything to send.
5. ✅ **A stuck message expires rather than escalating.** The one-hour expiry means a message the
   dispatcher cannot deliver is eventually dropped, and nothing in the workspace reports the drop —
   which is the same shape of blindness `ADR-001` §4.2 records for the absent dead-letter destinations.
6. ✅ **Deduplication is by message identifier only.** It protects against redelivery of the same
   message; it does not protect against a publisher emitting two messages for one logical event, and it
   verifies nothing about the payload (`ADR-003`).

### 4.3 Neutral / follow-on

1. ✅ The three deployables without the outbox are consistent with their shapes rather than being gaps:
   `pricing-service` publishes and consumes no messages at all (`ADR-001`), `operations-service` holds
   its state in a cache rather than in a database (`ADR-CANDIDATE-014`), and `ordermaker-service` is a
   saga coordinator whose durability problem is a different one that the outbox does not solve
   (`ADR-CANDIDATE-011`).
2. ✅ The event-handler decorator and the command-handler decorator are byte-for-byte the same logic
   over two interfaces, duplicated across seven repositories — fourteen near-identical files. That is a
   direct consequence of `ADR-002`'s no-shared-library rule, not a defect of this decision.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/data/transactional-outbox-handler-decorator.md` (Status: `Candidate`,
   evidence Strong). Rules 1 and 2 in §2 are that pattern adopted as binding; rules 3 and 4 are the two
   conditions this record adds before it should be approved.
2. **Constrains** `patterns/domain/aggregate-buffered-domain-events.md`: aggregates buffer their events
   and publish only after persistence, and the outbox is what makes that publication survive a failure.
   The two patterns are halves of one guarantee and should be read together.
3. **Constrains** `patterns/integration/service-owned-topic-exchange-messaging.md`: the reliability this
   record provides is what makes at-least-once delivery over that topology safe to depend on.
4. **Deliberately diverges** from the pattern's own name in one respect — transactions are disabled —
   and §2 rule 3 bounds the divergence to the single-node deployment that justifies it rather than
   leaving it as an unexplained setting.
5. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*). On approval, the disabled transactions
   become drift the moment the database gains a replica set, which is what rule 3 exists to catch.
6. **Pattern Update Proposal:** add outbox depth and oldest-message age to
   `patterns/observability/structured-logging-with-property-redaction.md`'s companion metrics guidance,
   or to the outbox pattern directly. Rule 4 in §2 cannot be satisfied by any existing pattern, because
   no catalogued pattern covers metrics conventions at all (`patterns/index.md`, *Categories*).

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | Each layered service decorates its command-handler and event-handler interfaces with outbox decorators in its composition root | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:62-63`; same at `…Availability.Infrastructure/Extensions.cs:67-68` |
| E2 | The decorator captures the inbound message identifier, generates one when absent, and either wraps the handler in the outbox or calls it directly based on one enabled flag | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:19-35` |
| E3 | The event-handler decorator is the same logic over the event interface | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Decorators/OutboxEventHandlerDecorator.cs:19-35` |
| E4 | Fourteen decorator files exist — a command and an event variant in each of seven services | ✅ | `*/Infrastructure/Decorators/Outbox{Command,Event}HandlerDecorator.cs` in Availability, Customers, Deliveries, Identity, Orders, Parcels and Vehicles |
| E5 | The outbox is registered as one toolkit call backed by the service's own document database, beside the broker registration | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:72-73`; same at `…Availability.Infrastructure/Extensions.cs:82` |
| E6 | The outbox writes an inbox collection and an outbox collection inside the service's own database, with sequential processing, a one-hour expiry and a two-second interval | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:104-112`; same values at `…Availability.Api/appsettings.json:102-110` |
| E7 | All seven services set the transaction-disabling flag | ✅ | The `outbox` section of `…Api/appsettings.json` in Availability, Customers, Deliveries, Identity, Orders, Parcels and Vehicles — identical seven values in each |
| E8 | `operations-service`, `ordermaker-service` and `pricing-service` have no outbox configuration and no decorator files | ✅ | No `outbox` section in their settings; no `Decorators/` folder in `…Operations`, `…OrderMaker` or `…Pricing` sources |
| E9 | The document database runs as a single container with no replication | ✅ | `hianshul100_Pacco/compose/infrastructure.yml:53-65` |
| E10 | No replica set is configured anywhere in the workspace | ✅ | Workspace-wide search of all settings, Compose and connection-string values — no replica-set parameter present |
| E11 | Nothing reports outbox depth or age; the metrics registration is a single toolkit call with no outbox-specific counters | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:77`; no outbox metric in any repository or in the scrape configuration under `hianshul100_Pacco/compose/prometheus` |

### 6.1 Documentation-versus-code conflicts

None found for this decision. `docs/architecture-inventory/repo-inventory.md` §2.2 and §2.3 and
`docs/architecture-inventory/baselines/architecture-baseline.md` §10.4 and §9.1 describe the same
collections, the same transaction-disabling setting and the same single-instance infrastructure the
source shows. The code adds one detail the inventory does not state: the three deployables without an
outbox are precisely the three that do not persist domain state to a document database, which is what
makes their absence a consequence rather than a gap (§4.3 item 1).

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | With transactions disabled, the state change and the outbox write are two separate writes, so a crash between them can still lose a message | It is what "disable transactions" means for a mechanism whose guarantee comes from writing state and message atomically. The toolkit that implements it is a package reference with no source in this workspace, so the exact behaviour is not observable here | If the toolkit compensates some other way, §4.2 item 1 overstates the weakness and rule 3 in §2 is unnecessary | Read the toolkit's outbox implementation at the pinned version, or kill a service between the state write and the dispatch interval and check whether the message is ever sent |
| A2 | The dispatcher publishes from the outbox on the configured interval, and deduplicates consumers using the inbox collection by message identifier | The two collections are named for those roles, the decorator supplies exactly a message identifier, and the interval and expiry settings are dispatch-shaped | The mechanism might dedupe on something else, or not at all, and rule 2 in §2 would be protecting handlers that are not actually protected | Send the same message identifier twice into a running service and confirm the handler executes once |
| A3 | The transaction-disabling flag was set because the database is a single node rather than for an unrelated reason | The database has run as a single container with no replica set in every environment definition in the workspace, and multi-document transactions require a replica set | It might have been set to work around a defect or a performance problem, and rule 3 in §2 would prescribe re-enabling something that was deliberately avoided | Ask whoever set it; if nobody remembers, enable transactions against a replica set in a test environment and observe |
| A4 | A message that cannot be dispatched within the one-hour expiry is discarded rather than retained | The expiry setting is the only lifetime control on the outbox and nothing else in the workspace acts on aged messages | Stuck messages might accumulate indefinitely instead, which changes the operational picture from silent loss to unbounded growth | Block the broker for longer than the expiry window in a test environment and inspect the outbox collection afterwards |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so nobody can approve this record or own the seven services' reliability settings | This ADR leaving `Proposed`, and rules 3 and 4 in §2, which both need someone accountable when the database deployment or the metrics surface changes | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should the database become a replica set so the outbox can use transactions? | Without one, the outbox narrows the dual-write window but does not close it, and the platform's strongest reliability mechanism is running in its weakest mode. It is also the same single-node database every service depends on for durability generally | Yes, if the platform is to run anywhere beyond a developer machine — and turn the transaction-disabling flag off in all seven services in the same change, so the two never drift apart | Platform owner |
| Q2 | **[ACTION NOW]** What should happen to a message that cannot be dispatched before the one-hour expiry? | Today it is dropped and nobody is told. That is the same silent-loss shape `ADR-001` records for the absent dead-letter destinations, and it applies to every message the platform publishes | Alert on outbox depth and oldest-message age before the expiry window, and decide the retention policy together with the dead-letter question in `ADR-001` Q1 so the platform gets one answer rather than two | Platform owner |
| Q3 | **[ACTION NOW]** How does a reviewer know a service is still protected? | The guarantee is two registration lines and a flag. Removing them compiles and passes every test in the repository, so a refactor can silently remove reliability from a service | Add a test in each service asserting that a handler resolved from the container is wrapped by the decorator. That is a small test per repository and it is the only thing that would catch the removal | Owners of the seven layered services (unassigned — see B1) |
| Q4 | **[handled later by adr_generation]** Should the saga coordinator gain outbox-style durability, or is its problem genuinely different? | `ordermaker-service` has no outbox and also no configured saga persistence, so it is the one place where in-flight cross-service state can be lost entirely. It is easy to read its missing outbox as the same gap | Keep them separate. The outbox protects a single handler's write-and-publish; the saga needs its own state store across many messages. Decide the saga half in `ADR-CANDIDATE-011` and cross-reference this record | `adr_generation` stage, in `ADR-CANDIDATE-011` |
