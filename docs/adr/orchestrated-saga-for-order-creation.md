# ADR-011: An orchestrated saga for order creation, and its state durability

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-011` |
| **Backlog candidate** | `ADR-CANDIDATE-011` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Orchestration / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-001` (the choreography this is the exception to), `ADR-003` (the contracts the coordinator publishes), `ADR-012` (the reliability guarantee this service does *not* have), `ADR-014` (the channel that reports the outcome to a caller) |

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

`ADR-001` made choreography the platform's default: a service publishes an event and interested
services react, with nobody owning the sequence. Order creation is the one process that does not fit
that shape, because it spans four service boundaries in a fixed order and a late failure has to undo
earlier steps.

1. ✅ **Exactly one process on the platform is orchestrated.** `ordermaker-service` runs a single
   state machine declared over one start message and four follow-on messages
   (`hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs:16-22`).
   No other repository references a saga library.
2. ✅ **The coordinator owns no aggregate and no database.** Its settings file declares no document
   database at all, and the repository contains no repository class, no document class and no domain
   project. Its whole state is one plain data class holding the order, customer, vehicle, reservation
   and the two parcel lists (`…/Sagas/AIMakingOrderData.cs:7-17`).
3. ✅ **It publishes commands onto four other services' exchanges.** The saga emits `CreateOrder` and
   `AddParcelToOrder` and `AssignVehicleToOrder` to Orders, `ReserveResource` to Availability, and
   `CancelOrder` on compensation — messages owned by services other than itself. This is the single
   platform-wide exception to the ownership rule `ADR-001` otherwise holds.
4. ✅ **It also makes two synchronous calls mid-process.** On the parcel-completion step it calls
   Vehicles for the best vehicle and Availability for the next free date before publishing the next
   command (`…/Sagas/AIOrderMakingSaga.cs:92-101`), so the saga is not purely message-driven.
5. ✅ **Saga state is held by whatever the library defaults to.** The composition root registers the
   saga library with one call and no persistence configuration
   (`…/Pacco.Services.OrderMaker/Extensions.cs:43`), and the project references the saga package but no
   persistence package for it (`…/Pacco.Services.OrderMaker.csproj:8-28`).
6. ✅ **Four of the five compensation methods do nothing.** Only the parcel step compensates, by
   publishing a cancellation; the other four return a completed task
   (`…/Sagas/AIOrderMakingSaga.cs:134-152`).
7. ✅ **The saga's only entry point is a direct HTTP call to the coordinator.** The command that starts
   it is bound to a `POST` route on the service itself (`…/Pacco.Services.OrderMaker/Program.cs:24-26`),
   the route carries no authentication, and no publisher for that command exists anywhere in the
   thirteen repositories. The coordinator has no module in any of the four gateway configurations, so
   the route is reachable only on the service's own container port.
8. ✅ **The coordinator's caller identity is empty by construction.** The saga sets a correlation
   context whose user context is a new empty object (`…/Sagas/AIOrderMakingSaga.cs:39-42`), so every
   command it publishes carries no caller — the bypass `ADR-006` records, demonstrated concretely.

### 1.1 What this record does *not* cover

This record fixes that order creation is orchestrated, and what the coordinator's state durability is.
It does not fix the message topology (`ADR-001`), the contract layer the published commands rely on
(`ADR-003`), the outbox that the layered services have and this service does not (`ADR-012` §4.3), or
how a caller learns the outcome (`ADR-014`). It also does not decide whether `ordermaker-service`
should be deployed at all; that question belongs to `ADR-017` and is carried here as blocker B2.

## 2. Decision

**Pacco keeps choreography as the default for cross-service behaviour, and uses an orchestrated saga
for exactly one process — order creation — implemented as a dedicated coordinator service that owns no
domain, reacts to the participants' events, publishes commands into the participants' exchanges, and
compensates on failure. The coordinator's state is held in the saga library's default in-memory store,
which is a durability position this record states rather than endorses.**

Five rules follow from the decision and are part of it:

1. ✅ **Orchestration is the exception and requires a named reason.** A process qualifies only when it
   has an identifiable start, an ordered sequence across more than two services, and a partial-completion
   state that needs undoing. Order creation is the only process that meets this today.
2. ✅ **The coordinator owns no business data and makes no business decision.** It sequences; the
   participants decide. This is why it has no database and no aggregate, and it is what keeps the
   exception from becoming a service-layer god object.
3. ✅ **Participants stay unaware they are in a saga.** They receive ordinary commands on their own
   exchanges and publish ordinary events; nothing in Orders, Parcels, Vehicles or Availability
   references the coordinator.
