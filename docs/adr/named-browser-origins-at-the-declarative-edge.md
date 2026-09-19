# ADR-023: Named browser origins at the declarative edge, replacing the wildcard

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-19 |
| **ADR id** | `ADR-023` |
| **Backlog candidate** | — (no candidate exists; this record originates from work item 13155, `intents/13155.md`) |
| **Source** | `DO3` — Pacco Browser Origin Enablement at the Ntrada Edge; `DO1` — Pacco Common Login & Role-Aware Welcome Landing |
| **Originating NFRs** | `NFR-9`, `NFR-18` (decision `needs_change`); `NFR-19` (operational constraint honoured); `NFR-15` (`at_risk`) |
| **Category / Impact** | Security / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-004` (the declarative edge this changes, and whose Q3 this record answers), `ADR-006` (the access enforcement this must not weaken), `ADR-021` (the surface whose origin is named), `ADR-022` (the session established across this boundary) |
| **Known defect** | This record's starting point is a live defect: `allowCredentials: true` combined with `allowedOrigins: ['*']` is the combination the CORS specification forbids, and it is identical in all four gateway manifests (§1) |

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
4. [Decision](#4-decision)
5. [Options Considered](#5-options-considered)
6. [Consequences](#6-consequences)
7. [Compliance Considerations](#7-compliance-considerations)
8. [Non-Functional Requirements & Testing](#8-non-functional-requirements--testing)
9. [Relationship to the implementation pattern catalog](#9-relationship-to-the-implementation-pattern-catalog)
10. [Follow-Up Actions](#10-follow-up-actions)
11. [Evidence](#11-evidence)
12. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Context

`ADR-004` left one question open and tagged it for this stage: *"Should the edge remain purely
declarative if a browser client is built?"* A browser client is now being built (`ADR-021`), so the
question is live, and the edge's cross-origin policy is where it first bites. That record's own
proposed answer — "Keep the declarative edge and add a separate backend-for-frontend service if
aggregation is needed… do not extend the YAML dialect to cover it" — is the position this record
adopts, with the note that no aggregation is needed and so no backend-for-frontend is introduced.

1. ✅ **The policy is declared once and repeated four times.** `extensions.cors`
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:27-41`) is served by
   `Ntrada.Extensions.Cors` (`Pacco.APIGateway.csproj:15`) and is **static and identical in all four
   manifests** (`docs/architecture-inventory/component-internals/api-gateway.md` §3.18):

   ```yaml
   allowCredentials: true
   allowedOrigins: ['*']
   allowedMethods: [post, put, delete]
   allowedHeaders: ['*']
   exposedHeaders: [Request-ID, Resource-ID, Trace-ID, Total-Count]
   ```

2. ✅ **The wildcard-with-credentials combination is forbidden.** The component inventory states it
   directly: "`allowCredentials: true` combined with `allowedOrigins: ['*']` is the combination the
   CORS specification forbids: a compliant browser rejects a wildcard origin on a credentialed
   request." Depending on how the extension materialises the wildcard, this is *either* an accidental
   allow-any-origin-with-credentials policy *or* a policy that silently fails every credentialed
   cross-origin call — "both outcomes are defects and the resolution is the same: name real origins"
   (`…/component-internals/api-gateway.md` §3.18).
3. ✅ **Which of the two it is cannot be settled from this workspace.** `Ntrada 0.4.*` and its six
   extension packages are NuGet references with no source in any of the fourteen clones, so CORS
   wildcard handling is unverifiable here (`…/component-internals/api-gateway.md` B-1, Q-2).
4. ✅ **`get` is absent from the method allow-list.** Every read route on the platform is a `GET`.
   Simple `GET` requests are not preflighted so they generally still work, but "a `GET` that becomes
   non-simple (e.g. because a client adds a custom header) would be preflighted and **rejected**.
   This is a latent, hard-to-diagnose browser-only failure"
   (`…/component-internals/api-gateway.md` §3.18).
