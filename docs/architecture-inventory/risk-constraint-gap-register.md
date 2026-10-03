# Pacco — Risk, Constraint & Gap Register

| Field | Value |
|-------|-------|
| Project | Pacco |
| Document role | The platform's single living queue of open architecture risks, binding constraints and evidenced gaps. Items are opened here, resolved here, and pruned from here |
| First authored | 2026-09-22, `architecture_evolution_generation`, work item 13652 |
| Last revised | 2026-10-03, `architecture_evolution_generation`, work item 14830 |
| Base ref of all cited source | `feature/13652/aidlc` for `R-01`…`R-14`, `C-01`…`C-09` and `G-01`…`G-06`; `feature/14830/aidlc` for `R-15`…`R-30`, `C-10`…`C-17` and `G-07`…`G-11` |
| Scope of the current FMEA | The architecture authored for work item 14830 — `ADR-024`…`ADR-030` and the design they record — scored as `R-15`…`R-30` and merged into the single queue below. The 13652 items `R-01`…`R-14` are carried forward unchanged: none was closed by this run, and nothing in `ADR-024`…`ADR-030` changes their exposure. Pre-existing platform constraints are carried in §3 and are not re-scored unless this run changes their exposure |
| Open questions promoted from 14830 | **None.** All five open questions raised by work item 14830 (`OQ-1`…`OQ-5`) arrived resolved — `OQ-3` by `ASM-18` and `OQ-5` by `ASM-20`, with the remaining three resolved with Product and Architecture before this stage. Nothing was left unresolved to promote into this register |

## How this register works

This file is **not** a per-ticket artifact and it is never copied per work item. It is one shared
queue with a lifecycle:

1. **Open** — an item is added with an id that is never reused.
2. **Resolve** — when an item is genuinely closed, its row is rewritten in place with
   `**RESOLVED <date> by <stage or person>**` and a sentence saying what closed it. It stays visible
   for one revision so a reader can see the disposition.
3. **Prune** — a resolved item is deleted at the next revision that touches this file. A deleted item
   leaves no id gap behind in the numbering.

Every row states, in plain words, **what it is · who must act · what breaks if it is ignored · and
whether it is needed now or later**, and carries **[ACTION NOW]** or **[handled later by \<stage\>]**.

## Contents

