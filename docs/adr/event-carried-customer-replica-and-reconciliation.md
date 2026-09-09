# ADR-009: Event-carried customer replicas in place of synchronous customer reads

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-009` |
| **Backlog candidate** | `ADR-CANDIDATE-009` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Storage / medium |
| **Supersedes / Superseded by** | — (nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-008` (the ownership rule this works around), `ADR-003` (the unguarded contract this rides on), `ADR-001` (the transport); paired with `ADR-010`, which records the *other* answer to the same question |

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

`ADR-008` forbids a service from reading another service's database. Two ways around that rule exist on
this platform, and the platform uses both against the same owning service. This record covers the
replication half; `ADR-010` covers the synchronous half.

What the source actually shows, which is narrower than the backlog anticipated:

1. ✅ **Exactly two services hold a customer replica** — `orders-service` and `parcels-service`. Each
   declares a `Customer` entity, an `ICustomerRepository`, a `CustomerDocument` and a Mongo repository
   registered in its own composition root
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:54`,
   `hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Infrastructure/Extensions.cs:52`).
2. ✅ **The replica carries one field: the customer id.** Both `Customer` entities have a single `Id`
   property set through the constructor, and both `CustomerDocument` classes declare `Id` and nothing
   else
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Core/Entities/Customer.cs:5-13`,
   `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs:6-9`).
   This is not a copy of customer *data*. It is a local index of which customer ids exist.
3. ✅ **One event feeds it, and one operation reads it.** `CustomerCreatedHandler` inserts the id after
   an existence check
   (`…Orders.Application/Events/External/Handlers/CustomerCreatedHandler.cs:17-25`); the only read
   anywhere is `ExistsAsync`, used to gate order creation
   (`…Orders.Application/Commands/Handlers/CreateOrderHandler.cs:31-34`) and parcel addition
   (`…Parcels.Application/Commands/Handlers/AddParcelHandler.cs:41`).
4. ✅ **`availability-service` declares the same handler and deliberately does nothing with it.** Its
   `CustomerCreatedHandler` returns a completed task, with a source comment stating that customer data
   *could* be saved depending on business requirements
   (`…Availability.Application/Events/External/Handlers/CustomerCreatedHandler.cs:6-12`). It is a
   subscriber, not a replica holder.