5. ✅ **The headers a browser needs to read are already exposed, but one of them is never set.**
   `exposedHeaders` carries `Request-ID`, `Resource-ID`, `Trace-ID` and `Total-Count`; the component
   inventory records that `Total-Count` is exposed even though no route in the gateway repository sets
   it, and `…/baselines/architecture-baseline.md` carries the wider gap that the correlation header
   is declared but never set by any code in the workspace
   (`…/component-internals/api-gateway.md` §3.18).
6. ✅ **The change takes effect only on a gateway restart.** No code path modifies routes and the
   configuration is not hot-reloaded, so the origin change is a scheduled restart in each
   environment, not a live edit (`NFR-19`; `ADR-004`).
7. ✅ **Nobody has named the origins.** The component inventory's own Q-8 asks "Is there a client
   application (`Pacco.Web`) whose origin should replace the CORS wildcard?" and answers that the
   repository is empty (`…/component-internals/api-gateway.md` Q-8). `OQ-2` in the work item asks the
   same question per environment and is unresolved.

### 1.1 What this record does *not* cover

It adds no route to any manifest, opens no new unauthenticated surface beyond the already-public
sign-in route, introduces no backend-for-frontend, and changes no identity-service or authorization
code (`ASM-6`). It does not change `allowCredentials`, `allowedHeaders` or `exposedHeaders` — §10 F4
carries the case for revisiting the first of those separately. It does not decide the concrete origin
*values*, which are `OQ-2` and belong to a human (see Q1).

## 2. Decision Drivers

| # | Driver | Source |
| --- | --- | --- |
| D1 | The UI's origin must be named explicitly; the current wildcard either allows any origin with credentials or silently fails credentialed cross-origin calls | `NFR-9`; `…/component-internals/api-gateway.md` §3.18 |
| D2 | Every HTTP method the UI issues cross-origin must appear in the method allow-list, which today omits `get` | `NFR-18` |
| D3 | The change must not weaken the edge's access enforcement — no new route, no new unauthenticated surface | `DO3`; `ADR-006` §2 |
| D4 | The edge must stay declarative; no gateway code and no client-shaping layer | `ADR-004` §2; `ADR-004` Q3 |
| D5 | The change is a scheduled gateway process restart per environment, not a live edit | `NFR-19`; `ASM-6` |
| D6 | The permitted scope is exactly one edge configuration change | `ASM-6`; work item 13155 input gate question 2, answered by a human |

## 3. Architecture Fit Evaluation

**This decision introduces no new component.** The extension point already exists, is already
declarative, is already owned by the gateway, and already holds precisely this concern. The fit
analysis below is recorded to show that the reuse conclusion was reached by evaluation rather than by
default.

| # | Existing component or extension point | Evaluated as | Outcome |
| --- | --- | --- | --- |
| 1 | `extensions.cors` in the four `ntrada*.yml` manifests | The declared cross-origin policy for the whole gateway, served by `Ntrada.Extensions.Cors` | **Reused. This is the correct and only home for the change.** Its documented extension procedure is exactly this operation: "Replace `allowedOrigins: ['*']` with the actual client origins… and add `get` to `allowedMethods`" (`…/component-internals/api-gateway.md` §3.18) |
| 2 | A new route or route group in the manifests | Would let the surface reach something it cannot reach today | Rejected. No new capability is needed — `POST /identity/sign-in` is already public and routed. A new route would widen the edge's surface for no gain and is forbidden by `ASM-6` |
| 3 | A reverse proxy or static host placed in front of both the UI and the gateway, collapsing them to one origin | Would remove the cross-origin problem entirely rather than configuring it | Rejected. It introduces an infrastructure component the platform does not have: there is no orchestration, no mesh and no infrastructure-as-code in any of the fourteen repositories (`ADR-006` §3 Alternative 2; `ADR-017`). It would also contradict the agreed scope, which assumes a cross-origin UI |
| 4 | A backend-for-frontend terminating the browser's requests same-origin | The route `ADR-004` leaves open for a future client | Rejected for the same reason as in `ADR-021` §3: there is no aggregation or shaping to do, and `ADR-004` §3 Alternative 1 already rejected a hand-written backend-for-frontend |
| 5 | Gateway code — a middleware or handler computing the allowed origin | Would allow per-request origin logic | Rejected. It ends the edge's declarative property, which is the whole of `ADR-004`. A static per-environment list needs no logic |

