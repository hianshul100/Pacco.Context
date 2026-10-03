# ADR-029: The ownership guard fails closed, and edge-bound customer identity is carried into both the reserve and the release leg

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
| **Infrastructure recommendation applied** | `INF-5` — carry edge-bound customer identity into the reserve and release legs with a fail-closed guard, governed by `ADR-006`, delegated to **HLS** |
| **Related** | `ADR-006` (the authorization decision whose recorded fail-open form this departs from), `ADR-005` (the gateway and its payload binding), `ADR-026` (the replica that supplies CAP-09 the customer id), `ADR-024` (the release leg that carries no identity today), `ADR-030` (the not-owned rejection code) |

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

**The release leg carries no identity at all.** `ReleaseResourceReservation` carries the resource and
the date; it does not carry a customer ✅. `ADR-024` makes a reschedule a release followed by a take
in one mutation, so the release leg is now part of a customer-initiated operation — and CAP-04 cannot
tell whose reservation it is releasing. `ReleaseReservation` also returns silently when nothing
matches ✅, so releasing the wrong thing and releasing nothing look identical from the outside.

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
| D5 | The release leg is now part of a customer-initiated operation and carries no identity | `ADR-024`, `availability-service.md` |
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
into every leg of the operation, including the release.

- **For.** It is the only option that satisfies `NFR-2` and `NFR-22` as written. It removes the
  unauthenticated-caller hole on the new paths and makes the release leg checkable at CAP-04.
- **Against.** It departs from the fail-open form `ADR-006` records as current behaviour, so two
  shapes of guard exist in CAP-07 until `FA2` converges them. It adds a field to an existing message
  contract, which `ADR-003`'s naming-convention-only compatibility makes a careful change.

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
customer identity is carried into both the reserve and the release leg of a reschedule.** Seven rules
follow.

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

**Rule 4 — the customer identity is carried into the reserve leg and the release leg.**
`ReleaseResourceReservation` gains the customer identity so CAP-04 can verify that the reservation
being released belongs to the caller. A release that does not match is an explicit refusal, not the
current silent return ✅. The field is additive and follows the platform's naming convention, per
`ADR-003` and `ADR-030`.

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
- **A message contract gains a field.** `ADR-003` records that compatibility rests on naming
  convention with no build-time check, so a publisher and consumer can disagree silently. Rule 4's
  field is additive and `N6` asserts it end to end, but the class of risk is real and is `R-24`.
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
| A caller acts only on their own order, delivery and reservation | `NFR-22` | Rules 1, 4, 6 |
| A customer sees only their own deliveries | `NFR-1` | Rules 6 and 7 |
| Identity is established from a validated token at the edge and bound into the payload | `ADR-005`, `ADR-006` | Rule 5 asserts the binding instead of assuming it |
| No shared library is introduced | `ADR-002` | Rule 3 is one guard per service, not one across services |
| Message fields follow the platform naming convention | `ADR-003`, `NFR-15`, `ADR-030` | Rule 4 |
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
| `N6` | The release leg carries the customer identity, and CAP-04 refuses a release that does not match | Rule 4 | Release with a mismatched customer. Assert an explicit refusal, not a silent return |
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

### 10.1 Documentation-versus-code conflicts

- **The guard reads as an authorization check and behaves as a partial one.** The implemented
  condition ✅ looks correct at a glance and admits the unauthenticated caller. The catalog records
  the behaviour; the code does not announce it. §9 asks for the catalog entry to say so plainly.

## 11. Follow-Up Actions

| # | Action | Owner | By |
|---|--------|-------|-----|
| `FA1` | **[ACTION NOW]** Treat the unauthenticated-caller bypass in the six existing CAP-07 handlers as a live finding and give it an owner and a date, independent of this feature. This record prevents new occurrences; it does not fix the existing ones (`R-18`) | Platform owner with the platform architect | Decision before `DO2` ships; fix on its own schedule |
| `FA2` | **[handled later by LLD]** Converge CAP-07's six guard copies onto the single fail-closed guard from Rule 3, once `FA1` has an owner | `DO2` implementer | During `DO2` low-level design |
| `FA3` | **[ACTION NOW]** Decide whether the existing unauthorized `GET /deliveries/{deliveryId}` is tightened, and identify who consumes it today. It is a live contract and this record deliberately leaves it alone | Platform owner with the product owner | Before `DO1` ships |
| `FA4` | **[handled later by HLS]** Fix the additive customer-identity field name on the release message and confirm publisher and consumer agree, per `ADR-030` and `NFR-15` | `DO2` implementer | During `DO2` high-level design |
| `FA5` | **[handled later by DevOps]** Add the gateway-binding assertion from `N7` to the pipeline that gates gateway configuration changes, so a typo in `ntrada.yml` fails a build rather than a customer | Platform owner | Before `DO2` reaches a shared environment |

## Assumptions, Blockers & Open Questions

### Assumptions

- **A1 ❓** The validated token's subject is a stable customer identifier that matches the customer id
  stored on the order and replicated to the delivery. The gateway binds `@user_id` into `customerId`
  ✅ and CAP-07 compares the identity's id to `order.CustomerId` ✅, which only works if the two are
  the same identifier space. That they are is implied by the working comparison and is not separately
  proven here.
- **A2 `[INFERRED]`** CAP-04 can determine the owner of a reservation once Rule 4's field is carried,
  because the reservation is taken on behalf of a customer in the same operation. The `Reservation`
  struct's current fields are date and priority ✅, so recording the owner may require an additive
  field there — that is `FA4`'s territory and is why Rule 4 names the contract change explicitly.

### Blockers

- None that block this decision. `FA1` and `FA3` are decisions about existing behaviour that need
  named owners, and they block the release conversation rather than this record.

### Open Questions

- **Q1** Should the admin path on the new routes be audited separately from the customer path?
  `NFR-22` is written about customers, and an admin acting on a customer's reservation is an action
  no record currently requires to be logged. Platform architect, after the first release.
