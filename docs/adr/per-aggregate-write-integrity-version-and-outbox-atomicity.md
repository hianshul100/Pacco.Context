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
| **Related** | `ADR-012` (the outbox and write-integrity decision this applies), `ADR-008` (hand-mapped documents, no migration tooling), `ADR-002` (no shared library), `ADR-024` (the atomic day move that depends on this), `ADR-025` (the new field on `Order` this protects) |

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
result is inspected, and the domain write and the outbox record commit together or not at all. The
implementation is a per-repository pattern, replicated, not a shared library.** Seven rules follow.

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

**Rule 5 — the domain write and the outbox record are one transaction.** `disableTransactions` must
be **false** for the three services in scope, and the setting is asserted in configuration rather
than assumed. Where a service's deployment cannot support transactional writes, that is a blocker to
raise, not a setting to flip back.

**Rule 6 — a conflict is retried at most once, by reloading, and then surfaced.** The reschedule is a
customer-initiated action with a `DO2` rejection vocabulary (`ADR-030`). Blind retry loops on a path
that takes and releases reservations can double-apply; one reload-and-retry, then an explicit
rejection, is the bound.

**Rule 7 — it is replicated per repository, deliberately.** `ADR-002` records that the platform has
no shared library and rejects introducing one. The pattern is therefore written three times, and
`patterns/data/` carries the canonical description so the three copies are the same shape. The cost —
three places to fix a bug in — is accepted and recorded.

## 6. Consequences

### 6.1 Positive

- `ADR-024`'s atomic release-and-take becomes genuinely atomic. A concurrent reschedule on the same
  resource now fails visibly rather than erasing the other.
- The worst failure mode on the platform — a lost update that still publishes its events ✅ — is
  closed for the three aggregates this feature writes.
- `Delivery.Version` stops being decorative. The source stops claiming a safety property the runtime
  does not have.
- A pattern exists for the other eight services, with a worked implementation in three.

### 6.2 Negative

- **The platform is left inconsistent.** Three aggregates are safe, eight are not, and nothing in the
  code marks the boundary. A developer who has seen `Resource` will reasonably assume `Parcel`
  behaves the same way. `R-30`.
- **Callers see conflicts they have never seen before.** This is the correct behaviour and it is
  still a behaviour change on existing write paths for `Order` and `Delivery`, including paths this
  feature does not touch. `FA2` scopes that blast radius.
- **Transactional writes have a deployment precondition.** Turning `disableTransactions` to false
  requires the datastore deployment to support transactions. `FA3` confirms it per environment before
  the setting changes; discovering this in production is the failure this action prevents.
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

## 8. Non-Functional Requirements & Testing

| # | Requirement | NFR | How it is verified |
|---|-------------|-----|--------------------|
| `N1` | Two concurrent writers to one `Resource` produce one success and one explicit conflict | `NFR-4` | Load twice, write twice. Assert one conflict |
| `N2` | Two concurrent writers to one `Order` produce one success and one explicit conflict | `NFR-4` | Same, on `Order`. This is the case that silently loses data today |
| `N3` | Two concurrent writers to one `Delivery` produce one success and one explicit conflict | `NFR-4` | Same, on `Delivery` |
| `N4` | **A conflicted write publishes no event** | `NFR-21`, Rule 2 | Force the non-match. Assert the outbox is empty and no message was dispatched. This is the regression test for the current CAP-04 defect |
| `N5` | A `Delivery` reloaded from the store carries the version it was saved with | Rule 3 | Save, reload, assert non-zero. Guards the `Delivery.Version` counter-example |
| `N6` | Every mutating method increments the version | Rule 4 | Call each mutator. Assert the version advances each time |
| `N7` | The same reschedule request applied twice results in one move | `NFR-7` | Replay the request. Assert one reservation and one history entry |
| `N8` | The domain write and the outbox record fail together | `ADR-012`, Rule 5 | Fault-inject between them. Assert neither is visible |
| `N9` | `disableTransactions` is false for the three services in scope | Rule 5 | Assert the effective configuration value in a test, not by reading the file |
| `N10` | A conflict is retried at most once before a rejection is returned | Rule 6 | Force a persistent conflict. Assert exactly two attempts and a `DO2` rejection code |

## 9. Relationship to the implementation pattern catalog

Extends `patterns/data/optimistic-concurrency-on-aggregate-write.md` with the rule the platform's own
implementation is missing — **the result of the conditional write must be inspected** — and names the
three in-scope aggregates so the eight out-of-scope ones are not mistaken for compliant.

Applies `patterns/integration/transactional-outbox-and-inbox-by-handler-decorator.md`, and records
Option C's finding against it: the inbox decorator does not cover HTTP-initiated commands, so it is
not an exactly-once mechanism for `DO2`'s edge write.

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

### 10.1 Documentation-versus-code conflicts

- **`Delivery` source versus `Delivery` runtime.** The entity's `Version` and `IncrementVersion`
  state a concurrency property the persistence layer does not implement ✅. Anyone reading the entity
  alone would conclude the aggregate is protected. Rules 3 and 4 close this, and `N5` keeps it closed.

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
| `FA3` | **[ACTION NOW]** Confirm, per environment, that the datastore deployment supports transactional writes before `disableTransactions` is set to false. A single-node deployment does not, and the setting will fail at runtime rather than at startup | Platform owner | **2027-01-30** — before `DO2` ships |
| `FA4` | **[handled later by DevOps]** Make CAP-04's pipeline execute its tests (`INF-6`, `ADR-018`). Every test in §8 that runs in CAP-04 is otherwise written and never run (`R-25`) | Platform owner | **2027-01-30** — before `DO2` ships |
| `FA5` | **[handled later by DevOps]** Add outbox depth and oldest-unpublished-age signal for the three services in scope (`INF-4`, `NFR-21`). An outbox that stops draining is currently indistinguishable from an idle one (`R-23`) | Platform owner | **2027-01-16** — before `DO2` reaches a shared environment |
| `FA6` | **[handled later by LLD]** Write the canonical pattern entry for scoped write integrity, and record in it which aggregates are covered and which are not, so the inconsistency in §6.2 is discoverable from the catalog (`R-30`) | `DO2` implementer | **2026-12-22** — during `DO2` low-level design |

## Assumptions, Blockers & Open Questions

### Assumptions

- **A1 ❓** The three services' datastore deployments can support transactional writes. This is the
  precondition for Rule 5 and is unverified from the workspace — `docker-compose` describes the local
  topology, not the deployed one. `FA3` verifies it per environment; if the answer is no anywhere,
  that environment cannot host `DO2` until it changes.
- **A2 `[INFERRED]`** An absent version element reads as zero on load, so existing documents need no
  backfill. This follows from the hand-mapped document model ✅ and the absence of a required-element
  contract, not from an observed migration.

### Blockers

- None that block this decision. `FA3` can block a specific environment, and `FA4` blocks meaningful
  verification in CAP-04.

### Open Questions

- **Q1** When are the remaining eight aggregates brought in? `AD-4` deliberately scopes this
  increment, and nothing currently schedules the rest. Platform architect, after `DO2` ships
  (`R-30`).