**Why no new boundary is required.** The concern — which browser origins may call this edge — is
already a first-class, named, declarative field in a component that already owns it. Creating
anything new here would be inventing a boundary where one exists.

## 4. Decision

**The gateway's `allowedOrigins` wildcard is replaced by an explicit list of the Pacco browser
surface's origins, one entry per environment, in all four `ntrada*.yml` manifests. The edge stays
declarative: no route is added, no gateway code is written, and no backend-for-frontend is
introduced. The change is applied as a scheduled gateway process restart in each environment.**

This answers `ADR-004`'s Q3 — *should the edge remain purely declarative if a browser client is
built?* — as **yes**. The arrival of a browser client changes one configuration value and nothing
about the edge's nature.

Six rules follow from the decision and are part of it:

1. 🎯 **No environment carries a wildcard origin.** Not local, not the Docker stack, not production.
   `['*']` is removed everywhere, not narrowed in some manifests and left in others.
2. 🎯 **One origin list per environment, and the four manifests stay in lockstep.** The policy is
   duplicated four times by construction (`ADR-004` §4.2); a change applied to fewer than four
   manifests produces an edge that admits the surface in one mode and refuses it in another.
3. 🎯 **The method allow-list must cover every verb the surface issues cross-origin before that verb
   is issued.** Today the list is `post`, `put`, `delete`; the login flow issues only `POST`, so it is
   covered. The first `GET` this surface or any later one needs requires `get` to be added first
   (`NFR-18`).
4. 🎯 **No new route and no new unauthenticated surface.** `POST /identity/sign-in` is already public;
   nothing else becomes public as a consequence of this change (`ADR-006` §2).
5. 🎯 **The change is scheduled as a restart, per environment, and treated as a change to the
   platform's single north-south edge.** Every one of the forty-one routes passes through the process
   being restarted (`NFR-19`).
6. 🎯 **The preflight path is proven before the surface is declared working.** `POST
   /identity/sign-in` with a JSON content type is a preflighted request, and no browser client has
   ever exercised this edge's preflight handling. See B2 — this is the single unproven mechanism the
   whole feature rests on.

## 5. Options Considered

| # | Option | Why it was rejected |
| --- | --- | --- |
| 1 | **An explicit per-environment origin list replacing the wildcard** | **Chosen.** It is the resolution the component inventory itself prescribes for a defect that is a defect under either reading of the wildcard's behaviour, and it is the only option inside the scope agreed at the work item's input gate (question 2: "CORS allowed-origin changes only — no new edge routes") |
| 2 | **Leave the wildcard and rely on it working** | Rejected because it is spec-forbidden with `allowCredentials: true` and because its actual behaviour is unverifiable from this workspace. If it echoes the request origin, every origin on the internet can make credentialed calls to the edge; if it emits a literal `*`, credentialed cross-origin calls fail. Neither is a state to build a login surface on |
| 3 | **Set `allowCredentials: false` and keep the wildcard** | Rejected as out of scope: `ASM-6` permits the allowed-origins change only. It would also be a weaker fix — it makes the combination legal without making the policy correct, leaving every origin able to call the edge with a bearer token in a header. Recorded as F4 for separate consideration, not as this decision |
| 4 | **Collapse UI and API onto one origin, removing CORS entirely** | Rejected. It is `OQ-1` option (c), already rejected at the `AD-1` gate, and it requires either a reverse proxy the platform has no infrastructure to run or the gateway serving static assets, which `ADR-021` §3 row 2 rejected |
| 5 | **Compute allowed origins in gateway code from a registry** | Rejected because it ends the declarative property that `ADR-004` records as the edge's defining characteristic, in exchange for flexibility a fixed per-environment list does not need |

## 6. Consequences

### 6.1 Positive