4. 🎯 **Saga state must be durable before this service is deployed anywhere shared.** The default store
   loses in-flight orders on restart. Until a persistence backend is configured, an order can be left
   half-created with no record that the process ever started.
5. 🎯 **Every step that can be undone must have a compensation.** Four of the five compensation methods
   are empty today, so a failure after the reservation step leaves a scarce resource reserved for an
   order that will never exist.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Extend choreography to order creation** — let each participant react to the previous participant's event with no coordinator. | Rejected because the process has a real ordering and a real failure mode. With choreography, no service knows the process reached step three, nothing can answer "where is this order now?", and compensation would have to be distributed across four services that each know only their own step. The platform would still need a place to hold the parcel list across steps, and that place is the coordinator by another name. |
| 2 | **Orchestrate with a durable workflow engine** — a hosted engine with persisted state, retries, timers and a visual history. | Rejected in the implementation, not on the merits: nothing in the workspace references one, and adopting it would add an infrastructure component to a platform that already runs nine of them on a single host (`ADR-017`). It remains the strongest answer to rule 4 and should be reconsidered when saga durability is fixed rather than dismissed. |
| 3 | **Put the sequence inside `orders-service`** — let the order aggregate drive its own creation across the other services. | Rejected because it would make Orders a caller of Parcels, Vehicles and Availability, turning the platform's narrowest coupling into its widest, and because a long-running multi-message process inside an aggregate breaks the buffered-event model every layered service uses (`patterns/domain/aggregate-buffered-domain-events.md`). |
| 4 | **A synchronous composite endpoint** — one request that calls the four services in order and returns the finished order. | Rejected because it reintroduces exactly the coupling `ADR-001` was taken to avoid: four synchronous hops, availability equal to the product of four services' availability, and no compensation story beyond an exception. The process is also genuinely long-running — it waits on a reservation date — which no request timeout accommodates. |

## 4. Consequences

### 4.1 Positive

1. ✅ One place answers where an order is in its creation sequence, and one file describes the whole
   process end to end.
2. ✅ Compensation has an owner. The parcel step's cancellation is written once in the coordinator
   rather than reconstructed by four services.
3. ✅ Participants are unchanged by the existence of the saga; the coordinator can be removed and the
   four services still work independently.
4. ✅ The exception is bounded to one process and one repository, so its cost is contained and its
   presence is obvious to anyone reading the repository list.
5. ✅ The coordinator stamps a state header on every message it publishes, which is what lets the
   platform's status projection report saga progress without knowing what a saga is (`ADR-014` §1).

### 4.2 Negative

1. ✅ **In-flight orders are lost on restart.** `[INFERRED]` no persistence is configured and no
   persistence package is referenced, so state lives in the process. A restart between step one and
   step five leaves an order created, parcels possibly attached, and nothing that will ever finish it.
2. ✅ **A late failure strands a reserved resource.** With four of five compensations empty, a failure
   after the reservation step releases nothing.
3. ✅ **The coordinator breaks the exchange-ownership rule.** It publishes into four other services'
   exchanges, so `ADR-001`'s "a service publishes only its own messages" is true of ten deployables and
   false of one — and a reader of the message topology cannot tell from the exchange who published.
4. ✅ **It acts with no caller identity.** Commands it publishes carry an empty user context, so the
   ownership guards in the receiving handlers have nothing to check (`ADR-006` §2).
5. ✅ **Two synchronous calls sit inside an asynchronous process.** If Vehicles or Availability is
   unavailable at the parcel-completion step, the saga step throws rather than retrying, and there is no
   compensation on that step to unwind what came before.
6. ✅ **Three declared contracts are dead.** An approval command and a rejection event are declared and
   never published, and an event is subscribed and forwarded to the coordinator that no saga action
   receives — the participant that acts on it is `orders-service`, which approves the order itself.

### 4.3 Neutral / follow-on

1. ✅ The coordinator has no outbox, unlike the seven layered services. That is consistent with its
   shape — it persists no domain state — but it means its publications have no reliability guarantee at
   all, which is a different gap from the one `ADR-012` closes.
2. ✅ The coordinator is defined in both container-compose service stacks and scraped for metrics, and
   is absent from both process manifests and from all four gateway configurations. Whether it is
   intended to run is `ADR-017`'s question, carried here as blocker B2.
3. ✅ The saga resolves its identity from the order id on every message, so the process is keyed by the
   thing it creates. That is the right key and it also means two concurrent sagas for one order id would
   collide silently.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates `orchestration/saga-process-manager.md`.** This record is the decision that pattern
   describes: one coordinator, no owned domain, participants unaware, compensation on the coordinator.
   The pattern is catalogued with confidence *Moderate* and one deployable using it, which matches the
   evidence here exactly.
