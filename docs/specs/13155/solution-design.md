# Solution Design — Work item 13155: Pacco common login and role-aware welcome landing

| Field | Value |
|-------|-------|
| Work item | 13155 |
| Delivery outcomes | `DO1` Pacco Common Login & Role-Aware Welcome Landing · `DO2` Pacco Session Continuity, Landing-Page Guard & Logout · `DO3` Pacco Browser Origin Enablement at the Ntrada Edge |
| Records authored | `ADR-021`, `ADR-022`, `ADR-023`, `ADR-024` |
| Records relied on | `ADR-004`, `ADR-006`, `ADR-007`, `ADR-018`, `ADR-020` |
| Capability registered | `CAP-17 — Browser Presentation & Authenticated Entry` |
| Date | 2026-09-19 |

---

## 1. Scope and summary

Pacco has no browser client. `Pacco.Web` is an empty repository with one tracked file, and the
platform's only piece of user-facing HTML is a static SignalR diagnostic page served from inside
`operations-service` — no build, no version, no independent deployment, and no storage of anything.
Every one of the platform's forty-one edge routes was designed for a machine caller.

This work item adds the first browser surface: one login screen, one role-aware welcome landing
page, and nothing else. There is no dashboard, no widget and no business functionality behind the
landing page. Three delivery outcomes make that possible — the surface itself, the client-side
session and guard behaviour on it, and one configuration change at the edge that lets a browser
reach the platform cross-origin.

**What this design changes about the platform.** Four firsts, each of which is why an architecture
record was needed rather than a design note:

1. A deployable that is not a .NET host, and the first UI that is independently built and released.
2. The first client-side persistence of a credential anywhere on the platform.
3. The first change to an existing gateway configuration value made for a feature.
4. The first caller for which the edge's CORS policy is load-bearing rather than latent.

**What this design deliberately does not change.** No new edge route. No backend-for-frontend. No
`identity-service` change. No authorization code change. No new unauthenticated surface beyond the
sign-in route that is already public. The edge remains the only way in, and `ADR-006`'s rule that
the gateway validates the token and binds caller identity is untouched.

**Honest summary of readiness.** The design is complete and it is not executable today. Two values
it needs do not exist — the hostname the surface is served from, and the hosting arrangement that
serves it — and one behaviour it rests on has never been observed: whether this gateway answers a
cross-origin preflight at all. Section 6 lists those and the rest by name.

---

## 2. Applicable ADRs

| ADR id | Governing or new this run | One-line why it applies |
|--------|---------------------------|--------------------------|
| `ADR-021` — A standalone browser surface in `Pacco.Web` | **new this run** | Establishes where the surface lives, that it is independently built and released, and that it reaches backends only through the edge |
| `ADR-022` — Client-owned browser session custody | **new this run** | Establishes that the session lives entirely in the client: one tab-scoped store, a client-side guard, a client-side logout, no refresh |
| `ADR-023` — Named browser origins at the declarative edge | **new this run** | Replaces the CORS wildcard with named per-environment origins so a credentialed browser call is possible at all |
| `ADR-024` — Toolchain currency for non-.NET deployables | **new this run** | Supplies the toolchain-currency rule that `ADR-020` does not reach, so the surface is governed by a record rather than by nothing. It names no toolchain and no version |
| `ADR-006` — Edge-enforced authentication with fail-open authorization | governing | Fixes the authentication boundary at the gateway, which is why the client-side guard is an affordance and not a control |
| `ADR-007` — Split JWT trust root, gateway and services | governing | Records that a revoked token is still accepted by the gateway and all eight services, which is why logout is a client-side discard and says so |
| `ADR-004` — Declarative configuration-driven API gateway | governing | Its Q3 asks whether the edge stays purely declarative if a browser client is built and directs that a separate backend-for-frontend be added only if aggregation is needed — this design needs none |
| `ADR-018` — Build and release automation | governing | Records that the eleven pipelines build and push images and that nothing deploys them, which the twelfth release path inherits |
| `ADR-020` — .NET Core 3.1 as platform runtime baseline | governing, with a recorded exception | Pins one runtime for every .NET deployable. A browser bundle has no .NET runtime, so `ADR-021` records an `ARCHITECTURE_ALIGNMENT_EXCEPTION` rather than claiming conformance. `ADR-024` states the scope split that exception implies, and `ADR-020` itself is unchanged |