5. ✅ **`customers-service` publishes three customer events; only one is consumed anywhere.**
   `customer_created`, `customer_became_vip` and `customer_state_changed` are all declared in the
   platform manifest
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:31-33`). A
   workspace-wide search for the two state-bearing types outside their owning repository returns no
   handler, no external event class and no subscription.
6. ✅ **The owning aggregate has state the replicas cannot see.** `Customer` carries `State`
   (`Incomplete`, `Valid`, `Locked`, `Suspicious`) and `IsVip`, and raises `CustomerStateChanged` on
   every transition and `CustomerBecameVip` on promotion
   (`…Customers.Core/Entities/Customer.cs:16-17, 67-91`).
7. ✅ **No delete or deactivation event exists at all.** `customers-service` declares no removal
   command and no removal event, so nothing could remove a replicated id even if a consumer wanted to.

Points 5, 6 and 7 combine into the finding that makes this a decision rather than a mechanism note:
**a customer who is `Locked` or `Suspicious` still passes both gates.** Both call sites ask only
whether the id exists, and the event that would have told them the state changed has no consumer.

### 1.1 What this record does *not* cover

It does not cover synchronous cross-service reads — that is `ADR-010`, and the two records are written
to be read together, each stating when the other applies. It does not cover delivery reliability of the
feeding event, which is `ADR-012`, nor the naming convention that binds publisher to consumer, which is
`ADR-003`.

## 2. Decision

**A service that needs only to know whether a customer *exists* keeps a local, id-only replica of
customer identifiers, populated by subscribing to the customer-created event and stored in its own
database. A service that needs any customer attribute — state, VIP status, address — must not replicate
it, and must obtain it by the synchronous point-read recorded in `ADR-010`.**

That rule describes what the code does today. Three obligations are attached and are part of the
decision, because the current shape is only safe while the rule above holds exactly:

1. 🎯 **Any decision gated on a customer must state which of the two access modes it needs.** Today
   `CreateOrder` and `AddParcel` gate on existence alone, which silently accepts a locked or suspicious
   customer. Whether that is intended is question Q1.
2. 🎯 **Every replicated fact must have a named reconciliation path.** An id-only replica has one
   defensible path — it is append-only and ids are never reused — but that is an argument, not a
   mechanism, and it must be written down before any second field is added.
3. 🎯 **No attribute may be added to these replicas without first consuming the events that maintain
   it.** Adding `IsVip` or `State` to `CustomerDocument` while `customer_became_vip` and
   `customer_state_changed` remain unconsumed would create a field that is wrong from its second write
   onward.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Synchronous point-read for every customer check**, as `pricing-service` and `availability-service` already do. | Rejected for these two call sites because it puts `customers-service` on the critical path of order creation and parcel addition — the platform's two highest-volume writes. An outage or a slow response there would fail writes that only needed to know an id was valid. This is the trade-off `ADR-010` accepts elsewhere and this record declines here. |
| 2 | **Replicate the full customer aggregate** — email, name, address, state, VIP flag — and maintain it from all three customer events. | Rejected as written, and it is the alternative the platform most visibly gave up. It would have made VIP pricing and state gating local and fast. It was not taken, and the evidence is that the two state-bearing events have no consumer at all: the platform kept the replica trivial rather than take on the reconciliation duty a real projection requires. Section 4.2 records what that costs. |
| 3 | **No local customer knowledge at all** — let the write fail downstream when the customer turns out not to exist. | Rejected because the failure would surface as a rejected event on a separate channel rather than as a synchronous error, so the caller would learn about an invalid customer id only after the write was acknowledged (`ADR-014`). Validating cheaply and locally at the point of the write is the better caller experience. |
| 4 | **A shared read model or customer cache** that all services query. | Rejected because it reintroduces a shared data store — the thing `ADR-008` exists to prevent — and creates a component that every write path depends on, with no owner. It would trade eight private failure domains for one shared one. |

## 4. Consequences

### 4.1 Positive

1. ✅ Order creation and parcel addition validate the customer without a network call, so
   `customers-service` is not on the critical path of either write.
2. ✅ The replica is as small as a replica can be, so the class of bug where a copied field drifts from
   its source cannot occur for any field other than existence itself.
3. ✅ Because ids are never reused and no delete event exists, the append-only replica is
   `[INFERRED]` correct for the one question it is asked, as long as the rule in §2 holds.
4. ✅ Each replica lives inside the owning service's own database, so it inherits the ownership,
   credential scoping and backup position established by `ADR-008` with no extra infrastructure.

### 4.2 Negative

1. ✅ **A `Locked` or `Suspicious` customer can still create an order and add a parcel.** Both gates
   test existence only, and the event that reports the transition has no consumer. This is the
   sharpest consequence of the decision and it is live today, not hypothetical.
2. ✅ **A replica is written once and never touched again.** There is no update path, no delete path
   and no reconciliation job in either service.
3. ✅ **A customer created before either consumer was deployed is invisible to it.** The replica is
   built only from events observed since the subscription started; no backfill or replay path exists
   anywhere in the workspace.
4. ✅ **The platform holds two different answers for the same owning service.** `orders-service` and
   `parcels-service` replicate; `availability-service` and `pricing-service` call synchronously
   (`ADR-010`). Nothing in the repositories states which a new service should choose — that rule is
   what §2 supplies.
5. ❓ **A dropped or unprocessed customer-created message leaves a permanently unusable customer.**
   That customer can never create an order, and nothing detects or repairs the omission. The inbox
   decorator (`ADR-012`) protects against duplicates, not against loss.

### 4.3 Neutral / follow-on

1. ✅ `availability-service`'s no-op handler is a deliberate placeholder, not an oversight — the source
   comment says so. It costs one queue binding and one message round trip per customer created.
2. ✅ `deliveries-service` holds no external event handler of any kind, so it neither replicates nor
   point-reads customer data. It has no need for it today.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/data/event-carried-reference-replica.md` (Status: `Candidate`, evidence
   Moderate) — but in its most minimal possible form. The pattern describes "a minimal local replica of
   another service's data"; here the replica holds no data beyond the identifier.