2. **Constrains the pattern on durability.** The catalog's recommendation is to adopt the shape only
   once saga state is durable. Decision rule 4 turns that recommendation into a condition of this
   record: the coordinator may be developed against the in-memory default, and may not be relied on in a
   shared environment until state survives a restart.
3. **Constrains the pattern on compensation completeness.** The catalog notes four of five compensation
   methods are empty. Decision rule 5 makes filling them a precondition rather than an improvement,
   because the reservation step consumes a scarce resource.
4. **Deliberately diverges from `integration/service-owned-exchange-topology.md`.** Every other
   deployable publishes only messages it owns. The coordinator publishes into four foreign exchanges by
   design, because a coordinator that owned its own commands would need every participant to subscribe
   to it — which would make the participants saga-aware and defeat rule 3. The divergence is accepted
   and is scoped to this one service.
5. **Deliberately diverges from `reliability/transactional-outbox-inbox-decorator.md`.** The coordinator
   applies neither decorator. That is coherent with owning no database — there is no local transaction to
   make atomic with the publish — but it means the coordinator is the least reliable publisher on the
   platform, and the divergence is a gap rather than a simplification.
6. **Pattern Drift: not applicable.** Every entry in `docs/architecture-inventory/patterns/index.md`
   currently carries status `Candidate`. Drift is reportable only against an `Approved` pattern, so no
   drift is recorded for this ADR.
7. **Pattern Update Proposal.** `orchestration/saga-process-manager.md` should record two facts this
   record establishes and the pattern file does not: that the process's only trigger is an
   unauthenticated HTTP route on the coordinator itself with no publisher anywhere in the workspace, and
   that the coordinator deliberately publishes into foreign exchanges. Its **Related ADRs** entry should
   change from `None` to `ADR-011`.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| 1 | One start action and four follow-on actions declare the saga | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs:16-22` |
| 2 | Every message is keyed on the order id | ✅ | `…/Sagas/AIOrderMakingSaga.cs:45-54` |
| 3 | Step one publishes the create command | ✅ | `…/Sagas/AIOrderMakingSaga.cs:63-68` |
| 4 | Step two publishes one add-parcel command per parcel | ✅ | `…/Sagas/AIOrderMakingSaga.cs:73-81` |
| 5 | Step three calls Vehicles and Availability synchronously, then publishes the assign command | ✅ | `…/Sagas/AIOrderMakingSaga.cs:92-109` |
| 6 | Step four publishes the reservation command | ✅ | `…/Sagas/AIOrderMakingSaga.cs:113-119` |
| 7 | Step five publishes completion and closes the saga | ✅ | `…/Sagas/AIOrderMakingSaga.cs:124-131` |
| 8 | Each publication carries a saga-state header | ✅ | `…/Sagas/AIOrderMakingSaga.cs:23` |
| 9 | The correlation context is built with an empty user context | ✅ | `…/Sagas/AIOrderMakingSaga.cs:39-42` |
| 10 | Four compensations are no-ops; only the parcel step cancels the order | ✅ | `…/Sagas/AIOrderMakingSaga.cs:134-152` |
| 11 | Saga data holds order, customer, vehicle, reservation and both parcel lists | ✅ | `…/Sagas/AIMakingOrderData.cs:7-17` |
| 12 | The saga library is registered with no persistence configuration | ✅ | `…/Pacco.Services.OrderMaker/Extensions.cs:43` |
| 13 | The project references the saga package and no persistence package for it | ✅ | `…/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj:8-28` |
| 14 | Five external events are subscribed at startup | ✅ | `…/Pacco.Services.OrderMaker/Extensions.cs:58-62` |
| 15 | Six message types are forwarded to the coordinator, including one with no matching saga action | ✅ | `…/Handlers/AIOrderMakingHandler.cs:10-42` |
| 16 | The start command is bound to an unauthenticated `POST` route on the service | ✅ | `…/Pacco.Services.OrderMaker/Program.cs:24-26` |
| 17 | The service declares no document database and, in the local profile, no name-based routing | ✅ | `…/Pacco.Services.OrderMaker/appsettings.json:23-34` |
| 18 | The approval command and the rejection event are declared and never published | ✅ | `…/Commands/External/ApproveOrder.cs`, `…/Events/Rejected/MakeOrderRejected.cs` (workspace-wide search finds no publisher) |
| 19 | Order approval is performed by `orders-service`, not by the coordinator | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Events/External/Handlers/ResourceReservedHandler.cs:24-36` |