Four records were authored rather than one. Three establish a distinct inheritable constraint with
a distinct owner: where a browser deployable lives and how it ships (`ADR-021`, platform owner), how
a browser holds a credential (`ADR-022`, platform security owner), and what the edge admits
(`ADR-023`, `api-gateway`). Those three map one-to-one onto `DO1`, `DO2` and `DO3`. The fourth,
`ADR-024`, maps onto none of them: it exists because `ADR-021`'s alignment exception left the new
deployable outside every toolchain rule the platform has, which risk `R9` scores at RPN 280 with an
`adr` mitigation owned by this stage. No record was created merely because more than one option
existed.

---

## 3. Finalized design decisions

| Decision | Chosen option | Rationale | Source | Rejected alternatives and why |
|----------|---------------|-----------|--------|-------------------------------|
| Where the browser surface lives and how it ships | A standalone, independently deployable surface with its own build artifact, versioning and pipeline, hosted in the existing `Pacco.Web` repository | The repository already exists and is named for exactly this. A new repository would add a fifteenth clone with no owner and no pipeline, which is the condition the platform already suffers from | **human-gate** — `AD-1`, option A, `chosen_by: human` | Static assets inside the `operations-service` image — binds UI availability to a domain service's process and forces an operations rebuild for any frontend change. Same-origin assets served by the gateway — sits in direct tension with the stated cross-origin scope and would extend a declarative edge into asset serving. A new backend-for-frontend — nothing to aggregate; one call to one existing route. A new repository — no owner, no pipeline, and a fourteen-clone scope fixed elsewhere. `identity-service` as host — a domain service serving a frontend inverts the platform's ownership model |
| Whether a platform frontend standard is authored now | Baseline this surface only, and defer a platform frontend standard until a second surface exists | One surface is not enough evidence to generalise a platform rule from, and generalising prematurely sets rules for a second surface nobody has seen | **human-gate** — `AD-2`, option A, `chosen_by: auto_default`. Because it was an auto-default rather than a human choice, it is recorded as an **assumption** in `ADR-021` A1, not as a decision | Author a platform frontend standard now — rejected as premature. Carried as `GAP-13155-09` with a named trigger: the second surface's intake |
| How the session is stored | `sessionStorage`, one tab-scoped key holding the access token only | Survives a page refresh and ends with the tab, which is the stated requirement, and introduces no cookie and no server-side session — neither of which exists on any Pacco route | **human-gate** — recorded answer to the session-persistence question | A cookie — the platform sets none anywhere and `identity-service` is explicitly not a session store. `localStorage` — outlives the tab, widening the exposure window for a credential nothing can revoke. In-memory only — fails the refresh requirement |
| What logout does | A client-side discard of the held token, with copy that claims nothing more | `ADR-007` §2 rule 4 records that a revoked token is still accepted by the gateway and by all eight domain services. A client-side discard is the only enforceable logout that exists | **governance** — `ADR-007`, plus the recorded human answer | Call the revocation endpoint — it has no gateway route in any of the four manifests, and even if routed it would change nothing at the gateway. Claim server-side termination in the copy — untrue |
| How the landing page is protected | A client-side route guard reading the `role` claim from the locally held token, with no backend call | `ADR-006` places the authentication boundary at the edge and leaves in-service authorization fail-open, so a client-side check cannot be a control and must not be built as one. The landing page carries no data to protect | **human-gate** — recorded answer to the landing-page validation question, consistent with **governance** `ADR-006` | A server-validated `GET /identity/me` on every protected-route entry — the route exists and is authenticated, but it would introduce the scope's only cross-origin `GET` against a method allow-list that omits `get`, and would imply a control the edge already owns |
| How session expiry is detected | Client-side, from the token's expiry, with a redirect to Login carrying a session-expired message | The landing page issues no backend call, so no failure response exists from which expiry could be learned | **fit-verdict** — a direct consequence of the guard decision above | Learn it from a failed backend call — there is no backend call to fail |
| Whether tokens are refreshed | No refresh. The refresh token in the sign-in response is discarded on receipt | `POST refresh-tokens/use` and `POST refresh-tokens/revoke` have no gateway route in any of the four manifests, so a browser cannot reach them | **governance** — the edge route inventory | Route the refresh endpoints — would add new edge routes, which the recorded scope answer explicitly forbids |
| What changes at the edge | `allowedOrigins: ['*']` is replaced by named per-environment origins in all four `ntrada*.yml` manifests. Nothing else in the CORS block changes and no route is added | `allowCredentials: true` with `allowedOrigins: ['*']` is the combination the CORS specification forbids, so naming origins is required for a credentialed browser call to work at all — not merely preferred | **governance** — the CORS specification and `…/component-internals/api-gateway.md` §3.18's documented extension procedure; **human-gate** — the recorded answer permitting CORS allowed-origin changes only | Leave the wildcard — forbidden with credentials, and the observed behaviour is either an open-CORS vulnerability or a broken-CORS bug. Drop `allowCredentials` — changes the edge's contract for every existing caller. Add a same-origin proxy route — a new edge route, which the scope forbids. Build a backend-for-frontend to sidestep the origin problem — a new deployable for a configuration value |
| Whether `get` is added to the method allow-list | Not in this scope | The route guard makes no backend call, so this scope issues no cross-origin `GET`. Adding a method the design does not use would be an unevidenced change to a security-relevant allow-list | **fit-verdict** | Add `get` now, as the component's documented extension procedure suggests — deferred deliberately, with the trap recorded so the next surface meets a note rather than a silent preflight rejection |
| Whether a new component is introduced at the edge | No. `ADR-023` changes a configuration value and introduces no component | The fit analysis concluded reuse. Every capability needed already exists at the edge | **fit-verdict** | A backend-for-frontend, a proxy, or a second ingress — each would add a boundary, an owner and a deployable to carry one configuration value |
| Accessibility target | WCAG 2.1 AA on both screens | No accessibility standard is catalogued anywhere on the platform, and the only quality gate is a build that a failing test suite does not break. A target had to be established rather than inherited | **fit-verdict**, recorded in `ADR-021` obligation 6 | Inherit a platform standard — none exists. Leave it unstated — leaves the platform's first user-facing surface with no target at all |

