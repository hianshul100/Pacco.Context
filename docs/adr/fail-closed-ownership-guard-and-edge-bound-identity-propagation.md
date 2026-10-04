# ADR-029: The ownership guard fails closed, and edge-bound customer identity is carried on every message of the reschedule chain

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-10-03 |
| **ADR id** | `ADR-029` |
| **Backlog candidate** | None. `adr-candidates.md` runs to `ADR-CANDIDATE-020` and records no candidate for the guard; the fail-open form is recorded in `ADR-006` as current behaviour |
| **Category / Impact** | Security & Access / critical |
| **Supersedes / Superseded by** | — (a deliberate departure from the fail-open position `ADR-006` records, for the paths named here. `ADR-006` is not superseded) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (`ADR-021` B1, `risk-constraint-gap-register.md` `G-01`) |
| **Base ref of all cited source** | `feature/14830/aidlc` |
| **Source** | `DO1`, `DO2` and `DO3`, work item **14830**, `intents/14830.md`. Every objective in this intent is customer-facing and operates on a resource the customer must own |
| **Non-functional requirements** | `NFR-2` (every new customer-facing route authenticates and authorizes server-side), `NFR-22` (a caller can act only on their own order, delivery and reservation), `NFR-1` (a customer sees only their own deliveries) |
| **Infrastructure recommendation applied** | `INF-5` — *"carry edge-bound customer identity into the reserve and release legs with a fail-closed guard"*, governed by `ADR-006`, delegated to **HLS**. The recommendation is quoted as received; the mechanism it names was written before `ADR-024` Rule 1 fixed the reschedule as a single move operation. Rule 4 below applies the recommendation's intent — identity carried from the edge, checked by the service holding the data — to the operation that actually exists |
| **Related** | `ADR-006` (the authorization decision whose recorded fail-open form this departs from), `ADR-005` (the gateway and its payload binding), `ADR-026` (the replica that supplies CAP-09 the customer id), `ADR-024` (the single move operation this guards, and Rule 8's owner fields on `Reservation` that make the guard possible), `ADR-030` (the message payloads that carry the identity, §5.2, and the not-owned rejection code) |

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

This intent puts a customer in direct control of a delivery date. Every one of its routes answers the
question "may this caller act on this thing?" — and the platform's current answer to that question
has three holes.

**The ownership guard is fail-open.** The same guard is written out by hand in six CAP-07 handlers
✅ (`orders-service.md` §3.8):

> if the identity is authenticated **and** the identity is not the order's customer **and** the
> identity is not an admin, then refuse.

An **unauthenticated** caller fails the first condition and therefore passes the whole guard. The
check is written as "refuse known-bad callers" rather than "admit known-good callers", so the unknown
caller is admitted. It is duplicated six times, so there is no single place to fix it and no way to
be sure every copy is identical.

**The gateway's protection is a spelling-dependent string.** The platform binds the customer id from
the validated token into the request payload — `customerId: @user_id` at `ntrada.yml:103` ✅ — which
overwrites whatever the client supplied and is the mechanism that prevents cross-customer
reservation. The catalog also records the matching known gap: **if the bind name is misspelled, the
binding silently does nothing and the client's own value is used**. There is no schema, no startup
validation and no test that would notice. A one-character typo in a YAML file is the difference
between an authorization control and no authorization control.

**CAP-04 cannot tell whose reservation it holds.** `Reservation` is a date and a priority ✅ — it
records no owner at all, so no amount of identity carried *to* CAP-04 lets it check anything until
that changes. `ADR-024` Rule 8 adds the owner fields; this record is what uses them. Two related
facts sit alongside: `ReleaseResourceReservation` carries the resource and the date and no customer
✅, and `ReleaseReservation` returns silently when nothing matches ✅, so releasing the wrong thing
and releasing nothing look identical from the outside.

**On the release leg, and a correction.** The first revision of this record treated the reschedule as
a release followed by a take and put the ownership check on `ReleaseResourceReservation`. `ADR-024`
Rule 1 settles that the reschedule is a **single** operation inside CAP-04 that issues neither
`ReserveResource` nor `ReleaseResourceReservation`. A guard added to the release command would
therefore have been correct code on a path this feature never takes. The unguarded standalone release
route is nonetheless a real platform defect — it is simply a different one, and it is carried as
`FA4` with its own owner rather than folded into `DO2` and assumed closed.

Add to this that `GET /deliveries/{deliveryId}` performs **no authorization whatsoever** ✅ and
returns whatever document it finds, and `DO1`'s customer-facing delivery read is being built on a
service with no concept of who is asking.

## 2. Decision Drivers

| # | Driver | Where it comes from |
|---|--------|---------------------|
| D1 | Every new customer-facing route must authorize server-side | `NFR-2` |
| D2 | A caller may act only on their own order, delivery and reservation | `NFR-22` |
| D3 | A customer must not see another customer's delivery | `NFR-1` |
| D4 | The existing guard admits unauthenticated callers | `orders-service.md` §3.8 |
| D5 | CAP-04 records no owner on a reservation, so it cannot authorize a move against one | `Reservation` entity ✅; `ADR-024` Rule 8 |
| D7 | The reschedule is one operation, not two legs, so the guard must sit on that operation | `ADR-024` Rule 1 |
| D8 | Identity must be carried, not re-derived, because the saga path drops it today | `architecture-views.md` §6 `GAP-13` |
| D6 | The gateway binding is the only cross-customer control and it is a string literal | `ntrada.yml:103`, catalog known gap |

## 3. Architecture Fit Evaluation

Two binding fit verdicts bear on this record, both `can_extend_existing_component: true` with all
three `new_*_required` flags false.

| Verdict | Dimension | Evolution type | Consequence here |
|---------|-----------|----------------|------------------|
| 4 | Customer-facing eligible-delivery read with a server-side ownership guard | `service_extension`, CAP-09 | The guard this record defines is what makes that read safe |
| 10 | Edge exposure of the new routes | `configuration`, CAP-02 — **GOVERNED** | The new routes are gateway configuration, and `ASM-18` makes them inherit the existing write mode. The bind is configuration too, which is why Rule 5 tests it |

No new deployable service, bounded context or runtime component is required, and none is proposed.
The extension is to handler code in CAP-07 and CAP-09, a message field in CAP-04's contract, and
declarative configuration in CAP-02.

## 4. Options Considered

### Option A — fail-closed guard at each service, with edge-bound identity carried end to end *(chosen)*

Deny unless the caller is proven to be the owner or an admin, and carry the bound customer identity
on every message of the chain so the service holding the data can check it against what it recorded.

- **For.** It is the only option that satisfies `NFR-2` and `NFR-22` as written. It removes the
  unauthenticated-caller hole on the new paths and makes the reschedule checkable at CAP-04, where
  the reservation actually lives.
- **Against.** It departs from the fail-open form `ADR-006` records as current behaviour, so two
  shapes of guard exist in CAP-07 until `FA2` converges them. It requires identity fields on four new
  message contracts, which `ADR-003`'s naming-convention-only compatibility makes a careful change,
  and it cannot be implemented until `ADR-024` Rule 8 gives `Reservation` an owner to compare against.

### Option B — rely on the gateway binding alone

**Rejected.** The binding is a single string in a YAML file with no validation, and the catalog
records the exact failure: a misspelled bind name silently restores the client's value. An
authorization model whose entire strength is a correctly-spelled configuration key is not an
authorization model. Defence in depth is the point, and `NFR-2` says *server-side*.

### Option C — reuse the existing guard as written

**Rejected.** It admits unauthenticated callers ✅. Copying it into new customer-facing routes would
propagate a known bypass into the most sensitive routes this intent adds.

### Option D — introduce a shared authorization library used by all services

**Rejected.** `ADR-002` records that the platform has no shared library and has decided against one.
The convergence this needs is `FA2`'s single guard per service, not a new cross-service dependency.

### Option E — centralize ownership checks in the gateway

**Rejected.** Ntrada is declarative routing and payload binding ✅; it has no access to order or
delivery ownership, which lives in each service's own store. The check must happen where the data is.

## 5. Decision

**Every route this intent adds authorizes server-side with a fail-closed guard, and the edge-bound
customer identity is carried on every message of the reschedule chain so that the service holding the
data can check ownership itself.** Seven rules follow.

**Rule 1 — deny by default.** The guard admits only a caller who is authenticated **and** is the
owner of the thing being acted on, or is an admin. An unauthenticated caller is refused, always. The
condition is written as an admit-list, not a refuse-list, because the refuse-list form is exactly how
the current hole was created ✅.

**Rule 2 — an absent or empty caller context is a refusal, not a pass.** This is Rule 1 restated
because it is the specific case that fails today. An empty identity must be the *most* restrictive
input, not the least.

**Rule 3 — the guard is written once per service.** One place in CAP-07 and one in CAP-09, called by
every handler that needs it. The existing six copies ✅ are the reason: six copies cannot be audited
and cannot be fixed once. New handlers call the single guard; `FA2` converges the existing six.

**Rule 4 — the customer identity is carried on every message of the reschedule chain, and the
ownership check happens where the data lives.** The first revision of this record described the
reschedule as a reserve leg and a release leg and put the guard on
`ReleaseResourceReservation`. That description was wrong, and the guard was in the wrong place.
`ADR-024` Rule 1 fixes the reschedule as **one** operation inside CAP-04 — it issues no
`ReserveResource` and no `ReleaseResourceReservation`, so a guard added to the release command would
never execute on this path at all. Four obligations replace it:

1. **Every message in the chain carries `CustomerId`.** `ADR-030` §5.2 fixes the four payloads, and
   `CustomerId` is a required field on all of them. It is bound from the validated token at the edge
   (Rule 5) and copied forward unchanged; no service re-derives it, and no service trusts a value a
   client supplied.
2. **CAP-04 checks ownership against what it has recorded, not against what it was told.** The
   caller's `CustomerId` and `OrderId` are compared to the owner fields `ADR-024` Rule 8 adds to
   `Reservation`. A move whose current-day reservation is held by a **different** order is refused
   explicitly — `ADR-024` Rule 4's third row and `ADR-030` §5.3's `MINE` branch are the same check.
   A refusal is a rejection code, never the current silent return ✅.
3. **A reservation with no recorded owner is not mine.** Rule 2's principle applied to data: absent
   ownership is the most restrictive input, not the least. `ADR-024` Rule 8 makes the fields nullable
   and additive precisely so that the legacy case fails closed rather than passing.
4. **The standalone release route is a separate defect and is not fixed here.** `ReleaseResourceReservation`
   today accepts a release without proving the caller holds the reservation ✅. That is real and it
   remains open; it is simply not on the reschedule path. It is `FA4`, with its own owner and date,
   so that closing `DO2` is not mistaken for closing it.

The new owner fields and the message fields are additive and follow the platform's naming convention,
per `ADR-003`, `NFR-15` and `ADR-030`.

**Rule 5 — the gateway binding is asserted by a test, not by reading the file.** A test exercises the
route with a client-supplied `customerId` that differs from the token's subject and asserts the
server observed the **token's** value. This is the only mechanism that catches the misspelled-bind
gap, which no schema, compiler or startup check will catch.

**Rule 6 — the delivery read is scoped by the replicated customer id.** CAP-09 gains the owning
customer from `ADR-026`'s replica, and the read returns a delivery only when the replica's customer
matches the caller. A delivery with **no** replicated customer is not returned to a customer caller —
absence is not ownership. This is the first authorization CAP-09 has ever performed on this path ✅.

**Rule 7 — not-owned and not-found are deliberately indistinguishable to a customer caller.** A
customer probing identifiers must not be able to tell a delivery that is not theirs from one that
does not exist. The distinction is recorded in logs under the correlation identifier, where an
operator can see it and a caller cannot.

## 6. Consequences

### 6.1 Positive

- The new customer-facing routes do not inherit the unauthenticated-caller bypass ✅. `NFR-2` and
  `NFR-22` are satisfiable on the paths this intent adds.
- CAP-04 can refuse a release that does not belong to the caller, instead of releasing silently and
  reporting nothing ✅.
- The gateway binding stops being an untested assumption; `N7` fails loudly if the bind name is ever
  misspelled or removed.
- CAP-09 performs authorization for the first time, which is the precondition for `NFR-1`.

### 6.2 Negative

- **CAP-07 ends this increment with two guard shapes.** New handlers fail closed; the six existing
  copies still fail open ✅. Anyone reading one and assuming the other is wrong. `R-18` carries the
  existing bypass, and `FA2` is the convergence that closes it — this record does not close it,
  because changing six existing authorization paths is a blast radius this feature should not absorb
  unannounced.
- **Four message contracts carry an identity field, not one.** `ADR-003` records that compatibility
  rests on naming convention with no build-time check, so a publisher and consumer can disagree
  silently, and `GAP-17` records that the same logical message is already three independent classes.
  Rule 4's fields are additive and `N6b` asserts the value end to end, but the class of risk is real
  and is `R-24`; `ADR-030` Rule 8 is what makes the assertion cover payload fields rather than only
  routing keys.
- **This record now depends on a data change it does not own.** Rule 4 clause 2 cannot be implemented
  until `ADR-024` Rule 8 puts `OrderId` and `CustomerId` on `Reservation`. Until then CAP-04 has
  nothing to compare the caller against — the current `Reservation` is a date and a priority ✅ — and
  the guard degrades to trusting the message. That is the `B2` blocker below, and it is a sequencing
  constraint on implementation, not an open question about the design.
- **The standalone release route stays open for longer than it should.** Moving the ownership check
  off `ReleaseResourceReservation` is correct for this flow and means `DO2` no longer incidentally
  fixes that route. `FA4` carries it with its own owner, and anyone reading only this record's first
  revision would have assumed it was already handled.
- **Rule 7 costs diagnosability for callers.** A customer who mistypes an identifier gets the same
  answer as one probing someone else's. The correlation-identifier log entry is the compensation, and
  it is only useful where logs are actually reachable — `ADR-021` records the platform's limits there.
- **The existing unauthorized delivery read stays as it is.** `GET /deliveries/{deliveryId}` is a
  live contract with unknown consumers; Rule 6 governs the customer-facing path. Tightening the
  existing route is `FA3`, with a named decision about who is currently relying on it.

### 6.3 Neutral and follow-on

- Admin access is preserved exactly as the current guard expresses it. This record does not change
  what an admin may do.
- `ASM-18` means the new routes inherit the existing gateway write mode, so no new edge mechanism is
  introduced by this record.

## 7. Compliance Considerations

| Obligation | Source | How this record complies |
|------------|--------|--------------------------|
| Every new customer-facing route authenticates and authorizes server-side | `NFR-2` | Rules 1, 2, 6 |
| A caller acts only on their own order, delivery and reservation | `NFR-22` | Rules 1, 4, 6. The reservation half is Rule 4 clause 2, checked inside CAP-04 against `ADR-024` Rule 8's owner fields |
| An ownership decision is made by the service that holds the data | `NFR-2`, `ADR-029` Option E rejection | Rule 4 clause 2. A caller-side check is not a control; `ADR-027` Rule 6 applies the same principle to the revalidation read |
| Absent data fails closed | `NFR-2` | Rules 2 and 4 clause 3; `N5` and `N6a` |
| A customer sees only their own deliveries | `NFR-1` | Rules 6 and 7 |
| Identity is established from a validated token at the edge and bound into the payload | `ADR-005`, `ADR-006` | Rule 5 asserts the binding instead of assuming it |
| No shared library is introduced | `ADR-002` | Rule 3 is one guard per service, not one across services |
| Message fields follow the platform naming convention | `ADR-003`, `NFR-15`, `ADR-030` | Rule 4 clause 1, against the payloads fixed in `ADR-030` §5.2 |
| No secrets, tokens or personal data appear in logs or error payloads | `ADR-021`, `NFR-11` | Rule 7 logs the distinction, not the credential. `N9` |
| New routes inherit the existing gateway write mode | `ASM-18` | No new edge mechanism |

## 8. Non-Functional Requirements & Testing

| # | Requirement | NFR | How it is verified |
|---|-------------|-----|--------------------|
| `N1` | An unauthenticated caller is refused on every new route | `NFR-2`, Rule 1 | Call each new route with no token. Assert refusal. This is the regression test for the current bypass |
| `N2` | A caller with an empty or absent identity context is refused | Rule 2 | Inject an empty identity directly at the handler. Assert refusal |
| `N3` | A customer cannot reschedule another customer's delivery | `NFR-22` | Authenticate as A, target B's delivery. Assert the not-owned rejection and assert **no** reservation call was made |
| `N4` | A customer's eligible list contains no other customer's delivery | `NFR-1`, Rule 6 | Two customers. Assert disjoint lists |
| `N5` | A delivery with no replicated customer id is returned to no customer caller | Rule 6 | Pre-replica delivery. Assert refusal for every caller |
| `N6` | **CAP-04 refuses a move whose current-day reservation is held by another order** | `NFR-22`, Rule 4 clause 2 | Reserve the day for order A. Send a reschedule naming order B and the same resource and day. Assert an explicit `not eligible` rejection, assert A's reservation is **intact**, and assert nothing was published. This replaces the first revision's release-leg test, which exercised a command the reschedule never issues (`ADR-024` Rule 1) |
| `N6a` | A reservation with no recorded owner is treated as not-mine | Rule 4 clause 3 | Write a reservation with null owner fields. Attempt a move against it. Assert refusal, not a pass |
| `N6b` | `CustomerId` survives every hop of the chain unmodified | Rule 4 clause 1, `ADR-030` §5.2 | Drive one reschedule end to end. Assert the same `CustomerId` on `M1`, `M2`, `M3` and `M4`, and assert the value equals the token's subject, not any client-supplied field |
| `N7` | A client-supplied `customerId` is overwritten by the token's subject at the gateway | Rule 5, `ADR-005` | Send a forged `customerId` through the real gateway configuration. Assert the server saw the token's value. This is the misspelled-bind detector |
| `N8` | Not-owned and not-found are indistinguishable in the customer-facing response | Rule 7 | Compare both responses byte for byte on the customer path |
| `N9` | No token, credential or personal field appears in any log line on a refusal path | `NFR-11`, `ADR-021` | Capture the sink across `N1` to `N8`. Assert absence |
| `N10` | An admin retains the access the current guard grants | §6.3 | Admin caller against another customer's order. Assert unchanged behaviour |

## 9. Relationship to the implementation pattern catalog

No new pattern, and one correction to an existing one. The catalog's description of the ownership
guard reflects the implemented refuse-list form; it should record that the form **admits
unauthenticated callers**, and that new code uses the admit-list form in Rule 1. Leaving the catalog
describing the defective shape as the house style is how the next six copies get written.

Applies `patterns/observability/structured-logging-with-property-redaction.md` for Rule 7's operator
-only distinction.

## 10. Evidence

| # | Claim | Source |
|---|-------|--------|
| E1 | The ownership guard is duplicated in six CAP-07 handlers and admits unauthenticated callers | `orders-service.md` §3.8 |
| E2 | The gateway binds `customerId: @user_id` from the validated token, overwriting the client's value | `ntrada.yml:103`; catalog AuthPolicy *Payload Binding for User ID* |
| E3 | A misspelled bind name silently restores the client-supplied value | Catalog KnownGap *Silent Authorization Bypass* |
| E4 | `ReleaseResourceReservation` carries no customer identity | `availability-service.md` §3.9; the message contract |
| E5 | `ReleaseReservation` returns silently when nothing matches | `Availability/.../Core/Entities/Resource.cs`; `availability-service.md` §3.9 |
| E6 | `GET /deliveries/{deliveryId}` performs no authorization and returns the document to any caller | `deliveries-service.md` §3.21 |
| E7 | CAP-09 holds no customer id today, so it cannot authorize | `Deliveries/.../Core/Entities/Delivery.cs`; `deliveries-service.md` §3.1 |
| E8 | Message compatibility rests on naming convention with no build-time check | `ADR-003`; catalog `GAP-6` |
| E9 | The platform has no shared library and has decided against one | `ADR-002` |
| E10 | `Reservation` carries a date and a priority only, so CAP-04 cannot identify a reservation's owner today | `Availability/.../Core/Entities/Reservation.cs`; `availability-service.md`; `architecture-views.md` §5.1 |
| E11 | The reschedule is one operation inside CAP-04 and issues neither `ReserveResource` nor `ReleaseResourceReservation` | `ADR-024` Rule 1; `ADR-030` §5.2 `M2` |
| E12 | Saga-dispatched commands carry an empty `CorrelationContext.UserContext`, so identity does not survive that path today | `architecture-views.md` §6 `GAP-13` |

### 10.1 Documentation-versus-code conflicts

- **The guard reads as an authorization check and behaves as a partial one.** The implemented
  condition ✅ looks correct at a glance and admits the unauthenticated caller. The catalog records
  the behaviour; the code does not announce it. §9 asks for the catalog entry to say so plainly.
- **Internal to this patch, resolved.** Rule 4 previously placed the reschedule's ownership check on
  `ReleaseResourceReservation`, while `ADR-024` Rule 1 states the reschedule never issues that
  command. Both could not be true, and the consequence of the first revision shipping as written
  would have been a guard that compiles, tests green on the release route, and never runs on the
  path it was written for. Rule 4 is now expressed in terms of the single move operation, and the
  release route is carried separately as `FA4`.

## 11. Follow-Up Actions

**Reading the `By` column.** Each entry carries a calendar date followed by the delivery milestone
that date is derived from. The dates come from the one work-item 14830 wave calendar in
[`../specs/14830/solution-design.md`](../specs/14830/solution-design.md) §5.1, so every record in
`ADR-024`…`ADR-030` resolves the same milestone to the same date. If the wave calendar moves, that
section is the single place to change and these dates move with it; the milestone is what binds.

| # | Action | Owner | By |
|---|--------|-------|-----|
| `FA1` | **[ACTION NOW]** Treat the unauthenticated-caller bypass in the six existing CAP-07 handlers as a live finding and give it an owner and a date, independent of this feature. This record prevents new occurrences; it does not fix the existing ones (`R-18`) | Platform owner with the platform architect | **2027-01-30** — decision before `DO2` ships; fix on its own schedule |
| `FA2` | **[handled later by LLD]** Converge CAP-07's six guard copies onto the single fail-closed guard from Rule 3, once `FA1` has an owner | `DO2` implementer | **2026-12-22** — during `DO2` low-level design |
| `FA3` | **[ACTION NOW]** Decide whether the existing unauthorized `GET /deliveries/{deliveryId}` is tightened, and identify who consumes it today. It is a live contract and this record deliberately leaves it alone | Platform owner with the product owner | **2026-12-05** — before `DO1` ships |
| `FA4` | **[ACTION NOW]** Treat the unguarded standalone `ReleaseResourceReservation` route as a live finding in its own right: it accepts a release today without proving the caller holds the reservation, and `ReleaseReservation` then returns silently when nothing matches ✅. `DO2` does **not** fix it — `ADR-024` Rule 1 removes that command from the reschedule path entirely — so it needs its own owner and date or it will be assumed closed. Decide the guard, and whether the silent return becomes an explicit refusal for existing callers | Platform owner with the platform architect | **2026-12-08** — decision during `DO2` high-level design; fix on its own schedule |
| `FA5` | **[handled later by DevOps]** Add the gateway-binding assertion from `N7` to the pipeline that gates gateway configuration changes, so a typo in `ntrada.yml` fails a build rather than a customer | Platform owner | **2027-01-16** — before `DO2` reaches a shared environment |
| `FA6` | **[handled later by HLS]** Fix the `CustomerId` field name and type on all four reschedule messages and confirm every publisher and consumer agrees, per `ADR-030` §5.2, `ADR-030` Rule 8 and `NFR-15`. `GAP-17` is why this cannot be assumed: the same logical message is already three independent classes on this platform | `DO2` implementer | **2026-12-08** — during `DO2` high-level design |
| `FA7` | **[handled later by HLS]** Decide whether identity must survive a saga-dispatched command before any part of this chain is routed through a saga. `CorrelationContext.UserContext` is empty on that path today (`GAP-13`), which would silently defeat Rule 4 clause 1. `ADR-024` `FA6` raises the same constraint from the other side | `DO2` implementer with the platform architect | **2026-12-08** — during `DO2` high-level design |

## Assumptions, Blockers & Open Questions

### Assumptions

- **A1 ❓** The validated token's subject is a stable customer identifier that matches the customer id
  stored on the order and replicated to the delivery. The gateway binds `@user_id` into `customerId`
  ✅ and CAP-07 compares the identity's id to `order.CustomerId` ✅, which only works if the two are
  the same identifier space. That they are is implied by the working comparison and is not separately
  proven here.
- **A2 — resolved, and it was not an assumption.** The first revision assumed CAP-04 could determine
  a reservation's owner once the identity was carried on the message, because the reservation is
  taken on behalf of a customer in the same operation. That inference does not survive contact with
  the entity: `Reservation` is a date and a priority ✅ and persists nothing about who holds it, so
  CAP-04 comparing a carried identity against its own state had nothing to compare against. The
  owner fields are now an explicit decision — `ADR-024` Rule 8 — not an inference drawn here.
- **A3 ❓** The token's subject is the same identifier CAP-04 will see in `Reservation.CustomerId`
  once backfill or re-reservation has populated it. Where it is null, Rule 4 clause 3 refuses, so a
  wrong answer here fails closed rather than open; `ADR-024` `FA5` decides the backfill.

### Blockers

- **B1 — Rule 4 clause 2 cannot be implemented before `ADR-024` Rule 8 lands.** The check compares
  the caller against owner fields that do not exist on `Reservation` today ✅. Until they do, CAP-04
  can only trust what the message told it, which is the control this record exists to avoid. This is
  a sequencing constraint between two records in the same wave, and `ADR-024` `B1` is its other half.
- `FA1` and `FA3` are decisions about existing behaviour that need named owners; they block the
  release conversation rather than this record. `FA4` is now in the same category and must not be
  read as something `DO2` closes.

### Open Questions

- **Q1** Should the admin path on the new routes be audited separately from the customer path?
  `NFR-22` is written about customers, and an admin acting on a customer's reservation is an action
  no record currently requires to be logged. Platform architect, after the first release.
- **Q2** Does an admin rescheduling on a customer's behalf pass the customer's identity or their own
  on the chain? Rule 4 clause 1 says `CustomerId` is bound from the token; for an admin the token's
  subject is not the owner, so CAP-04's check in clause 2 would refuse. Rule 1 admits admins at the
  edge, which means the two rules disagree on this one case. No `DO1`/`DO2` route is specified as
  admin-facing, so nothing in this increment exercises it — but it should be settled before an
  operator route is added. Platform architect, with `ADR-030` §5.2's payload review (`FA6`).
