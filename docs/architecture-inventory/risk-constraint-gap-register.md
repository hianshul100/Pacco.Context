# Risk, Constraint & Gap Register

> **What this file is.** A single living queue for every risk, constraint, gap and unanswered
> question that the platform's architecture records could not close. It is shared across work items,
> not owned by one of them: entries are **opened** when a record cannot settle something, **resolved**
> when a named person or a later stage settles it, and **pruned** when they stop being true. An entry
> is never deleted to make the register look shorter — resolution is recorded with the answer that
> closed it, so a reader can see why a constraint stopped applying.
>
> **What it is not.** It is not a to-do list for one delivery, and it is not a substitute for the ADRs
> that reference it. Each entry states what is unknown and who must act. The reasoning behind an entry
> lives in the record that opened it.

| Field | Value |
|-------|-------|
| Owner | This register has no single owner, which is itself an open entry — see `GAP-13155-14` |
| First opened | 2026-09-19, work item 13155 |
| Sources | `docs/adr/` records `ADR-001` … `ADR-024`; `docs/architecture-inventory/` baselines, component internals and views; `intents/13155.md` |
| Scoring | Failure-mode scoring in §5 uses S, O and D on 1–10 with `RPN = S × O × D` |

---

## Table of contents

1. [How to read this register](#1-how-to-read-this-register)
2. [Open entries](#2-open-entries)
3. [Quality attributes carrying risk](#3-quality-attributes-carrying-risk)
4. [Load-bearing assumptions ledger](#4-load-bearing-assumptions-ledger)
5. [Architecture risk assessment](#5-architecture-risk-assessment)
6. [Unresolved items in plain language](#6-unresolved-items-in-plain-language)

---

## 1. How to read this register

**Entry ids.** An id is stable for the life of the entry and is never reused. Ids opened by a work
item carry that work item's number — `GAP-13155-03` was opened by work item 13155 — but the entry
itself is not scoped to that work item, and a later work item may resolve it. Ids without a work
item number were opened by an earlier inventory stage and are referenced by their original label.

**State.** Exactly one of:

| State | Meaning |
|-------|---------|
| `open` | Nobody has answered it. The entry names who must |
| `resolved` | Answered. The entry records the answer and what closed it |
| `accepted` | Understood, not going to be fixed, and the consequence is carried deliberately |
| `pruned` | No longer true. The entry records why it stopped being true |

**Urgency.** Exactly one of **[ACTION NOW]** or **[handled later by \<named stage\>]**. The rule for
choosing between them is not how soon the answer is needed — it is who owns the answer. **An entry
is [ACTION NOW] whenever it depends on a system, a team or a capability outside the owning work
item's control**, because no later stage of that work item can produce the answer by working harder.
An entry is deferred only when a named later stage genuinely can settle it from within its own scope.

**Traceability.** Every entry names the record that opened it, the requirement or open question it
came from, and what breaks if it is ignored. An entry that cannot name what breaks is not a gap — it
is a preference, and it does not belong here.

---

## 2. Open entries

### 2.1 Summary

| id | Title | Category | State | Urgency | Owner |
|----|-------|----------|-------|---------|-------|
| `GAP-13155-01` | Where the browser surface ships and how it is built | architecture | `resolved` | — | Settled by human gate `AD-1` |
| `GAP-13155-02` | The cross-origin preflight path has never been exercised | security / integration | `open` | **[ACTION NOW]** | Platform owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-03` | The concrete browser origin per environment is unnamed | security / operations | `open` | **[ACTION NOW]** | Product and Architecture jointly |
| `GAP-13155-04` | A script-readable token store with no compensating controls | security | `open` | **[ACTION NOW]** | Platform security owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-05` | Three implementations compare the same `role` claim differently | security | `open` | **[ACTION NOW]** | Platform security owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-06` | `Request-ID` is declared CORS-exposed and set by nothing | observability | `open` | **[handled later by the `hls` stage]** | `hls` stage |
| `GAP-13155-07` | Nothing governs the browser toolchain or its version | maintainability | `resolved` | — | Settled by `ADR-024` |
| `GAP-13155-08` | No `docs/standards/` exists, so eleven rule families are uncovered | governance | `open` | **[ACTION NOW]** | Architecture owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-09` | A platform frontend standard was deferred by auto-default | governance | `open` | **[handled later by the second browser surface's intake]** | Architecture owner |
| `GAP-13155-10` | What a token with no `role` claim at all shows the user | product | `open` | **[ACTION NOW]** | Product owner |
| `GAP-13155-11` | No test-coverage expectation exists to inherit | quality | `open` | **[ACTION NOW]** | QA owner |
| `GAP-13155-12` | Logout cannot terminate a session anywhere on the platform | security | `accepted` | **[ACTION NOW]** | Platform security owner |
| `GAP-13155-13` | No total request timeout is configured at the edge | performance | `open` | **[ACTION NOW]** | Platform owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-14` | No repository has a named owner | governance | `open` | **[ACTION NOW]** | Whoever commissions this platform's work |
| `GAP-13155-15` | Nothing deploys, and a twelfth artifact inherits that | operations | `open` | **[ACTION NOW]** | Platform owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-16` | `POST /identity/sign-up` accepts `"role": "admin"` from the body | security | `open` | **[ACTION NOW]** | Platform security owner (unassigned — `GAP-13155-14`) |
| `GAP-13155-17` | How a static artifact receives per-environment configuration | architecture | `open` | **[handled later by the `hls` stage]** | `hls` stage |
| `GAP-13155-18` | Which of the four gateway manifests is authoritative per environment | operations | `open` | **[ACTION NOW]** | Platform owner (unassigned — `GAP-13155-14`) |

Seven of the eighteen — `GAP-13155-12` through `GAP-13155-16`, plus `GAP-13155-18` and the
standards absence in `GAP-13155-08` — are **pre-existing platform conditions, not defects this work
item introduced**. They appear here because a browser surface is the first caller for which they
become load-bearing, and because a register that listed only newly-created risk would understate
what a reviewer has to act on.

### 2.2 Entry detail

---

**`GAP-13155-01` — Where the browser surface ships and how it is built and released**

| | |
|-|-|
| **State** | `resolved` |
| **Opened by** | `intents/13155.md` OQ-1; `ADR-021` |
| **What it was** | `Pacco.Web` is an empty repository with one tracked file. Three hosting options were live: a new standalone deployable, static assets inside an existing service image, or same-origin assets served by the gateway |
| **How it was answered** | Human decision gate `AD-1`, option A: a standalone independently deployable browser surface with its own build artifact, versioning and pipeline, reaching backends only through the existing edge, hosted in the existing `Pacco.Web` repository rather than a new one |
| **Recorded in** | `ADR-021`; `architecture-baseline.md` §2.2, §7.4, §9.5; `capability-baseline.md` CAP-17 |
| **Why it stays in the register** | Its resolution is the premise of `GAP-13155-03`, `GAP-13155-07`, `GAP-13155-15` and `GAP-13155-17`. Pruning it would leave four entries with no stated origin |

---

**`GAP-13155-02` — The cross-origin preflight path has never been exercised**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-023` §6.2 item 3, B3; `…/component-internals/api-gateway.md` B-1, Q-2 |
| **What it is** | No browser client has ever made a cross-origin call to this gateway. `OPTIONS` appears in no manifest, and `Ntrada.Extensions.Cors` — the component that would answer a preflight — has its source outside the workspace, so nothing here shows how it behaves |
| **Who must act** | The platform owner, once named (`GAP-13155-14`). This cannot be settled by reading the workspace. It requires running a credentialed cross-origin `POST` against a gateway with a named origin and observing the response |
| **What breaks if ignored** | The entire work item fails at its first call, with a browser-side CORS error and no server-side log line explaining it. Every other decision in this run assumes this path works |
| **Needed** | Before `DO1` is declared working, not before it is built |
| **Verification** | `must_verify`. Accepted configuration is not proven behaviour: the manifests accepting a named origin proves the YAML parsed, not that a preflight is answered |

---

**`GAP-13155-03` — The concrete browser origin per environment is unnamed**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `intents/13155.md` OQ-2; `ADR-023` B2, Q1; `…/component-internals/api-gateway.md` Q-8 |
| **What it is** | `ADR-023` fixes the mechanism — named origins replace `allowedOrigins: ['*']` in all four `ntrada*.yml` manifests — and fixes nothing about the values. No hostname is named for any environment |
| **Who must act** | Product and Architecture jointly. The answer depends on where the artifact is hosted, which `GAP-13155-15` has not settled either |
| **What breaks if ignored** | `DO3` has no content. The decision is complete and unexecutable, and because `DO1` depends on `DO3` for cross-origin reachability, `DO1` cannot be demonstrated either |
| **Needed** | Before the gateway configuration change is applied |
| **Note** | A later stage must not invent these values. Inventing an origin would mean inventing the hosting answer, which is exactly what `GAP-13155-15` records as unknown |

---

**`GAP-13155-04` — A script-readable token store with no compensating controls**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-022` §6.2, F2; `…/baselines/ui-inventory.md` §9 |
| **What it is** | `ADR-022` puts the access token in `sessionStorage`, which any script running on the page can read. This is the platform's first client-side persistence of a credential — the existing browser asset stores nothing at all. The controls that normally accompany such a store — a content-security policy and dependency scanning in the build — are specified nowhere, because no security standard for browser clients exists on this platform |
| **Who must act** | The platform security owner, once named (`GAP-13155-14`) |
| **What breaks if ignored** | A single cross-site scripting defect or one compromised build-time dependency yields a valid 60-minute bearer token that the gateway will accept and that nothing can revoke — see `GAP-13155-12` |
| **Needed** | Before the surface serves a real token in an environment reachable by real users |
| **Note** | The store choice itself is a settled human decision and is not reopened here. What is open is the absence of the controls around it |

---

**`GAP-13155-05` — Three implementations compare the same `role` claim differently**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-022` §6.2, F4; `intents/13155.md` ASM-2 |
| **What it is** | The same `role` claim is compared in three places with two different rules. The edge's claim gate uses exact-string equality on five admin routes. `IdentityContext.IsAdmin` compares case-insensitively. The browser guard in `ADR-022` compares case-insensitively. Nothing keeps the three in step, and no test covers the disagreement |
| **Who must act** | The platform security owner, once named (`GAP-13155-14`) |
| **What breaks if ignored** | A token whose `role` differs in case from `admin` produces a user who sees the admin landing page and is refused by the edge on every admin route — an incoherent experience that reads as a UI defect and is actually a platform inconsistency. The reverse ordering is worse in principle but is not reachable here, because this surface gates no data |
| **Needed** | Before a second surface puts real functionality behind the guard. For this work item the landing page carries only a welcome message, which bounds the damage but does not remove the divergence |

---

**`GAP-13155-06` — `Request-ID` is declared CORS-exposed and set by nothing**

| | |
|-|-|
| **State** | `open` · **[handled later by the `hls` stage]** |
| **Opened by** | `ADR-023` §6.2 item 5, F5, Q4; `intents/13155.md` ASM-12; `…/component-internals/api-gateway.md` §3.18 |
| **What it is** | The gateway's CORS policy lists `Request-ID`, `Resource-ID`, `Trace-ID` and `Total-Count` as exposed headers, and a recorded gap notes that no code in the workspace sets `Request-ID`. The requirement asks the surface to capture it for failed sign-ins |
| **Who must act** | The `hls` stage, which can settle it by searching for an emitter and recording the finding either way |
| **What breaks if ignored** | Either capture logic is built for a header that never arrives, or `NFR-15`'s traceability target is quietly unmet with nobody noticing. The surface is already required to degrade cleanly without it, so the failure is silent rather than loud — which is what makes it worth writing down |
| **Needed** | Before the sign-in failure path's diagnostics are specified |

---

**`GAP-13155-07` — Nothing governs the browser toolchain or its version**

| | |
|-|-|
| **State** | `resolved` — by `ADR-024`, 2026-09-19 |
| **Opened by** | `ADR-021` §7 `ARCHITECTURE_ALIGNMENT_EXCEPTION`, F4 |
| **What it is** | `ADR-020` requires every Pacco deployable to target one runtime version and forbids a service from selecting its own. The rule is written for .NET deployables and a browser bundle has no .NET runtime to pin, so the surface fell outside it — and nothing replaced it. No framework, bundler, package manager or version policy governed the twelfth deployable |
| **What resolved it** | `ADR-024` (`docs/adr/toolchain-currency-for-non-dotnet-deployables.md`). Every deployable is now covered by exactly one toolchain-currency record: `ADR-020` keeps those with a .NET runtime, unchanged, and `ADR-024` takes those without. Its rules are one declaration per repository, a committed lockfile the pipeline installs from, a version in vendor support on the day it is committed, and a review scheduled before that support ends |
| **What is still not settled** | Which toolchain and which version. `ADR-024` deliberately names none — that is `lld`'s choice, tracked as `ADR-021` Q3. `ADR-024` is `Proposed` and cannot be ratified until an owner exists (`GAP-13155-14`), and its B2 records that the eleven existing deployables would fail its support-window rule today |
| **Who must act** | No further architecture action. Ratification waits on `GAP-13155-14`; compliance is checked when the first manifest is committed |
| **What breaks if ignored** | *(as recorded when open)* The one governance rule that has kept eleven deployables on a single pinned toolchain stops applying at exactly the point a second toolchain enters the platform. The failure is slow: divergence accumulates without a signal, in the same way `C9` pinned the platform to a runtime that is now years out of support |
| **Needed** | Before the surface's dependency manifest is committed — met |
| **Alignment** | The `ARCHITECTURE_ALIGNMENT_EXCEPTION` in `ADR-021` §7 stands as written: `ADR-020` still does not reach the browser surface. What has changed is that the surface is no longer ungoverned — it is governed by a different record. `ADR-024` §4 states the split so that "outside `ADR-020`" can no longer be read as "outside any rule" |

---

**`GAP-13155-08` — No `docs/standards/` exists, so eleven rule families are uncovered**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-021` §7, `ADR-022` §7, `ADR-023` §7 |
| **What it is** | There is no `docs/standards/` directory in this repository and no equivalent catalogue anywhere in the fourteen clones. Eleven rule families were checked for covering content and none was found: API contracts; async and eventing; database; multi-tenancy; security and privacy; observability; feature flags and configuration; frontend state ownership, host/shell contract and accessibility; AI governance; clinical and regulated concerns; deployment and infrastructure |
| **Who must act** | The architecture owner, once named (`GAP-13155-14`) |
| **What breaks if ignored** | Every design decision that would normally cite a standard either cites nothing or silently imports an outside default. Three records in this work item had to establish targets by decision that a standard would ordinarily supply — WCAG 2.1 AA in `ADR-021`, token-store rules in `ADR-022`, and the origin policy in `ADR-023`. Each is now a precedent set by a single feature rather than a platform rule |
| **Needed** | Before a second browser surface, which will otherwise either copy these three records by accident or contradict them |
| **Note** | No family was defaulted from an outside convention without saying so. Where a target had to be set, the record says it was set by decision and names what it was derived from |

---

**`GAP-13155-09` — A platform frontend standard was deferred by auto-default**

| | |
|-|-|
| **State** | `open` · **[handled later by the second browser surface's intake]** |
| **Opened by** | Decision gate `AD-2`, option A, `chosen_by: auto_default`; `ADR-021` assumption A1, F5 |
| **What it is** | The choice to baseline this surface alone and defer a platform frontend standard until a second surface exists was reached by auto-default, not by a human. It is recorded as an assumption in `ADR-021` rather than as a decision, precisely because nobody chose it |
| **Who must act** | The architecture owner, at the point a second browser surface is proposed |
| **What breaks if ignored** | The second surface inherits three records written for one feature as if they were platform rules, or ignores them and diverges. Neither is a decision anybody made |
| **Needed** | Later, but the trigger must be watched: the deferral is only safe while exactly one browser surface exists |

---

**`GAP-13155-10` — What a token with no `role` claim at all shows the user**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `intents/13155.md` OQ-3; `ADR-022` Q1 |
| **What it is** | `NFR-5` and `ADR-022` treat an absent claim and an unrecognised claim value identically — both resolve to the normal-user experience. Whether the product wants them distinguished is a product judgement, not an architectural one |
| **Who must act** | The product owner |
| **What breaks if ignored** | Nothing at runtime — the safe reading is already implemented by the rule. What breaks is acceptance: a reviewer who expected a distinction finds none and reads it as a missing requirement |
| **Needed** | At spec sign-off |
| **Recommended answer** | Treat both identically as a normal user and show the plain welcome, never the admin view. It is the safe reading of the stated guardrail and needs no backend behaviour |

---

**`GAP-13155-11` — No test-coverage expectation exists to inherit**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `intents/13155.md` OQ-4 |
| **What it is** | The platform's test gate is vacuous: a failing suite does not break the build, the suite is not run in continuous integration, and the badge records confidence that is not earned. There is no house standard for a new surface to inherit |
| **Who must act** | The QA owner |
| **What breaks if ignored** | The twelfth pipeline reproduces the vacuous gate by default, and the first surface with a real user-facing failure mode ships with no enforced coverage at all |
| **Needed** | At spec sign-off |
| **Recommended answer** | Unit and component tests for the login form, validation, processing state and route guard, plus one happy-path end-to-end run per role — and a gate that actually fails the build |

---

**`GAP-13155-12` — Logout cannot terminate a session anywhere on the platform**

| | |
|-|-|
| **State** | `accepted` · **[ACTION NOW]** |
| **Opened by** | `ADR-007` §2 rule 4; `ADR-022` §6.2, B2; `NFR-7` |
| **What it is** | A revoked access token is still accepted by the gateway and by all eight domain services, because revocation is consulted only by the issuing service. No client-side decision can change this |
| **Why `accepted` and still **[ACTION NOW]**** | `accepted` describes this work item's disposition: `ADR-022` responds by making the logout copy honest rather than by attempting a fix, and that response is complete. **[ACTION NOW]** describes the platform condition, which is not this work item's to close and does not become less true by being accepted here |
| **Who must act** | The platform security owner. The fix is a platform change — consulting revocation at the boundary that enforces authentication — not a UI change |
| **What breaks if ignored** | A user who logs out on a shared machine leaves a token that remains valid for up to 60 minutes. For this surface the exposure is bounded because the landing page carries no functionality. For the next surface it will not be |

---

**`GAP-13155-13` — No total request timeout is configured at the edge**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `NFR-21`; `intents/13155.md` ASM-16; `ADR-022` rule 7 |
| **What it is** | No total request timeout is configured anywhere at the edge, and the gateway's HTTP retry policy can add several seconds on top of the downstream service's own timeouts. The upper bound on a sign-in call is unmeasured |
| **Who must act** | The platform owner, once named (`GAP-13155-14`), for the edge configuration. The client-side bound is settled by `ADR-022` and is not open |
| **What breaks if ignored** | The client-side timeout has to be sized against an unknown, so it is either set too low and aborts calls that would have succeeded, or set too high and leaves the user watching a processing state for longer than anyone intended |
| **Needed** | Before the client-side timeout value is fixed, which `ADR-022` Q3 delegates to the `lld` stage |

---

**`GAP-13155-14` — No repository has a named owner**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-021` B1, `ADR-022` B1, `ADR-023` B1, `ADR-024` B1 — all four independently |
| **What it is** | None of the fourteen repositories has a recorded owner, so no follow-up action in any of the four new records has an accountable person, and none of the four can leave `Proposed` |
| **Who must act** | Whoever commissions work on this platform. This is the one entry that cannot be delegated to any stage, because delegation is precisely what it lacks |
| **What breaks if ignored** | Ten of the eighteen entries in this register name an owner that does not exist — `GAP-13155-07` left that set when `ADR-024` resolved it. The register becomes a list of things nobody is accountable for, which is indistinguishable from not having written it |
| **Needed** | Before any of `ADR-021`, `ADR-022`, `ADR-023` or `ADR-024` is ratified |

---

**`GAP-13155-15` — Nothing deploys, and a twelfth artifact inherits that**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-018` §6; `ADR-021` B3; `architecture-baseline.md` §9.5 |
| **What it is** | The eleven existing pipelines build and push images and nothing deploys them. How an image reaches a running environment is unknown. `ADR-021` adds a twelfth release path producing static assets, and how those assets reach a served origin is unknown for the same reason |
| **Who must act** | The platform owner, once named (`GAP-13155-14`) |
| **What breaks if ignored** | "Independently deployable" stays an intention rather than a property, and `GAP-13155-03` cannot be answered because nobody can name an origin for a hosting arrangement that does not exist |
| **Needed** | Before any claim that this surface reaches an environment |

---

**`GAP-13155-16` — `POST /identity/sign-up` accepts `"role": "admin"` from the request body**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `ADR-022` B3; `…/component-internals/identity-service.md` §3.4 |
| **What it is** | The sign-up route is public and accepts a `role` value from the caller's request body, so an anonymous caller can create an administrator account |
| **Who must act** | The platform security owner, once named (`GAP-13155-14`). The fix belongs to `identity-service`, which this work item is explicitly forbidden to change |
| **What breaks if ignored** | The role-aware landing page faithfully shows the admin view to an account that escalated itself. The surface is not the defect and cannot mitigate it — it would be displaying a claim the platform legitimately issued |
| **Needed** | Before the platform is reachable by untrusted callers. It is listed here because a browser login screen makes sign-up reachable in a way a machine-caller-only edge did not |

---

**`GAP-13155-17` — How a static artifact receives per-environment configuration**

| | |
|-|-|
| **State** | `open` · **[handled later by the `hls` stage]** |
| **Opened by** | `ADR-021` obligation 2, assumption A2, Q1; `NFR-17` |
| **What it is** | Every other Pacco deployable is a process that reads configuration at start-up. A static bundle is not a process, and no mechanism for configuring one exists anywhere on the platform. `ADR-021` requires the gateway base URL to be per-environment configuration rather than compiled into the asset, and does not say how |
| **Who must act** | The `hls` stage, which can settle it within its own scope once the hosting arrangement is known |
| **What breaks if ignored** | The shipped asset ends up with a hard-coded absolute backend URL — which is exactly the anti-pattern `NFR-17` exists to prevent, and exactly what the existing browser asset does with `http://localhost:5005/pacco` |
| **Needed** | Before the build is specified |
| **Dependency** | Partly blocked by `GAP-13155-15`: some mechanisms are only available if the hosting arrangement supplies them |

---

**`GAP-13155-18` — Which of the four gateway manifests is authoritative per environment**

| | |
|-|-|
| **State** | `open` · **[ACTION NOW]** |
| **Opened by** | `architecture-views.md` §6 GAP-3, escalated by `ADR-023` assumption A2 |
| **What it is** | `NTRADA_CONFIG` selects one of `ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml` and `ntrada-async.docker.yml`. Which is live in which environment is unknown. This predates the work item |
| **Who must act** | The platform owner, once named (`GAP-13155-14`) |
| **What breaks if ignored** | `ADR-023` requires the origin change in all four manifests. If a reader assumes only one is live and edits that one, the platform gets a mode-dependent failure that appears only when the environment selects a different manifest — and no test on this platform would catch it |
| **Needed** | Before the gateway configuration change is applied |
| **Why it is escalated rather than inherited** | As a documentation gap it was tolerable. As soon as a configuration value must be identical across four files for a feature to work, an unknown about which file is live becomes a live failure mode |

---

## 3. Quality attributes carrying risk

`architecture-baseline.md` §13 records the disposition of all twenty-three quality attributes raised
by work item 13155. Three of them were marked `at_risk`, meaning they cannot be met from within the
owning work item because something outside it must change first. They are reproduced here so that a
reader of the register sees them without leaving it.

| NFR | Attribute | Why it is at risk | Register entry | Bounded by |
|-----|-----------|-------------------|----------------|------------|
| `NFR-7` | Logout must not claim server-side session termination | Revocation is consulted only by the issuing service, so no logout anywhere on this platform terminates anything | `GAP-13155-12` | `ADR-022` rule 4 makes the logout copy honest, which is the only control available to a client |
| `NFR-15` | Failed sign-in traceable through a correlation identifier | `Request-ID` is declared CORS-exposed and no code in the workspace sets it | `GAP-13155-06` | `ASM-12` and `ADR-023` require opportunistic capture, so the surface degrades cleanly rather than failing |
| `NFR-21` | The sign-in call bounded client-side | No total request timeout exists at the edge and the gateway retry policy can add several seconds, so the bound must be sized against an unmeasured figure | `GAP-13155-13` | `ADR-022` rule 7 sets an explicit client-side bound, which guarantees the processing state clears even if the value is imperfect |

The pattern across all three is worth naming: in each case the work item can make the surface
**honest about** a platform limitation but cannot **remove** it. That distinction is what keeps these
three out of the `needs_change` column in §13 — there is no change to this work item's artifacts
that would satisfy them.

---

## 4. Load-bearing assumptions ledger

An assumption is load-bearing when a decision in this run fails if the assumption is false. Each row
below names the component-internals section and the `file:function` citation that was read to prove
it, states whether a false assumption fails **loudly** (something visibly breaks) or **silently**
(something appears to work and does not), and carries a verdict.

**The "accepted is not usable" rule.** For any seam where this run writes a value that something
else must later read back, proving the write was accepted is not proof the read works. Every such
seam is verified on the read side or marked `BLOCKING`.

### 4.1 Ledger

| # | Assumption | Component-internals evidence | Citation | Failure mode | Verdict |
|---|-----------|------------------------------|----------|--------------|---------|
| L1 | `POST /identity/sign-in` is a declared public route through the edge and needs no new route | `…/component-internals/api-gateway.md` §6.1 route table | `ntrada.yml:263-269`, row `POST /identity/sign-in` marked **public** | Loud — the sign-in call returns an auth failure at the edge and nothing works | `proven` |
| L2 | The sign-in response carries the role and an expiry the client can read | `…/component-internals/identity-service.md` §3.11 | `AuthDto` = `(string AccessToken, string RefreshToken, string Role, long Expires)` | Loud — the landing page cannot resolve a role and the guard has nothing to read | `proven` |
| L3 | The `role` claim has a closed two-value vocabulary, lower-cased | `…/component-internals/identity-service.md` §3.4 | `Role` closed vocabulary `{user, admin}` | Silent — an unexpected value resolves to the normal-user experience, which is the intended safe default, so nothing signals the surprise | `proven`, with `GAP-13155-05` carried for the comparison divergence |
| L4 | The access token's absolute lifetime is 60 minutes | `…/component-internals/identity-service.md` §3.10; `architecture-baseline.md` §8.1 | `jwt.expiryMinutes: 60` | Loud — sessions end earlier or later than the stated target | `proven` |
| L5 | The edge exposes `Request-ID` to a cross-origin reader | `…/component-internals/api-gateway.md` §3.18 | `exposedHeaders: [Request-ID, Resource-ID, Trace-ID, Total-Count]` | Silent — the header is exposed and never populated, so capture logic finds nothing and reports nothing | `proven` that it is **declared**; **not proven** that anything sets it — `GAP-13155-06` |
| L6 | The current CORS policy is `allowedOrigins: ['*']` with `allowCredentials: true` | `…/component-internals/api-gateway.md` §3.18 | The verbatim `extensions.cors` block | Loud — the premise of `ADR-023` would be wrong | `proven` |
| L7 | `allowedMethods` omits `get` | `…/component-internals/api-gateway.md` §3.18 | `allowedMethods: [post, put, delete]` | Silent for this scope, because no cross-origin `GET` is issued. Loud for the next surface that issues one | `proven` |
| L8 | Naming origins is the documented extension procedure for this component | `…/component-internals/api-gateway.md` §3.18 extension procedure, §7.7 item 2 | "Replace `allowedOrigins: ['*']` with the actual client origins… and add `get` to `allowedMethods`" | Loud — if the extension does not honour a named list, the feature fails at its first call | **`BLOCKING`** — the procedure is documented, the behaviour is unverified. `Ntrada.Extensions.Cors` source is absent (`…/component-internals/api-gateway.md` B-1). See `GAP-13155-02` |
| L9 | Gateway configuration is read at process start and is not hot-reloaded | `…/component-internals/api-gateway.md` §3; `architecture-baseline.md` §4.3 | No code path modifies routes; `NTRADA_CONFIG` read at start-up | Loud — an operator expecting a live edit sees no change and concludes the configuration is wrong | `proven` |
| L10 | Nothing on the platform stores anything client-side today | `…/baselines/ui-inventory.md` §9 | "**Token storage: None.** No `localStorage`, `sessionStorage`, `document.cookie`, or in-memory persistence beyond the live DOM input value" | Silent — if a store already existed, `ADR-022`'s "first client-side persistence" framing would be wrong without anything failing | `proven` |
| L11 | The existing browser asset is not independently deployable | `…/baselines/ui-inventory.md` §11.2 | "**Independently deployable: No.** There is no separate build, no separate artifact, no separate image, no separate pipeline stage, and no separate version" | Loud — `ADR-021`'s rejection of the operations-service host rests on it | `proven` |
| L12 | Revocation is not honoured at the gateway or at the domain services | `ADR-007` §2 rule 4 | "a revoked token is still accepted by the gateway and by all eight domain services, because only the issuing service checks the revocation store" | Silent and severe — a logout that appears to work and does not is the worst possible shape for this failure | `proven` |
| L13 | Authentication is enforced at the edge, and in-service authorization fails open | `ADR-006` §2, §4.2.1 | 37 routes require auth, 5 gated on an admin role claim; the ownership guard passes for unauthenticated callers | Loud for the edge, silent for the service-side guard | `proven` |
| L14 | A static browser artifact can be given per-environment configuration | No component-internals section covers this — no static artifact exists on the platform | — | Loud at build time, or silent if someone compiles the URL in and nobody notices until a second environment | **`BLOCKING`** — no mechanism is evidenced anywhere. `ADR-021` A2 states it as an assumption. See `GAP-13155-17` |

Twelve of the fourteen are `proven`. The two that are not — L8 and L14 — are the two that describe
behaviour of things **not in this workspace**: an Ntrada extension whose source is elsewhere, and a
hosting arrangement that does not exist. Neither can be proven by reading harder.

### 4.2 Cross-outcome contract edges

`intents/13155.md` declares three contract edges between delivery outcomes. Each is resolved below
to a named producer, a named consumer, an interaction mechanism and the owner of that mechanism. An
edge is only resolved when the consumer is provably invoked at runtime.

| Edge | Producer | Consumer | Mechanism | Mechanism owner | Consumer invoked at runtime | Verdict |
|------|----------|----------|-----------|-----------------|------------------------------|---------|
| `DO1` produces-for `DO2` — the authenticated session token and resolved role claim held client-side | The login screen in the `Pacco.Web` browser surface | The route guard, the session-expiry check and the logout action, all in the same browser surface | `sessionStorage`, one tab-scoped key, written on sign-in and read on every protected-route entry | `Pacco.Web` — both ends of the edge are inside one deployable | Yes. The guard runs on landing-page entry, which is the only way to reach the landing page. No network hop and no second process is involved | `resolved` |
| `DO1` produces-for `DO3` — the deployed UI browser origin per environment | The hosting arrangement that serves the `Pacco.Web` build artifact | The `allowedOrigins` list in the four `ntrada*.yml` manifests | A configuration value applied by a human and taking effect on a gateway restart. Not a runtime call | `api-gateway` for the manifests; the platform owner for the values | Not applicable — this edge is a build-and-release-time value, not a runtime invocation. It is consumed when an operator edits the manifests | `resolved` as a mechanism, **unexecutable** — the value is unnamed. See `GAP-13155-03` and `GAP-13155-15` |
| `DO3` produces-for `DO1` — edge cross-origin reachability for the named UI origin | `api-gateway`, via `Ntrada.Extensions.Cors` | The browser, which enforces the response headers before handing the response to the surface | CORS response headers on the preflight and on the actual `POST` | `api-gateway` — the extension is referenced at `Pacco.APIGateway.csproj:15` | **Unverified.** The consumer is the browser's own enforcement, which is certain to run. What is unverified is whether the producer emits headers it will accept | **`BLOCKING`** · `must_verify`. See `GAP-13155-02` |

**Why the third edge is blocking and the second is not.** The second edge is unexecutable because a
human has not chosen a value — the mechanism is understood and will work once the value exists. The
third edge is blocking because the mechanism itself is unproven: naming an origin is the documented
procedure, but no browser has ever exercised it against this gateway and the code that would answer
is not in the workspace. A value can be supplied by a decision. A behaviour has to be observed.

**This is the "accepted is not usable" rule applied concretely.** Writing named origins into four
manifests and watching the gateway start without error proves the YAML was accepted. It does not
prove a preflight is answered, and the two outcomes are distinguishable only by making a real
cross-origin call. That call is `must_verify` before `DO1` is declared working.

---

## 5. Architecture risk assessment

A failure-mode analysis of the design authored by this stage — `ADR-021`, `ADR-022`, `ADR-023` and
`ADR-024`, and the baseline edits that accompany them. It assesses the **design**, not a running system, so
every occurrence figure is a judgement about how the design will behave rather than an observation
of how it has behaved.

**Scoring.** Each of severity (**S**), occurrence (**O**) and detection (**D**) is scored 1–10, and
`RPN = S × O × D`. Severity is the impact if the failure happens. Occurrence is how likely the
failure mode is to arise. **Detection is scored so that a high number is bad**: `D = 1` means the
failure is immediate and unmissable, `D = 10` means it can persist indefinitely with nobody noticing.
That convention is what pushes silent failures up the ranking, which is the behaviour wanted — a
loud failure that stops the feature is easier to manage than a quiet one that does not.

### 5.1 Ranked summary

| id | Title | Category | S | O | D | RPN | Mitigation type | Owner stage | Residual | `must_verify` |
|----|-------|----------|---|---|---|-----|-----------------|-------------|----------|---------------|
| `R4` | A logged-out token stays valid | security | 7 | 10 | 9 | **630** | `accept` | architecture | high | no |
| `R2` | The origin is named in fewer than four manifests | operations | 7 | 6 | 8 | **336** | `process` | devops | medium | yes |
| `R3` | The stored token is exfiltrated from the browser | security | 9 | 4 | 9 | **324** | `architecture_change` | lld | high | no |
| `R9` | The browser toolchain drifts with no governing rule | maintainability | 5 | 7 | 8 | **280** | `adr` | architecture | low (`ADR-024`) | no |
| `R10` | Failed sign-ins carry no correlation identifier | observability | 4 | 9 | 7 | **252** | `infra` | hls | medium | yes |
| `R13` | An anonymous caller creates an admin account | security | 9 | 3 | 8 | **216** | `architecture_change` | architecture | high | no |
| `R15` | Three feature records become a de-facto platform standard | governance | 4 | 6 | 9 | **216** | `adr` | architecture | medium | no |
| `R6` | The gateway base URL is compiled into the shipped asset | portability | 6 | 6 | 4 | **144** | `architecture_change` | hls | low | yes |
| `R11` | The client-side timeout is mis-sized against an unmeasured bound | performance | 4 | 6 | 5 | **120** | `infra` | lld | medium | yes |
| `R1` | The edge does not answer the cross-origin preflight | integration | 9 | 5 | 2 | **90** | `infra` | devops | high | yes |
| `R5` | The guard and the edge disagree about the same role value | security | 5 | 3 | 6 | **90** | `architecture_change` | architecture | medium | no |
| `R7` | A legitimate origin is refused because only one was named | operations | 6 | 4 | 3 | **72** | `process` | devops | low | yes |
| `R8` | An origin change takes the whole edge down for a restart | operations | 7 | 5 | 2 | **70** | `process` | devops | low | no |
| `R12` | Tab-scoped sessions are read as a defect | usability | 3 | 7 | 3 | **63** | `process` | hls | low | no |
| `R14` | The artifact is built and never reaches an environment | operations | 6 | 8 | 1 | **48** | `infra` | devops | high | no |

**Reading the ranking.** The top of this table is not where the design is weakest — it is where the
platform is. `R4`, `R13` and `R14` are pre-existing conditions that a browser surface inherits;
none is caused by a decision in this run, and none can be fixed by one. The highest-ranked risk the
design itself creates is `R3`, and it ranks where it does because of `D = 9`: a token read out of
`sessionStorage` leaves no trace anywhere on this platform.

**On `R9`.** `R9` is the one entry in this table whose `mitigation_type` of `adr` was left
unfulfilled when the table was first written, on the reasoning that the governing rule belonged to
a platform owner who does not yet exist. That reasoning does not survive the threshold: an `adr`
mitigation owned by `architecture` at RPN 280 has to be answered by a record, not deferred to one.
`ADR-024` is that record. Its residual is `low` rather than nil because the record is `Proposed` and
binds a repository that holds no manifest yet — see §5.2. The S, O and D figures are left at their
original values throughout this table: they score the failure mode, not the state of its mitigation.

### 5.2 Detail

**`R4` — A logged-out token stays valid** · `security` · S 7 · O 10 · D 9 · **RPN 630**

- **Failure mode:** the user chooses logout, the client discards the token, and the token continues
  to be accepted by the gateway and by all eight domain services until its natural expiry.
- **Effects:** a session that the user believes ended has not ended. On a shared machine, anyone who
  captured the token before logout has up to 60 minutes of authenticated access.
- **Causes:** revocation is consulted only by the issuing service (`ADR-007` §2 rule 4). No
  client-side action can change that, and the token-revocation route is not reachable through the
  edge in any of the four manifests.
- **Occurrence is 10** because this is not a probabilistic failure: it happens on every logout.
- **Mitigation:** `ADR-022` rule 4 requires the logout copy to claim only what is true — the token
  is discarded from this browser — and forbids any claim of server-side termination.
- **`mitigation_type`:** `accept`. The mitigation makes the system honest, not correct.
- **`adr_recommendation`:** none new. `ADR-007` already states the rule and owns the fix; this run
  adds no record that would duplicate it.
- **`owner_stage`:** `architecture` — a platform change, not a feature change.
- **Residual risk:** high, and bounded only by this surface carrying no business functionality.
- **Affected:** `ADR-022`, `NFR-7`, `GAP-13155-12`, `DO2`.

---

**`R2` — The origin is named in fewer than four manifests** · `operations` · S 7 · O 6 · D 8 · **RPN 336**

- **Failure mode:** an operator edits `ntrada.yml` and `ntrada.docker.yml`, misses the two async
  manifests, and the gateway refuses the browser origin whenever `NTRADA_CONFIG` selects one of them.
- **Effects:** the login surface works in one environment and fails in another with an identical
  build, producing a defect report against the UI for a gateway configuration cause.
- **Causes:** the CORS policy is duplicated across four files by construction (`ADR-004` §4.2 already
  carries that duplication as a cost), and which manifest is live per environment is unknown
  (`GAP-13155-18`).
- **Detection is 8** because nothing on this platform compares the four manifests, no test covers
  them, and the symptom appears only in the environment that selected the unedited file.
- **Mitigation:** `ADR-023` §4 requires all four to change together and names them explicitly.
  The durable fix is a check that the four CORS blocks are identical.
- **`mitigation_type`:** `process`.
- **`adr_recommendation`:** none new — `ADR-023` covers it. A check belongs in the pipeline, which
  is `GAP-13155-15`'s territory.
- **`owner_stage`:** `devops`.
- **Residual risk:** medium until a check exists.
- **`must_verify`:** yes — confirm all four manifests carry the identical origin list after the edit.
- **Affected:** `ADR-023`, `ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`,
  `ntrada-async.docker.yml`, `GAP-13155-18`, `DO3`.

---

**`R3` — The stored token is exfiltrated from the browser** · `security` · S 9 · O 4 · D 9 · **RPN 324**

- **Failure mode:** script running on the page — injected, or shipped by a compromised build-time
  dependency — reads the access token out of `sessionStorage` and sends it elsewhere.
- **Effects:** a valid bearer token in an attacker's hands, accepted by the gateway for up to 60
  minutes, and unrevocable for the reason `R4` describes.
- **Causes:** `sessionStorage` is script-readable by design. No content-security policy is specified,
  no dependency scanning is specified, and no platform standard supplies either (`GAP-13155-04`,
  `GAP-13155-08`).
- **Detection is 9** because a read from client storage produces no server-side signal of any kind
  and this platform has no client-side telemetry.
- **Mitigation:** `ADR-022` rule 2 confines the token to one store and the `Authorization` header,
  which minimises the surface but does not protect the store. The real mitigation is a
  content-security policy and dependency scanning in the `Pacco.Web` pipeline — `ADR-022` F2.
- **`mitigation_type`:** `architecture_change`.
- **`adr_recommendation`:** a record establishing browser-client security controls, authored when
  `GAP-13155-08` is addressed. Not authored here: writing it now would set a platform-wide security
  rule from one feature's viewpoint, which is exactly what `GAP-13155-09` warns against.
- **`owner_stage`:** `lld` for the controls; `architecture` for the standard.
- **Residual risk:** high until compensating controls exist.
- **Affected:** `ADR-022`, `NFR-22`, `NFR-10`, `GAP-13155-04`, `DO1`, `DO2`.

---

**`R9` — The browser toolchain drifts with no governing rule** · `maintainability` · S 5 · O 7 · D 8 · **RPN 280**

- **Failure mode:** the surface's framework, bundler and dependencies age without any rule requiring
  them to be current, in the same way the platform's .NET runtime did.
- **Effects:** an unsupported toolchain, unpatchable dependencies, and a build nobody can reproduce.
- **Causes:** `ADR-020` pins one runtime for every .NET deployable and does not reach a browser
  bundle. Nothing replaces it (`GAP-13155-07`).
- **Detection is 8** because drift has no event — the platform already demonstrates this, having run
  on a runtime that reached end of support in December 2022 (`architecture-baseline.md` §11.2 C9).
- **Mitigation:** `ADR-021` obligation 3 requires a committed dependency manifest, build and lint
  baseline, which makes the toolchain visible. `ADR-024` makes it governed: every deployable is
  covered by exactly one toolchain-currency record, `ADR-020` keeps the .NET ones and `ADR-024`
  takes the rest, with one declaration per repository, a committed lockfile the pipeline installs
  from, a version in vendor support on the day it is committed, and a review scheduled before that
  support ends. `ADR-024` names no version, so the choice of toolchain stays with `lld`.
- **`mitigation_type`:** `adr`.
- **`adr_recommendation`:** **Answered.** The recommendation offered two courses — extend `ADR-020`,
  or author a companion record. The second was taken: `ADR-024` leaves `ADR-020`'s scope untouched
  (it is a current-state record written to be superseded when the runtime moves) and supplies the
  rule for deployables it does not reach. What is *not* decided here is which toolchain or which
  version, because that is not a rule this stage owns.
- **`owner_stage`:** `architecture`.
- **Residual risk:** low. The rule now exists and the drift has a scheduled event. Two things keep it
  from being nil: `ADR-024` is `Proposed` and cannot be ratified until an owner is named
  (`GAP-13155-14`), and it binds a repository that holds no manifest yet, so nothing enforces it
  until the first one is committed.
- **Affected:** `ADR-021`, `ADR-020`, `ADR-024`, `NFR-23`, `GAP-13155-07`.

---

**`R10` — Failed sign-ins carry no correlation identifier** · `observability` · S 4 · O 9 · D 7 · **RPN 252**

- **Failure mode:** the surface captures `Request-ID` on a failed sign-in and finds no header,
  because nothing on the platform emits one.
- **Effects:** a failed sign-in cannot be correlated to anything server-side, so the generic
  non-disclosing error message — which is required — becomes undiagnosable rather than merely opaque.
- **Causes:** `Request-ID` is declared in `exposedHeaders` and set by no code in the workspace.
- **Occurrence is 9** because on current evidence the header is absent from every response.
- **Mitigation:** `ASM-12` and `ADR-023` require opportunistic capture, so the surface degrades
  cleanly. That controls the symptom and not the cause.
- **`mitigation_type`:** `infra` — something must emit the header.
- **`adr_recommendation`:** none. This is a defect to fix or a fact to record, not a decision.
- **`owner_stage`:** `hls`, which can establish whether an emitter exists.
- **Residual risk:** medium.
- **`must_verify`:** yes — search for an emitter before building capture logic.
- **Affected:** `ADR-023`, `NFR-15`, `GAP-13155-06`, `DO1`.

---

**`R13` — An anonymous caller creates an admin account** · `security` · S 9 · O 3 · D 8 · **RPN 216**

- **Failure mode:** a caller posts `"role": "admin"` to the public sign-up route and receives an
  administrator token, which this surface then honours by showing the admin landing page.
- **Effects:** privilege escalation with no authentication required. The surface is not the defect
  and cannot mitigate it — it would be displaying a claim the platform legitimately issued.
- **Causes:** the sign-up command accepts a role from the request body
  (`…/component-internals/identity-service.md` §3.4).
- **Detection is 8** because the resulting account is indistinguishable from a legitimate one.
- **Mitigation:** none available to this work item, which is forbidden to change `identity-service`.
- **`mitigation_type`:** `architecture_change`.
- **`adr_recommendation`:** none authored here. The fix belongs to `identity-service` and to whoever
  owns `ADR-006`'s enforcement boundary.
- **`owner_stage`:** `architecture`.
- **Residual risk:** high.
- **Affected:** `ADR-022` B3, `GAP-13155-16`, `CAP-01`.

---

**`R15` — Three feature records become a de-facto platform standard** · `governance` · S 4 · O 6 · D 9 · **RPN 216**

- **Failure mode:** the second browser surface copies `ADR-021`, `ADR-022` and `ADR-023` as if they
  were platform rules — or contradicts them — without anybody deciding which.
- **Effects:** a platform frontend standard that nobody authored, assembled by precedent from one
  feature's constraints.
- **Causes:** gate `AD-2` deferred the standard by `auto_default` rather than by a human choice, and
  a deferral with no trigger is indistinguishable from an omission.
- **Detection is 9** because the failure is visible only in retrospect, at the point two surfaces
  already disagree.
- **Mitigation:** `ADR-021` records the deferral as assumption A1 rather than as a decision, and F5
  names the trigger — the second surface's intake.
- **`mitigation_type`:** `adr`.
- **`adr_recommendation`:** a platform frontend standard, authored at that trigger. Authoring it now
  would violate the same principle: one surface is not enough evidence to generalise from.
- **`owner_stage`:** `architecture`.
- **Residual risk:** medium, and entirely dependent on the trigger being watched.
- **Affected:** `ADR-021`, `GAP-13155-09`, `GAP-13155-08`.

---

**`R6` — The gateway base URL is compiled into the shipped asset** · `portability` · S 6 · O 6 · D 4 · **RPN 144**

- **Failure mode:** with no mechanism for configuring a static artifact, the build hard-codes an
  absolute backend URL.
- **Effects:** one artifact per environment instead of one artifact promoted across environments,
  which removes most of what `ADR-021` obligation 1 was for.
- **Causes:** every other deployable is a process that reads configuration at start-up. A static
  bundle is not, and no precedent exists (`GAP-13155-17`, ledger row L14).
- **Mitigation:** `ADR-021` obligation 2 forbids it and `NFR-17` targets zero hard-coded backend
  URLs. The mechanism is delegated to the `hls` stage.
- **`mitigation_type`:** `architecture_change`.
- **`adr_recommendation`:** none new — the obligation exists; the mechanism is a design question.
- **`owner_stage`:** `hls`.
- **Residual risk:** low, because the constraint is explicit and the precedent to avoid is named:
  the existing browser asset's hard-coded `http://localhost:5005/pacco`.
- **`must_verify`:** yes — confirm the shipped artifact contains no absolute backend URL.
- **Affected:** `ADR-021`, `NFR-17`, `GAP-13155-17`, `DO1`.

---

**`R11` — The client-side timeout is mis-sized against an unmeasured bound** · `performance` · S 4 · O 6 · D 5 · **RPN 120**

- **Failure mode:** the timeout is set below the edge's real worst case and aborts sign-in calls that
  would have succeeded, or far above it and leaves the user watching a processing state.
- **Effects:** intermittent sign-in failures attributed to the wrong component, or a poor experience
  on a slow path.
- **Causes:** no total request timeout is configured at the edge and the gateway's retry policy can
  add several seconds on top of the downstream's own timeouts. The figure to size against is unknown
  (`GAP-13155-13`).
- **Mitigation:** `ADR-022` rule 7 requires an explicit bound, which guarantees the processing state
  always clears whatever value is chosen. The value itself is delegated to the `lld` stage.
- **`mitigation_type`:** `infra` — the durable fix is an edge timeout.
- **`adr_recommendation`:** none new.
- **`owner_stage`:** `lld` for the value; `devops` for the edge configuration.
- **Residual risk:** medium.
- **`must_verify`:** yes — measure the edge's worst-case sign-in latency before fixing the value.
- **Affected:** `ADR-022`, `NFR-21`, `NFR-12`, `GAP-13155-13`.

---

**`R1` — The edge does not answer the cross-origin preflight** · `integration` · S 9 · O 5 · D 2 · **RPN 90**

- **Failure mode:** the browser sends an `OPTIONS` preflight before the sign-in `POST` and
  `Ntrada.Extensions.Cors` does not answer it acceptably, so the actual request is never sent.
- **Effects:** the whole work item fails at its first call.
- **Causes:** no browser has ever called this edge cross-origin, `OPTIONS` appears in no manifest,
  and the extension's source is outside the workspace (`GAP-13155-02`, ledger row L8).
- **Detection is 2** — this is the loudest failure in the register. Nothing works at all, the browser
  console states the cause, and it is discovered on the first attempt.
- **Mitigation:** `ADR-023` rule 6 requires the preflight path to be proven before the surface is
  declared working, which converts an assumption into a test.
- **`mitigation_type`:** `infra`.
- **`adr_recommendation`:** none new.
- **`owner_stage`:** `devops`.
- **Residual risk:** high until exercised, then zero. It is either true or it is not.
- **`must_verify`:** yes — a credentialed cross-origin `POST` from a named origin against a running
  gateway. This is the register's single most important verification.
- **Affected:** `ADR-023`, `NFR-9`, `GAP-13155-02`, `DO1`, `DO3`.

---

**`R5` — The guard and the edge disagree about the same role value** · `security` · S 5 · O 3 · D 6 · **RPN 90**

- **Failure mode:** a `role` value differing in case from `admin` passes the browser guard's
  case-insensitive comparison and fails the edge's exact-string claim gate.
- **Effects:** a user who sees the admin landing page and is refused on every admin route.
- **Causes:** three implementations, two comparison rules, no test covering the disagreement
  (`GAP-13155-05`).
- **Occurrence is 3** because the issuing service lower-cases the value, so a mixed-case claim
  requires a token minted outside the normal path.
- **Mitigation:** `ADR-022` rule 3 fixes a single comparison and resolves every non-matching value,
  including an absent claim, to the normal-user experience — the safe direction.
- **`mitigation_type`:** `architecture_change` for the reconciliation, which `ADR-022` F4 owns.
- **`adr_recommendation`:** none new — a reconciliation, not a decision.
- **`owner_stage`:** `architecture`.
- **Residual risk:** medium, bounded here because the landing page gates no data.
- **Affected:** `ADR-022`, `NFR-5`, `ASM-2`, `GAP-13155-05`.

---

**`R7` — A legitimate origin is refused because only one was named** · `operations` · S 6 · O 4 · D 3 · **RPN 72**

- **Failure mode:** an environment serves the surface from more than one hostname and only one is on
  the list, so calls from the other are refused by the browser.
- **Effects:** the surface works at one address and fails at another with no server-side signal.
- **Causes:** `ADR-023` assumption A3 takes one origin per environment, following the work item's
  recommended default. Nobody has confirmed the hosting arrangement (`GAP-13155-03`).
- **Mitigation:** `allowedOrigins` is a list and takes more than one entry. The mitigation is to
  confirm the hostnames when `GAP-13155-03` is answered, not to design for it now.
- **`mitigation_type`:** `process`.
- **`adr_recommendation`:** none new.
- **`owner_stage`:** `devops`.
- **Residual risk:** low.
- **`must_verify`:** yes — confirm each environment serves from exactly one hostname.
- **Affected:** `ADR-023` A3, `GAP-13155-03`, `DO3`.

---

**`R8` — An origin change takes the whole edge down for a restart** · `operations` · S 7 · O 5 · D 2 · **RPN 70**

- **Failure mode:** naming a UI origin requires restarting the single north-south gateway, which
  carries all forty-one routes.
- **Effects:** a UI hosting change becomes a platform-wide operational event affecting every caller.
- **Causes:** Ntrada configuration is read at process start and is not hot-reloaded, and no code path
  modifies routes.
- **Detection is 2** — a gateway restart is not subtle.
- **Mitigation:** `ADR-023` §4 schedules the change as a restart rather than a live edit, which is
  the whole content of `NFR-19`. Scheduling it is the control.
- **`mitigation_type`:** `process`.
- **`adr_recommendation`:** none new.
- **`owner_stage`:** `devops`.
- **Residual risk:** low, and unchanged from the platform's existing operating model.
- **Affected:** `ADR-023`, `NFR-19`, `DO3`.

---

**`R12` — Tab-scoped sessions are read as a defect** · `usability` · S 3 · O 7 · D 3 · **RPN 63**

- **Failure mode:** a user opens the landing page in a second tab, `sessionStorage` is empty there,
  and the guard sends them to Login.
- **Effects:** a second sign-in that the user did not expect, reported as a bug.
- **Causes:** `sessionStorage` is tab-scoped by definition. The stated semantics — survives a
  refresh, ends with the tab — are a settled human decision.
- **Mitigation:** none technical, and none wanted: changing the store would reverse a human decision
  and would widen `R3`. The control is agreement — `ADR-022` Q4 asks the product to confirm it.
- **`mitigation_type`:** `process`.
- **`adr_recommendation`:** none.
- **`owner_stage`:** `hls`, at acceptance-criteria definition.
- **Residual risk:** low.
- **Affected:** `ADR-022` A3, `NFR-22`, `DO2`.

---

**`R14` — The artifact is built and never reaches an environment** · `operations` · S 6 · O 8 · D 1 · **RPN 48**

- **Failure mode:** the twelfth pipeline builds and versions a static artifact, and nothing deploys
  it, exactly as nothing deploys the eleven images.
- **Effects:** "independently deployable" remains an intention rather than a property.
- **Causes:** no CD stage exists anywhere in the workspace (`architecture-baseline.md` §9.5,
  `GAP-13155-15`).
- **Detection is 1** — the surface is simply not reachable. Nothing is ambiguous about it.
- **Mitigation:** none within this work item. `ADR-021` B3 records it as a blocker rather than
  claiming the surface reaches an environment.
- **`mitigation_type`:** `infra`.
- **`adr_recommendation`:** none authored here — a deployment decision for the whole platform, not
  for one artifact.
- **`owner_stage`:** `devops`.
- **Residual risk:** high, and identical to the residual risk carried by all eleven existing
  deployables.
- **Affected:** `ADR-021`, `ADR-018`, `GAP-13155-15`, `NFR-20`.

---

## 6. Unresolved items in plain language

Everything above, restated without jargon. One row per item that is unproven, risky, deferred,
`BLOCKING` or `must_verify`. Each says what it is, who must act, what breaks if it is ignored, and
whether it is needed now or later.

**The rule for "now" versus "later".** An item is **[ACTION NOW]** whenever it depends on a system,
a team or a capability outside this run's ownership — not because it is urgent, but because no later
stage of this work can produce the answer on its own. An item is deferred only when a named stage
can genuinely settle it from within its own scope. "Downstream" is not a name and is not used here.

### 6.1 Reviewer action required now

> **One item left this list on 2026-09-19.** "Nothing says which tools or versions the browser build
> may use" was item 9 and is now settled by `ADR-024`; the entry itself is not deleted — it is
> `GAP-13155-07` in §2.2, marked `resolved` with the record that closed it. The items after it are
> renumbered, so a citation of "item 13" written before that date means item 12 here.

| # | What it is | Who must act | What breaks if ignored | Tag |
|---|-----------|--------------|------------------------|-----|
| 1 | Nobody owns any of the fourteen repositories, so no follow-up in the four new records has an accountable person | Whoever commissions work on this platform | Ten entries in this register name owners who do not exist, and none of `ADR-021`, `ADR-022`, `ADR-023` or `ADR-024` can be ratified | **[ACTION NOW]** |
| 2 | No browser has ever made a cross-origin call to this gateway, and the code that would answer the preflight is not in the workspace | Platform owner, by running a credentialed cross-origin `POST` from a named origin | The feature fails at its very first call, and every other decision in this run assumes it does not | **[ACTION NOW]** |
| 3 | The actual web address the login screen will be served from has not been chosen, for any environment | Product and Architecture together | The gateway configuration change has no value to apply, so `DO3` cannot be executed and `DO1` cannot be demonstrated | **[ACTION NOW]** |
| 4 | Nothing on this platform deploys anything, so there is no arrangement to serve the login screen from | Platform owner | The artifact is built, versioned and unreachable — and the address in item 3 cannot be chosen because there is nowhere to host it | **[ACTION NOW]** |
| 5 | The token will sit in browser storage that any script on the page can read, with no content-security policy and no dependency scanning specified | Platform security owner | One scripting defect or one bad dependency yields a working token for up to an hour that nobody can revoke | **[ACTION NOW]** |
| 6 | Logging out does not end the session anywhere on this platform — the token keeps working until it expires | Platform security owner | A user logging out on a shared machine leaves a usable credential behind. The screen can only be honest about it | **[ACTION NOW]** |
| 7 | Anyone can create an administrator account by asking for one on the public sign-up route | Platform security owner | The login screen will faithfully show the admin area to an account that gave itself the role | **[ACTION NOW]** |
| 8 | Three different parts of the platform compare the same role value by two different rules | Platform security owner | A user can see the admin screen and be refused on every admin action, which reads as a UI bug and is not one | **[ACTION NOW]** |
| 9 | There is no standards directory at all, so eleven categories of rule that a design would normally cite do not exist | Architecture owner | Every choice either cites nothing or quietly imports an outside convention. Three targets in this run had to be set by decision because no standard supplied them | **[ACTION NOW]** |
| 10 | The gateway has no overall request timeout, so there is no measured figure to size the screen's own timeout against | Platform owner | The timeout is guessed — too short and it cancels calls that would have worked, too long and the user waits | **[ACTION NOW]** |
| 11 | Nobody knows which of the four gateway configuration files is actually used in each environment | Platform owner | The origin change must be made in all four. Editing only the one assumed live produces a failure that appears in one environment and not another | **[ACTION NOW]** |
| 12 | It has not been decided what a user whose token carries no role at all should see, as distinct from an unrecognised role | Product owner | Nothing breaks at runtime — the safe behaviour is already specified — but acceptance has an unanswered question in it | **[ACTION NOW]** |
| 13 | There is no testing expectation to inherit: a failing test suite does not break the build and the suite is not run automatically | QA owner | The new pipeline copies a gate that checks nothing, on the platform's first user-facing surface | **[ACTION NOW]** |
| 14 | It has not been confirmed that requiring a second sign-in in a second browser tab is acceptable | Product owner | A direct consequence of the agreed storage choice gets reported as a defect because nobody agreed to it | **[ACTION NOW]** |

### 6.2 Delegated to a named later stage — no action now

| # | What it is | Which stage settles it | What breaks if that stage skips it | Tag |
|---|-----------|------------------------|-------------------------------------|-----|
| 15 | How a set of static files receives a different backend address in each environment — every other deployable is a running process that can read configuration, and this one is not | `hls` | The address gets built into the files, which is the exact practice the requirement exists to prevent and the exact mistake the existing browser page already makes | **[handled later by the `hls` stage]** |
| 16 | Whether anything on this platform actually sets the correlation identifier the screen is asked to capture | `hls` | Either capture logic is written for something that never arrives, or the traceability target is quietly unmet | **[handled later by the `hls` stage]** |
| 17 | Whether session expiry is read from the token's own claim or from the value in the sign-in response | `hls` | Two places end up disagreeing about when the session ended | **[handled later by the `hls` stage]** |
| 18 | What number the screen's own request timeout is set to | `lld` | The processing state has no defined bound in practice, which is what the requirement was written to prevent | **[handled later by the `lld` stage]** |
| 19 | Which framework, bundler and component library the surface uses | `lld` | The toolchain baseline cannot be committed, and whatever is chosen becomes a precedent by default | **[handled later by the `lld` stage]** |
| 20 | Whether a cross-origin `GET` is ever added, and whether the method list is reviewed as a whole when it is | `lld`, at the point the first one is introduced | The next surface meets a preflight rejection with no server-side log line explaining it | **[handled later by the `lld` stage]** |
| 21 | Whether a platform-wide frontend standard is written, at the point a second browser surface is proposed | `architecture`, at the second surface's intake | Three records written for one feature become a platform standard nobody authored | **[handled later by the `architecture` stage]** |
| 22 | Whether a static bundle belongs in the Compose service list and the process manifests, or is a third kind of artifact neither describes | `devops` | A third kind of deployable is added to two inventories that already disagree with each other | **[handled later by the `devops` stage]** |

### 6.3 What a reviewer should take from this

Fourteen items need action before this design can be executed, and **only three of them are about
this feature** — items 3, 12 and 14. The other eleven are platform conditions that a browser surface
is simply the first thing to run into: no owners, no deployment, no standards, no test gate, a
revocation model that does not revoke, and a sign-up route that hands out administrator accounts.

That distribution is the single most useful thing this register says. The design authored here is
small and its own risk is modest. What it does is make a set of long-standing platform gaps
load-bearing for the first time, because a browser is the first caller that cannot route around any
of them.