---

## 4. Target design and solution shape

### 4.1 The shape in one paragraph

A static browser artifact, built and versioned in `Pacco.Web`, served from a named per-environment
origin. It calls exactly one backend route — `POST /identity/sign-in`, already public — through
`api-gateway`, cross-origin, with credentials. It stores the returned access token in
`sessionStorage`, reads the `role` claim from that token to choose between two welcome messages, and
uses the same claim in a client-side route guard that makes no network call. Logout clears the
store. Expiry is detected from the token. The edge changes by one configuration value in four files.

### 4.2 Components and ownership

| Component | Status in this design | Owner | What it does here |
|-----------|----------------------|-------|-------------------|
| `Pacco.Web` browser surface | **new** | `Pacco.Web`, per `ADR-021` | The login screen, the landing page, the guard, the store and the logout action |
| `api-gateway` | reused, one configuration value changed | `api-gateway` | Terminates the cross-origin call, answers the preflight, routes sign-in downstream unchanged |
| `identity-service` | reused, unchanged | `identity-service` | Verifies credentials and mints the token, exactly as it does today |
| `sessionStorage` | new usage of a browser-supplied capability | `Pacco.Web` | The single token store, tab-scoped |

**Justification for the one new component.** `ADR-021` §3 evaluated five existing hosts before
concluding a new boundary was required: the `operations-service` static host (rejected — binds UI
availability to a domain service's process and forces an unrelated rebuild for any frontend change),
same-origin assets from `api-gateway` (rejected — extends a declarative routing edge into asset
serving and contradicts the stated cross-origin scope), a new backend-for-frontend (rejected —
nothing to aggregate), a new repository (rejected — a fifteenth clone with no owner and no
pipeline), and `identity-service` (rejected — inverts the platform's ownership model). The new
boundary is required because a browser surface has a different release cadence, a different
toolchain and a different availability profile from every .NET service, and because `ADR-021`'s
central obligation — independent deployability — is unachievable inside any existing deployable by
definition. It stays stable because its dependency surface is exactly one route through one edge.
Extending an existing service to host it would violate cohesion (a domain service owning
presentation), ownership (`CAP-11` owning a `CAP-17` concern) and deployment independence (the
property the decision exists to obtain).

### 4.3 Resolved cross-outcome wiring

| Edge | Producer | Consumer | Mechanism | Mechanism owner | Verdict |
|------|----------|----------|-----------|-----------------|---------|
| `DO1` → `DO2` | The login screen | The route guard, the expiry check and the logout action | `sessionStorage`, one tab-scoped key, written on sign-in and read on every protected-route entry | `Pacco.Web` — both ends are inside one deployable | **resolved.** The consumer is provably invoked: the guard runs on landing-page entry, which is the only route to the landing page |
| `DO1` → `DO3` | The hosting arrangement serving the build artifact | The `allowedOrigins` list in four `ntrada*.yml` manifests | A configuration value applied by a human, taking effect on a gateway restart — not a runtime call | `api-gateway` for the manifests; the platform owner for the values | **resolved as a mechanism, unexecutable as a value.** The hostname is unnamed — `GAP-13155-03` |
| `DO3` → `DO1` | `api-gateway` via `Ntrada.Extensions.Cors` | The browser's own CORS enforcement | CORS response headers on the preflight and on the actual `POST` | `api-gateway` — the extension is referenced at `Pacco.APIGateway.csproj:15` | **`BLOCKING` · `must_verify`.** The consumer certainly runs. Whether the producer emits headers it accepts has never been observed — `GAP-13155-02` |

The distinction between the second and third rows matters for planning. The second is waiting on a
**decision**: once a human names a hostname, it works. The third is waiting on an **observation**:
naming origins is the documented extension procedure, but no browser has exercised it against this
gateway and the extension's source is not in the workspace. A value can be supplied. A behaviour has
to be seen. Writing origins into the manifests and watching the gateway start proves the YAML
parsed, not that a preflight is answered.

### 4.4 What the design refuses to do, and why it is written down

Four absences are deliberate and each is the kind of thing a later stage could reasonably "restore":

1. **No `GET /identity/me` on guard evaluation.** The route exists and is authenticated. Calling it
   would be the obvious way to build a guard, and it is excluded — which is also what keeps the
   scope free of any cross-origin `GET`.
2. **No refresh.** The refresh token is discarded on receipt. The consequence is a hard 60-minute
   session ceiling, accepted knowingly.
3. **No revocation call on logout.** Unroutable through the edge, and it would not change the
   gateway's behaviour if it were.
4. **No SignalR connection.** The existing browser asset opens a hub connection to a hard-coded
   address and sends its token after the socket is open. That is a different surface with a different
   token model, and this one calls only through the edge.

---

## 5. Downstream design obligations

### 5.1 For the `hls` stage

| # | Obligation | Why it lands here |
|---|-----------|-------------------|
| H1 | Name the mechanism by which a static artifact receives a per-environment gateway base URL | `ADR-021` obligation 2 requires configuration rather than a compiled-in URL, and no precedent exists on a platform where every other deployable is a process. `GAP-13155-17` |
| H2 | Establish whether anything sets the `Request-ID` response header, and record the finding either way | The header is CORS-exposed and set by no code in the workspace. Building capture logic for it without checking is the failure mode. `GAP-13155-06` |
| H3 | Decide whether expiry is read from the token's own claim or from the sign-in response `Expires` value, and state which | Both are available, nothing verifies that they agree, and two code paths reading different sources is how they silently diverge |
| H4 | Write acceptance criteria that state the tab-scoped session behaviour explicitly | A second tab requiring a second sign-in is a direct consequence of the agreed store. Unstated, it is reported as a defect |
| H5 | Specify the failure-path behaviour for every route: invalid credentials, service failure, network failure, and the client-side timeout elapsing | `NFR-12` requires all four to leave the form usable and the processing state cleared |

### 5.2 For the `lld` stage

| # | Obligation | Why it lands here |
|---|-----------|-------------------|
| L1 | Choose the framework, bundler and component library, and commit the dependency manifest, build and lint baseline | `ADR-021` obligation 3 and `NFR-23`. `ADR-024` constrains the choice without making it: one declaration per repository, a committed lockfile, and a version still in vendor support on the day it is committed. Whatever is chosen becomes a precedent — see `GAP-13155-09` |
| L2 | Fix the client-side request timeout value, sized above the gateway's retry behaviour | `ADR-022` rule 7 requires an explicit bound. `GAP-13155-13` records that the figure to size against is unmeasured, so state the assumption used |
| L3 | Implement the role comparison as exactly one case-insensitive check against `admin`, resolving every other value and an absent claim to the plain welcome | `NFR-5`, `ASM-2`, `ADR-022` rule 3. One comparison, one place |
| L4 | Ensure the token is written to `sessionStorage` and the `Authorization` header and nowhere else — not a URL, a log line, an analytics event or an error report | `NFR-10`, `ASM-10`, `ADR-022` rule 2 |
| L5 | Implement WCAG 2.1 AA on both screens: labelled inputs, programmatically associated error text, keyboard operability, visible focus | `ADR-021` obligation 6, `NFR-14`, `ASM-11` |
| L6 | Block duplicate submissions for the whole time a sign-in request is in flight | `NFR-11` |
| L7 | When a cross-origin `GET` is first needed, raise the method allow-list question rather than adding `get` silently | `ADR-023` rule 3 defers it deliberately. `GAP-13155-17`'s neighbour trap is in `ADR-023` Q3 |

### 5.3 For the coding and `devops` stages

| # | Obligation | Why it lands here |
|---|-----------|-------------------|
| D1 | Apply the named origins to **all four** manifests — `ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` — and verify the four CORS blocks are identical afterwards | `ADR-023` §4. Editing fewer than four produces a failure that appears only in the environment selecting the unedited file. `R2`, the second-highest-scored risk in the register |
| D2 | Schedule the origin change as a gateway process restart in each environment | `NFR-19`. Configuration is read at process start and is not hot-reloaded |
| D3 | Prove the cross-origin preflight before declaring the surface working: a credentialed cross-origin `POST` from a named origin against a running gateway | `ADR-023` rule 6, `R1`, `GAP-13155-02`. This is the single most important verification in the work item |
| D4 | Build the `Pacco.Web` pipeline as a genuine build-test-release path, not a copy of a .NET pipeline | `ADR-021` obligation 1. The eleven existing pipelines are `language: csharp` with `dotnet: 3.1.100` and do not transfer |
| D5 | Verify the shipped artifact contains no absolute backend URL | `NFR-17`, `R6` |
| D6 | Decide whether the static bundle is a member of `compose/services.yml` and the PM2 manifests, or a third kind of artifact | `ADR-021` Q2. The two deployable sets already disagree with each other |

### 5.4 Carried constraints that every stage inherits

- Every call goes through `api-gateway`. No direct service address, ever (`NFR-16`).
- No new edge route, no new unauthenticated surface, no backend-for-frontend, no `identity-service`
  or authorization code change.
- The client-side guard gates no data beyond what the held token already carries (`NFR-8`).
- Sign-in failure copy is one generic message for every invalid-credential outcome (`NFR-4`).
- No raw backend exception or stack trace is rendered (`NFR-3`).
- The landing page carries the welcome message and nothing else.

---

## 6. Open items and escalations

### 6.1 Reviewer action required now

Each row names a verb, a named owner or role, and what is blocked. None can be settled by a later
stage of this work item, because each depends on a system, a team or a capability outside it.

> **One action left this list on 2026-09-19.** "Decide what governs the browser toolchain and its
> version" was item 9 and is answered by `ADR-024`; `GAP-13155-07` in the register carries the
> resolution. The items after it are renumbered.

| # | Action | Owner | What is blocked |
|---|--------|-------|-----------------|
| 1 | **Name an owner** for each of the fourteen repositories | Whoever commissions work on this platform | Ratification of `ADR-021`, `ADR-022`, `ADR-023` and `ADR-024`, and every follow-up action in all four. Ten register entries currently name owners who do not exist. `GAP-13155-14` |
| 2 | **Run** a credentialed cross-origin `POST` from a named origin against a running gateway and record what the preflight returns | Platform owner | `DO1` and `DO3` both. The feature's first call. `GAP-13155-02`, risk `R1` |
| 3 | **Choose** the concrete browser origin hostname for each environment | Product and Architecture jointly | `DO3` entirely — the decision is complete and has no value to apply — and therefore `DO1`'s demonstration. `GAP-13155-03` |
| 4 | **Decide** how the built artifact reaches a served origin, given that nothing on this platform deploys anything | Platform owner | Item 3, which cannot be answered without it, and any claim that the surface reaches an environment. `GAP-13155-15` |
| 5 | **Specify** compensating controls for a script-readable token store — at minimum a content-security policy and dependency scanning in the `Pacco.Web` pipeline | Platform security owner | Serving a real token to real users. `GAP-13155-04`, risk `R3` |
| 6 | **Acknowledge** that logout terminates nothing platform-side, or fund the fix at the boundary that enforces authentication | Platform security owner | Nothing in this work item — `ADR-022` already responds by making the copy honest. It is listed because accepting it is a decision someone must actually take. `GAP-13155-12`, risk `R4` |
| 7 | **Close** the public sign-up route's acceptance of `"role": "admin"` from the request body | Platform security owner | Nothing in this work item technically. The surface will faithfully show the admin area to a self-escalated account, and no client-side change can prevent that. `GAP-13155-16`, risk `R13` |
| 8 | **Reconcile** the three role-comparison behaviours — gateway exact-string, `IdentityContext` case-insensitive, browser guard case-insensitive | Platform security owner | Any future surface that puts real functionality behind the guard. `GAP-13155-05`, risk `R5` |
| 9 | **Author or commission** a standards catalogue — eleven rule families have no covering content anywhere | Architecture owner | Nothing immediately. Three targets in this run had to be set by decision because no standard supplied them, and each is now a precedent from a single feature. `GAP-13155-08` |
| 10 | **Configure** a total request timeout at the edge, or state that there will not be one | Platform owner | `L2` above — the client-side timeout is otherwise sized against an unmeasured figure. `GAP-13155-13`, risk `R11` |
| 11 | **Confirm** which of the four gateway manifests is live in each environment | Platform owner | `D1` above. Editing only the manifest assumed live produces an environment-specific failure. `GAP-13155-18`, risk `R2` |
| 12 | **Answer** what a user whose token carries no `role` claim at all sees, as distinct from an unrecognised value | Product owner | Spec sign-off. Nothing breaks at runtime — the safe behaviour is already specified. `GAP-13155-10` |
| 13 | **Set** the expected automated coverage for this feature, since the platform's test gate checks nothing | QA owner | Spec sign-off, and `D4` above. `GAP-13155-11` |
| 14 | **Confirm** that requiring a second sign-in in a second browser tab is acceptable | Product owner | Acceptance. It is a direct consequence of the agreed storage choice. `ADR-022` Q4 |

### 6.2 Delegated downstream — no action now

Each of these has a named stage that can settle it from within its own scope. They are listed so a
reviewer can see they were considered and assigned, not forgotten.

| # | Item | Named stage | Tag |
|---|------|-------------|-----|
| 15 | The mechanism by which a static artifact receives per-environment configuration | `hls` | **[handled later by the `hls` stage]** |
| 16 | Whether anything sets the `Request-ID` response header | `hls` | **[handled later by the `hls` stage]** |
| 17 | Whether expiry is read from the token claim or the response `Expires` value | `hls` | **[handled later by the `hls` stage]** |
| 18 | The concrete client-side timeout value | `lld` | **[handled later by the `lld` stage]** |
| 19 | Framework, bundler and component library selection | `lld` | **[handled later by the `lld` stage]** |
| 20 | Whether `get` is added alone or the method list is reviewed as a whole, when the first cross-origin `GET` appears | `lld` | **[handled later by the `lld` stage]** |
| 21 | Whether a platform frontend standard is authored, at the second browser surface's intake | `architecture` | **[handled later by the `architecture` stage]** |
| 22 | Whether the static bundle joins `compose/services.yml` and the PM2 manifests or is a third artifact kind | `devops` | **[handled later by the `devops` stage]** |

### 6.3 Escalation summary

Fourteen items need action before this design is executable, and three of them concern this feature —
items 3, 12 and 14. The remaining eleven are long-standing platform conditions: no repository
owners, no deployment of anything, no standards catalogue, a test gate that checks nothing, a
revocation model that does not revoke, and a public route that issues administrator accounts.

None of the eleven was introduced here, and none can be closed here. They appear because a browser
client is the first caller that cannot route around any of them: it cannot deploy itself, it cannot
be reached without a named origin, it cannot revoke a token, and it cannot decline to display a role
the platform issued. The full detail, with failure-mode scoring, is in
`docs/architecture-inventory/risk-constraint-gap-register.md`.