1. 🎯 The platform's most-cited configuration defect is closed. The wildcard-with-credentials
   combination stops being a question about an unreadable extension's behaviour and becomes a
   readable list.
2. 🎯 The edge stays declarative with a browser client attached, which resolves `ADR-004` Q3 in the
   direction that preserves that record rather than superseding it.
3. 🎯 An unnamed origin is refused by the browser, so the surface's reachability becomes an explicit,
   reviewable, per-environment statement instead of an accident.
4. 🎯 The change is inspectable in source review. A reviewer can see which origins the platform
   admits by reading four YAML fields, with no runtime observation required.

### 6.2 Negative

1. 🎯 **Four manifests change together, forever.** The policy is duplicated by construction, and
   `ADR-004` §4.2 already carries that duplication as a cost. Adding a second browser surface later
   means four more edits, and an edit applied to three of the four produces a mode-dependent failure
   that no test on the platform would catch.
2. 🎯 **Every origin change is a gateway restart.** The single north-south edge for all forty-one
   routes is restarted to admit a UI origin, so a UI hosting change becomes a platform-wide
   operational event.
3. ❓ **The preflight path has never been exercised.** No browser client has ever made a cross-origin
   call to this edge, the extension's source is absent from the workspace, and `OPTIONS` appears in
   no manifest. Whether `Ntrada.Extensions.Cors` answers the preflight for `POST /identity/sign-in`
   with a JSON content type is unproven, and if it does not, the whole feature fails at its first
   call. Carried as B3 and as `GAP-13155-02`.
4. 🎯 **`get` remains absent and stays a trap.** This scope needs no cross-origin `GET`, because the
   route guard reads the held token and issues no backend call (`ADR-022` §4 rule 5). The moment a
   later surface adds one — or adds a custom header to one — it meets a preflight rejection with no
   server-side log line to explain it (`…/component-internals/api-gateway.md` §3.18).
5. 🎯 **The correlation header the UI is asked to capture may never arrive.** `Request-ID` is CORS-
   exposed but recorded as never set by any code in the workspace, so `NFR-15`'s traceability target
   depends on a header whose emission is unproven. `ASM-12` already requires the surface to capture
   it opportunistically and not depend on it. Carried as `GAP-13155-06`.

### 6.3 Neutral

1. `allowCredentials`, `allowedHeaders` and `exposedHeaders` are unchanged. Naming the origins makes
   the `allowCredentials: true` value legal without making it necessary; F4 carries that separately.
2. No route's authentication or claim requirements change. The five administrator-gated routes stay
   administrator-gated, and the thirty-seven authenticated routes stay authenticated.
3. The asynchronous manifests receive the same origin list as the synchronous ones even though the
   login flow uses only the synchronous path, because rule 2 requires lockstep and a divergent
   manifest is worse than an unused entry.

## 7. Compliance Considerations

| Governing source | Rule as it applies here | How this decision conforms |
| --- | --- | --- |
| `docs/adr/declarative-configuration-driven-api-gateway.md` §2 — the edge is a YAML document rather than gateway code; obligation 3 forbids per-client shaping | The arrival of a browser client is the test of that rule | Conforms, and answers that record's Q3. The change is one configuration field in four documents; no code, no shaping, no aggregation |
| `docs/adr/edge-enforced-authentication-with-fail-open-authorization.md` §2 — authentication is enforced at the gateway, routes gated on claims | A CORS change must not become an authorization change | Conforms. Rule 4 forbids any new route or newly public surface. CORS governs which origins a browser will let talk to the edge; it changes no route's `auth` or `claims` value |
| `docs/architecture-inventory/component-internals/api-gateway.md` §3.18 extension procedure — "Replace `allowedOrigins: ['*']` with the actual client origins… and add `get` to `allowedMethods`" | The documented procedure for this exact change | Conforms on the first half. The second half is deferred to F3 with a stated reason — this scope issues no cross-origin `GET` — rather than silently omitted |
| CORS specification, on credentialed requests | A wildcard origin is not permitted on a credentialed request | Conforms once rule 1 is applied. The clause is quoted here at second hand through `…/component-internals/api-gateway.md` §3.18 rather than from the specification text, which is not available in this workspace; the inventory's wording is reproduced verbatim in §1 item 2 above |
| API contract standards, security standards | No `docs/standards/` directory exists in this repository; no API or security rule family is catalogued | No silent default is taken. Recorded as `GAP-13155-08` in `docs/architecture-inventory/risk-constraint-gap-register.md` |
| Deployment and infrastructure standards | Not catalogued; `ADR-017` records compose-and-process-manager deployment with no production orchestration | Rule 5 states the operational consequence explicitly rather than assuming a rolling restart the platform cannot perform |

