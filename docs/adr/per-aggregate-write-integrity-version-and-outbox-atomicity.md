# ADR-028: Write integrity is scoped to Resource, Order and Delivery — a version-conditioned update whose result is checked, in one transaction with the outbox

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-10-03 |
| **ADR id** | `ADR-028` |
| **Backlog candidate** | None. `adr-candidates.md` runs to `ADR-CANDIDATE-020` and records no candidate for scoped write integrity |
| **Category / Impact** | Data & Reliability / high |
| **Supersedes / Superseded by** | — (narrows how `ADR-012` is applied; supersedes nothing. `ADR-008` remains the mapping decision) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (`ADR-021` B1, `risk-constraint-gap-register.md` `G-01`) |
| **Base ref of all cited source** | `feature/14830/aidlc` |
| **Source** | `DO2` — *Customer Delivery Reschedule Confirmation & Slot Safe-Move*, work item **14830**, `intents/14830.md`. The reschedule is a release-and-take inside one aggregate (`ADR-024`), which is only safe if a concurrent writer cannot erase it |
| **Non-functional requirements** | `NFR-7` (**at risk**) — a reschedule is applied once and only once under concurrency and re-delivery; `NFR-4` (a concurrent write is not silently lost); `NFR-21` (a write that fails to publish its event is detectable) |
| **Infrastructure recommendation applied** | `INF-2` — per-aggregate write integrity, governed by `ADR-012`, delegated to **HLS**; and `INF-4` — outbox depth and age observability, governed by `ADR-012`, delegated to **HLS** |
| **Resolved decision applied** | `AD-4` option **B**, chosen by human, confidence high. Binding and settled — not re-opened here: *"Scope per-aggregate optimistic concurrency and domain-write/outbox atomicity to Resource, Order and Delivery for this increment, implemented as a reusable pattern that can later be adopted platform-wide."* |
| **Fit verdict applied** | Verdict 8 — per-aggregate write integrity realized against `ADR-012` in preference to `ADR-008`, `evolution_type: component_extension`, `can_extend_existing_component: true` |
| **Related** | `ADR-012` (the outbox and write-integrity decision this applies), `ADR-008` (hand-mapped documents, no migration tooling), `ADR-002` (no shared library), `ADR-024` (the atomic day move that depends on this), `ADR-025` (the new field on `Order` this protects, and the version this record maintains which `ADR-025` Rule 6 publishes as `ScheduleRevision`), `ADR-026` (the consumer that orders by that revision, and whose history write shares Rule 5's transaction), `ADR-030` (the message contracts whose `RequestId` Rule 8 records) |

## Notation

| Symbol | Meaning |
| --- | --- |
| ✅ | Confirmed current behaviour — observed in source at the cited path and line range |
| 🎯 | Target state — intended design, not in production today |
| ❓ | Needs validation — assumed or inferred, not observed in source |
| `[INFERRED]` | Conclusion drawn from observed evidence rather than stated by it |

## Contents

1. [Context](#1-context)
2. [Decision Drivers](#2-decision-drivers)
3. [Architecture Fit Evaluation](#3-architecture-fit-evaluation)
4. [Options Considered](#4-options-considered)
5. [Decision](#5-decision)
6. [Consequences](#6-consequences)
7. [Compliance Considerations](#7-compliance-considerations)
8. [Non-Functional Requirements & Testing](#8-non-functional-requirements--testing)
9. [Relationship to the implementation pattern catalog](#9-relationship-to-the-implementation-pattern-catalog)
10. [Evidence](#10-evidence)
11. [Follow-Up Actions](#11-follow-up-actions)
12. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Context

`ADR-024` makes a reschedule one atomic release-and-take inside the `Resource` aggregate. That is
correct only if the written aggregate cannot be silently overwritten by a concurrent writer. On this
platform today, it can be — in three different ways, each worse than the last.

**CAP-04 has the machinery and throws the answer away.** The repository writes with
`ReplaceOneAsync(r => r.Id == resource.Id && r.Version < resource.Version, ...)` ✅ — a correct
optimistic-concurrency predicate. **The result is never inspected.** When the predicate does not
match, Mongo reports zero documents modified, the method returns normally, the handler proceeds, and
the integration events publish as though the write had happened ✅
(`availability-service.md` §3.13). A lost update is therefore not merely silent: it is *announced*.
Downstream services act on a reservation that does not exist.

**CAP-07 has no machinery at all.** `Order` updates are whole-document replaces keyed on `Id` alone
✅ (`orders-service.md` §3.17). Two concurrent writers both win and the second erases the first.
`ADR-025` adds a field to this aggregate, which widens what a lost update can erase.

**CAP-09 declares the machinery and never uses it.** `Delivery` declares `Version` and
`IncrementVersion`, and **nothing ever calls `IncrementVersion`** ✅. `Version` is not in
`DeliveryDocument`, so it is not persisted, and `UpdateAsync` has no version predicate ✅
(`deliveries-service.md` §3.2, §3.18). The aggregate looks concurrency-safe in the source and is
last-writer-wins in production — the most expensive kind of wrong, because it survives code review.

Underneath all three, the outbox is not reliably transactional. At least one service runs with
`outbox.disableTransactions: true` ✅ (`Availability/.../Api/appsettings.json:129`), which means the
domain write and the outbox record are not one atomic unit. A crash between them loses the event
while keeping the write, or the reverse.

`AD-4` decides the scope: fix this properly for the three aggregates this feature actually writes,
build it as a pattern others can adopt, and do not attempt a platform-wide sweep in this increment.

## 2. Decision Drivers

| # | Driver | Where it comes from |
|---|--------|---------------------|
| D1 | `ADR-024`'s atomic move is meaningless if the written aggregate can be silently clobbered | `ADR-024`, `NFR-4` |
| D2 | A reschedule must apply exactly once under concurrency and message re-delivery | `NFR-7`, recorded **at risk** |
| D3 | A write that does not publish its event must be detectable | `NFR-21` |
| D4 | A lost update that still publishes events is worse than one that fails loudly | `availability-service.md` §3.13 |
| D5 | A platform-wide sweep is out of scope for this increment | `AD-4` option B, chosen by human |
| D6 | No shared library may carry the implementation | `ADR-002` |

## 3. Architecture Fit Evaluation

The binding fit verdict is `can_extend_existing_component: true`, `evolution_type:
component_extension`, realized against **`ADR-012`** rather than `ADR-008`, confidence high. All
three `new_*_required` flags are false.

| Dimension | Verdict | Consequence for this record |
|-----------|---------|-----------------------------|
| Governing record | `ADR-012`, not `ADR-008` | `ADR-008` governs how a document is mapped. The decision here is about *when a write is allowed to succeed* and *what is published when it does*, which is `ADR-012`'s subject |
| Owner | Each of CAP-04, CAP-07 and CAP-09, inside its own repository | No cross-service component is created |
| New deployable service | **Not required** | None is proposed |
| New bounded context | **Not required** | None is proposed |
| New runtime component | **Not required** | None is proposed |

The extension is to existing repositories and existing handler wiring. CAP-04 already has the
predicate and needs the check; CAP-09 already has the field and needs it persisted and enforced;
CAP-07 needs both. None of this is new architecture — it is the architecture `ADR-012` already
records, applied where it currently is not.

## 4. Options Considered

### Option A — fix write integrity platform-wide in this increment

Apply the version-conditioned write and the transactional outbox to all eleven services.

**Rejected by `AD-4`, and the rejection is sound.** Eleven services, each with hand-mapped documents
and no migration tooling ✅, and `ADR-018` records that pipelines do not reliably run tests — CAP-04's
in particular never runs them ✅. A sweep of this size with that verification floor is how a
correctness fix becomes an outage.

### Option B — scope to `Resource`, `Order` and `Delivery`, built as a reusable pattern *(chosen; `AD-4` option B, chosen by human)*

- **For.** It covers exactly the aggregates `DO2` writes, so the feature is safe. It is small enough
  to verify. It leaves a worked pattern and a catalog entry so later adoption is mechanical rather
  than exploratory.
- **Against.** The platform ends this increment with three safe aggregates and eight unsafe ones,
  which is an inconsistency somebody will trip over. `R-30` records it rather than pretending the
  inconsistency is temporary by default.

### Option C — rely on the inbox de-duplication decorator for exactly-once

**Rejected on evidence.** The inbox de-duplicates *messages*. The reschedule confirmation arrives
over HTTP, not as a message — the gateway is an HTTP edge ✅ — so the inbox decorator is not on that
path at all. Relying on it would leave `NFR-7` unmet in the exact case it is written for. This is
recorded as `R-15`.

### Option D — serialize writes with a distributed lock in Redis

**Rejected.** Redis is present on the platform, but no service uses it for locking and there is no
lock-lease or fencing discipline anywhere. A lock without fencing is a correctness illusion, and it
would be a new coordination mechanism where a version check — already present in one service — does
the job with no new infrastructure.

## 5. Decision

**For `Resource`, `Order` and `Delivery`, every aggregate write is a version-conditioned update whose
result is inspected, the outcome of the request that caused it is recorded with it, and the domain
write and the outbox record commit together or not at all. The implementation is a per-repository
pattern, replicated, not a shared library.** Eight rules follow.

**Rule 1 — every write carries a version predicate.** The update matches on the aggregate id **and**
the version the writer loaded. This already exists in CAP-04 ✅ and must be added to CAP-07 and
CAP-09.

**Rule 2 — the result is inspected, and a non-match is an error.** This is the rule that actually
changes behaviour. Zero documents modified means a concurrent writer won: the handler raises a
conflict, **no event is published**, and the caller is told. The current CAP-04 behaviour — discard
the result, publish anyway — is the defect this record exists to end.

**Rule 3 — the version is persisted.** The version must be in the document, in `AsDocument` and in
`AsEntity`. `Delivery.Version` is the worked counter-example: declared on the entity, absent from the
document, therefore always zero on reload, therefore a predicate built on it would match everything
✅. Persisting the version is a precondition for Rule 1, not a detail of it.

**Rule 4 — the version is incremented on every mutation.** `IncrementVersion` exists on `Delivery`
and is never called ✅. A version that never changes is a predicate that never fails.

**Rule 5 — the domain write, the outbox record and any companion document written by the same
handler are one transaction.** `disableTransactions` must be **false** for the three services in
scope, and the setting is asserted in configuration rather than assumed. Where a service's
deployment cannot support transactional writes, that is a blocker to raise, not a setting to flip
back — `FA3` is that check, and it is a **precondition of `DO2`**, not a late verification. The
transaction is explicitly not limited to two documents: `ADR-026` Rule 7 writes a history entry into
a separate collection alongside the delivery change, and `ADR-026` Rule 9 and Rule 8 below write a
recorded outcome. All of them are inside this one commit. A standalone datastore node cannot start a
transaction at all, so an environment that fails `FA3` cannot host `DO2` until it changes
(`ADR-026` `B2`).

**Rule 6 — a conflict is retried at most once, by reloading, and then surfaced.** The reschedule is a
customer-initiated action with a `DO2` rejection vocabulary (`ADR-030`). Blind retry loops on a path
that takes and releases reservations can double-apply; one reload-and-retry, then an explicit
rejection, is the bound.

**Rule 7 — it is replicated per repository, deliberately.** `ADR-002` records that the platform has
no shared library and rejects introducing one. The pattern is therefore written three times, and
`patterns/data/` carries the canonical description so the three copies are the same shape. The cost —
three places to fix a bug in — is accepted and recorded.

**Rule 8 — the outcome of an idempotent request is recorded in the same write.** Where a handler acts
on a client-supplied `RequestId` (`ADR-030` §5.2 — every reschedule message carries one), the
handler records that `RequestId` together with the outcome it produced — applied, or the specific
rejection — as part of the same transaction as the domain write. A later arrival of a `RequestId`
already recorded returns the recorded outcome and performs **no second mutation**. Three constraints
bind this:

1. **It is recorded with the aggregate being written, not in a side table written separately.** A
   separate write is a second transaction and reintroduces exactly the gap this rule closes. Whether
   the aggregate document is the right home for CAP-04 specifically — a `Resource` would accumulate
   one entry per reschedule against it, which grows without bound — is `FA7`.
2. **It is not the inbox de-duplication decorator.** The decorator de-duplicates *messages*; the
   customer's reschedule enters over HTTP ✅, so the decorator is not on that path (`R-15`, Option C
   above). This rule is what makes `NFR-7` hold for the HTTP edge.
3. **A recorded outcome is terminal.** Replaying a `RequestId` that was rejected returns the same
   rejection; it does not re-evaluate the guards against current state. A retry must not be able to
   turn a refusal into an acceptance, or an acceptance into a refusal, because the world moved
   between the two attempts.

The two consumers of this rule are `ADR-026` Rule 9 (CAP-09, keyed on the delivery write) and
`ADR-024` Rule 4 (CAP-04, keyed on the reservation move); `ADR-030` §5.3's `SEEN` branch is the
decision point both of them implement. Rule 8 and Rule 1 answer different questions and neither
replaces the other: the version predicate decides which of two *concurrent* writers wins, and the
recorded outcome decides what a *repeat of the same request* is told. A delivery retried after a
timeout is not a concurrent writer; it is the same writer asking again, and a version check lets it
through.

## 6. Consequences

### 6.1 Positive

- `ADR-024`'s atomic release-and-take becomes genuinely atomic. A concurrent reschedule on the same
  resource now fails visibly rather than erasing the other.
- The worst failure mode on the platform — a lost update that still publishes its events ✅ — is
  closed for the three aggregates this feature writes.
- `Delivery.Version` stops being decorative. The source stops claiming a safety property the runtime
  does not have.
- A pattern exists for the other eight services, with a worked implementation in three.
- **`NFR-7` acquires a mechanism it did not have.** Before Rule 8, "applied once and only once" rested
  on the inbox decorator, which is not on the HTTP path the customer actually uses (`R-15`). Rule 8
  places the guarantee where the request arrives, and `ADR-030`'s `RequestId` is what keys it.
- **The version becomes a published fact, not only a private guard.** `ADR-025` Rule 6 publishes the
  `Order` version as `ScheduleRevision` on `order_delivery_date_changed`, and `ADR-026` Rule 5 orders
  on it. Rule 4 — increment on every mutation — is therefore what makes the consumer's ordering sound,
  not only what makes the predicate fail. A version that stalls now produces silently discarded events
  downstream, which is a stronger reason to keep `N6` green than this record originally had.

### 6.2 Negative

- **The platform is left inconsistent.** Three aggregates are safe, eight are not, and nothing in the
  code marks the boundary. A developer who has seen `Resource` will reasonably assume `Parcel`
  behaves the same way. `R-30`.
- **Callers see conflicts they have never seen before.** This is the correct behaviour and it is
  still a behaviour change on existing write paths for `Order` and `Delivery`, including paths this
  feature does not touch. `FA2` scopes that blast radius.
- **Transactional writes are a deployment *blocker*, not a precondition to check late.** Turning
  `disableTransactions` to false requires the datastore deployment to support transactions; a
  standalone node cannot, and it fails when the first transaction is attempted rather than at
  startup. Three separate rules now depend on it — Rule 5, `ADR-026` Rule 7's history write and
  Rule 8's recorded outcome — so an environment that cannot support it cannot host `DO2` in a
  degraded form either. `FA3` is accordingly a blocker (see **Blockers** below), and its date has
  been pulled forward to the shared-environment milestone rather than the ship milestone, because
  the shared environment is where it will first be exercised.
- **Rule 8 adds a write that grows.** Each recorded outcome is retained, and nothing in this record
  prunes them. For `Order` and `Delivery` the volume is bounded by reschedules per order, which is
  small; for `Resource` it is not. `FA7` decides the retention and the home.
- **The transaction is now multi-collection.** `ADR-026` Rule 7 writes a history entry in a separate
  collection inside this commit. That is a larger transaction than the two-document domain-plus-outbox
  write this record originally scoped, and it widens what `FA3` must confirm.
- **The outbox is still a background dispatcher with no depth or age signal.** `INF-4` carries that
  to HLS. Until it lands, an outbox that stops draining looks like a quiet day. `R-23`.

### 6.3 Neutral and follow-on

- Existing documents have no persisted version. A missing version reads as zero, which is a valid
  starting point for every aggregate — no backfill is needed, which is fortunate given there is no
  migration tooling ✅.
- CAP-04's predicate uses `Version < resource.Version` ✅ rather than strict equality. Rule 1 does not
  mandate a rewrite of a working predicate; Rule 2's result check is what makes either form correct.
  `FA1` records the form chosen so the three copies agree.

## 7. Compliance Considerations

| Obligation | Source | How this record complies |
|------------|--------|--------------------------|
| Domain write and event publication are atomic | `ADR-012` | Rule 5 |
| No shared library is introduced | `ADR-002` | Rule 7, deliberately |
| Fields are added, never renamed or removed | `ADR-008`, `NFR-17` | Rule 3 adds the version field to three documents |
| A failed write publishes nothing | `ADR-012`, `NFR-21` | Rule 2 |
| One service, one database; no cross-service writes | `ADR-008` C5 | Each repository changes only its own store |
| A service publishes only to its own exchange | `ADR-001` C1 | No new exchange or routing change |
| Pipelines must actually execute the tests that prove this | `ADR-018`, `INF-6` | `FA4`, because CAP-04's pipeline does not ✅ |
| A reschedule is applied once and only once under re-delivery | `NFR-7` | Rule 8, on the HTTP path the decorator does not cover (`R-15`); `N7` and `N11` |
| An ordering key published to other contexts advances monotonically | `ADR-025` Rule 6, `ADR-026` Rule 5 | Rule 4; `N6` is the test that keeps `ScheduleRevision` sound downstream |
| A history entry is never visible without the change it records | `ADR-026` Rule 7 | Rule 5, which names the companion document as inside the same transaction |

## 8. Non-Functional Requirements & Testing

| # | Requirement | NFR | How it is verified |
|---|-------------|-----|--------------------|
| `N1` | Two concurrent writers to one `Resource` produce one success and one explicit conflict | `NFR-4` | Load twice, write twice. Assert one conflict |
| `N2` | Two concurrent writers to one `Order` produce one success and one explicit conflict | `NFR-4` | Same, on `Order`. This is the case that silently loses data today |
| `N3` | Two concurrent writers to one `Delivery` produce one success and one explicit conflict | `NFR-4` | Same, on `Delivery` |
| `N4` | **A conflicted write publishes no event** | `NFR-21`, Rule 2 | Force the non-match. Assert the outbox is empty and no message was dispatched. This is the regression test for the current CAP-04 defect |
| `N5` | A `Delivery` reloaded from the store carries the version it was saved with | Rule 3 | Save, reload, assert non-zero. Guards the `Delivery.Version` counter-example |
| `N6` | Every mutating method increments the version | Rule 4 | Call each mutator. Assert the version advances each time |
| `N7` | The same reschedule request applied twice results in one move | `NFR-7`, Rule 8 | Replay the same `RequestId`. Assert one reservation, one history entry, and that the second call returns the **recorded** outcome rather than re-evaluating |
| `N8` | The domain write and the outbox record fail together | `ADR-012`, Rule 5 | Fault-inject between them. Assert neither is visible |
| `N9` | `disableTransactions` is false for the three services in scope | Rule 5 | Assert the effective configuration value in a test, not by reading the file |
| `N10` | A conflict is retried at most once before a rejection is returned | Rule 6 | Force a persistent conflict. Assert exactly two attempts and a `DO2` rejection code |
| `N11` | **A replayed `RequestId` that was rejected returns the same rejection** | Rule 8 clause 3 | Reject a request on a guard, change the world so the guard would now pass, replay the same `RequestId`. Assert the original rejection is returned and nothing is written. A retry must not be able to launder a refusal into an acceptance |
| `N12` | The recorded outcome and the domain write are visible together or not at all | Rule 8 clause 1, Rule 5 | Fault-inject between the domain update and the outcome record. Assert neither is visible, and that a replay therefore re-executes rather than returning a half-recorded result |
| `N13` | A history entry written by `ADR-026` Rule 7 is not visible when its delivery write is rolled back | Rule 5 | Fault-inject after the history insert and before commit. Assert the history collection is empty |
| `N14` | A conflicted write leaves no recorded outcome behind | Rule 2, Rule 8 | Force the version non-match. Assert no `RequestId` entry was persisted, so the caller's retry is still able to succeed |

## 9. Relationship to the implementation pattern catalog

Extends `patterns/data/optimistic-concurrency-on-aggregate-write.md` with the rule the platform's own
implementation is missing — **the result of the conditional write must be inspected** — and names the
three in-scope aggregates so the eight out-of-scope ones are not mistaken for compliant.

Applies `patterns/integration/transactional-outbox-and-inbox-by-handler-decorator.md`, and records
Option C's finding against it: the inbox decorator does not cover HTTP-initiated commands, so it is
not an exactly-once mechanism for `DO2`'s edge write. Rule 8 is the replacement for the edge, and the
catalog entry is amended to say so rather than leaving the decorator looking sufficient.

## 10. Evidence

| # | Claim | Source |
|---|-------|--------|
| E1 | CAP-04 writes with a version predicate and discards the result; events publish regardless | `Availability/.../Mongo/Repositories/ResourceMongoRepository.cs`; `availability-service.md` §3.13 |
| E2 | `Order` updates are whole-document replaces keyed on id alone, with no version | `Orders/.../Mongo/Repositories/OrderMongoRepository.cs:45`; `orders-service.md` §3.17 |
| E3 | `Delivery` declares `Version` and `IncrementVersion`; nothing calls `IncrementVersion` | `Deliveries/.../Core/Entities/Delivery.cs`; `deliveries-service.md` §3.2 |
| E4 | `Version` is absent from `DeliveryDocument` and `UpdateAsync` has no version predicate | `deliveries-service.md` §3.18 |
| E5 | At least one service runs its outbox with `disableTransactions: true` | `Availability/.../Api/appsettings.json:129`; `availability-service.md` §3.14 |
| E6 | The platform has no shared library and has decided against one | `ADR-002` |
| E7 | No migration framework exists in any repository | `ADR-008`; `architecture-baseline.md` C8 |
| E8 | CAP-04's pipeline does not execute its tests | `ADR-018`; `availability-service.md` §5 |
| E9 | The gateway edge is HTTP, so inbox de-duplication is not on the reschedule command path | `ADR-005`, `ADR-013`; `ntrada.yml` |
| E10 | No message on the platform carries a client-supplied request identifier today; `RequestId` is new in `ADR-030` §5.2 | `architecture-views.md` §6 `GAP-13`, `GAP-17`; `ADR-030` §5.2 |
| E11 | The outbox implementation is a background dispatcher with no depth or age signal | `ADR-012`; `architecture-baseline.md`; `R-23` |

### 10.1 Documentation-versus-code conflicts

- **`Delivery` source versus `Delivery` runtime.** The entity's `Version` and `IncrementVersion`
  state a concurrency property the persistence layer does not implement ✅. Anyone reading the entity
  alone would conclude the aggregate is protected. Rules 3 and 4 close this, and `N5` keeps it closed.
- **`NFR-7` versus the inbox decorator.** The requirement is written as though message-level
  de-duplication satisfies it, and the catalog entry for the decorator does not say what it does not
  cover. The customer's reschedule arrives over HTTP ✅, where the decorator never runs. This was
  already recorded as `R-15`; this revision resolves it in the architecture rather than leaving it
  as a noted risk, by making Rule 8 the mechanism and `N7`/`N11` the proof.

## 11. Follow-Up Actions

**Reading the `By` column.** Each entry carries a calendar date followed by the delivery milestone
that date is derived from. The dates come from the one work-item 14830 wave calendar in
[`../specs/14830/solution-design.md`](../specs/14830/solution-design.md) §5.1, so every record in
`ADR-024`…`ADR-030` resolves the same milestone to the same date. If the wave calendar moves, that
section is the single place to change and these dates move with it; the milestone is what binds.

| # | Action | Owner | By |
|---|--------|-------|-----|
| `FA1` | **[handled later by HLS]** Fix one predicate form — strict equality or the existing less-than — and apply it identically in all three repositories, so Rule 7's three copies do not drift on day one | `DO2` implementer with the platform architect | **2026-12-08** — during `DO2` high-level design |
| `FA2` | **[ACTION NOW]** Enumerate the existing `Order` and `Delivery` write paths that will begin returning conflicts, and confirm each caller handles one. This changes behaviour on paths `DO2` does not otherwise touch | `DO2` implementer with the platform owner | **2027-01-30** — before `DO2` ships |
| `FA3` | **[ACTION NOW — blocker]** Confirm, per environment, that the datastore deployment supports transactional writes spanning **multiple collections**, before `disableTransactions` is set to false. A single-node deployment does not, and the setting fails at the first transaction rather than at startup. This is no longer a pre-ship check: Rule 5, Rule 8 and `ADR-026` Rule 7 all require it, so an environment that fails it cannot host `DO2` at all. Record the answer per environment in the deployment notes, not in a ticket comment | Platform owner | **2026-11-21** — before `DO1` reaches a shared environment, so the answer is known before `DO2` design closes |
| `FA4` | **[handled later by DevOps]** Make CAP-04's pipeline execute its tests (`INF-6`, `ADR-018`). Every test in §8 that runs in CAP-04 is otherwise written and never run (`R-25`) | Platform owner | **2027-01-30** — before `DO2` ships |
| `FA5` | **[ACTION NOW]** Add outbox depth and oldest-unpublished-age signal for the three services in scope (`INF-4`, `NFR-21`). An outbox that stops draining is currently indistinguishable from an idle one (`R-23`). The reschedule chain is four messages across three services (`ADR-030` §5.2), so a stalled outbox now presents to the customer as a confirmed reschedule that never settles — the failure is no longer internal | Platform owner | **2027-01-16** — before `DO2` reaches a shared environment |
| `FA6` | **[handled later by LLD]** Write the canonical pattern entry for scoped write integrity, and record in it which aggregates are covered and which are not, so the inconsistency in §6.2 is discoverable from the catalog (`R-30`) | `DO2` implementer | **2026-12-22** — during `DO2` low-level design |
| `FA7` | **[handled later by HLS]** Fix where CAP-04's recorded outcome lives and how long it is kept. Rule 8 clause 1 requires it inside the aggregate's transaction; a `Resource` accumulates one entry per reschedule against that resource and has no natural bound, unlike `Order` and `Delivery`. Decide the retention window and whether the entries are capped, pruned or moved, and state the chosen form so all three repositories implement the same shape | `DO2` implementer with the platform architect | **2026-12-08** — during `DO2` high-level design, because Rule 8 cannot be implemented without it |
| `FA8` | **[handled later by LLD]** Amend the outbox-and-inbox catalog entry to state explicitly that the inbox decorator does not cover HTTP-initiated commands, and to point at Rule 8 for that path. Leaving the entry as it stands is how `R-15` happened in the first place | `DO2` implementer | **2026-12-22** — during `DO2` low-level design |

## Assumptions, Blockers & Open Questions

### Assumptions

- **A1 ❓ — now carried as `B1`.** The three services' datastore deployments can support transactional
  writes **across more than one collection**. This is the precondition for Rule 5, Rule 8 and
  `ADR-026` Rule 7, and it is unverified from the workspace — `docker-compose` describes the local
  topology, not the deployed one. It was recorded here as an assumption on the first revision; with
  three rules now resting on it and `ADR-026` `B2` raising it independently, an unverified assumption
  is the wrong instrument. It is restated as a blocker below. `FA3` is the action.
- **A2 `[INFERRED]`** An absent version element reads as zero on load, so existing documents need no
  backfill. This follows from the hand-mapped document model ✅ and the absence of a required-element
  contract, not from an observed migration.
- **A3 `[INFERRED]`** A recorded outcome is small — a request identifier, a result and a timestamp —
  so Rule 8 does not materially change document size for `Order` or `Delivery`. This does not hold
  for `Resource`, which is why `FA7` exists.

### Blockers

- **B1 — the environments must support multi-collection transactional writes, and this is unverified.**
  Rule 5, Rule 8 and `ADR-026` Rule 7 all commit more than one document together. A standalone node
  cannot begin a transaction, and the failure surfaces at the first write, not at startup. Until
  `FA3` returns an answer per environment, `DO2` has no confirmed home. This blocker is owned by the
  platform owner and is raised identically in `ADR-026` `B2`; the two are the same blocker seen from
  the consumer and the mechanism.
- `FA4` blocks meaningful verification in CAP-04: every gate in §8 that runs there is otherwise
  written and never executed (`R-25`).
- **B2 — Rule 8 cannot be implemented in CAP-04 until `FA7` fixes where the outcome lives.** The rule
  requires the outcome inside the aggregate's transaction; for `Resource` the obvious home has no
  bound. This is a design decision owed before high-level design closes, not during implementation.

### Open Questions

- **Q1** When are the remaining eight aggregates brought in? `AD-4` deliberately scopes this
  increment, and nothing currently schedules the rest. Platform architect, after `DO2` ships
  (`R-30`).
- **Q2** Does the recorded outcome of Rule 8 need to survive beyond the retry window it exists to
  serve? Nothing in `DO2` reads it except a replay, and a replay arriving a month later is not a
  retry. If the answer is no, `FA7`'s retention window can be short and the `Resource` growth problem
  largely disappears; if a reschedule's outcome is wanted for audit, that belongs in `ADR-026`
  Rule 7's history, not here. Platform architect, with `FA7`.
