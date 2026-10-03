# Solution Design Report — Work item 14830

| Field | Value |
|-------|-------|
| Work item | **14830** — Pacco Delivery Rescheduling and Delivery Preferences |
| Platform | Pacco |
| Stage | `architecture_evolution_generation` |
| Date | 2026-10-03 |
| Branch | `feature/14830/architecture-evolution-generation` |
| Base ref of all cited source | `feature/14830/aidlc` |
| Delivery outcomes | `DO1` Eligible Delivery Schedule & Alternative-Day View (wave 1) · `DO3` Delivery Instructions & Rescheduling History (wave 1) · `DO2` Customer Delivery Reschedule Confirmation & Slot Safe-Move (wave 2) |
| Evolution pattern | `enhancement` — feature work on the platform as it stands. No new target stack, no coexistence layer, no migration |
| Architecture impact | `significant` — seven new decision records, three capabilities materially extended |
| Writable repository | `Pacco.Context` (this repository). The other thirteen clones are read-only context and none was modified |

## Contents

1. [Scope and summary](#1-scope-and-summary)
2. [Applicable ADRs](#2-applicable-adrs)
3. [Finalized design decisions](#3-finalized-design-decisions)
4. [Target design and solution shape](#4-target-design-and-solution-shape)
5. [Downstream design obligations](#5-downstream-design-obligations)
6. [Open items and escalations](#6-open-items-and-escalations)

---

## 1. Scope and summary

Work item 14830 lets a customer move their own delivery to a different day, see the days they could
move it to, leave delivery instructions, and look back at what was rescheduled and when.

**What makes this an architecture task rather than three screens and a handler.** The four
facts this feature needs do not exist on the platform.

- **The delivery record has no customer and no date.** `Delivery` is `{Id, OrderId, Status, Notes,
  registrations}`. "Show me my deliveries" has no field to filter on and "is this in the future" has
  no field to compare. `GET /deliveries/{deliveryId}` performs no authorization at all and returns
  whatever document it finds to whoever asks.
- **Nothing records which availability resource a delivery's day is held on.** The platform relies on
  a convention that a vehicle id and a resource id are the same value — recorded in the capability
  catalog as an assumption explicitly *not* established anywhere in the repository — confirmed by an
  exact-equality match on `(VehicleId, DeliveryDate)` that works only because both sides truncate to
  midnight.
- **A concurrent write is silently lost, and in the worst case still publishes its events.**
  `availability-service` writes with a correct version predicate and discards the result: a
  non-matching write returns normally and the handler publishes its reservation events anyway.
  `orders-service` has no version mechanism at all. `Delivery` declares `Version` and
  `IncrementVersion`, never increments it, and never persists it — the aggregate reads as
  concurrency-safe in source and is last-writer-wins in production.
- **The ownership guard admits unauthenticated callers.** The check duplicated across six
  `orders-service` handlers refuses a caller who *is* authenticated and is not the owner. A caller
  with no identity passes it.

A customer-facing reschedule sits on top of all four. That is why this is an architecture stage.

**What this stage produced.** Seven new decision records, `ADR-024` through `ADR-030`, and in-place
edits to the four canonical inventory documents. No new deployable service, no new bounded context
and no new runtime component — all ten architecture fit verdicts returned
`can_extend_existing_component: true` with every `new_*_required` flag false. Three capabilities are
materially extended: CAP-04 (the atomic day move and the inspected version check), CAP-07 (the
recorded resource id and its first optimistic concurrency), and CAP-09, which changes most — its
first inbound subscription, its first outbound synchronous call, and its first authorization.

**What this stage did not decide.** The four decisions marked `AD-1` to `AD-4` arrived already
chosen by a human and are applied as given, not re-opened. All five open questions arrived resolved.
Where this report records something as still open, it is a consequence discovered while designing
against those answers — §6 separates the ones a reviewer must act on now from the ones a named later
stage owns.

## 2. Applicable ADRs

| ADR id | Governing or new this run | Why it applies |
|--------|---------------------------|----------------|
| `ADR-024` | **New this run** | Makes a reschedule one atomic same-resource day move and bounds the alternative-day read to 14 ascending calendar days |
| `ADR-025` | **New this run** | Records the reserved availability resource on the order, replacing an unverified vehicle-to-resource naming correspondence |
| `ADR-026` | **New this run** | Gives `deliveries-service` an event-carried replica of the customer, delivery date and resource id, and its first inbound subscription |
| `ADR-027` | **New this run** | Revalidates a delivery's reservation at read time and reports "schedule lost — needs rescheduling" |
| `ADR-028` | **New this run** | Version-conditioned writes whose result is inspected, atomic with the outbox, scoped to `Resource`, `Order` and `Delivery` |
| `ADR-029` | **New this run** | Fail-closed ownership guard on new routes, and edge-bound customer identity carried into the reserve and release legs |
| `ADR-030` | **New this run** | Fixes the reschedule message names and the five rejection classes as contracts, asserted across the boundary |
| `ADR-001` | Governing | A service publishes only to its own exchange — constrains where every new message may go |
| `ADR-002` | Governing | No shared library, so `ADR-028`'s pattern is replicated per repository by decision rather than by oversight |
| `ADR-003` | Governing | Message compatibility rests on naming convention with no build-time signal — the reason `ADR-030` Rule 5 exists |
| `ADR-005` | Governing | The declarative edge and its payload binding, which is where `customerId` is bound from the token |
| `ADR-006` | Governing | Authentication at the edge with in-service ownership re-checks; `ADR-029` departs from its recorded fail-open form on new paths only |
| `ADR-008` | Governing | Hand-mapped documents and no migration tooling — every persistence change is additive and nullable, and nothing backfills |
| `ADR-009` | Governing | The event-carried replica precedent `ADR-026` follows, including its never-reconciled gap |
| `ADR-010` | Governing | Cross-service reads are narrow point reads carrying service identity — `ADR-027` is its first read-path instantiation |
| `ADR-011` | Governing | Aggregate-buffered domain events, which is what lets `ADR-024`'s take-and-release be one mutation |
| `ADR-012` | Governing | The transactional outbox decision `ADR-028` applies; chosen over `ADR-008` as the governing record for write integrity |
| `ADR-013` | Governing | Subscription and handler wiring — eight registration points, one automatic, for CAP-09's first subscription |
| `ADR-014` | Governing | Operation-status projection and its 300-second sliding expiry, which bounds the asynchronous edge outcome |
| `ADR-018` | Governing | Per-repository pipelines — and the recorded fact that CAP-04's does not run its tests |
| `ADR-020` | Governing | The pinned runtime baseline; nothing in this work requires a runtime change |
| `ADR-021` | Governing | `Pacco.Web` as the client boundary, which is where the reschedule screens land and where no frontend standard exists |
| `ADR-023` | Governing | The error and response shape `ADR-030`'s rejection codes are expressed within |

## 3. Finalized design decisions

| # | Decision | Chosen option | Rationale | Source | Rejected alternatives, and why |
|---|----------|---------------|-----------|--------|-------------------------------|
| `D1` | How a reschedule moves a day | **`AD-1` option A** — one atomic same-resource move inside the `Resource` aggregate: take the new day, then release the old one, in one command and one write | `ASM-15` fixes the bookable set to the delivery's currently assigned resource, so the move never crosses an aggregate boundary. Taking before releasing means a failure leaves the customer with the day they started with | **human-gate** — chosen by human, confidence high | Release-then-reserve across two calls: a failure between them leaves the customer with no day at all. A saga: `ASM-15` keeps everything inside one aggregate, so there is nothing to coordinate. A new scheduling component: the fit verdict excludes a new service |
| `D2` | Where the reserved resource is recorded | **`AD-2` option A** — one additive nullable field on the `Order` aggregate, written where the platform already consumes `resource_reserved` | It converts an unverified naming correspondence into a recorded fact at the exact point the platform already has it, without coupling two identity spaces | **human-gate** — chosen by human, confidence high | Declaring `resourceId == vehicleId` a platform invariant: permanently couples two capabilities' identity spaces to make one feature work, and is unverifiable today. Holding it on the delivery: CAP-09 receives no reservation event. Querying availability at reschedule time: there is no such query, and adding one means a collection scan |
| `D3` | How the delivery side learns the schedule and the owner | **`AD-3` option A** — event-carried replica of the order's customer and delivery date onto CAP-09, following the `ADR-009` precedent, with CAP-07 explicitly remaining the system of record per `ASM-10` | The eligible-delivery list is a list; answering it with cross-service calls means one call per delivery and a customer-facing read that dies when another service does | **human-gate** — chosen by human, confidence high | Synchronous read from CAP-07 per delivery: the coupling `ADR-009` exists to avoid. CAP-07 owning the read: it has no delivery status, which `ASM-11` makes part of eligibility. Client-side composition: puts an authorization decision in a browser. A new read-model service: the fit verdict excludes it |
| `D4` | How far write integrity extends | **`AD-4` option B** — scope optimistic concurrency and domain-write/outbox atomicity to `Resource`, `Order` and `Delivery` for this increment, built as a reusable pattern for later platform-wide adoption | Covers exactly the aggregates this feature writes, is small enough to verify, and leaves a worked pattern | **human-gate** — chosen by human, confidence high | A platform-wide sweep now: eleven services with hand-mapped documents and a pipeline that does not reliably run tests is how a correctness fix becomes an outage. Relying on the inbox decorator: it does not cover the HTTP command path. A Redis lock: a lock without fencing is a correctness illusion |
| `D5` | Who owns the atomic move and the bounded read | CAP-04 — `service_extension` and `component_extension` | The reservation set and the priority rule already live in the `Resource` aggregate; handlers only orchestrate | **fit-verdict** — `can_extend_existing_component: true`, confidence high | A new scheduling service, a new bounded context, a new runtime component — all three flags returned false |
| `D6` | Where the result of the version-conditioned write is decided | `ADR-012`, not `ADR-008` | `ADR-008` governs how a document is mapped. The question here is *when a write may succeed* and *what is published when it does* | **fit-verdict** — `component_extension`, confidence high | Treating it as a mapping concern would have left the discarded-result defect in place, because the mapping is already correct |
| `D7` | Where the reservation truth is re-established | At delivery **read** time, against CAP-04, with the delivery contract unchanged | `ASM-20`, which resolves `OQ-5`. The answer is computed from current state rather than from a replica that may never have been updated — and the release path returns silently on a non-match, so the event that would update a replica is exactly the one that may not fire | **governance** — `ASM-20`, binding | A reservation-released event updating the replica: would keep the read dependency-free, and is rejected because the release is silent and the replica is never reconciled. A background sweep: no scheduler or job host exists anywhere. A contract change: `ASM-20` forbids it |
| `D8` | What a customer-facing read does when revalidation fails | Degrade to an explicitly labelled **revalidation unavailable**, never fail and never present an unverified answer as a confirmed one | `NFR-20` is recorded at risk precisely because the obvious implementation makes a customer read die when a dependency is slow. Three outcomes — confirmed, lost, unverified — must be distinguishable | **fit-verdict** — verdict 6, confidence **medium**, because CAP-09 has no outbound client, no timeout policy and no service identity today | Failing the read: turns a dependency blip into an outage on a customer surface. Returning the recorded values unlabelled: tells the customer their day is safe when nobody checked |
| `D9` | How the ownership guard is written on new routes | Admit-list, not refuse-list: deny unless the caller is authenticated **and** is the owner or an admin; an empty caller context is the most restrictive input | The existing refuse-list form is exactly how the unauthenticated-caller hole was created, and it is duplicated six times so no single fix exists | **governance** — `NFR-2`, `NFR-22`, and `INF-5` | Relying on the gateway binding alone: its entire strength is a correctly spelled YAML key. Reusing the existing guard: propagates a known bypass onto the most sensitive new routes. A shared auth library: `ADR-002`. Centralizing in the gateway: the edge has no access to ownership data |
| `D10` | What a rejected reschedule returns | Exactly five classes — day no longer available, held at equal or higher priority, not eligible, not owned, system error — each with a stable code and a customer-safe message | `NFR-14` asks for an actionable reason. Every CAP-09 error is currently HTTP 400 carrying the exception message, which is both unactionable and an internal leak | **governance** — `NFR-14`, `ADR-023` | Improving the exception text: makes clients string-match on prose. Status codes alone: cannot separate capacity exhaustion from priority displacement. A shared contracts package: `ADR-002` |
| `D11` | What is *not* changed | The existing `GET /deliveries/{deliveryId}` authorization, the six existing fail-open guards, the two existing date truncations, the unvalidated `OrderId`, and the eight aggregates outside `AD-4`'s scope | Each is a live behaviour with unknown consumers. Changing them is a blast radius this feature should not absorb unannounced — but leaving them silent would be worse, so each carries a register entry and a named decision owner | **governance** — scope discipline, `AD-4` | Fixing them inside this feature: couples a customer feature to five independent corrections, none of which it needs |

## 4. Target design and solution shape

### 4.1 The shape in one paragraph

A customer reads their eligible deliveries from CAP-09's own store, scoped to their identity by a
replicated customer id, with each delivery's reservation revalidated against CAP-04 by a point read
keyed on a recorded resource id. They pick from at most fourteen ascending calendar days. The
confirmation enters through the edge with the customer id bound from the validated token, is checked
fail-closed against the delivery's replicated owner, and becomes one atomic take-then-release inside
the `Resource` aggregate, written under a version predicate whose result is inspected, in one
transaction with the outbox. CAP-07 updates the authoritative delivery date from the resulting event.
CAP-09 appends an immutable history entry. Anything that fails returns one of five named classes.

### 4.2 Cross-outcome wiring — every contract edge resolved

Each edge below names its producer, its consumer, the mechanism, who owns that mechanism, and the
proof that the consumer is actually invoked at runtime. "Accepted" is not "retrievable": the
round-trip column states what must be readable back after the write, because several of these edges
are new paths on services that have never had one.

| Edge | Producer | Consumer | Mechanism | Mechanism owner | Consumer actually invoked at runtime | Write-to-read-back round trip |
|------|----------|----------|-----------|-----------------|--------------------------------------|-------------------------------|
| `E1` | CAP-02 edge | CAP-09 reschedule handler | HTTP route, or a command on the `deliveries` exchange where the environment's gateway is asynchronous — `ASM-18` inherits the mode | CAP-02 configuration | **Yes, with a precondition.** The route must exist in the `ntrada*.yml` file the running environment loads, and `G-02` records that the environment-to-file mapping is itself unrecorded. The gateway already carries `deliveries` routes, so the mechanism is proven; the specific route is not | Confirmation accepted → the delivery read must return the new date. `ADR-030` `N1` and `ADR-026` `N10` |
| `E2` | CAP-09 reschedule handler | CAP-04 `Resource` aggregate | Reserve and release commands carrying the customer identity | CAP-04 contract, extended additively by `ADR-029` Rule 4 | **Yes.** CAP-04 already consumes `reserve_resource` and `release_resource_reservation` from the gateway and from CAP-10. The additive customer field is the only new element, and `ADR-030` Rule 5 asserts the name across the boundary | Move accepted → CAP-04's point read must show the new day held and the old one released. `ADR-024`'s evidence gates and `ADR-027` `N1` |
| `E3` | CAP-04 outbox | CAP-07 order | `resource_reserved` and `reservation_canceled` on the `availability` exchange | CAP-04 exchange, `ADR-001` | **Yes — observed.** CAP-07 subscribes to both today and advances the order on them. This edge is not new; what is new is `ADR-028` Rule 2 making it fire **only** when the write actually committed | Reservation moved → the order's authoritative delivery date must change. `ADR-028` `N4` proves the negative case: a conflicted write publishes nothing |
| `E4` | CAP-07 outbox | CAP-09 replica handler | Order events carrying the customer, the delivery date and the reserved resource id, on the `orders` exchange | CAP-07 exchange, `ADR-001`; consumer wiring per `ADR-013` | **This is the edge that does not exist.** CAP-09 subscribes to nothing today. It is wired at eight registration points of which one is automatic, and a partially wired subscription compiles, deploys and silently receives nothing. `ADR-026` `N7` asserts the publisher's routing key against the consumer's binding in **one** test, because each-side-against-its-own-constant proves nothing | Order date changed → the customer's eligible list must show it. `ADR-026` `N1` and `N5`. **If this edge is misbound the symptom is an empty list, not an error** — `ADR-026` Rule 4 excludes a delivery with no replica, so a broken edge looks like a correct answer. That is why `R-24` scores 7 on detection |
| `E5` | CAP-09 read handler | CAP-04 resource point read | Synchronous HTTP point read by resource id, Fabio-mediated, carrying service identity | `ADR-010`; CAP-09 has no `httpClient.services` entry and no service-identity certificate today | **Not yet.** This is CAP-09's first outbound synchronous call. `ADR-027` Rule 6 fails closed without the certificate, and `ADR-027` `B1` / `G-09` record that as the precondition it is | Reservation displaced → the next delivery read must say so. `ADR-027` `N2`; `N4` proves the degraded branch |
| `E6` | CAP-09 reschedule handler | CAP-09 delivery document | Append-only history entry in the same aggregate write | CAP-09's own store, `ADR-028` Rule 5 | **Yes** — same service, same transaction as the delivery write | Reschedule confirmed → the history read must show one new immutable entry, and three reschedules must show three. `ADR-026` `N10` |
| `E7` | CAP-04 and CAP-09 outboxes | CAP-11 operation status | Existing universal subscription across all eight exchanges | CAP-11, pre-existing | **Yes — observed**, and with one caveat worth carrying: CAP-11 binds the names listed in its message manifest. A new message that is not in that manifest is published but **not observed**, which the platform already has a worked example of | Asynchronous confirmation → the outcome must be observable before the operation-status record expires. `NFR-9`, `ADR-014` |

**`DO1` and `DO3` are wave 1; `DO2` is wave 2.** The wiring respects that order: `E4` and `E5` are
`DO1`'s edges and must work before `DO2`'s `E1`/`E2` have anything meaningful to confirm against. A
reschedule confirmation whose replica never arrived would be rejected by `ADR-025` Rule 4 and
`ADR-026` Rule 4 rather than guessing, which is correct but useless — so wave 1's edges are load
bearing for wave 2's feature, not merely adjacent to it.

### 4.3 Load-bearing assumptions ledger

Every assumption this design rests on, proven from source before being depended on. "Loud" means a
violation produces an error somebody sees; "silent" means it does not.

| # | Assumption | Proven from | Loud or silent if wrong | Verdict |
|---|------------|-------------|-------------------------|---------|
| `L1` | `Delivery` holds no customer and no delivery date, so a customer-scoped delivery read cannot be written against the current model | `Deliveries/.../Core/Entities/Delivery.cs`; `component-internals/deliveries-service.md` §3.1 | Loud — there is no field to compile against | **proven** |
| `L2` | `GET /deliveries/{deliveryId}` performs no authorization and returns the document to any caller; a null returns 200 with an empty body | `component-internals/deliveries-service.md` §3.21 | Silent — it returns data to the wrong caller with no error | **proven** |
| `L3` | CAP-09 subscribes to no external event and has no HTTP client, so `ADR-026`'s and `ADR-027`'s edges are both firsts | `component-internals/deliveries-service.md` §3.34; the capability catalog's known gap; `architecture-views.md` §2.1, where `deliveries` has no outbound synchronous edge | Silent — a missing subscription receives nothing and logs nothing | **proven** |
| `L4` | `Delivery.Version` and `IncrementVersion` exist, `IncrementVersion` is never called, `Version` is absent from `DeliveryDocument`, and `UpdateAsync` has no version predicate | `component-internals/deliveries-service.md` §3.2, §3.18 | Silent — the source reads as concurrency-safe and the runtime is last-writer-wins | **proven** |
| `L5` | `OrderId` on the delivery document is unindexed and not unique; a restarted delivery inserts a second document and the lookup by order is non-deterministic | `component-internals/deliveries-service.md` §3.6, §3.19 | Silent — a query returns one of two valid-looking documents | **proven** |
| `L6` | Every CAP-09 error is HTTP 400 and the reason field carries the exception message | `component-internals/deliveries-service.md` §3.23 | Silent as a contract defect, loud only to a customer reading an exception | **proven** |
| `L7` | CAP-04 writes with `ReplaceOneAsync(r => r.Id == resource.Id && r.Version < resource.Version, ...)` and **discards the result**, so a lost update publishes its events anyway | `component-internals/availability-service.md` §3.13 | Silent, and worse than silent — the failure is announced as a success to every downstream consumer | **proven** |
| `L8` | `ReleaseReservation` returns silently when nothing matches, so releasing the wrong thing and releasing nothing are indistinguishable | `component-internals/availability-service.md` §3.9 | Silent | **proven** |
| `L9` | `AsDaysSinceEpoch` yields whole days since `0001-01-01` and discards the time of day, so the value an event carries is not the value the store holds | `component-internals/availability-service.md` §3.12 | Silent | **proven** |
| `L10` | At least one service runs its outbox with `disableTransactions: true`, so the domain write and the outbox record are not one atomic unit | `Availability/.../Api/appsettings.json:129`; `component-internals/availability-service.md` §3.14 | Silent — a crash between the two loses one half | **proven** |
| `L11` | `Order` has no optimistic concurrency: updates are whole-document replaces keyed on id alone | `Orders/.../Mongo/Repositories/OrderMongoRepository.cs:45`; `component-internals/orders-service.md` §3.17 | Silent | **proven** |
| `L12` | `Order.SetDeliveryDate` assigns `d.Date`, truncating silently with no `DateTimeKind` normalisation, and the `(VehicleId, DeliveryDate)` correlation works only because both sides truncate | `component-internals/orders-service.md` §3.11, §3.16 | Silent in the ordinary case; silent **and undiagnosable** if either side ever emits a non-midnight value | **proven** |
| `L13` | The ownership guard is duplicated in six CAP-07 handlers and admits an unauthenticated caller | `component-internals/orders-service.md` §3.8 | Silent — the unauthorized request succeeds | **proven** |
| `L14` | `CancellationReason` exists on the `Order` entity, is absent from `OrderDocument`, and therefore never persists — the platform's worked example of a field lost to an incomplete mapper | `component-internals/orders-service.md` §3.1, §3.15 | Silent | **proven** |
| `L15` | The gateway binds `customerId: @user_id` from the validated token, overwriting the client's value, and a misspelled bind name silently restores it | `ntrada.yml:103`; the catalog's auth-policy and known-gap records | Silent — the control disappears with no error | **proven** |
| `L16` | `ReleaseResourceReservation` carries no customer identity, so CAP-04 cannot tell whose reservation it is releasing | `component-internals/availability-service.md` §3.9; the message contract | Silent, compounding `L8` | **proven** |
| `L17` | Adding a handler or subscription requires eight registration points, only one of which is automatic | `ADR-013`; `component-internals/deliveries-service.md` §3.30 | Silent — a partially wired subscription builds and deploys | **proven** |
| `L18` | No migration framework exists in any repository, so nothing can backfill the new fields | `ADR-008`; `architecture-baseline.md` C8 | Loud at the point someone looks for the tool; silent in its consequence, which is records that never get the field | **proven** |
| `L19` | CAP-04's pipeline does not execute its tests, so verification commissioned there does not run | `ADR-018`; `component-internals/availability-service.md` §5 | Silent — the build is green | **proven** |
| `L20` | CAP-04's point read returns the resource's reservation set in a form that answers "is this day held, and at what priority" | **Not proven.** The `Resource` entity holds `HashSet<Reservation>` and the read is a point read by id, but the API projection's shape is read from the endpoint contract, not observed | Loud — the revalidation would not compile or would return a shape it cannot interpret | **Needs validation at HLS.** `ADR-027` `A1`. This is why fit verdict 6 came back at medium confidence; if the projection omits reservations, `ADR-027` `FA1` grows an additive CAP-04 projection |

`L20` is the one load-bearing item not proven from source. It is listed as needing validation rather
than asserted, and it fails loudly, which is why it is not a blocker: the `DO1` implementer meets it
at the first compile rather than a customer meeting it in production.

## 5. Downstream design obligations

| # | Obligation | Owner stage | What must be verified |
|---|-----------|-------------|-----------------------|
| `X1` | Wire CAP-09's first inbound subscription at every registration point, and confirm whether the inbox decorator's de-duplication is actually active for its configuration | `hls` — `INF-1`, `ADR-026` `FA3` | `ADR-026` `N5`, `N6`, `N7`: idempotence, staleness rejection, and the publisher key asserted against the consumer binding in one test |
| `X2` | Implement the version-conditioned write **with the result inspected** in all three repositories, with one agreed predicate form, and turn the outbox transactional | `hls` — `INF-2`, `ADR-028` `FA1` | `ADR-028` `N1`-`N4`, `N8`, `N9`. `N4` is the one that matters: a conflicted write publishes nothing |
| `X3` | Build CAP-09's outbound client with an explicit timeout, the degraded branch, and the service-identity certificate | `hls` — `INF-3`, `ADR-027` `FA1`-`FA3` | `ADR-027` `N3`, `N4`, `N5`, `N7`, `N9` |
| `X4` | Carry the edge-bound customer identity into the reserve **and** release legs, and name the additive field so publisher and consumer agree | `hls` — `INF-5`, `ADR-029` `FA4` | `ADR-029` `N6`: a release with a mismatched customer is explicitly refused, not silently returned |
| `X5` | Fix the five rejection code literals, the customer-facing message per class, and the new message names on each exchange | `hls` — `ADR-030` `FA1`, `FA2` | `ADR-030` `N1`, `N2`, `N6`, `N7` |
| `X6` | Fix the instruction length bound as a number and add the instruction field to the redaction set | `hls` — `ADR-026` `FA4`, `G-11` | `ADR-026` `N8`, `N9`: over-length text rejected and nothing persisted; the text never appears in a log line |
| `X7` | Decide whether the recorded resource id is exposed on `OrderDto`, and whether the schedule-lost observation is persisted on the delivery record | `hls` — `ADR-025` `FA2`, `ADR-027` `FA3` | If persisted, a fresh read always wins over the cached observation |
| `X8` | Converge CAP-07's six ownership-guard copies onto the single fail-closed guard, once `E1` below has an owner | `lld` — `ADR-029` `FA2` | `ADR-029` `N1`, `N2`, `N10`: unauthenticated refused, empty context refused, admin access unchanged |
| `X9` | Write the canonical pattern entry for scoped write integrity, naming which aggregates are covered and which are not | `lld` — `ADR-028` `FA6` | That the boundary in `R-30` is discoverable from the catalog rather than from reading three repositories |
| `X10` | Update the pattern catalog entries this work corrects: the point-read entry gains a read-path failure policy, the replica entry gains the staleness rule, the exchange-naming entry gains the single-literal rule, and the ownership-guard entry records that the implemented form admits unauthenticated callers | `lld` — `ADR-026` §9, `ADR-027` §9, `ADR-029` §9, `ADR-030` §9 | That a developer reading the catalog meets the evidence before the defect |
| `X11` | Implement the mapping changes for every new field at all four touch-points — property, document property, `AsEntity`, `AsDocument` | `codegen` — `ADR-025` Rule 2, `ADR-026` Rule 1 | `ADR-025` `N1`, `N2`: a save-and-reload round trip, and a pre-change document still loading with nulls. This is the `CancellationReason` regression test (`L14`) |
| `X12` | Implement the admit-list guard, the five rejection codes and the bounded ascending day read exactly as the records state | `codegen` — `ADR-024`, `ADR-029`, `ADR-030` | `ADR-024` `N1`-`N9`, `ADR-029` `N1`-`N10`, `ADR-030` `N1`-`N9` |
| `X13` | Make CAP-04's pipeline execute its tests and fail the build on failure | `devops` — `INF-6`, `R-25` | Push a deliberately failing test and confirm the build fails and no image is published. **Every CAP-04 verification above is contingent on this** |
| `X14` | Emit outbox depth and oldest-unpublished-age for the three services in scope, and alert ahead of the expiry window | `devops` — `INF-4`, `R-23` | Stop the dispatcher, write a reschedule, confirm both signals move and an alert fires before the operation-status record expires |
| `X15` | Issue CAP-09's service-identity client certificate and record where service certificates are stored | `devops` — `INF-3`, `G-09` | `ADR-027` `N7`: CAP-04 refuses the revalidation call when it carries no service identity |
| `X16` | Add consumer-lag and queue-depth signal for CAP-09's new queue, and success-rate and latency signal for the new CAP-09-to-CAP-04 edge | `devops` — `ADR-026` `FA5`, `ADR-027` `FA5` | That a persistently failing revalidation is distinguishable from nobody rescheduling |
| `X17` | Add the gateway-binding assertion and the cross-boundary message-name assertion to the pipelines that gate gateway configuration and CAP-04/CAP-09 builds | `devops` — `ADR-029` `FA5`, `ADR-030` `FA4` | `ADR-029` `N7` and `ADR-030` `N6`: a typo in `ntrada.yml` fails a build rather than a customer |

## 6. Open items and escalations

### 6.1 Reviewer action required now

| # | What it is | Who must act | What is blocked if it is ignored |
|---|-----------|--------------|----------------------------------|
| `E1` | **The ownership guard admits unauthenticated callers in six live CAP-07 handlers.** This run prevents new occurrences and deliberately does not fix the existing six, because changing six live authorization paths is not something a delivery feature should do unannounced. It needs its own owner and its own date | **Decide:** the platform owner with the platform architect. `ADR-029` `FA1`, `R-18` | Nothing in this feature. A live authorization bypass stays open with nobody accountable for it |
| `E2` | **No revalidation timeout exists, and no availability or latency target exists to justify one.** An unconfigured client default is effectively unbounded, so a slow CAP-04 makes a customer's delivery list hang rather than degrade | **Fix the number:** the platform owner with the `DO1` implementer. `ADR-027` `FA2`, `G-10`, `R-21` | `DO1`'s revalidation path. `NFR-20` cannot be assessed against an unspecified timeout |
| `E3` | **Nothing backfills the orders and deliveries written before this ships.** Pre-existing deliveries are not reschedulable and are excluded from the eligible list. Both are *correct* behaviours that will look like bugs. The options are to accept it or to commission a hand-written backfill with a named owner | **Choose one:** the product owner with the platform architect. `ADR-025` `FA1`, `R-29` | `DO2`'s launch communication. The feature ships either way; what changes is whether customers are told |
| `E4` | **The delivery-side replica has no staleness detector.** Full reconciliation is out of scope and nobody is asking for it. A dropped message currently produces a permanently empty list for that customer, which is indistinguishable from a correct answer | **Decide the minimum detector:** the platform architect with the platform owner. `ADR-026` `FA1`, `R-26` | `DO1`'s customer-facing list. Shipping it with no staleness signal is a product risk that needs a named acceptance |
| `E5` | **One order can have two delivery documents, and nothing decides what the customer sees.** Today the answer is "whichever the store returns first" | **Decide:** the product owner with the `DO1` implementer. `ADR-026` `FA2`, `R-27` | `DO1`'s list. A duplicate customer-facing row, or a reschedule that moves the wrong one of two documents |
| `E6` | **CAP-04's pipeline does not run its tests.** Most of the verification this run commissions lands in CAP-04 | **Fix the pipeline:** the platform owner. `INF-6`, `R-25`, `X13` | Every "must verify" in CAP-04 — the inspected version check, the atomic move, the priority classes, the release-leg identity check. They would be written and never run |
| `E7` | **Confirm per environment that the datastore supports transactional writes** before `disableTransactions` is set to false. A single-node deployment does not, and it fails at runtime rather than at startup | **Confirm:** the platform owner. `ADR-028` `FA3` | `DO2` in any environment that cannot support it. Discovering this in production is the failure this action prevents |
| `E8` | **The accessibility bar has no owner, no method and no implementation to hold it.** `NFR-12` names WCAG 2.1 AA against a platform with no frontend standard of any kind | **Fix a testable standard:** the platform architect with the product owner. `R-17`, `G-04` | Nothing immediately. Retrofitting after a component library and interaction model are chosen is materially more expensive, which is why the window is now |
| `E9` | **Decide whether CAP-09's existing unauthorized delivery read and exception-message-as-reason behaviour are corrected in this increment.** Both are live contracts with consumers this run cannot enumerate | **Decide:** the platform owner with the product owner. `ADR-029` `FA3`, `ADR-030` `FA3` | Nothing in this feature. Two error shapes and two authorization postures coexist in one service until somebody decides |
| `E10` | **Record, from a running environment, whether a vehicle id and an availability resource id actually hold the same value.** This run removes the reschedule's dependence on the answer; the existing order-advancing correlation still rests on it entirely | **Observe and record:** the platform owner. `ADR-025` `FA3`, `G-08` | Nothing in this feature. A platform-wide correlation keeps resting on an assumption nobody has checked |
| `E11` | **All twenty-nine ADRs are `Proposed` with `Deciders: Unassigned`**, because no repository has an owner anywhere in the fourteen clones | **Name owners:** the platform owner. `G-01`, `R-12` | Approval of `ADR-024`…`ADR-030`. The records are complete and nobody can accept them |

### 6.2 Delegated downstream — no action now

| # | Item | Owning stage |
|---|------|--------------|
| `Y1` | Confirm whether the inbox decorator's de-duplication is active for CAP-09's configuration, and record the answer | `hls` — the `DO1` implementer |
| `Y2` | Agree one predicate form for the version-conditioned write and apply it identically in all three repositories | `hls` — the `DO2` implementer with the platform architect |
| `Y3` | Enumerate the existing `Order` and `Delivery` write paths that will begin returning conflicts, and confirm each caller handles one | `hls` — the `DO2` implementer with the platform owner |
| `Y4` | Decide how schedule-lost and revalidation-unavailable are expressed inside the unchanged delivery contract | `hls` — the `DO1` implementer with the platform architect |
| `Y5` | Decide whether the recorded resource id is exposed on `OrderDto` | `hls` — the `DO2` implementer with the platform architect |
| `Y6` | Fix the five rejection code literals and the new message names on each exchange | `hls` — the `DO2` implementer with the product owner |
| `Y7` | Fix the instruction length bound and the redaction entry | `hls` — the `DO3` implementer with the product owner |
| `Y8` | Validate `L20` — that CAP-04's point read exposes the reservation set in an interpretable form — and grow an additive projection if it does not | `hls` — the `DO1` implementer |
| `Y9` | Converge CAP-07's six guard copies onto the single fail-closed guard | `lld` — the `DO2` implementer |
| `Y10` | Write the scoped-write-integrity pattern entry naming covered and uncovered aggregates, and apply the four pattern-catalog corrections in `X10` | `lld` — the `DO2` implementer |
| `Y11` | Decide, after the first release, whether the embedded rescheduling history needs to move out of the delivery document | `architecture` — the platform architect, `R-28` |
| `Y12` | Decide, after `DO2` ships, when the remaining eight aggregates gain write integrity | `architecture` — the platform architect, `R-30` |
| `Y13` | Decide, after the first release, whether a repeatedly failing revalidation should open a circuit rather than pay the timeout on every read | `architecture` — the platform architect |
| `Y14` | Decide, after the first release, whether the `(vehicleId, deliveryDate)` correlation migrates onto the recorded resource id | `architecture` — the platform architect |
| `Y15` | Decide whether the unvalidated `OrderId` on `deliveries-service` is in scope for `DO1` | `architecture` — the platform owner, `G-07` |
| `Y16` | Decide whether an admin acting on a customer's reservation should be audited separately | `architecture` — the platform architect |

### 6.3 What this stage deliberately did not do

- **It did not re-open the four human-gated decisions or the five resolved open questions.** `AD-1`
  to `AD-4` arrived chosen with high confidence and are applied as given. Option exploration ran only
  on the dimensions they do not cover.
- **It did not fix the five pre-existing defects it depends on.** The fail-open guards, the
  unauthorized delivery read, the two date truncations, the unvalidated `OrderId`, and the eight
  unprotected aggregates each stay as they are, each carries a register entry, and each has a named
  decision owner in §6.1 or §6.2. Fixing them inside a delivery feature would couple five independent
  corrections to one release.
- **It did not propose a new service, a new bounded context, a new runtime component, a new
  datastore, a shared library, a scheduler, or a circuit breaker.** Every fit verdict returned
  `can_extend_existing_component: true`, and each of those additions was considered and rejected with
  its reason recorded in the relevant ADR's Options Considered.
- **It did not write a per-ticket risk note, delta document or generation summary.** The risk register
  is one shared living queue and this run added to it in place.