## 8. Non-Functional Requirements & Testing

| NFR | Target | How this decision addresses it | How it is verified |
| --- | --- | --- | --- |
| `NFR-9` [security] | Explicit origin list, no wildcard | §4 rules 1 and 2 — the wildcard is removed from all four manifests in every environment | A search of all four `ntrada*.yml` files finds no `'*'` in `allowedOrigins`; a credentialed cross-origin call from the named origin succeeds and one from an unnamed origin is refused by the browser |
| `NFR-18` [portability_infra] | The method allow-list covers 100% of the UI's cross-origin verbs | §4 rule 3 — the verb set is checked against the list before the verb is issued | An enumeration of the surface's outbound calls is compared against `allowedMethods`; today the enumeration is `POST` alone and the list covers it |
| `NFR-19` [operational_readiness] | The origin change is scheduled as a gateway restart, not a live edit | §4 rule 5 | The change is applied, the gateway is restarted, and the new policy is observed in the `Access-Control-Allow-Origin` response header — confirming that the pre-restart configuration was not in effect |
| `NFR-15` [observability] | A correlation `Request-ID` captured for every failed sign-in where the edge emits one | Unchanged by this record; `Request-ID` is already exposed, and §6.2.5 records that it may never be set | Observe the response headers of a failed sign-in from the named origin and record whether `Request-ID` is present. A negative result is a finding, not a failure of the surface |
| `NFR-16` [portability_infra] | Zero direct-to-service calls from the browser | §4 rule 4 — no new route and no new surface; the edge remains the only reachable path | No service container port appears in any origin, base URL or call target in the surface |

**Testing obligations this decision creates.** None of these can be satisfied by a unit test; all
require a running gateway, and the platform's test gate would not catch any of them
(`OQ-4`):

1. **A preflight test against a running gateway** — `OPTIONS` for `POST /identity/sign-in` from the
   named origin with a JSON content type, asserting a successful preflight response carrying the
   named origin. This is rule 6 and it is the highest-value test in this work item, because it is the
   only one that proves the unverifiable extension behaves as assumed.
2. **A negative origin test** — the same call from an origin not on the list, asserting the browser
   refuses it.
3. **A four-manifest consistency check** — the origin list is identical across `ntrada.yml`,
   `ntrada.docker.yml`, `ntrada-async.yml` and `ntrada-async.docker.yml`. A configuration test, not a
   runtime one, and the only mechanical guard on rule 2.

## 9. Relationship to the implementation pattern catalog

| Pattern | Relationship |
| --- | --- |
| [`integration/declarative-configuration-driven-api-gateway.md`](../architecture-inventory/patterns/integration/declarative-configuration-driven-api-gateway.md) | **Confirmed under stress.** The pattern's central claim is that the edge is configuration rather than code. This decision is the first test of that claim against a client type the platform has never had, and it holds: one field, four files, no code |
| [`security/edge-enforced-authentication-with-identity-binding.md`](../architecture-inventory/patterns/security/edge-enforced-authentication-with-identity-binding.md) | **Unchanged.** No route's `auth` or `claims` value is touched, and no identity binding is added or removed |
| [`deployment/composable-per-concern-environment-stacks.md`](../architecture-inventory/patterns/deployment/composable-per-concern-environment-stacks.md) | **Strained.** The per-environment origin list is the first configuration value on the platform that must differ per environment *and* be identical across the four manifests within an environment. The pattern has no mechanism for that, which is why rule 2 is stated as a rule rather than assumed |