1. [Architecture Risk Assessment — FMEA](#1-architecture-risk-assessment--fmea)
2. [Risk detail](#2-risk-detail)
3. [Binding constraints](#3-binding-constraints)
4. [Evidenced gaps](#4-evidenced-gaps)
5. [Resolved, awaiting prune](#5-resolved-awaiting-prune)

---

## 1. Architecture Risk Assessment — FMEA

Scoring is the standard FMEA scale, 1 to 10 on each axis, `RPN = S × O × D`.

- **S — Severity.** How bad the effect is if the failure happens.
- **O — Occurrence.** How likely the failure mode is, given the design as recorded.
- **D — Detection.** How hard it is to notice **before** it causes harm. A failure that breaks
  loudly and immediately scores **low**. A failure that is silent, or that only shows up in
  production, scores **high**.

`RPN` ranks attention, it does not decide acceptance. `R-03` is the clearest example: it carries the
highest RPN in this register and is **deliberately accepted**, because removing it was placed out of
scope by an explicit reviewer decision.

`mitigation type` is one of `architecture_change`, `adr`, `infra`, `process` or `accept`. Where the
type is `adr` and the owner is the architecture stage, the record was authored **on this patch** —
`ADR-024`…`ADR-030` exist in `docs/adr/`, and the owner column then names the stage that carries the
remaining exposure, because a decision is not an implementation.

| id | title | category | S | O | D | RPN | mitigation type | owner stage | residual |
|----|-------|----------|---|---|---|-----|-----------------|-------------|----------|
| `R-03` | Client-side logout leaves the issued token valid at the edge | security | 7 | 10 | 8 | **560** | accept | Platform owner — reopen only as a separate edge decision | **High — accepted** |
| `R-18` | The ownership guard admits an unauthenticated or empty caller context | security | 9 | 6 | 8 | **432** | adr — `ADR-029`, authored this patch | lld — converge the six existing copies | Medium until `FA1` has an owner |
| `R-19` | A lost update on `Resource` is silent **and still publishes its events** | correctness | 8 | 6 | 9 | **432** | adr — `ADR-028`, authored this patch | hls | Low once the result is inspected |
| `R-15` | Inbox de-duplication does not cover the HTTP reschedule command, so redelivery is not de-duplicated for it | reliability | 7 | 6 | 8 | **336** | adr — `ADR-028`, authored this patch | hls | Medium — handler idempotence is the only control |
| `R-22` | A misspelled gateway `bind:` name silently restores the client-supplied `customerId` | security | 9 | 4 | 9 | **324** | adr — `ADR-029` Rule 5, authored this patch | devops — add the assertion to the gateway pipeline | Low once asserted |
| `R-13` | The JWT signing key is committed in the repository, and the browser client makes the edge it protects end-user facing | security | 9 | 5 | 7 | **315** | transfer | Platform owner | High |
| `R-20` | A submitted slot's intra-day precision is silently truncated on both sides | correctness | 6 | 7 | 7 | **294** | adr — `ADR-024` Rule 5, authored this patch | hls | Low once rejection replaces truncation |
| `R-24` | Routing key and queue binding diverge, and the only symptom is messages that never arrive | maintainability | 8 | 5 | 7 | **280** | adr — `ADR-030` Rules 4 and 5, authored this patch | devops — run the cross-boundary assertion | Medium — existing divergences stay open |
| `R-02` | The gateway returns downstream exception messages to the browser | security | 6 | 7 | 6 | **252** | mitigate — client-side | `DO1` implementation | Low once mitigated |
| `R-09` | No frontend standard exists, so the first client sets platform conventions by accident | governance | 4 | 9 | 7 | **252** | mitigate — write the minimum set | Platform architect | Medium |
| `R-23` | The outbox has no depth or oldest-age signal, so a stuck message looks like a quiet day | observability | 7 | 4 | 9 | **252** | infra — `INF-4` | devops | Medium until the signal exists |
| `R-25` | `availability-service`'s pipeline does not execute its tests, so every new test there is written and never run | operational readiness | 7 | 9 | 4 | **252** | infra — `INF-6` | devops | High until the pipeline runs them |
| `R-12` | No repository has an owner, so the gateway's public-contract change has no reviewer | governance | 6 | 8 | 5 | **240** | mitigate — name owners | Platform owner | Medium |
| `R-17` | The accessibility bar is asserted against a client that does not exist | usability | 5 | 8 | 6 | **240** | process — fix the bar before the first screen | Platform architect with the product owner | Medium |
| `R-26` | The delivery-side replica is never reconciled, so a dropped message is permanently invisible | correctness | 6 | 5 | 8 | **240** | adr — `ADR-026` §6.2, authored this patch | hls — the minimum staleness detector | **Medium — accepted in part** |
| `R-27` | One order can have two delivery documents, which becomes a duplicate row in a customer-facing list | correctness | 7 | 4 | 8 | **224** | process — decide the list behaviour | hls | Medium |
| `R-08` | Credentials or tokens leak into browser storage, logs or URLs | security | 9 | 4 | 6 | **216** | mitigate — client-side | `DO1` implementation | Low once mitigated |
| `R-30` | Three aggregates get write integrity and eight do not, with nothing marking the boundary | maintainability | 6 | 6 | 6 | **216** | accept — scoped by `AD-4`, chosen by human | Platform architect | **Medium — accepted** |
| `R-10` | The refresh token cannot be redeemed at the edge, so sessions end abruptly at expiry | usability | 5 | 9 | 4 | **180** | accept for now, decide later | Platform owner | Medium |
| `R-05` | The exact CORS origin is applied to a gateway configuration file the environment does not load | operability | 7 | 5 | 4 | **140** | mitigate — record the mapping | Platform owner | Low once recorded |
| `R-06` | An unknown or unsupported role renders the admin landing message | correctness | 9 | 3 | 5 | **135** | mitigate — closed-vocabulary check | `DO2` implementation | Low once mitigated |
| `R-28` | The append-only rescheduling history grows without bound inside a size-capped document | reliability | 6 | 3 | 7 | **126** | accept — `ASM-13` requires full retention | Platform architect | **Medium — accepted** |
| `R-29` | Orders and deliveries written before this change carry no resource id and no replica, and nothing backfills them | correctness | 5 | 8 | 3 | **120** | process — decide accept or hand-write a backfill | Platform owner with the product owner | Medium |
| `R-01` | The exact `Pacco.Web` local origin is not fixed, so no browser call succeeds | operability | 8 | 6 | 2 | **96** | mitigate — fix the value | Platform owner with the `Pacco.Web` implementer | Low |
| `R-07` | The landing page is reachable without an authenticated session | security | 6 | 4 | 4 | **96** | mitigate — client-side guard | `DO2` implementation | Medium — see detail |
| `R-16` | Reservation eligibility blocks on `customers-service`, the platform's synchronous leaf | availability | 6 | 5 | 3 | **90** | adr — `ADR-027` Rule 4 applied to the reschedule path | hls | Low once degradation is proven |
| `R-21` | A customer-facing delivery read now depends on `availability-service` being up | availability | 6 | 5 | 3 | **90** | adr — `ADR-027` Rules 3 and 4, authored this patch | hls — fix the timeout | Low once the timeout is fixed |
| `R-14` | The gateway library's actual CORS behaviour is unverified, because its source is not in the workspace | operability | 7 | 4 | 3 | **84** | mitigate — observe it | Platform owner | Low |
| `R-04` | The edge's allowed-method list omits `get`, blocking the first browser read surface | operability | 5 | 4 | 3 | **60** | defer | Platform architect | Low |
| `R-11` | A duplicate sign-in submission is accepted while a request is in flight | correctness | 4 | 5 | 3 | **60** | mitigate — client-side | `DO1` implementation | Low once mitigated |

---

## 2. Risk detail

Each entry names the failure mode, its effects, its causes, the mitigation, the ADR that governs it,
and what must be verified before the risk can be called closed.

### `R-03` — Client-side logout leaves the issued token valid at the edge · **[ACTION NOW — as an acceptance, not a fix]**

- **What it is.** Logout clears the browser's copy of the access token. It does not invalidate the
  token. Anyone holding a copy — a browser extension, a proxy log, a screenshot of a network tab, a
  shared machine — can keep using it against the gateway and every domain service until it expires.
- **Failure mode.** A token survives the session that created it.
- **Effects.** A user who believes they have logged out has not, from the platform's point of view.
  On a shared machine this is a real access-control failure, not a theoretical one.
- **Causes.** `ADR-007` obligation 4: only `identity-service` consults the revocation store, and its
  three token-management routes have **no gateway route in any of the four configurations**. There
  is no mechanism at the edge to revoke against.
- **Affected.** `Pacco.Web` (CAP-17), `api-gateway` (CAP-02), every authenticated route.
- **Mitigation · type `accept`.** Accepted by explicit reviewer decision: `DO2` holds the
  authentication and authorization backend out of scope, and exposing revocation at the edge would
  turn a login feature into a gateway architecture change. The mitigation is honesty — `ADR-021` §5
  rule 5 and §6.2 item 1 state the limitation in the record rather than implying it is solved.
- **ADR.** `ADR-021` §5 rule 5. Reversing it requires a **new** ADR that changes `ADR-007`'s
  revocation position — not an amendment to `ADR-021`.
- **Who must act.** Platform owner, if and when token revocation at the edge becomes in scope. **No
  action is required to ship `DO1` and `DO2`.**
- **Must verify.** That the limitation is visible to whoever accepts it: capture a token, log out,
  replay it against an authenticated route, and confirm it still succeeds. The test is there to keep
  the residual risk measured, not to pass.
- **Residual: High, accepted.**

### `R-18` — The ownership guard admits an unauthenticated or empty caller context · **[ACTION NOW]**

- **What it is.** The ownership check duplicated across six `orders-service` handlers reads "if the
  caller is authenticated **and** is not the owner **and** is not an admin, refuse". A caller with no
  identity fails the first clause and therefore passes the whole guard. The check is written as a
  refuse-list, so the unknown caller is admitted.
- **Failure mode.** An unauthenticated or empty caller context is treated as permitted.
- **Effects.** Any path that reaches a service without traversing the gateway operates on another
  customer's order with no check at all. This intent puts customers in direct control of a delivery
  date, so the guard now protects a customer-facing action rather than an internal one.
- **Causes.** The guard's clause order, and the fact that it is copied six times rather than written
  once — there is no single place to fix it and no way to confirm the six copies are identical.
- **Affected.** CAP-07 (six handlers), and by inheritance any new handler copied from them. CAP-09
  has no guard at all today.
- **Mitigation · type `adr`.** `ADR-029` Rules 1, 2 and 3, authored on this patch: deny by default,
  treat an empty caller context as the most restrictive input, and write the guard once per service.
  It governs **new** paths. It deliberately does not rewrite the six existing copies, because
  changing six live authorization paths is a blast radius this feature should not absorb unannounced.
- **ADR.** `ADR-029`, with `FA1` and `FA2`.
- **Who must act.** The platform owner with the platform architect must give `ADR-029` `FA1` — the
  existing bypass — an owner and a date, independently of this feature. The `DO2` implementer then
  converges the six copies during low-level design.
- **Must verify.** `ADR-029` `N1` and `N2`: call every new route with no token and assert refusal,
  and inject an empty identity directly at the handler and assert refusal.
- **Residual: Medium** until `FA1` has a named owner. Low on the new paths from the moment `N1`
  passes.

### `R-19` — A lost update on `Resource` is silent **and still publishes its events** · **[ACTION NOW]**

- **What it is.** `availability-service` writes with `ReplaceOneAsync(r => r.Id == resource.Id &&
  r.Version < resource.Version, ...)` — a correct optimistic-concurrency predicate — and never
  inspects the result. When the predicate does not match, zero documents are modified, the method
  returns normally, and the handler publishes its integration events as though the write had
  happened.
- **Failure mode.** A write that did not happen is announced as though it did.
- **Effects.** Worse than an ordinary lost update. Downstream services act on a reservation that does
  not exist: an order is advanced against a slot nobody holds. `orders-service` has no version
  predicate at all, so the same class of failure there is a plain silent overwrite.
- **Causes.** The discarded `ReplaceOneResult`; the absence of any version mechanism on `Order`; and
  `Delivery.Version`, which is declared, never incremented and never persisted.
- **Affected.** CAP-04, CAP-07, CAP-09, and every consumer of the events CAP-04 publishes.
- **Mitigation · type `adr`.** `ADR-028` Rules 1 to 4, authored on this patch: the predicate stays,
  **the result is inspected**, a non-match raises a conflict and publishes nothing, and the version is
  persisted and incremented on `Order` and `Delivery` as well.
- **ADR.** `ADR-028`, scoped by `AD-4` option B to `Resource`, `Order` and `Delivery`.
- **Who must act.** The `DO2` implementer during high-level design, per `INF-2`.
- **Must verify.** `ADR-028` `N4` above all: force the predicate to miss and assert the outbox is
  empty and no message was dispatched. `N1` to `N3` prove one success and one conflict per aggregate.
- **Residual: Low** for the three aggregates in scope once `N4` passes. See `R-30` for the eight that
  stay as they are.

### `R-15` — Inbox de-duplication does not cover the HTTP reschedule command · **[ACTION NOW]**

- **What it is.** The requirement that a redelivered reschedule command produces no second
  reservation names the platform's transactional inbox decorator as the mechanism. That decorator
  de-duplicates **messages**. The reschedule confirmation arrives over HTTP through the gateway, so
  the decorator is not on that path at all.
- **Failure mode.** A retried or duplicated confirmation is processed twice.
- **Effects.** Two reservations, or a move applied twice with the second release cancelling the first
  take. The customer sees a day they did not pick, or no day at all.
- **Causes.** A stated reliance on a mechanism that does not cover the path in question. Where the
  gateway runs in asynchronous mode the decorator does apply, which makes the exposure
  environment-dependent and therefore easy to miss in testing.
- **Affected.** CAP-02 (the edge write mode), CAP-04, CAP-09.
- **Mitigation · type `adr`.** `ADR-028` Rules 5 to 7 and `ADR-026` Rule 5, authored on this patch:
  handler-level idempotence is the control, not the decorator. The same request applied twice must be
  indistinguishable from applying it once, and a conflict is retried at most once before an explicit
  rejection.
- **ADR.** `ADR-028`; the rejection vocabulary is `ADR-030`.
- **Who must act.** The `DO2` implementer during high-level design. `ADR-026` `FA3` separately
  confirms whether the decorator is active at all for CAP-09's configuration, so the answer is
  recorded rather than assumed.
- **Must verify.** `ADR-028` `N7`: replay the same reschedule request and assert one reservation and
  one history entry. Run it in **both** gateway write modes, because only one of them has the
  decorator.
- **Residual: Medium.** Handler idempotence is a discipline enforced by tests, not by infrastructure.

### `R-22` — A misspelled gateway `bind:` name silently restores the client-supplied `customerId` · **[ACTION NOW]**

- **What it is.** The gateway binds `customerId: @user_id` from the validated token, overwriting
  whatever the client sent. That one line at `ntrada.yml:103` is the control that prevents a customer
  reserving against another customer's identity. If the bind name is misspelled, the binding does
  nothing and the client's own value is used.
- **Failure mode.** A one-character typo in a YAML file turns an authorization control into no
  control, with no error anywhere.
- **Effects.** A caller supplies any `customerId` and acts as that customer on the reservation path.
  Combined with `R-18`, there is then no server-side check to catch it either.
- **Causes.** No schema for the gateway configuration, no startup validation, no test, and four
  separate routing tables with none marked authoritative.
- **Affected.** CAP-02, CAP-04, and every customer whose identity could be impersonated.
- **Mitigation · type `adr`.** `ADR-029` Rule 5, authored on this patch: the binding is asserted by a
  test that sends a forged `customerId` through the real gateway configuration and checks the server
  observed the **token's** value. Nothing else on the platform would catch this.
- **ADR.** `ADR-029`, with `FA5`.
- **Who must act.** The platform owner must add that assertion to whatever pipeline gates gateway
  configuration changes, so a typo fails a build rather than a customer.
- **Must verify.** `ADR-029` `N7`, run against each of the four `ntrada*.yml` files that an
  environment might load — `G-02` records that the environment-to-file mapping is itself unrecorded.
- **Residual: Low** once the assertion runs in the pipeline. High until then, because the control is
  invisible when it fails.

### `R-13` — The committed signing key, now protecting an end-user-facing edge · **[ACTION NOW]**

- **What it is.** A symmetric `issuerSigningKey` is committed in all four `ntrada*.yml` files and in
  `Pacco.Services.Operations.Api/appsettings.json`. Anyone with repository access can mint a token
  the gateway accepts. This predates `ADR-021` — what changes is the exposure: the edge it protects
  stops being machine-only and starts carrying end-user sessions.
- **Failure mode.** A forged token is accepted as an authenticated user.
- **Effects.** Complete bypass of authentication and of every claim gate, for any route.
- **Causes.** The key was committed rather than injected. `ADR-006` obligation 4 already records
  that it must leave source control.
- **Affected.** `api-gateway` (CAP-02), `identity-service` (CAP-01), every service behind the edge.
- **Mitigation · type `transfer`.** Move the key to the platform's existing secret mechanism (Vault
  is already in the stack for database credentials) and rotate it. This is `ADR-006`'s obligation,
  not a new one, and `ADR-021` does not discharge it.
- **ADR.** `ADR-006` obligation 4. `ADR-021` §6 does not change it.
- **Who must act.** Platform owner. Not a blocker for `DO1` or `DO2` in local development, and a
  blocker for any non-local environment.
- **Must verify.** That no key material remains in any `ntrada*.yml` or `appsettings.json`, and that
  the gateway still validates tokens after the key is injected rather than committed.
- **Residual: High** until the key is moved.

### `R-20` — A submitted slot's intra-day precision is silently truncated on both sides · **[handled later by HLS]**

- **What it is.** Two independent truncations sit on the reschedule path. `Order.SetDeliveryDate`
  assigns `d.Date`, discarding the time of day and performing no `DateTimeKind` normalisation.
  `availability-service` converts a reservation date with `AsDaysSinceEpoch`, which yields whole days
  since `0001-01-01` and discards the time entirely — so the value a reservation event carries is not
  the value the store holds.
- **Failure mode.** A submitted value is reinterpreted rather than rejected.
- **Effects.** A customer picking a slot with any time component gets a different slot than they
  chose, silently. Worse, the `(vehicleId, deliveryDate)` correlation that advances orders works only
  because **both** sides truncate the same way; if either side ever emits a non-midnight or
  differently-kinded value, every correlation lookup fails and affected orders stop advancing with no
  error.
- **Causes.** Date handling implemented twice, differently, with no shared representation and no
  validation at the edge.
- **Affected.** CAP-04, CAP-07, and `DO2`'s confirmation path.
- **Mitigation · type `adr`.** `ADR-024` Rule 5, authored on this patch: intra-day precision the
  reservation store cannot hold is **rejected** at the edge rather than discarded. The slot is a
  calendar day by `ASM-9`, so a value carrying a time is a caller error, not something to round.
- **ADR.** `ADR-024`; the read-side counterpart is `ADR-027` `N6`.
- **Who must act.** The `DO2` implementer during high-level design.
- **Must verify.** `ADR-024`'s rejection test, plus `ADR-027` `N6` — drive the revalidation
  comparison with a timestamped date and assert it yields the same verdict as the midnight case or an
  explicit rejection, never a silent mismatch.
- **Residual: Low** once rejection replaces truncation on the new path. The two existing truncations
  stay as they are, which is why `orders-service` `Q-6` remains open.

### `R-24` — Routing key and queue binding diverge, and the only symptom is messages that never arrive · **[ACTION NOW]**

- **What it is.** Message compatibility on this platform rests on a naming convention with no shared
  artifact and no build-time signal. A divergence between the routing key a publisher emits and the
  key a consumer binds already exists on the platform, and its only symptom is that messages are
  never delivered. Nothing fails, nothing logs.
- **Failure mode.** A publisher and a consumer disagree about a string, and the feature simply does
  not work.
- **Effects.** `ADR-026` gives CAP-09 its first inbound subscription. If its binding does not match
  what CAP-07 publishes, the replica is never populated, so every delivery is excluded from every
  customer's eligible list — which, by `ADR-026` Rule 4, is a *correct-looking* empty list rather
  than an error.
- **Causes.** No shared contracts package, by decision; convention-only naming; and the key being
  retyped on each side rather than referenced from one literal.
- **Affected.** CAP-04, CAP-07, CAP-09, CAP-11.
- **Mitigation · type `adr`.** `ADR-030` Rules 4 and 5, authored on this patch: one literal per name,
  referenced by both sides, and **one test per new message that asserts the publisher's key against
  the consumer's binding across the boundary**. A test that checks each side against its own constant
  proves nothing.
- **ADR.** `ADR-030`, with `FA4`; `ADR-026` `N7`.
- **Who must act.** The platform owner must get that assertion into the CAP-04 and CAP-09 pipelines —
  and `R-25` records that CAP-04's pipeline does not run tests at all today.
- **Must verify.** `ADR-030` `N6` and `N7`.
- **Residual: Medium.** The assertion covers the new names. Existing divergences, including the one
  already recorded on the platform, stay open and are not closed by this run.

### `R-02` — The gateway returns downstream exception messages to the browser · **[ACTION NOW]**

- **What it is.** `extensions.customErrors.includeExceptionMessage` is `true` in all four gateway
  configurations (`ntrada.yml:24-25` and the three siblings), so when a downstream service throws,
  the exception message travels to the caller. `DO1` targets **zero raw backend exceptions or stack
  traces shown to users**, so this is directly load-bearing on an acceptance target.
- **Failure mode.** A backend exception message is rendered to an end user.
- **Effects.** `DO1`'s target is missed. Exception text can disclose internal structure — service
  names, validation internals, and on a database or serialization error, field names.
- **Causes.** An edge configuration value set for machine callers, unchanged now that the caller is
  a browser.
- **Affected.** `Pacco.Web` Login screen (CAP-17), `api-gateway` (CAP-02).
- **Mitigation · type `mitigate`, client-side.** `Pacco.Web` maps every non-success response to a
  fixed non-technical message and **never renders a response body verbatim**. This is the mitigation
  `ADR-021` §8 `N2` requires. Turning `includeExceptionMessage` off at the edge would be a broader
  change affecting every existing machine caller, and is **not** proposed here.
- **ADR.** `ADR-021` §8 `N2`. The edge setting itself is governed by `ADR-004`.
- **Who must act.** The `DO1` implementer, in the Login screen's error handling.
- **Must verify.** Force invalid credentials, a stopped `identity-service`, and a malformed request.
  Assert the fixed message renders in all three and that the raw body appears nowhere in the DOM.
- **Residual: Low** once the client-side mapping exists. The edge setting remains as it is.

### `R-09` — No frontend standard exists, so the first client sets conventions by accident · **[ACTION NOW]**

- **What it is.** The platform has no frontend standard of any kind: no state-ownership rule, no
  accessibility level, no error-presentation rule, no client logging or redaction rule, no dependency
  or lockfile policy. `ADR-021` §7.1 records this rather than importing an outside convention.
- **Failure mode.** Whatever the Login screen happens to do becomes the platform's convention.
- **Effects.** The second and third surfaces inherit accidents. Accessibility in particular is very
  expensive to retrofit and nearly free to build in.
- **Causes.** No client ever existed, so no standard was ever needed.
- **Affected.** CAP-17 and every future browser surface.
- **Mitigation · type `mitigate`.** Write the minimum set **as the client is built**, not after:
  accessibility level, error-presentation rule, no-credential-logging rule, dependency policy.
- **ADR.** `ADR-021` §7.1 and `Q4`, FA6 for the related design-system question.
- **Who must act.** Platform architect, with the `DO1` implementer.
- **Must verify.** That the four rules above exist in writing somewhere the platform can cite before
  a second surface is started.
- **Residual: Medium** until written.

### `R-23` — The outbox has no depth or oldest-age signal, so a stuck message looks like a quiet day · **[handled later by DevOps]**

- **What it is.** Schedule-change events leave a service through a background outbox dispatcher.
  Nothing emits the queue depth or the age of the oldest unpublished record, and nothing alerts on
  either. At least one service additionally runs its outbox with transactions disabled, so the domain
  write and the outbox record are not one atomic unit.
- **Failure mode.** The outbox stops draining and the platform looks idle.
- **Effects.** A confirmed reschedule never reaches `orders-service`, so the authoritative delivery
  date never moves. The customer was told the move succeeded. Nobody is paged, because an outbox with
  a thousand stuck records and an outbox with none emit exactly the same signal — none.
- **Causes.** No metric, no alert, and no position recorded anywhere on availability or latency
  against which "stuck" could be defined.
- **Affected.** CAP-04, CAP-07, CAP-09.
- **Mitigation · type `infra`.** `INF-4`: emit outbox depth and oldest-unpublished-age for the three
  services in scope, and alert ahead of the expiry window. `ADR-028` Rule 5 makes the write and the
  outbox record atomic, which removes the half-written case but not the never-dispatched one.
- **ADR.** `ADR-028`, with `FA5`.
- **Who must act.** The platform owner, before `DO2` reaches a shared environment.
- **Must verify.** Stop the dispatcher, write a reschedule, and confirm depth and age both move and
  that an alert fires before the operation-status record expires.
- **Residual: Medium** until the signal exists. This is the detection axis, which is why the score is
  9 on `D` despite a modest severity.

### `R-25` — `availability-service`'s pipeline does not execute its tests · **[ACTION NOW]**

- **What it is.** `availability-service` has the deepest test coverage on the platform — five test
  projects including a load-test project — and its pipeline does not run them. Tests that compile and
  never execute are documentation.
- **Failure mode.** A regression ships with a green build.
- **Effects.** Most of the verification this run commissions lands in CAP-04: the inspected version
  check, the atomic take-then-release, the priority refusal classes, the release-leg identity check.
  All of it would be written and none of it would run. The platform would gain the *appearance* of
  verification on its highest-RPN correctness risks.
- **Causes.** The pipeline definition, independently per repository with no shared template.
- **Affected.** CAP-04, and every risk whose "must verify" lands there — `R-19`, `R-20`, `R-24`,
  `R-15`.
- **Mitigation · type `infra`.** `INF-6`: make the pipeline execute the test step and fail the build
  on failure, so the published image is blocked.
- **ADR.** `ADR-018`; carried by `ADR-028` `FA4` and `ADR-030` `FA4`.
- **Who must act.** The platform owner, before `DO2` ships.
- **Must verify.** Push a deliberately failing test to a branch and confirm the build fails and no
  image is published.
- **Residual: High** until the pipeline runs them. Everything else in this register that says "must
  verify" in CAP-04 is contingent on this one.

### `R-12` — The gateway's public-contract change has no reviewer · **[ACTION NOW]**

- **What it is.** `ADR-004` §2 obligation 1 makes the gateway configuration a reviewed architectural
  artifact: a change to it changes the platform's public contract. The CORS change `ADR-021` requires
  is exactly such a change. **No repository in the workspace has a `CODEOWNERS`, a contributing guide
  or any team metadata**, so there is no named reviewer to satisfy that obligation.
- **Failure mode.** A public-contract change ships unreviewed, and `ADR-021` stays `Proposed` forever.
- **Effects.** The governing obligation is unenforceable in practice. Every ADR in the corpus —
  all twenty-one — is blocked from moving past `Proposed` for the same reason.
- **Causes.** No ownership metadata has ever existed. Carried as `adr-candidates.md` B4 since the
  ADR corpus was written.
- **Affected.** `api-gateway` (CAP-02), `Pacco.Web` (CAP-17), the whole ADR corpus.
- **Mitigation · type `mitigate`.** Name an owner for `Pacco.APIGateway` and for `Pacco.Web`. Two
  names unblock this.
- **ADR.** `ADR-004` §2 obligation 1, `ADR-021` B3, `adr-candidates.md` B4.
- **Who must act.** Platform owner. Needed **before** `ADR-021` can be approved.
- **Must verify.** That a named person or team is recorded against both repositories.
- **Residual: Medium.**

### `R-17` — The accessibility bar is asserted against a client that does not exist · **[ACTION NOW]**

- **What it is.** `NFR-12` requires the new reschedule screens to meet WCAG 2.1 AA. No accessibility
  standard is recorded anywhere on the platform, the only client capability has no implementation,
  and the requirement's own source is an assumption at low confidence.
- **Failure mode.** A quality bar that nobody owns, nobody tests and nobody can fail.
- **Effects.** The screens ship meeting whatever bar the first implementer happens to apply.
  Retrofitting accessibility after a component library and an interaction model are chosen is
  materially more expensive than choosing them with it in mind, which is why the occurrence score is
  high and the window is now.
- **Causes.** No frontend standard of any kind exists — this is the same absence `R-09` and `G-04`
  record, now with a named requirement attached to it.
- **Affected.** CAP-17, and the `DO1` and `DO3` customer-facing surfaces.
- **Mitigation · type `process`.** Fix the bar, the conformance method and who checks it **before**
  the first screen is written. This is not something architecture can decide by itself: it needs a
  product owner's acceptance of a testable standard.
- **ADR.** None. `ADR-021` records the client boundary and deliberately takes no position on frontend
  quality rules; inventing one here would be a training-prior default, not a decision.
- **Who must act.** The platform architect with the product owner, before `DO1`'s first screen.
- **Must verify.** That a named, testable conformance check exists and runs — not that someone has
  written "WCAG 2.1 AA" in a document.
- **Residual: Medium.** The requirement is real and the mechanism to hold it is absent.

### `R-26` — The delivery-side replica is never reconciled · **[ACTION NOW]**

- **What it is.** `ADR-026` adopts the platform's event-carried replica pattern, including the part
  the platform has never solved. The existing `customers` replicas in `orders-service` and
  `parcels-service` are written once at creation and never reconciled; the two events that would
  update them have no handler anywhere. The new delivery-side replica inherits that whole.
- **Failure mode.** A missed, dropped or mis-bound message leaves the copy permanently wrong, with no
  detection and no repair path.
- **Effects.** A delivery shows a date that is not its date, or — because `ADR-026` Rule 4 excludes a
  delivery with no replica — is permanently invisible to its own customer, as an empty list rather
  than an error. There is no reconciliation job anywhere on this platform and this run does not
  create one.
- **Causes.** At-least-once delivery with no reconciliation, no scheduler, and no job host.
- **Affected.** CAP-09, and the customer-facing reads in `DO1` and `DO3`.
- **Mitigation · type `adr`.** `ADR-026` §6.2 and `FA1`, authored on this patch: the gap is recorded
  rather than discovered later, Rule 5 makes handlers idempotent and ordering-tolerant so that
  re-delivery repairs rather than corrupts, and `FA1` asks for the **minimum viable detector** — a
  count comparison or an age alarm on the replicated date — not full reconciliation.
- **ADR.** `ADR-026`, with `FA1`; the precedent and its gap are `ADR-009`.
- **Who must act.** The platform architect with the platform owner must decide the minimum acceptable
  detection before `DO1` ships. Shipping a customer-facing list fed by an undetectable stale replica
  is a product risk that needs a named acceptance.
- **Must verify.** Drop a message in a test environment and confirm the detector notices within the
  agreed window. Without a detector there is nothing to verify, which is the point.
- **Residual: Medium — accepted in part.** The reconciliation gap is platform-wide and out of this
  increment's scope. The detection gap is not.

### `R-27` — One order can have two delivery documents · **[handled later by HLS]**

- **What it is.** `OrderId` on the delivery document is unindexed and not unique. A delivery that
  fails and is restarted inserts a **second** document with the same `OrderId`, and the lookup by
  order returns a non-deterministic one of them.
- **Failure mode.** The same order has two deliveries and the platform picks one arbitrarily.
- **Effects.** Today this is an internal oddity. Once `DO1` lists deliveries to customers it becomes
  a duplicate row in a customer-facing list, and a reschedule that targets "the" delivery for an
  order may move whichever document the query happened to return — leaving the other one showing the
  old date.
- **Causes.** No uniqueness constraint, no index, and a restart path that inserts rather than
  transitions.
- **Affected.** CAP-09, and `DO1`'s list.
- **Mitigation · type `process`.** `ADR-026` `FA2`: decide what the customer-facing list does when an
  order has two delivery documents — show one, show both, or refuse. Today the answer is "whichever
  the store returns first", which is not a decision anyone made.
- **ADR.** `ADR-026` §6.2. Adding a uniqueness constraint is **not** proposed here: existing data may
  already violate it, and there is no migration tooling to find out safely.
- **Who must act.** The product owner with the `DO1` implementer, before `DO1` ships.
- **Must verify.** `ADR-026` `N11`: insert two documents with one `OrderId` and assert the list
  behaviour is the decided one, not the incidental one.
- **Residual: Medium.** The underlying duplication is not closed by this run.

### `R-08` — Credentials or tokens leak into browser storage, logs or URLs · **[handled later by `DO1` implementation]**

- **What it is.** `DO1` targets zero password values logged or displayed and zero credentials
  hard-coded in the client. Browsers make all three leak paths easy: a console log, a token in
  `localStorage`, a credential in a query string that lands in history and in any intermediary's logs.
- **Failure mode.** A password or token is written somewhere it outlives the request.
- **Effects.** Credential disclosure. Combined with `R-03`, a leaked token is usable until it expires
  with no way to revoke it.
- **Causes.** No client logging or redaction rule exists (`R-09`).
- **Affected.** `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** The password is read from the form, sent once in the request
  body, and never persisted, logged, or placed in a URL. No credential is compiled into the client.
- **ADR.** `ADR-021` §8 `N1`.
- **Who must act.** The `DO1` implementer.
- **Must verify.** Sign in with a known password and inspect the console, the network tab, browser
  storage and the built bundle for the literal value. Grep the bundle for any credential literal.
- **Residual: Low** once verified.

### `R-30` — Three aggregates get write integrity and eight do not · **[ACTION NOW — as an acceptance, not a fix]**

- **What it is.** `AD-4` scopes version-conditioned writes and outbox atomicity to `Resource`,
  `Order` and `Delivery` for this increment. The platform's other eight aggregates keep
  last-writer-wins, and nothing in the code marks which is which.
- **Failure mode.** A developer who has read `Resource` reasonably assumes `Parcel` behaves the same
  way, and writes a concurrent path that silently loses data.
- **Effects.** An inconsistency that is invisible at the call site. The three protected aggregates
  look like the house style and are the exception.
- **Causes.** A deliberate scope decision, plus the replication-per-repository constraint — there is
  no shared library, so the pattern is written three times and nothing centrally records its reach.
- **Affected.** Every service outside CAP-04, CAP-07 and CAP-09.
- **Mitigation · type `accept`.** Accepted by the human decision recorded as `AD-4` option B, and the
  reasoning is sound: eleven services with hand-mapped documents, no migration tooling and a pipeline
  that does not reliably run tests is how a correctness fix becomes an outage. The compensation is
  discoverability — `ADR-028` `FA6` requires the pattern entry to **name which aggregates are covered
  and which are not**.
- **ADR.** `ADR-028`, §6.2 and `FA6`; the scope is `AD-4`.
- **Who must act.** The platform architect, after `DO2` ships, to decide when the remaining eight are
  brought in. Nothing currently schedules them. **No action is required to ship `DO2`.**
- **Must verify.** That the catalog entry states the boundary explicitly. The verification here is
  that the inconsistency is discoverable, not that it is gone.
- **Residual: Medium, accepted.**

### `R-10` — The refresh token cannot be redeemed at the edge · **[ACTION NOW — as a decision, not a fix]**

- **What it is.** Sign-in returns a `refreshToken` in `AuthDto`, but `POST refresh-tokens/use` has no
  gateway route in any of the four configurations. The client receives a token it has no way to use.
- **Failure mode.** The session ends the moment the access token expires, mid-task, with no renewal.
- **Effects.** Acceptable for two screens. Not acceptable for a working application, and the longer
  it goes unstated the more likely a later surface assumes renewal exists.
- **Causes.** The token-management routes were never exposed at the edge (GAP-21).
- **Affected.** `Pacco.Web` (CAP-17), `api-gateway` (CAP-02), `identity-service` (CAP-01).
- **Mitigation · type `accept` now, `decide` later.** `DO1` and `DO2` do not need renewal. What is
  needed now is a **stated position** — either route the refresh endpoint in a later record, or state
  that Pacco sessions end at access-token expiry by design.
- **ADR.** `ADR-021` §6.3 item 3 and `Q1`.
- **Who must act.** Platform owner — a decision, before the third browser surface, not before `DO1`.
- **Must verify.** That the position is written down, whichever way it goes.
- **Residual: Medium** until stated.

### `R-05` — The origin is applied to a configuration file the environment does not load · **[ACTION NOW]**

- **What it is.** Four `ntrada*.yml` files exist with identical CORS blocks, selected by the
  `NTRADA_CONFIG` environment variable. Nobody has recorded which file each environment loads
  (`ADR-004` B2). Under a wildcard this was invisible. Under an exact origin it decides whether the
  browser works.
- **Failure mode.** The exact origin lands in a file the running gateway is not reading.
- **Effects.** All browser calls are refused cross-origin. It fails loudly and early, which is why
  `D` is low, but it is easy to misdiagnose as a client bug.
- **Causes.** Four configurations, no recorded environment mapping.
- **Affected.** `api-gateway` (CAP-02), `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** Apply the change to **all four** files — they are identical today
  and `ADR-021` §5 rule 4 keeps them identical — and record the environment-to-file mapping.
- **ADR.** `ADR-004` §2 obligation 2, `ADR-021` FA2 and B2.
- **Who must act.** Platform owner. Compose loads `ntrada-async.docker.yml`
  (`compose/services.yml:4-13`), which settles the local case and not the others.
- **Must verify.** Diff the CORS block across all four files after the change, and confirm the
  running gateway's loaded configuration.
- **Residual: Low** once recorded.

### `R-06` — An unknown role renders the admin landing message · **[handled later by `DO2` implementation]**

- **What it is.** `DO2` targets zero unknown-role or normal users shown `Welcome to Admin Area`. The
  role vocabulary is closed and lower-cased — `Role.cs` defines exactly `user` and `admin` and
  validates case-insensitively — so anything else is unknown.
- **Failure mode.** A default-to-admin branch, a case-sensitivity slip, or a role inferred from an
  email address or username shows the admin message to a non-admin.
- **Effects.** A user is told they are in an admin area. The gateway's claim gates still refuse the
  admin routes, so this misleads rather than escalates — which is why `O` is low and `S` is high.
- **Causes.** Role comparison written permissively, or a fallback branch that treats unknown as admin.
- **Affected.** `Pacco.Web` landing page (CAP-17).
- **Mitigation · type `mitigate`.** Compare against the closed vocabulary explicitly. Unknown renders
  the non-admin message. Never infer a role from an identifier.
- **ADR.** `ADR-021` §8 `N4` and §5.2.
- **Who must act.** The `DO2` implementer.
- **Must verify.** Sign in as `admin`, as `user`, and with a token carrying an unrecognised role.
  Assert the message in each case, and repeat after a reload and a logout/login cycle.
- **Residual: Low** once verified.

### `R-28` — The append-only rescheduling history grows without bound · **[handled later by the platform architect]**

- **What it is.** `ASM-13` makes the rescheduling history an audit record with full retention —
  entries are added, never edited, never deleted. `ADR-026` Rule 7 embeds it in the delivery
  document, which has a hard size ceiling.
- **Failure mode.** A delivery rescheduled pathologically often eventually fails to save.
- **Effects.** The failure arrives on a write, at the worst moment, for the one customer whose record
  is largest. `ASM-17` sets no cap and no cool-off on how often a customer may reschedule, so nothing
  bounds the entry count from the product side either.
- **Causes.** Full retention plus document embedding plus no rate limit, each individually reasonable.
- **Affected.** CAP-09.
- **Mitigation · type `accept`.** Accepted for this increment. `ASM-13` forbids pruning, so the only
  other lever is *where* the history is stored, and moving it out of the document is a change this
  feature does not need. It is recorded here rather than mitigated by a silent cap, which would
  violate `ASM-13` quietly.
- **ADR.** `ADR-026` Rule 7 and `Q1`.
- **Who must act.** The platform architect, after the first release, if the entry distribution turns
  out to have a long tail. **No action is required to ship `DO3`.**
- **Must verify.** Measure the entry count distribution once real usage exists. There is nothing
  useful to verify before that.
- **Residual: Medium, accepted.**

### `R-29` — Nothing backfills the orders and deliveries written before this change · **[ACTION NOW]**

- **What it is.** `ADR-025` records the reserved resource id on the order and `ADR-026` replicates the
  customer and delivery date onto the delivery. Both are nullable additive fields, and both are
  populated only by events that arrive **after** the change ships. There is no migration tooling
  anywhere on the platform, so the only way to populate historical records is a hand-written script
  with no home in any repository.
- **Failure mode.** A correct refusal that looks like a bug.
- **Effects.** Deliveries already in flight when this ships cannot be rescheduled — `ADR-025` Rule 4
  rejects a reschedule for an order with no recorded resource id rather than guessing from the
  vehicle id — and are excluded from the customer's eligible list by `ADR-026` Rule 4. This is a real
  functional limitation of the first release, not an edge case.
- **Causes.** Additive-only persistence, which is itself required because there is no migration
  tooling.
- **Affected.** CAP-07, CAP-09, and every customer with an open delivery at release time.
- **Mitigation · type `process`.** Decide between two honest options: accept that pre-existing
  deliveries are not reschedulable until their order is reserved again, or commission a hand-written
  backfill and give it an owner. Both are legitimate; neither is the default.
- **ADR.** `ADR-025` §6.2 and `FA1`; `ADR-026` §6.2.
- **Who must act.** The product owner with the platform architect, before `DO2` ships. The decision
  changes what the launch communication has to say.
- **Must verify.** `ADR-025` `N3`: a reschedule against an order with no recorded resource id is
  rejected with its own distinct reason and makes no reservation call — so the limitation is
  explainable to a customer rather than arriving as a generic failure.
- **Residual: Medium** until the decision is made. Low afterwards either way, because both options
  are visible.

### `R-01` — The exact `Pacco.Web` local origin is not fixed · **[ACTION NOW]**

- **What it is.** `ADR-021` §5 rule 4 replaces the wildcard with `Pacco.Web`'s exact local origin.
  That origin — scheme, host and port — has not been chosen. `http://localhost:3000` is the intent's
  example, not a committed value.
- **Failure mode.** The allow-list names an origin the client does not serve on.
- **Effects.** Every credentialed browser call is refused. `DO1` cannot complete a sign-in round trip.
- **Causes.** The client's local runtime has not been chosen yet.
- **Affected.** `api-gateway` (CAP-02), `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** Fix the port when the client's local runtime is chosen, and apply
  it to all four files in one change together with `R-05`.
- **ADR.** `ADR-021` §5 rule 4, FA1, B1.
- **Who must act.** Platform owner with the `Pacco.Web` implementer, before `DO1` is exercised end to
  end.
- **Must verify.** A credentialed `POST /identity/sign-in` preflight from the allowed origin returns
  that origin in `Access-Control-Allow-Origin`, and one from a different origin does not.
- **Residual: Low** — it fails loud, immediately, at the browser.

### `R-07` — The landing page is reachable without an authenticated session · **[handled later by `DO2` implementation]**

- **What it is.** `DO2` targets zero unauthenticated requests reaching the landing page. The guard is
  a client-side route guard, because the landing page is a client-rendered screen with no server
  route behind it.
- **Failure mode.** A direct request, a reload, or a back-navigation after logout renders the landing
  screen without a session.
- **Effects.** The screen shows. **It shows nothing privileged** — `DO2` introduces no dashboard
  widgets and no business functionality, and any real data would come from a gateway call the missing
  token would refuse. The harm is a broken guarantee, not a data disclosure.
- **Causes.** A guard applied on navigation but not on direct load or on reload.
- **Affected.** `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** Guard on every entry into the route — navigation, direct load,
  reload and back-navigation — and redirect to Login. An expired token takes the same path with a
  session-expired message.
- **ADR.** `ADR-021` §8 `N3`.
- **Who must act.** The `DO2` implementer.
- **Must verify.** Request the landing route with no session, with a cleared session after logout,
  and with an expired token. Assert the redirect in all three and that it survives a reload.
- **Residual: Medium** by nature: a client-side guard is a UX guarantee, not a security boundary. The
  security boundary stays at the edge, where `ADR-006` puts it. Any future landing screen that shows
  real data must get that data through an authenticated gateway call, which the edge enforces
  independently of this guard.

### `R-16` — Reservation eligibility blocks on `customers-service` · **[handled later by HLS]**

- **What it is.** The reservation handler makes a synchronous, blocking HTTP call to
  `customers-service` to check customer state, even though a local replica exists.
  `customers-service` calls nothing itself and is read by two capabilities plus the gateway: it is
  the platform's synchronous leaf and single point of dependency for both reservation eligibility and
  pricing.
- **Failure mode.** A `customers-service` outage fails every reschedule confirmation.
- **Effects.** The reschedule path inherits this dependency unchanged — it is not introduced by this
  work, it is extended to a new customer-facing action. The requirement is that it degrade to a clear
  customer-readable retryable failure and never a partial reschedule, not that it be removed.
- **Causes.** A pre-existing blocking read where a replica already exists, outside this increment's
  scope to change.
- **Affected.** CAP-03, CAP-04, and `DO2`'s confirmation.
- **Mitigation · type `adr`.** `ADR-027` Rules 3 and 4 applied to this leg: an explicit timeout, and
  a degraded, labelled outcome rather than an unhandled error. `ADR-030`'s `system error` class is
  the vocabulary it surfaces through, and it is retryable.
- **ADR.** `ADR-010` for the call shape; `ADR-027` for the failure policy; `ADR-030` Rule 1.
- **Who must act.** The `DO2` implementer during high-level design.
- **Must verify.** Stub `customers-service` to time out, attempt a confirmation, and assert a
  retryable customer-readable rejection **and** that no reservation was taken or released — a partial
  reschedule is the outcome that must not occur.
- **Residual: Low** once the degradation is proven. The dependency itself stays.

### `R-21` — A customer-facing delivery read now depends on `availability-service` being up · **[ACTION NOW]**

- **What it is.** `ADR-027` gives `deliveries-service` its first outbound synchronous call, on a
  customer-facing read path, in a service that has never had one — no client, no configured timeout,
  no failure policy, and no service-identity certificate.
- **Failure mode.** A slow or unreachable `availability-service` makes a customer's delivery list
  slow or unavailable.
- **Effects.** A genuine reduction in the independence `deliveries-service` has today. With no
  configured timeout the default is effectively unbounded, so the read does not fail — it hangs. The
  platform has no circuit breaker anywhere, so a persistently slow dependency means every read pays
  the full timeout.
- **Causes.** `ASM-20` locates the revalidation at read time, which is the right call for
  correctness — the alternative relies on a release path that returns silently when nothing matches,
  so the event that would keep a replica honest is exactly the one that may not fire.
- **Affected.** CAP-09, CAP-04, and `DO1`'s customer-facing reads.
- **Mitigation · type `adr`.** `ADR-027` Rules 2, 3 and 4, authored on this patch: revalidate once
  per **distinct resource** rather than per delivery, declare an explicit timeout in configuration,
  and degrade to a labelled `revalidation unavailable` answer that is distinguishable from both a
  confirmed schedule and a lost one.
- **ADR.** `ADR-027`, with `FA2` and `FA5`.
- **Who must act.** The platform owner with the `DO1` implementer must **fix the timeout as a number,
  in configuration, before this path is enabled**. The requirement it serves cannot be assessed
  against an unspecified timeout.
- **Must verify.** `ADR-027` `N4` and `N5`: with the client stubbed to time out, the read still
  returns successfully, marked unavailable, and that response is distinguishable from a confirmed
  one. `N3` asserts one outbound call for five deliveries on one resource.
- **Residual: Low** once the timeout is fixed and `N4` passes.

### `R-14` — The gateway library's actual CORS behaviour is unverified · **[ACTION NOW]**

- **What it is.** Ntrada is a NuGet reference; its source is not in the workspace. That
  `extensions.cors.allowedOrigins` is an exact-match allow-list which echoes the matched origin with
  `Access-Control-Allow-Credentials: true` is read from the key names, not proven from code. The same
  limitation applies to today's wildcard-plus-credentials combination: nobody has observed whether
  the library accepts it, rejects it, or silently rewrites it.
- **Failure mode.** The configuration means something other than what it reads as.
- **Effects.** Either the browser path fails after the change, or the edge stays permissive after the
  wildcard is removed — the second being the dangerous one, because it is silent.
- **Causes.** A third-party configuration dialect with no source in the workspace (`ADR-004` A1).
- **Affected.** `api-gateway` (CAP-02).
- **Mitigation · type `mitigate`.** Observe it. Run the gateway with the exact origin and compare the
  response headers for an allowed and a disallowed origin. Record what the library actually does.
- **ADR.** `ADR-021` A1, FA3.
- **Who must act.** Platform owner, when `R-01` is applied.
- **Must verify.** `Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials` on a
  credentialed preflight from both origins.
- **Residual: Low** once observed.

### `R-04` — The allowed-method list omits `get` · **[handled later by HLS]**

- **What it is.** `extensions.cors.allowedMethods` lists `post`, `put` and `delete` only, in all four
  configurations. No cross-origin `GET` is permitted through the edge.
- **Failure mode.** The first browser surface that reads through the gateway is blocked.
- **Effects.** None for `DO1` or `DO2` — their only platform call is `POST /identity/sign-in`. A
  later read surface fails at the browser, loudly.
- **Causes.** A method list written for machine callers that never exercised browser CORS.
- **Affected.** `api-gateway` (CAP-02), any future browser read surface.
- **Mitigation · type `defer`.** Widen it in the same change that adds the first browser read
  surface. Widening it speculatively now enlarges the edge's accepted surface for no delivered need.
- **ADR.** `ADR-021` §6.2 item 5 and `Q2`.
- **Who must act.** Platform architect, when the first browser read surface is designed. **No action
  now.**
- **Must verify.** That whoever designs that surface knows this, which is what this row is for.
- **Residual: Low.**

### `R-11` — A duplicate submission is accepted while a request is in flight · **[handled later by `DO1` implementation]**

- **What it is.** `DO1` targets zero duplicate login submissions accepted while a request is in
  progress.
- **Failure mode.** Repeated clicks or an Enter key held down send several concurrent sign-in
  requests.
- **Effects.** Multiple tokens minted for one intent, multiple refresh tokens created, a confusing
  race between responses. `IdentityService.SignInAsync` has no idempotency guard and does not need
  one — the fix belongs in the client.
- **Causes.** A submit control left enabled during an in-flight request.
- **Affected.** `Pacco.Web` Login screen (CAP-17), `identity-service` (CAP-01).
- **Mitigation · type `mitigate`.** Disable submission for the duration of the request and show a
  processing state.
- **ADR.** `ADR-021` §8 `N7`.
- **Who must act.** The `DO1` implementer.
- **Must verify.** Submit repeatedly against a slowed sign-in response and assert exactly one request
  leaves the browser.
- **Residual: Low** once verified.

---

## 3. Binding constraints

Constraints are not risks. They are properties of the platform that any design must work within.
They are listed here so a designer does not have to rediscover them.

| id | Constraint | Source | What it forbids |
|----|-----------|--------|-----------------|
| `C-01` | Authentication is enforced at the edge. Services behind it trust the caller context they are given and re-check only resource ownership | `ADR-006` §2 | A client that authenticates on its own, or a browser path that bypasses the gateway |
| `C-02` | The gateway configuration is a reviewed architectural artifact. A change to it changes the platform's public contract | `ADR-004` §2 obligation 1 | Treating the CORS origin change as an operations edit |
| `C-03` | Response aggregation and per-client shaping must not be added to the gateway configuration. A client needing either gets a backend-for-frontend service and its own record | `ADR-004` §2 obligation 3 | Stretching the edge to shape responses for the browser |
| `C-04` | Revocation is consulted only by the issuing service, and its routes are not exposed at the edge | `ADR-007` §2 obligation 4 | A logout that claims to invalidate a token |
| `C-05` | Repository per component, with independent per-repository release | `ADR-018` | A client bundled into a backend image or onto a backend's release path |
| `C-06` | Deployment is Docker Compose stacks and PM2 manifests only. No Kubernetes, Helm or Terraform exists | `ADR-017` | Designing a frontend deployment pipeline against a target that does not exist |
| `C-07` | The role vocabulary is the closed, lower-cased set `user` and `admin`, validated case-insensitively | `Identity.Core/Entities/Role.cs` | Any third role, and any role inferred from an identifier |
| `C-08` | Every .NET deployable is pinned to `.NET Core 3.1`, which is out of support | `ADR-020` | Assuming a browser client inherits a platform runtime baseline — it does not |
| `C-09` | There are currently no Dev, QA, Staging or Production frontend environments | `ADR-021` §5 rule 6, reviewer decision | Pre-defining environment origins, DNS names or deployment targets now |
| `C-10` | A reschedule is one atomic same-resource day move: one command, one aggregate, one write, with the new day taken before the old one is released | `ADR-024` Rules 1 and 2, `ASM-15` | A two-call release-then-reserve sequence, and any design that changes the assigned resource |
| `C-11` | Customer reschedules use one fixed standard priority that never expropriates an existing hold, and a later higher-priority hold may take the customer's day | `ASM-14`, `ASM-16` | A priority ladder, a per-customer priority, and any design that promises the day cannot be lost |
| `C-12` | The alternative-day query returns at most the next 14 eligible calendar days, ascending, days only | `ASM-19`, `ADR-024` Rule 6, `NFR-19` | An unbounded or open-ended availability scan, and intra-day slot granularity |
| `C-13` | The order's delivery date is the authoritative schedule. Any delivery-side copy is a replica and never overrides it | `ASM-10`, `ADR-026` Rule 2 | A delivery-side write that changes the schedule without an order event |
| `C-14` | Persistence changes are additive and nullable: fields added, never renamed or removed, enum ordinals append-only, and nothing backfills | `ADR-008`, `NFR-17`, `C-08` of the baseline | A required new field, a renamed field, and any design that assumes historical records carry it |
| `C-15` | A cross-service read is a point read by id and carries service identity. `deliveries-service` has no service-identity certificate today | `ADR-010`, `ADR-027` Rules 1 and 6 | A collection scan, a per-delivery fan-out, and enabling the revalidation path before the certificate exists |
| `C-16` | Write integrity is scoped to `Resource`, `Order` and `Delivery` for this increment, replicated per repository | `AD-4` option B, `ADR-028` Rules 1-7, `ADR-002` | A platform-wide sweep in this increment, and a shared library to carry the pattern |
| `C-17` | A rejected reschedule returns one of exactly five classes, keyed on a stable code, with no internal detail | `ADR-030` Rules 1-3, `NFR-14`, `ADR-023` | A generic failure, a client that parses prose, and an exception message reaching a caller |

---

## 4. Evidenced gaps

A gap is a missing input, not a risk to score. Each one blocks a decision somebody will have to make.

| id | Gap | Why it matters | Who must supply it | Needed |
|----|-----|----------------|--------------------|--------|
| `G-01` | **[ACTION NOW]** No repository has an owner — no `CODEOWNERS`, no contributing guide, no team metadata anywhere in the fourteen clones | Nothing can be approved. All twenty-one ADRs are stuck at `Proposed`, and `C-02`'s review obligation is unenforceable | Platform owner | Now — it blocks `ADR-021`'s approval |
| `G-02` | **[ACTION NOW]** No environment-to-gateway-configuration mapping is recorded | An exact CORS origin must land in the file the running gateway loads (`R-05`) | Platform owner | Now — before `DO1` is exercised end to end |
| `G-03` | **[ACTION NOW]** No availability target, latency budget or error-rate objective is documented for any service, for the gateway, or for the platform | A client cannot justify a timeout, a retry policy or a loading-state threshold against nothing. Whatever `DO1` picks becomes the de facto target | Platform owner | Now, if the client is to have a justified timeout. Otherwise the first value chosen becomes the standard by accident |
| `G-04` | **[ACTION NOW]** No frontend standard of any kind exists — state ownership, accessibility, error presentation, client logging and redaction, dependency and lockfile policy | The first client sets all of them by accident (`R-09`) | Platform architect | Now — cheapest while the first screens are being written |
| `G-05` | **[handled later by the design-system decision]** `STYLE_README.md` and `pacco-material-you.css` have no versioning, no distribution mechanism and no consumer contract | A second client would copy them or force a design-system decision under time pressure | Platform architect | Later — forced by a second consumer, not before (`ADR-021` FA6) |
| `G-06` | **[handled later by HLS]** The message path into `identity-service` has no evidenced producer: `Program.cs` calls `SubscribeCommand<SignUp>()`, but no gateway route publishes to the `identity` exchange and no service publishes `sign_up` | Pre-existing (`architecture-views.md` GAP-4). It does not affect `DO1` or `DO2`, whose sign-up path is not in scope, but it is the kind of thing a client team will trip over | Platform owner | Later |
| `G-07` | **[ACTION NOW]** `deliveries-service` accepts `OrderId` with no validating call, so a delivery can exist for an order that does not exist | `ADR-026` makes the consequence smaller — a delivery with no matching order never receives a replica and is therefore never listed — but the underlying acceptance is unchanged, and it is now upstream of a customer-facing read | Platform owner, to decide whether it is in scope for `DO1` | Now — the decision, not necessarily the fix |
| `G-08` | **[ACTION NOW]** Whether a vehicle id and an availability resource id actually hold the same value in a running environment is unverified | The catalog records the correspondence as an assumption explicitly not established in the repository. `ADR-025` removes the reschedule's dependence on it, but the existing `(vehicleId, deliveryDate)` correlation that advances orders still rests on it entirely | Platform owner — the only role that can observe a running environment | Now — before `DO2` ships |
| `G-09` | **[handled later by DevOps]** No service-identity client certificate exists for `deliveries-service`, and no record states where service certificates are issued or stored | `ADR-027`'s revalidation path fails closed without one, so `DO1`'s schedule-lost behaviour cannot be enabled. This is `ADR-027` `B1` | Platform owner, carried by `INF-3` | Before `DO1` reaches a shared environment |
| `G-10` | **[ACTION NOW]** No revalidation timeout value exists, and the platform has no availability or latency target to justify one against | Without a number, the client default is effectively unbounded and `NFR-20` cannot be assessed. This is the same absence `G-03` records, now with a specific decision waiting on it | Platform owner with the `DO1` implementer | Now — before the revalidation path is enabled |
| `G-11` | **[ACTION NOW]** No instruction-length bound and no redaction entry exist for the new free-text delivery-instructions field | `NFR-11` requires both. The existing unbounded `Notes` field written verbatim to console, file and Seq is the precedent being avoided, and avoiding it requires two concrete values nobody has chosen | Product owner with the `DO3` implementer | Now — before `DO3` persists any customer text |

---

## 5. Resolved, awaiting prune

**None.** This register was first authored on 2026-09-22 and no item has yet been resolved. The
first revision that closes an item records it here with its closing date and the stage or person that
closed it, and the row is deleted at the revision after that.

**What the 2026-10-03 revision did and did not close.** It opened `R-15`…`R-30`, `C-10`…`C-17` and
`G-07`…`G-11`. It closed nothing. Two things are worth stating plainly rather than leaving to
inference:

- **`R-01`…`R-14` are carried forward unchanged.** They belong to the browser-client work recorded by
  `ADR-021`…`ADR-023`, and nothing in `ADR-024`…`ADR-030` changes their exposure. Two of them are
  adjacent to this run's work and still were not re-scored: `R-09` and `G-04` (no frontend standard)
  are the same absence `R-17` now attaches a named requirement to, and `R-02` (the gateway returns
  downstream exception messages) sits behind `ADR-030` Rule 3's obligation that no internal detail
  crosses the boundary. Re-scoring them would have been a judgement about the 13652 design, which
  this run has no evidence to revisit.
- **All five open questions raised by work item 14830 arrived resolved**, so none was promoted into
  this register. `OQ-3` is resolved by `ASM-18` (the edge write mode is inherited, not chosen) and
  `OQ-5` by `ASM-20` (the reservation is revalidated at delivery read time, contract unchanged); the
  other three were resolved with Product and Architecture before this stage. The items above are
  risks and gaps found while designing against those answers — they are not disguised open questions.
