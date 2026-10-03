# ADR-025: The reserved resource is recorded on the Order aggregate, replacing the assumed `resourceId == vehicleId` correspondence

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-10-03 |
| **ADR id** | `ADR-025` |
| **Backlog candidate** | None. `adr-candidates.md` runs to `ADR-CANDIDATE-020` and records no candidate for the vehicle-to-resource correspondence; it exists in the catalog only as an unverified assumption |
| **Category / Impact** | Data & Domain / high |
| **Supersedes / Superseded by** | — (supersedes nothing; no record has ever stated where the reserved resource is held) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (`ADR-021` B1, `risk-constraint-gap-register.md` `G-01`) |
| **Base ref of all cited source** | `feature/14830/aidlc` |
| **Source** | `DO1` — *Eligible Delivery Schedule & Alternative-Day View*, and `DO2` — *Customer Delivery Reschedule Confirmation & Slot Safe-Move*, work item **14830**, `intents/14830.md`: "the bookable set is the delivery's currently assigned resource only". The catalog records `resourceId == vehicleId` only as an Assumption node described as *not explicitly verified or established within the repository* |
| **Non-functional requirements** | `NFR-17` (additive, backward-compatible persistence), `NFR-5` (a failed reschedule leaves the order's delivery date in its pre-request state), `NFR-6` (no silent date reinterpretation) |
| **Resolved decision applied** | `AD-2` option **A**, chosen by human, confidence high. Binding and settled — not re-opened here |
| **Related** | `ADR-024` (the day move whose target resource this field names), `ADR-008` (hand-mapped documents, no migration tooling), `ADR-009` (the replica precedent), `ADR-026` (the CAP-09 replica that consumes this field), `ADR-028` (the version-conditioned write on `Order`) |

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

`ASM-15` fixes the bookable set for a reschedule as **the delivery's currently assigned resource
only**. For that sentence to be executable, something on the platform has to know which CAP-04
resource a given delivery's reservation is held on. Nothing does.

What exists instead is a correspondence by convention. CAP-07 holds `VehicleId` on the `Order`
aggregate ✅ (`Orders/.../Core/Entities/Order.cs:16`), set by `SetVehicle`, which is a bare
assignment with no guard of any kind ✅ (`Order.cs:75-78`). CAP-04 holds a `Resource` whose id is a
plain `Guid` and whose only type discriminator is its tag set ✅
(`Availability/.../Core/Entities/Resource.cs:10-35`). The two ids are treated as the same id by the
reservation flow, and the catalog records that correspondence as an **Assumption** — explicitly
described as not verified or established anywhere in the repository — alongside a Requirement node
recording that the correspondence *is not created within the repository*.

The correlation that does exist is weaker still and runs the other way. When CAP-04 publishes
`resource_reserved`, CAP-07 finds the affected order by querying
`o.VehicleId == vehicleId && o.DeliveryDate == deliveryDate.Date` ✅
(`Mongo/Repositories/OrderMongoRepository.cs:29-30`) — an exact-equality match on a pair of fields,
neither of which is an identity of the reservation. It works only because both sides are truncated to
midnight, and the component model records plainly that if `availability-service` ever emits a
reservation at a non-midnight instant or with a different `DateTimeKind`, every correlation lookup
fails and every affected order silently stops advancing ✅ (`orders-service.md` §3.11).

So the platform's answer to "which resource is this delivery's day held on?" is today a guess
derived from a vehicle id, confirmed by a date match, with no recorded fact behind either.

### 1.1 What this record does *not* cover

It does not change vehicle assignment, which stays exactly as it is — `ASM-15` is explicit that the
assigned vehicle never changes during a reschedule. It does not make CAP-07 the owner of reservation
state; CAP-04 remains that owner. It does not decide the delivery-side copy of the schedule, which is
`ADR-026`.

## 2. Decision Drivers

| # | Driver | Where it comes from |
|---|--------|---------------------|
| D1 | The reschedule must name a resource, and must name the *right* one, with no guessing | `ASM-15`, `DO2` |
| D2 | A fact the platform relies on should be recorded at the moment the platform already has it, rather than reconstructed later | `ADR-009`'s recorded reasoning for replicas |
| D3 | Persistence changes must be additive and backward-compatible, because no migration tooling exists | `ADR-008`, `NFR-17` |
| D4 | CAP-07's order delivery date stays the authoritative schedule | `ASM-10` |
| D5 | The existing `(vehicleId, deliveryDate)` correlation is fragile and must not be made load-bearing for a new customer-facing feature | `orders-service.md` §3.11, Q-6 |

## 3. Architecture Fit Evaluation

The binding fit verdict for this dimension is `can_extend_existing_component: true`,
`evolution_type: service_extension`, owner **CAP-07 Order Lifecycle Management**, confidence high.

| Dimension | Verdict | Consequence for this record |
|-----------|---------|-----------------------------|
| Owner | CAP-07 — orders already observes `resource_reserved` and already holds the delivery date | The field lands where the platform already receives the fact. No new consumer, no new subscription |
| New deployable service | **Not required** | None is proposed |
| New bounded context | **Not required** | None is proposed |
| New runtime component | **Not required** | None is proposed |

The verdict's reasoning is adopted verbatim as the fit argument: the `Order` aggregate already
observes `resource_reserved` and already holds the delivery date, so recording the reserved
`resourceId` converts a fragile cross-catalog naming convention into a recorded fact **at the exact
point the platform already has it**, without permanently coupling the vehicle and resource identity
spaces and without creating a second description of the schedule.

## 4. Options Considered

### Option A — record the reserved resource id on the `Order` aggregate *(chosen; `AD-2` option A, chosen by human)*

`Order` gains one additive nullable field holding the CAP-04 resource id the order's delivery day is
reserved on. It is written by the handler that already consumes `resource_reserved`.

- **For.** One additive field on an aggregate the platform already writes at that moment. No new
  message, no new subscription, no new service-to-service call. The fact becomes recorded rather than
  inferred.
- **Against.** It adds a field to the platform's most-coupled aggregate, and `Order`'s mapper has a
  documented history of silent field loss — `CancellationReason` is on the entity and absent from
  `OrderDocument`, so it never persists ✅ (`orders-service.md` §3.1, §3.15).

### Option B — declare `resourceId == vehicleId` as a platform invariant and enforce it

Make the correspondence a rule: CAP-05 and CAP-04 must mint the same id for the same physical thing.

**Rejected.** It permanently couples two capabilities' identity spaces to make one feature work, and
it is unverifiable today — nobody has established that the correspondence actually holds in a running
environment. Turning an unverified assumption into an enforced invariant is the opposite of recording
a fact.

### Option C — hold the resource id only on the delivery record in CAP-09

**Rejected.** CAP-09 does not receive `resource_reserved` — it subscribes to nothing at all ✅ — so
the fact would have to travel further to reach a place that has less claim to it. It would also make
CAP-09 the first and only holder of a reservation association, which sits uneasily beside `ASM-10`'s
decision that CAP-07 holds the authoritative schedule.

### Option D — query CAP-04 for the resource at reschedule time

**Rejected.** There is no query that answers "which resource holds a reservation for this customer on
this day" — CAP-04's only read is a point read by resource id ✅. Adding one would mean a search
across resources, which is precisely the unbounded collection read `ADR-010` C10 forbids.

## 5. Decision

**The CAP-04 resource against which an order's delivery day is reserved is recorded on the `Order`
aggregate as an additive, nullable field, written at the moment CAP-07 already consumes
`resource_reserved`.** Five rules follow.

**Rule 1 — one additive nullable field.** `Order` gains a nullable resource-id property with a
private setter, written only through a method on the aggregate. Nullable is deliberate: every order
written before this change has no value, and a null must read as "not recorded" rather than as
`Guid.Empty` — which `SetVehicle` demonstrates is an accepted value at the aggregate level today ✅.

**Rule 2 — all four mapping touch-points are changed in the same commit.** The property, the
document property, `AsEntity` and `AsDocument`. Omitting `AsDocument` drops the value on every write
and omitting `AsEntity` drops it on every read, both silently — `CancellationReason` is the living
proof that this failure mode is real on this exact aggregate ✅. Whether it is also exposed on
`OrderDto` is decided by `FA2`, not assumed.

**Rule 3 — it is written where the fact arrives.** The handler that already consumes
`resource_reserved` records the resource id on the order it correlates. No new subscription and no
new message are introduced by this record.

**Rule 4 — the reschedule reads the recorded field, and refuses when it is absent.** A reschedule
for an order with no recorded resource id is **rejected with a distinct reason**, not silently
retargeted at the vehicle id. Falling back to the vehicle id would reintroduce the unverified
assumption this record exists to remove, and would do so on the one path where it is least visible.

**Rule 5 — the vehicle id is untouched.** `VehicleId` keeps its current meaning, its current
population path and its role in the existing `(vehicleId, deliveryDate)` correlation. This record
adds a fact; it removes nothing and renames nothing, which is what `NFR-17` and `ADR-008` require.

## 6. Consequences

### 6.1 Positive

- `ASM-15`'s "the delivery's currently assigned resource" becomes a value the platform can read,
  rather than a correspondence it has to assume.
- The reschedule stops depending on the `(vehicleId, deliveryDate)` equality match, which the
  component model records as fragile in the ordinary case and undiagnosable in the failing case ✅.
- Orders written after this change carry an auditable record of which resource their day was held on,
  which is the same fact `ADR-026`'s delivery-side replica needs.

### 6.2 Negative

- **Every order created before this change has a null resource id, and there is no backfill.**
  `ADR-008` records that no migration tooling exists anywhere in the platform ✅, so the only way to
  populate historical orders is a hand-written script with no home in any repository. Rule 4 turns
  that into a visible rejection rather than a wrong answer, but it does mean **pre-existing deliveries
  cannot be rescheduled until their order is reserved again**. This is a real functional limitation of
  the first release and is recorded as `R-29` rather than hidden.
- **It adds a field to the platform's most-coupled aggregate.** CAP-07 is the heaviest caller and the
  busiest consumer on the platform; every change there has the widest blast radius.
- **The silent-mapping failure mode is real here.** Rule 2 exists because this aggregate has already
  lost a field exactly this way, and nothing in the build detects it.

### 6.3 Neutral and follow-on

- Whether `resourceId` and `vehicleId` actually hold the same value in a running environment stays
  unknown. This record removes the platform's *dependence* on the answer without establishing it;
  `G-08` keeps the question open for whoever can observe a running environment.
- `Order` has no optimistic concurrency today — `UpdateAsync` is a whole-document replace keyed on id
  alone ✅ (`orders-service.md` §3.17). Adding a field widens the data a lost update can erase.
  `ADR-028` closes that for `Order`.

## 7. Compliance Considerations

| Obligation | Source | How this record complies |
|------------|--------|--------------------------|
| Fields are added, never renamed or removed; enum ordinals append-only | `ADR-008`, `NFR-17` | Rule 1 adds one nullable field. Rule 5 leaves `VehicleId` untouched |
| One logical database per service; no cross-service database access | `ADR-008` C5 | The field lives in `orders-service`'s own database and is read by CAP-07 alone |
| The order's delivery date remains the authoritative schedule | `ASM-10` | Unchanged. This record adds a resource reference, not a schedule |
| A service publishes only to its own exchange | `ADR-001` C1 | This record introduces no message at all |
| No vehicle or driver reassignment behaviour is introduced | `DO2` scope, `ASM-15` | Rule 5 |
| Message compatibility rests on naming convention, with no build-time signal | `ADR-003`, C2 | No contract changes, so no compatibility surface is touched |

## 8. Non-Functional Requirements & Testing

| # | Requirement | NFR | How it is verified |
|---|-------------|-----|--------------------|
| `N1` | A resource id written to an order survives a save-and-reload round trip | `NFR-17` | Write, reload from Mongo, assert the value. This is the `CancellationReason` regression test, written for the new field |
| `N2` | An order document written before this change still loads, with a null resource id and no error | `NFR-17` | Load a fixture document lacking the element. Assert null and no exception |
| `N3` | A reschedule against an order with no recorded resource id is rejected with its own reason | `NFR-14`, Rule 4 | Attempt it. Assert the distinct code, and assert that **no** reservation call was made |
| `N4` | The recorded resource id is never silently replaced by the vehicle id | Rule 4 | Set a resource id different from the vehicle id. Assert the reschedule targets the recorded resource |
| `N5` | Recording the resource id does not change the existing `resource_reserved` correlation behaviour | `NFR-5` | Replay the existing reservation-approves-order flow. Assert the order still advances exactly as before |
| `N6` | A concurrent parcel addition is not erased by the write that records the resource id | `NFR-4` | Covered by `ADR-028` `N2`. Listed here because this record widens the exposure |

## 9. Relationship to the implementation pattern catalog

No new pattern. This is an application of
`patterns/data/database-per-service-with-document-mapping.md` — specifically its recorded obligation
that a new field is four coordinated edits, not one. The catalog entry should gain a pointer to
`CancellationReason` as the platform's worked example of what omitting one of the four costs, so the
next person adding a field to `Order` meets the evidence before the defect.

## 10. Evidence

| # | Claim | Source |
|---|-------|--------|
| E1 | The `resourceId == vehicleId` correspondence is recorded as an unverified assumption, not a fact | Capability catalog Assumption node: "not explicitly verified or established within the repository" |
| E2 | `SetVehicle` is an unguarded assignment that accepts `Guid.Empty` | `Orders/.../Core/Entities/Order.cs:75-78`; `orders-service.md` §3.10 |
| E3 | CAP-07 correlates reservations by exact `(VehicleId, DeliveryDate)` equality | `Mongo/Repositories/OrderMongoRepository.cs:29-30`; `orders-service.md` §3.11 |
| E4 | A non-midnight or differently-kinded reservation date breaks every correlation silently | `orders-service.md` §3.11, *Why this matters more than it looks* |
| E5 | `CancellationReason` exists on the entity, is absent from `OrderDocument`, and is therefore never persisted | `orders-service.md` §3.1, §3.15 |
| E6 | `Order` has no optimistic concurrency; updates are whole-document replaces keyed on id | `Mongo/Repositories/OrderMongoRepository.cs:45`; `orders-service.md` §3.17 |
| E7 | CAP-09 subscribes to no external event, so it cannot be the first receiver of this fact | Catalog KnownGap *deliveries-service subscribes to nothing and no service publishes a deliveries*; `deliveries-service.md` §3.34 |
| E8 | No migration framework exists in any repository | `ADR-008`; `orders-service.md` §5; `architecture-baseline.md` C8 |

### 10.1 Documentation-versus-code conflicts

None found for this record.

## 11. Follow-Up Actions

**Reading the `By` column.** Each entry carries a calendar date followed by the delivery milestone
that date is derived from. The dates come from the one work-item 14830 wave calendar in
[`../specs/14830/solution-design.md`](../specs/14830/solution-design.md) §5.1, so every record in
`ADR-024`…`ADR-030` resolves the same milestone to the same date. If the wave calendar moves, that
section is the single place to change and these dates move with it; the milestone is what binds.

| # | Action | Owner | By |
|---|--------|-------|-----|
| `FA1` | **[ACTION NOW]** Decide what happens to orders already in flight when this ships — accept that they cannot be rescheduled until reserved again, or commission a hand-written backfill. There is no migration tooling, so the second option is a script somebody has to own (`R-29`) | Product owner with the platform architect | **2027-01-30** — before `DO2` ships |
| `FA2` | **[handled later by HLS]** Decide whether the recorded resource id is exposed on `OrderDto`. It is an internal correlation today; exposing it changes a public read contract | `DO2` implementer with the platform architect | **2026-12-08** — during `DO2` high-level design |
| `FA3` | **[ACTION NOW]** Record, from a running environment, whether `resourceId` and `vehicleId` actually hold the same value (`G-08`). This record no longer depends on the answer, but `ADR-026`'s replica and the existing correlation both still do | Platform owner — the only role that can observe a running environment | **2027-01-30** — before `DO2` ships |

## Assumptions, Blockers & Open Questions

### Assumptions

- **A1 ❓** The handler that consumes `resource_reserved` receives the reserved resource's id in the
  event payload. The integration event is published by CAP-04 and carries the resource it concerns;
  that it is the *resource* id rather than a derived vehicle id is read from the publisher's contract
  and is not separately proven here.
- **A2 `[INFERRED]`** Because the field is nullable and read only by new code paths, no existing
  consumer of `OrderDocument` changes behaviour. Nothing in the workspace reads that document outside
  `orders-service` — `ADR-008` C5 forbids it — so this follows from the database-per-service rule
  rather than from an exhaustive search.

### Blockers

- None that block this decision. `FA1` blocks the release, not the record.

### Open Questions

- **Q1** Should the existing `(vehicleId, deliveryDate)` correlation be migrated onto the recorded
  resource id once enough orders carry it? That would close `orders-service.md` Q-6, and it is a
  change to an existing production flow that this feature does not need. Platform architect, after
  the first release.