Every pattern in the catalog is `Candidate`, so none of these relationships constitutes approval.

## 10. Follow-Up Actions

| # | Action | Owner | Due | Blocks |
| --- | --- | --- | --- | --- |
| F1 | **Name the concrete origin for each environment (local, Docker, production).** This is `OQ-2` and it cannot be answered by this stage — it depends on where the artifact is actually hosted, which follow-up F1 in `ADR-021` decides | Platform owner with the platform security owner (both unassigned — see B1) | 2026-10-10 | `DO3` in full; `NFR-9`; every rule in §4 |
| F2 | Exercise the preflight path against a running gateway from a named origin and record the observed `Access-Control-Allow-Origin` value, settling `…/component-internals/api-gateway.md` Q-2 | `hls` stage, against a running Docker stack | 2026-10-10 | Rule 6; B2; `DO1` acceptance |
| F3 | Add `get` to `allowedMethods` in all four manifests at the point the first cross-origin `GET` is introduced, and not before | `lld` stage of the first outcome that needs a read call | At that outcome's design | `NFR-18` for future surfaces |
| F4 | Decide whether `allowCredentials: true` is needed at all, given that no route on the platform uses a cookie or a session and the surface sends a bearer token in a header | Platform security owner (unassigned — see B1) | 2026-11-14 | Not blocking; reduces the blast radius of a future origin-list mistake |
| F5 | Confirm whether any code sets the `Request-ID` response header, or record it as declared-but-never-emitted | `hls` stage | 2026-10-10 | `NFR-15`; `GAP-13155-06` |