2. **Constrained by** `patterns/data/database-per-service-with-document-mapping.md`, which is why the
   replica lives in the consuming service's own database rather than in a shared one.
3. **Depends on** `patterns/integration/service-owned-topic-exchange-messaging.md` for the feed and on
   `patterns/data/transactional-outbox-handler-decorator.md` for at-least-once handling of it.
4. **Paired with** `patterns/integration/narrow-synchronous-point-read.md`, which the same platform
   uses for the same owning service. §2 of this record is the missing rule for choosing between them.
5. **Pattern Drift:** none reportable. Drift requires an `Approved` pattern to violate, and every entry
   in `patterns/index.md` carries status `Candidate` (*Governance*).
6. **Pattern Update Proposal (correction):** the catalog lists this pattern's related services as
   "Orders, Availability, Deliveries". Source shows **Orders and Parcels**. `availability-service`
   subscribes but stores nothing, and `deliveries-service` has no external event handler at all. The
   conflict is stated in §6.1 and the pattern's service list should be corrected.
7. **Pattern Update Proposal (addition):** the pattern should require every replicated field to name
   the event that maintains it, and should state that a replica must not carry a field whose
   maintaining event has no consumer — the exact defect §2 obligation 3 forbids.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | `orders-service` holds a customer replica behind a domain repository registered in its composition root | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:54` |
| E2 | `parcels-service` holds the same replica, registered the same way | ✅ | `hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Infrastructure/Extensions.cs:52` |
| E3 | The replicated entity carries only an identifier | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Core/Entities/Customer.cs:5-13`; `hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Core/Entities/Customer.cs:5-13` |
| E4 | The persisted document declares an id and no other field | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs:6-9` |
| E5 | The replica is populated by the customer-created event after an existence check | ✅ | `…Orders.Application/Events/External/Handlers/CustomerCreatedHandler.cs:17-25`; `…Parcels.Application/Events/External/Handlers/CustomerCreatedHandler.cs:18-26` |
| E6 | The only read of the replica is an existence test gating order creation | ✅ | `…Orders.Application/Commands/Handlers/CreateOrderHandler.cs:31-34` |
| E7 | The same existence test gates parcel addition | ✅ | `…Parcels.Application/Commands/Handlers/AddParcelHandler.cs:41` |
| E8 | `availability-service` subscribes to the customer-created event and deliberately stores nothing | ✅ | `…Availability.Application/Events/External/Handlers/CustomerCreatedHandler.cs:6-12` |
| E9 | The owning aggregate carries a four-value state and a VIP flag, raising an event on each change | ✅ | `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Core/Entities/Customer.cs:16-17, 67-91` |
| E10 | Three customer events are published platform-wide | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:31-33` |
| E11 | Neither state-bearing customer event has any consumer outside its owning repository | ✅ | Workspace-wide search for `CustomerBecameVip` and `CustomerStateChanged` outside `hianshul100_Pacco.Services.Customers` — zero matches |
| E12 | No customer removal command or event exists | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:24-37` — the customers exchange declares two commands, three events and two rejections, none of them a removal |
| E13 | `deliveries-service` declares no external event handler at all | ✅ | `hianshul100_Pacco.Services.Deliveries/src/…Application/Events/External/` — directory absent |

### 6.1 Documentation-versus-code conflicts

1. ✅ **The pattern catalog names the wrong services.**
   `docs/architecture-inventory/patterns/index.md` lists this pattern's related services as "Orders,
   Availability, Deliveries". The code shows **Orders and Parcels**: `availability-service`'s handler
   is a deliberate no-op (E8) and `deliveries-service` has no external event handler (E13). The code is
   followed here.
2. ✅ **The backlog describes the replica as holding customer data.**
   `docs/architecture-inventory/adr-candidates.md` (candidate 009) calls it "their own local copy of
   customer data" and frames the risk as staleness. Source shows an id-only record with no attribute to
   go stale (E3, E4). The real exposure is different and narrower: unconsumed *state* events mean a
   locked customer is never rejected. This record follows the code and states that exposure instead.
3. ✅ **The backlog names the events imprecisely.** It refers to "a state change and a VIP-promotion
   event"; the declared names are `customer_state_changed` and `customer_became_vip` (E10). The finding
   that neither is consumed is confirmed (E11).

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | Customer identifiers are never reused, so an append-only replica cannot go wrong for the one question it answers | Ids are GUIDs assigned at creation, and no removal command or event exists anywhere in `customers-service` (E12) | The append-only argument in §4.1 collapses, and a recycled id would let a new customer inherit an old one's order history | Confirm with the owner of `customers-service` that no administrative deletion or re-import path exists outside the message contracts |
| A2 | Accepting an order from a `Locked` or `Suspicious` customer is a defect rather than an intended business rule | Those states exist and are transitioned deliberately (E9), which implies someone expected them to have an effect; no code anywhere acts on them | If it is intended, obligation 1 in §2 and question Q1 are unnecessary work, and the negative consequence in §4.2 is not a consequence at all | Ask the product owner what `Locked` and `Suspicious` are meant to prevent, and confirm whether order creation is in scope |
| A3 | The customer-created event is delivered to both consumers at least once in practice | The inbox and outbox decorators are enabled in both services (`ADR-012`), which provides deduplication and reliable publication | A customer whose event was lost is permanently unable to order, and nothing reports it (§4.2 item 5) | Compare the customer id count in `customers-service` against the replica counts in `orders-service` and `parcels-service` in a running environment |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so nobody can approve this record or accept the reconciliation obligations in §2 | This ADR leaving `Proposed`, and any decision on whether the locked-customer gap is fixed | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has confirmed whether any environment holds customers in the `Locked` or `Suspicious` state. If it does, orders are being accepted from them right now | Prioritising the fix for §4.2 item 1 — an active exposure is an incident, an empty state set is a backlog item | Platform owner | Count customers by state in each deployed environment and report whether any non-`Valid` customer has a recent order | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should order creation and parcel addition reject a customer who is `Locked` or `Suspicious`? | This is the decision the id-only replica quietly made. Answering yes means consuming `customer_state_changed` and adding a state field; answering no means the four states have no effect on the two operations that matter most | Yes — consume `customer_state_changed`, store the state alongside the id, and gate both handlers on it. Do this before adding any other replicated field, per §2 obligation 3 | Platform owner, with the product owner of `customers-service` |
| Q2 | **[ACTION NOW]** What repairs a replica that missed its creation event? | Today nothing does, and the customer is permanently unable to order with no error anyone would notice (§4.2 item 5) | Add a reconciliation query that compares replica ids against `customers-service` and re-emits the creation event for the difference. It needs an owner before it needs a design | Platform owner |
| Q3 | **[handled later by the design stage]** Should VIP status be replicated, given that `pricing-service` already reads it synchronously? | It is the one customer attribute with a live consumer. Replicating it would remove a synchronous hop from the pricing path; not replicating it keeps the replica trivial and correct | Do not replicate it. `pricing-service` reads it on a path that already tolerates a synchronous call (`ADR-010`), and replicating it would require consuming `customer_became_vip` and owning its reconciliation for no availability gain | Owners named per service once B1 is resolved |
| Q4 | **[handled later by the design stage]** Should the two replicas be recognised as one shared concern rather than two independent copies? | `orders-service` and `parcels-service` hold identical entities, documents, repositories and handlers. Any fix from Q1 or Q2 has to be written twice, and `ADR-003` guarantees no build-time signal if they diverge | Keep them separate — a shared package would contradict `ADR-002` — but require that any change to one is applied to the other in the same change set, and note the duplication in both repositories | Owners named per service once B1 is resolved |