### 6.1 Documentation-versus-code conflicts

1. **A designed rejection path that does not exist.** The message manifest consumed by
   `operations-service` lists a rejection message under the coordinator's exchange
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`), and the
   coordinator declares the matching type — but nothing publishes it, in any repository. A caller
   watching for a rejection of order creation will wait indefinitely. Recorded as **Future/Intended
   State (Not Implemented)**; the documented contract is the intent and the code is the current truth.
2. **A subscription with no receiver.** `…/Pacco.Services.OrderMaker/Extensions.cs:58-62` subscribes to
   the resource-reserved event and `…/Handlers/AIOrderMakingHandler.cs:10-42` forwards it to the
   coordinator, but the saga declares no action for it (evidence 1). The step it appears to represent is
   performed by `orders-service` (evidence 19). Recorded as dead code rather than reconciled: the
   subscription reads as an intended sixth step that was implemented elsewhere.
3. **The service's own settings contradict the platform's security and storage records.** The
   coordinator has no credential-store section and no document-database section, while nine of the
   eleven deployables have both (`ADR-016` §1). It is unresolved whether the coordinator is exempt by
   design or simply unfinished. Recorded as **Unverifiable — Missing Source Evidence**; no design note
   exists in any clone that states an intent either way.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section records what this document assumes, what is blocking it, and what still needs an answer.
> Items here are not decided. Treat every entry as open until an owner closes it.

### Assumptions

| ID | Assumption | Why we made it | Impact if wrong |
| --- | --- | --- | --- |
| A1 | The saga library's default store keeps state in the running process only. | No persistence backend is configured and no persistence package is referenced (evidence 12, 13); an in-memory default is the only remaining possibility. | If the library defaults to a durable store, negative consequence 1 and decision rule 4 are unnecessary, and blocker B2's severity drops. |
| A2 | The coordinator is intended to run as part of the platform, not as a discarded experiment. | It is defined in both container stacks, given a port, and scraped for metrics. | If it is an experiment, this record should be marked superseded and the coordinator removed from the container stacks and the metrics job list. |
| A3 | Order creation is the only process on the platform that needs orchestration. | No other repository references the saga library, and no other multi-service ordered sequence appears in the message flow. | If a second such process exists or is planned, the "exception requires a named reason" rule needs a written test rather than a single example. |

### Blockers

| ID | Blocker | Who is affected | Owner |
| --- | --- | --- | --- |
| B1 | No repository in the workspace names an owner, a team or a review group, so this record cannot list deciders. | Every ADR in the set. | **[handled later by the architecture PR review stage]** — the reviewer assigns deciders when the batch is reviewed as a whole. |
| B2 | Saga state durability is unresolved, so it cannot be stated whether an in-flight order survives a restart. | Anyone deploying the coordinator outside a developer machine. | **[ACTION NOW]** — the author of this batch records the position in decision rule 4 and marks the service development-only until a persistence backend is configured. |
| B3 | The process has no producer: the only way to start it is an unauthenticated HTTP call to the coordinator's own port, and no gateway route reaches it. | Anyone trying to exercise order creation from a client. | **[ACTION NOW]** — recorded in context point 7 and consequence 4.3.2; the choice between adding a gateway route and removing the coordinator is `ADR-017`'s to make. |

### Open Questions

| ID | Question | Why it matters | Owner |
| --- | --- | --- | --- |
| Q1 | Which persistence backend should hold saga state, given the coordinator deliberately owns no database? | A store for the coordinator reopens the "owns no data" rule, so the answer shapes decision rule 2 as well as rule 4. | **[handled later by the batch 4 authoring stage, in the record covering storage and reliability]** |
| Q2 | Should the four empty compensations be filled, or should the failing steps be made impossible to reach instead? | Filling them is work; making them unreachable changes the participants. Both are valid and they lead to different services changing. | **[handled later by the architecture PR review stage]** |
| Q3 | Should the coordinator act as an identified caller rather than with an empty user context? | Every command it publishes bypasses the receiving handlers' ownership checks, which is the concrete form of the gap `ADR-006` records. | **[handled later by the batch 4 authoring stage, in the record covering service-to-service identity]** |
| Q4 | Is the resource-reserved subscription meant to be removed, or is `orders-service`'s approval meant to move into the coordinator? | One is deleting dead code; the other moves a business decision across a service boundary. | **[ACTION NOW]** — raised here with both readings and their evidence, for the deciders assigned under B1 to choose between. |