## 11. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | `extensions.cors` declares `allowCredentials: true`, `allowedOrigins: ['*']`, `allowedMethods: [post, put, delete]`, `allowedHeaders: ['*']`, `exposedHeaders: [Request-ID, Resource-ID, Trace-ID, Total-Count]` | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:27-41`; `docs/architecture-inventory/component-internals/api-gateway.md` §3.18 |
| E2 | The policy is static and identical in all four manifests | ✅ | `…/component-internals/api-gateway.md` §3.18 |
| E3 | The policy is served by `Ntrada.Extensions.Cors` | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Pacco.APIGateway.csproj:15` |
| E4 | Wildcard-with-credentials is forbidden by the CORS specification, and either reading of the extension's behaviour is a defect with the same resolution | ✅ | `…/component-internals/api-gateway.md` §3.18 |
| E5 | Ntrada's source is not in the workspace, so CORS wildcard handling cannot be settled here | ✅ | `…/component-internals/api-gateway.md` B-1, Q-2 |
| E6 | `get` is absent from `allowedMethods`, and a non-simple `GET` would be preflighted and rejected | ✅ | `…/component-internals/api-gateway.md` §3.18 |
| E7 | `Total-Count` is exposed although no route in the gateway repository sets it | ✅ | `…/component-internals/api-gateway.md` §3.18 |
| E8 | The documented extension procedure for this component is to replace the wildcard with actual client origins and add `get` | ✅ | `…/component-internals/api-gateway.md` §3.18, §7.7 item 2 |
| E9 | The gateway's intended client origin is unknown because `Pacco.Web` is empty | ✅ | `…/component-internals/api-gateway.md` Q-8 |
| E10 | `POST /identity/sign-in` is public and routed in the synchronous manifest at lines 263-269 | ✅ | `…/component-internals/api-gateway.md` §6.1 |
| E11 | Thirty-seven routes require authentication and five are gated on an administrator role claim | ✅ | `ADR-006` §1 |
| E12 | The platform has no orchestration, no mesh and no infrastructure-as-code in any of the fourteen repositories | ✅ | `ADR-006` §3 Alternative 2; `ADR-017` |
| E13 | `ADR-004` carries Q3 — "Should the edge remain purely declarative if a browser client is built?" — tagged `[handled later by the design stage]` | ✅ | `ADR-004` Open Questions, Q3 |

### 11.1 Documentation-versus-code conflicts

| # | Conflict | Resolution |
| --- | --- | --- |
| X1 | None found for this record. The four manifests, the component inventory and `ADR-004` agree on the policy's content and on its duplication | No resolution required |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | `Ntrada.Extensions.Cors` answers a CORS preflight for a routed `POST` and returns the matching origin when that origin is on the list | Naming origins is the documented extension procedure for this component, which implies the extension does the ordinary thing with a named list | The surface cannot call the edge at all, and the only permitted fix is out of scope — the defect would be in a package with no source in the workspace. `DO1`, `DO2` and `DO3` all fail together | F2 — exercise the preflight against a running gateway before the surface's first call is written |
| A2 | The four manifests are all reachable configurations that a deployed gateway may select, so all four need the origin list | `NTRADA_CONFIG` selects among them and all four carry the identical policy today | Two of the four edits would be unnecessary. This costs nothing — an unused origin entry is inert — which is why rule 2 prefers lockstep over precision | Confirm which manifest each environment selects when F1 names the environments |
| A3 | One origin per environment is sufficient, and no environment serves the surface from more than one hostname | The work item's recommended default is one explicit origin per environment | An origin the platform intended to admit is refused, producing a browser-only failure with no server-side log line | F1 — the owner naming the origins states whether any environment has more than one |
| A4 | Removing the wildcard does not break an existing consumer | No browser client exists on the platform today, and the only existing browser asset calls `operations-service` directly rather than through the gateway | An unknown consumer outside the fourteen clones loses access at the restart, with no error visible in gateway logs | Ask the platform owner whether any client outside the clone set calls this edge from a browser, before the production restart |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no follow-up action in §10 has an accountable person and this record cannot leave `Proposed` | F1, F4, and this ADR's ratification | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | At the work item review that receives this record — it blocks ratification |
| B2 | **[ACTION NOW]** The concrete origin values are unknown. `OQ-2` is unresolved and depends on where the artifact is hosted. Until a human names them, `DO3` has no content — the decision is complete but unexecutable | `DO3` in full, and therefore `DO1`'s first successful call | Platform owner with the platform security owner | F1, after `ADR-021` F1 settles hosting | 2026-10-10 |
| B3 | **[ACTION NOW]** No browser client has ever made a cross-origin call to this edge, and the extension's source is absent from the workspace. The preflight behaviour the entire feature rests on is assumed, not proven | Every outcome in this work item | `hls` stage, with access to a running Docker stack | F2 — exercise it and record the observed response headers | 2026-10-10 |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Which concrete origin or origins replace the wildcard, per environment? | This is the decision's only unfilled value. Without it the change cannot be applied, and no later stage can invent the answer without inventing the hosting arrangement it depends on | One explicit origin per environment — local, Docker and production — derived from where `ADR-021`'s artifact is served, with no wildcard in any environment | Platform owner |
| Q2 | **[ACTION NOW]** Does `Ntrada.Extensions.Cors` echo the request origin for `allowedOrigins: ['*']` with `allowCredentials: true`, or emit a literal `*`? | It decides whether the platform has been running an open-CORS vulnerability or a broken-CORS bug up to now. It does not change this decision, but it changes whether the change is a fix or an incident response | Unknown from this workspace. F2's observation settles it as a by-product; if the observed header echoes an arbitrary origin, escalate before the production restart rather than after | Platform security owner |
| Q3 | **[handled later by the `lld` stage]** When the first cross-origin `GET` is introduced, is `get` added alone or is the whole method list reviewed? | Rule 3 defers the addition deliberately; deferring without saying who picks it up is how latent traps survive | Review the whole list at that point against the then-current surface's verb set, rather than adding one verb reactively | `lld` stage of the first outcome needing a read call |
| Q4 | **[handled later by the `hls` stage]** Does any code set the `Request-ID` response header the surface is asked to capture? | `NFR-15` targets capturing it, `ASM-12` already hedges that it may never arrive, and building capture logic for a header that is never emitted is wasted work with a misleading test | Treat it as declared-but-unproven, capture it opportunistically, and never branch on its absence | `hls` stage, per F5 |
