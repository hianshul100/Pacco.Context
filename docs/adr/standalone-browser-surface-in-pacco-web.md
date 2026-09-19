# ADR-021: A standalone browser surface in `Pacco.Web`, independently built and released

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-19 |
| **ADR id** | `ADR-021` |
| **Backlog candidate** | — (no candidate exists; this record originates from work item 13155, `intents/13155.md`) |
| **Source** | `DO1` — Pacco Common Login & Role-Aware Welcome Landing; `DO3` — Pacco Browser Origin Enablement at the Ntrada Edge |
| **Originating NFRs** | `NFR-14`, `NFR-17`, `NFR-20`, `NFR-23` (decision `needs_adr`); `NFR-16` (constraint honoured) |
| **Category / Impact** | Deployment / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Architecture decision gate `AD-1`, option A, `chosen_by: human` (work item 13155). Repository ownership remains unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-018` (the per-repository release rule this extends), `ADR-004` (the edge this surface calls and must not stretch), `ADR-020` (the single-runtime rule this surface falls outside), `ADR-024` (the toolchain-currency rule that governs it instead), `ADR-022` (what the surface holds in the browser), `ADR-023` (the origin that makes it reachable) |

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

Pacco has eleven deployables, one declarative edge, and no application frontend. The platform is
about to acquire its first one, and nothing in the current structure says where it goes.

1. ✅ **There is no frontend.** No `package.json`, `angular.json` or `vite.config.*` exists in any of
   the fourteen clones; no `.cshtml` and no `.razor` file exists; exactly one `wwwroot` directory
   exists, inside `operations-service`
   (`docs/architecture-inventory/baselines/architecture-baseline.md` §7.1;
   `docs/architecture-inventory/baselines/ui-inventory.md` §1.1, §1.2, §1.5).
2. ✅ **`Pacco.Web` is named for this purpose and is empty.** `git -C hianshul100_Pacco.Web ls-files`
   returns `README.md`, containing the single line `# Pacco.Web`
   (`…/baselines/architecture-baseline.md` §7.1; `…/baselines/ui-inventory.md` §1.4).
3. ✅ **The one browser asset that exists is bundled into a backend image.** The static SignalR
   console at `Pacco.Services.Operations.Api/wwwroot/ui/` is published by `dotnet publish` into the
   `operations-service` output and baked into the `pacco.services.operations` container image. A
   change to `index.html` or `app.js` requires a rebuild and redeploy of the whole service, and the
   page's availability is exactly that service's availability
   (`…/baselines/ui-inventory.md` §11.2).
4. ✅ **That asset hard-codes its backend address.** `app.js` builds its connection against
   `http://localhost:5005/pacco` — plaintext, fixed host, and a direct address that bypasses the
   gateway entirely (`…/baselines/architecture-baseline.md` §7.2).
5. ✅ **It was authored without a toolchain.** No bundler, no transpiler, no minifier, no lockfile;
   the SignalR client is a hand-vendored webpack bundle whose version cannot be established
   (`…/baselines/architecture-baseline.md` §7.2; `…/baselines/ui-inventory.md` §5.1).
6. ✅ **The platform already releases one deployable per repository.** Eleven repositories each carry
   their own pipeline, their own image and their own release, with no mechanism for releasing several
   together. Three repositories carry no pipeline at all, and `Pacco.Web` is one of them
   (`ADR-018` §2, §6).
7. ✅ **The edge is closed to per-client shaping.** `ADR-004` §2 obligation 3 states that response
   aggregation or per-client shaping "must not be added to this configuration. If a future client
   needs either, introduce a separate backend-for-frontend service and record that decision rather
   than stretching this one."
8. ✅ **The edge already exposes what this surface needs.** `POST /identity/sign-in` is a public,
   synchronous, downstream-proxied route in the gateway's synchronous manifest
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:263-269`), and `identity`,
   `operations` and `pricing` have no asynchronous routes, so sign-in stays synchronous in every
   manifest (`docs/architecture-inventory/component-internals/api-gateway.md` §5.2, §6.1).

The question this record answers is narrow: **where does a browser login surface live, and what is
its release identity?** It was put to an architecture decision gate as `AD-1` and answered by a
human.

### 1.1 What this record does *not* cover

It does not decide what the surface holds in the browser — the token store, the route guard, the
logout semantics and the expiry handling are `ADR-022`. It does not decide which origins the edge
admits, which is `ADR-023`. It does not select a framework, a bundler or a component library, and it
does not establish a platform-wide frontend standard: that deferral is recorded as assumption A1.

## 2. Decision Drivers

| # | Driver | Source |
| --- | --- | --- |
| D1 | A UI change must not require rebuilding and redeploying a domain service, and UI availability must not be a domain service's availability | `NFR-20`; `…/baselines/ui-inventory.md` §11.2 |
| D2 | The backend address must be per-environment configuration, never compiled into a shipped asset | `NFR-17`; `ASM-13` |
| D3 | The surface must carry a documented dependency manifest, build and lint baseline rather than repeating the no-tooling authoring model | `NFR-23`; `ASM-17` |
| D4 | Every UI-to-backend call must traverse the Ntrada edge allow-list; no direct-to-service call | `NFR-16` |
| D5 | The platform's existing release rule — one repository, one deployable, one independent release — should extend to the new surface rather than be excepted for it | `ADR-018` §2 |
| D6 | The edge must not be stretched into a client-shaping layer, and no backend-for-frontend is in scope | `ADR-004` §2 obligation 3; `DO3` |
| D7 | The scope permits exactly one edge configuration change and no identity or authorization code change | `ASM-6` |

## 3. Architecture Fit Evaluation

**No existing component owns presentation.** The capability ontology runs `CAP-01` … `CAP-16` and
contains no presentation capability (`docs/architecture-inventory/baselines/capability-baseline.md`
§1). The only browser asset on the platform is classified capability-scoped to `CAP-11` and owned by
`operations-service` (`…/baselines/ui-inventory.md` §11.1, §12). Each candidate host was evaluated
against the drivers above before a new boundary was proposed.

| # | Existing component or extension point | Evaluated as | Primary reason for rejection |
| --- | --- | --- | --- |
| 1 | `operations-service` static-file host (`Extensions.cs:88` `.UseStaticFiles()`, `wwwroot/ui/`) | The only working static-asset host on the platform — extending it is the lowest-effort option | It binds the login surface's availability and release to a domain service. A login-page change would require an `operations-service` rebuild and redeploy, and the login screen would go dark whenever operation-status tracking did. It also puts the platform's front door inside the `CAP-11` boundary, which owns operation status and nothing else |
| 2 | `api-gateway` (Ntrada) serving assets same-origin | Would remove the cross-origin problem entirely | The gateway is a YAML document, not an application — it has no static-file middleware, no build step and no asset pipeline (`…/component-internals/api-gateway.md` §1). Serving UI assets there makes the edge the deployment unit for every UI release, and a gateway configuration change takes effect only on a process restart (`NFR-19`), so UI releases would become edge restarts. It also contradicts the scope, which assumes a cross-origin UI and names the CORS entry as the sole edge change |
| 3 | A new backend-for-frontend service between browser and edge | The route `ADR-004` explicitly leaves open for a future client | Nothing in scope needs it. The surface calls one public route and reads one response body; there is no aggregation, no fan-out and no shaping to do. `ADR-004` §3 Alternative 1 already rejected a hand-written backend-for-frontend, and adding one would introduce a twelfth .NET deployable to solve a problem that does not exist yet |
| 4 | A new repository dedicated to the login surface | The literal reading of "standalone deployable" | `Pacco.Web` already exists, is inside the catalogued fourteen-clone scope, and is named for exactly this. Creating a second repository would leave a permanently empty one whose name claims the role. Rejected at the `AD-1` gate |
| 5 | `identity-service` hosting a server-rendered login page | Would put the login form next to the credential store | It would make `identity-service` an HTTP presentation host, which it is not — it exposes a dispatcher-bound command/query surface only (`…/component-internals/identity-service.md` §1). It would also re-introduce the coupling in row 1 with a more sensitive service, and it contradicts `ASM-6`, which permits no identity-service change |

**Why a new boundary is required.** Presentation is a capability no Pacco deployable owns, and it
differs from every existing deployable on all four axes that make a boundary: a different runtime
(a browser, not a .NET host), a different toolchain (Node, not the .NET SDK), a different release
cadence (a UI copy change is not a domain release), and a different failure domain (a broken bundle
does not degrade any backend). Every candidate above resolves one of those axes by collapsing it
onto a backend's, which is precisely the coupling `NFR-20` exists to prevent.

**Why the boundary stays stable.** Its only outbound contract is the gateway's public HTTP
allow-list, which is declarative, versioned in `ntrada*.yml` and already governed by `ADR-004`. It
holds no domain state, owns no database, publishes to no exchange, and registers with no discovery
registry. Nothing downstream depends on it — the dependency runs one way only.

**Why extending an existing component would violate cohesion, ownership or deployment
independence.** Row 1 violates all three at once: the login surface would sit inside a capability
boundary that does not own identity, under a service team that does not own the front door, in a
release train that does not match its cadence. Row 2 violates deployment independence in the
sharpest form the platform has — every UI release becomes an edge restart, and the edge is the
single point every one of the forty-one routes passes through.

## 4. Decision

**The Pacco browser surface is a standalone, independently deployable presentation deployable hosted
in the existing `Pacco.Web` repository. It has its own build, its own artifact, its own version and
its own pipeline, and it reaches backend capabilities only through the existing Ntrada edge. No new
repository is created, no backend-for-frontend is introduced, and no edge route is added.**

This was decided at gate `AD-1`, option A, by a human. It is not re-opened by this record.

Five obligations are attached and are part of the decision:

1. 🎯 **`Pacco.Web` acquires a pipeline.** It is one of three repositories that carry none today
   (`ADR-018` §6). Until it has one, "independently deployable" is an intention rather than a
   property, and the surface has no release identity — which is the condition `NFR-20` measures.
2. 🎯 **The gateway base URL is a per-environment configuration value read at runtime, never a
   literal in the shipped bundle.** The existing browser asset's hard-coded
   `http://localhost:5005/pacco` is the anti-pattern this obligation names and forbids
   (`ASM-13`, `NFR-17`).
3. 🎯 **The surface ships a committed dependency manifest, build and lint/format baseline.** The
   platform has no frontend scaffold, template, contributing guide or architecture test to inherit,
   so the first surface authors the baseline it will be held to (`ASM-17`, `NFR-23`).
4. 🎯 **The browser talks to the edge and to nothing else.** No direct call to a service's published
   container port, and no route that is not already declared in the active `ntrada*.yml` manifest
   (`NFR-16`, `ASM-6`).
5. 🎯 **The deployable inventory is corrected when the surface ships.** `compose/services.yml`
   declares eleven services and the PM2 manifests declare ten; a twelfth deployable that appears in
   neither repeats the discrepancy already recorded in
   `…/baselines/architecture-baseline.md` §2.3.
6. 🎯 **The surface states WCAG 2.1 AA as its conformance target and ships the checks that hold it
   there.** No accessibility standard is catalogued for this platform and no quality gate would
   catch a regression, so the first surface authors the bar it is held to rather than inheriting one
   (`ASM-11`, `NFR-14`).

## 5. Options Considered

Option A was settled by a human at gate `AD-1`. The table records why the discarded options were
discarded, so the reasoning is inheritable rather than only the outcome.

| # | Option | Why it was rejected |
| --- | --- | --- |
| 1 | **A standalone independently deployable browser surface in `Pacco.Web`** | **Chosen.** Governing source: architecture decision gate `AD-1`, option A, `chosen_by: human`, work item 13155. It is the only option that satisfies D1, D2, D3 and D5 simultaneously, and it extends `ADR-018`'s existing rule rather than making an exception to it |
| 2 | **Static assets served from an existing service image** (`OQ-1` option b) | Rejected because it reproduces the coupling recorded in `…/baselines/ui-inventory.md` §11.2 exactly: no separate build, no separate artifact, no separate image, no separate pipeline, no separate version, and UI availability equal to a domain service's availability. It fails `NFR-20` by construction, and places the platform's front door inside a capability boundary that does not own identity |
| 3 | **Same-origin assets served by the gateway** (`OQ-1` option c) | Rejected because it sits in direct tension with the agreed scope, which assumes a cross-origin UI and names the CORS allowed-origins entry as the single permitted edge change (`ASM-6`). It also makes every UI release a gateway restart (`NFR-19`) and requires the edge to grow a static-file responsibility it has never had (`ADR-004`) |
| 4 | **A new backend-for-frontend deployable** | Rejected because no aggregation or per-client shaping is required by any outcome in scope, and `ADR-004` §3 Alternative 1 already rejected a hand-written backend-for-frontend on this platform. Revisit only when a client needs a response shape the edge may not produce |
| 5 | **A new repository outside the catalogued clone set** | Rejected at the `AD-1` gate. `Pacco.Web` exists, is in scope, and is named for this role; a second repository would leave a permanently empty one and would put the surface outside the fourteen-repository boundary every current artifact is written against |

## 6. Consequences

### 6.1 Positive

1. 🎯 A UI change is a UI release. No domain service is rebuilt, redeployed or restarted for a copy
   change, and no domain service's downtime takes the login screen with it.
2. 🎯 The platform's release rule becomes uniform again: twelve repositories, twelve deployables,
   twelve independent releases, consistent with `ADR-018` §2.
3. 🎯 The gateway stays a router. It gains one configuration value (`ADR-023`) and no new
   responsibility, so `ADR-004`'s declarative property survives the arrival of a browser client —
   which is the question `ADR-004` left open as its Q3.
4. 🎯 The surface's dependency graph is one edge wide. It calls the gateway; nothing calls it. That
   makes it the cheapest deployable on the platform to reason about, test and replace.

### 6.2 Negative

1. 🎯 **A second toolchain enters the platform.** Every current pipeline pins the same .NET SDK patch
   version and every image pins the same base images (`ADR-020` §2). A browser surface needs a Node
   toolchain, which `ADR-020`'s single-version rule does not reach and was never written to cover.
   The platform gains a second toolchain family. It is not ungoverned: `ADR-024` supplies the
   currency rule for every deployable `ADR-020` does not reach. What it does mean is that the
   platform now maintains two currency records instead of one, and that the uniformity `ADR-020`
   buys within .NET does not extend across the boundary.
2. 🎯 **`Pacco.Web` needs a pipeline built, not copied.** The platform's eleven pipelines are
   copies of one another, each running `build.sh`, `test.sh` and `dockerize.sh` against a .NET
   solution (`ADR-018` §6). None of them applies to a browser bundle, so obligation 1 is new work,
   not a copy-paste.
3. 🎯 **Nothing deploys today.** `ADR-018` §6 records that the pipelines build and push images and
   that nothing deploys them. A twelfth pipeline inherits that gap, so "independently deployable"
   will mean "independently built and published" until a deployment path exists.
4. ❓ **The deployable-set discrepancy grows.** `compose/services.yml` and the PM2 manifests already
   disagree about the platform's membership (`…/baselines/architecture-baseline.md` §2.3). Adding a
   deployable that is a static bundle rather than a process makes the question "what is a Pacco
   deployable" harder, not easier, until obligation 5 is done.
5. 🎯 **The platform frontend standard is deferred.** See assumption A1: this surface is baselined
   alone, and the second surface will either inherit its choices by default or re-litigate them.

### 6.3 Neutral

1. The existing SignalR console is untouched. It stays where it is, inside `operations-service`, as a
   capability-scoped developer harness (`…/baselines/ui-inventory.md` §11.1). This record neither
   migrates it nor blesses it.
2. No capability moves. `CAP-01` stays with `identity-service` and `CAP-02` stays with the gateway;
   the surface consumes both and owns neither.
3. The choice of framework, bundler and component library is left entirely open by this record and
   is not implied by any obligation in §4.

## 7. Compliance Considerations

| Governing source | Rule as it applies here | How this decision conforms |
| --- | --- | --- |
| `docs/adr/repository-per-service-independent-release.md` §2 — "Every deployable owns its own repository, its own solution, its own container image and its own release" | The surface is a deployable, so it owns a repository and a release | Conforms. `Pacco.Web` is the repository; obligation 1 gives it the release |
| `docs/adr/declarative-configuration-driven-api-gateway.md` §2 obligation 3 — "Response aggregation or per-client shaping must not be added to this configuration. If a future client needs either, introduce a separate backend-for-frontend service and record that decision rather than stretching this one." | A browser client has arrived; the rule decides whether the edge changes shape | Conforms. No aggregation or shaping is added to any manifest, and no backend-for-frontend is introduced. The only edge change is an allowed-origins value (`ADR-023`) |
| `docs/adr/edge-enforced-authentication-with-fail-open-authorization.md` §2 — "Pacco enforces authentication at the gateway… Services behind the edge trust the caller context they are given" | The browser is outside the boundary and must enter through it | Conforms. Obligation 4 forbids any direct-to-service call, so the surface is always on the authenticated side of the edge's rules |
| `docs/adr/dotnet-core-31-as-platform-runtime-baseline.md` §2 — "Every Pacco deployable targets one runtime version… No service selects its own runtime." | The rule is written for .NET deployables; a browser bundle has no .NET runtime to pin | `ARCHITECTURE_ALIGNMENT_EXCEPTION`, now with a stated replacement. The rule's scope is the .NET runtime of a Pacco service, and this deployable has none, so the exception stands as written. The gap it left — nothing governing the surface's toolchain version — was carried as `GAP-13155-07` and follow-up F4, and is closed by `ADR-024`, which takes every deployable `ADR-020` does not reach. "Outside `ADR-020`" therefore no longer means "outside any rule" |
| `docs/architecture-inventory/patterns/index.md` — "**Every pattern here is `Candidate`**" | No pattern in the catalog is binding, so none can be cited as a mandate | Honoured. `9` below relates this decision to the catalog without claiming approval the catalog does not grant |
| Frontend standards (state ownership, host/shell contract, accessibility) | No `docs/standards/` directory exists in this repository and no frontend rule family is catalogued | No silent default is taken. The absence is recorded as `GAP-13155-08` in the register, and the accessibility target is stated explicitly in §8 as an assumption (`ASM-11`) rather than inherited |

## 8. Non-Functional Requirements & Testing

| NFR | Target | How this decision addresses it | How it is verified |
| --- | --- | --- | --- |
| `NFR-20` [operational_readiness] | An independently deployable UI boundary | §4 obligation 1 — own build, artifact, version and pipeline stage in `Pacco.Web` | The surface's artifact is produced and published by a pipeline in `Pacco.Web` that references no other repository, and a UI-only change produces no new image for any `*-service` |
| `NFR-17` [portability_infra] | Zero hard-coded backend URLs; one configuration value per environment | §4 obligation 2 — the gateway base URL is runtime configuration | A grep of the built artifact finds no absolute backend host; the same artifact runs against two different gateway base URLs without a rebuild |
| `NFR-23` [maintainability] | A committed frontend build and lint baseline shipped with the surface | §4 obligation 3 — dependency manifest, build and lint/format committed. `ADR-024` adds what obligation 3 does not say: that the declared version must be in vendor support when committed and must carry a scheduled review | The repository contains a dependency manifest with a lockfile and a lint/format configuration, and the pipeline runs both. `ADR-024` §8 carries the currency checks |
| `NFR-16` [portability_infra] | Zero direct-to-service calls from the browser | §4 obligation 4 — the edge is the only reachable surface | Every network call the surface makes resolves to the configured gateway base URL; no call targets a `500x` service port |
| `NFR-14` [usability_accessibility] | WCAG 2.1 AA on the login screen and the landing page | §4 obligation 6 — the surface states the conformance target and ships the checks, because no platform standard exists to inherit | An automated accessibility check runs in the pipeline over both screens and fails the build on a violation, plus a manual keyboard-only pass covering labelled inputs, programmatically associated error text and visible focus |
| `NFR-2` [security] | Zero hard-coded credentials | The bundle carries configuration, not secrets; the token is obtained at runtime | Secret scanning over the built artifact and the repository returns no credential |

**Testing obligations this decision creates.** The platform has no house standard to inherit: the
build is not broken by a failing test suite, the suite is not run in CI, and the CI badge is
recorded as false confidence (`OQ-4`;
`docs/architecture-inventory/baselines/architecture-baseline.md` §9.5). That makes the following
explicit rather than assumed, and it is the `hls` and `lld` stages that must specify them:

1. The pipeline must fail the build when the test command fails. Inheriting the existing pipeline
   shape without this would reproduce the vacuous gate on a twelfth repository.
2. A build-output assertion that no absolute backend host string appears in the artifact — this is
   the only mechanical check that keeps `NFR-17` true over time.
3. One deployment-shape check that the artifact runs with the gateway base URL supplied externally.

## 9. Relationship to the implementation pattern catalog

| Pattern | Relationship |
| --- | --- |
| [`deployment/independent-per-repository-release.md`](../architecture-inventory/patterns/deployment/independent-per-repository-release.md) | **Extended.** The pattern is stated for the eleven .NET deployable repositories and their identical three-script pipeline. This decision extends the rule to a repository whose artifact is a browser bundle, where the three-script shape does not transfer. The pattern's `Related services/repos` entry no longer covers the full set once `Pacco.Web` ships |
| [`integration/declarative-configuration-driven-api-gateway.md`](../architecture-inventory/patterns/integration/declarative-configuration-driven-api-gateway.md) | **Unchanged and deliberately so.** The arrival of a browser client is the event the pattern's cost section anticipated; this decision keeps the edge declarative by putting the client outside it |
| [`security/edge-enforced-authentication-with-identity-binding.md`](../architecture-inventory/patterns/security/edge-enforced-authentication-with-identity-binding.md) | **Consumed, not changed.** The surface is a caller on the outside of this pattern's boundary. It adds no enforcement and removes none |

Every pattern in the catalog is `Candidate`, so none of these relationships constitutes approval.

## 10. Follow-Up Actions

| # | Action | Owner | Due | Blocks |
| --- | --- | --- | --- | --- |
| F1 | Build a pipeline for `Pacco.Web` that produces a versioned browser artifact and fails on a failing test command | Platform owner (unassigned — see B1) | 2026-10-17 | `NFR-20`; obligation 1 |
| F2 | Decide and record how the gateway base URL is supplied per environment for a static browser artifact, given that `Pacco.Web` has no runtime process to read environment variables at start-up | `hls` stage, with the platform owner | 2026-10-03 | `NFR-17`; obligation 2 |
| F3 | Add the surface to `compose/services.yml` and the PM2 manifests, or record explicitly why a static bundle is not a member of either set | Platform owner (unassigned — see B1) | 2026-10-24 | Obligation 5; closes part of `…/baselines/architecture-baseline.md` §2.3 |
| F4 | ~~Decide what governs the surface's toolchain version, since `ADR-020`'s single-runtime rule does not reach a Node toolchain~~ **Done, 2026-09-19:** `ADR-024` governs it. What remains is to satisfy that record when the first manifest is committed — declare one version for the repository, commit the lockfile, and record the support-end date | Platform owner (unassigned — see B1) | 2026-11-07 | `GAP-13155-07` — resolved |
| F5 | Revisit assumption A1 and decide whether a platform frontend standard is authored, at the point a second browser surface is proposed | Architecture owner (unassigned — see B1) | At the second surface's intake | `GAP-13155-09` |

## 11. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | `Pacco.Web` tracks exactly one file, `README.md`, containing `# Pacco.Web` | ✅ | `docs/architecture-inventory/baselines/architecture-baseline.md` §7.1; `docs/architecture-inventory/baselines/ui-inventory.md` §1.4 |
| E2 | No `package.json`, `angular.json` or `vite.config.*`, no `.cshtml` or `.razor`, and exactly one `wwwroot` exist across the clones | ✅ | `…/baselines/architecture-baseline.md` §7.1; `…/baselines/ui-inventory.md` §1.1, §1.2, §1.5 |
| E3 | The existing browser asset is published into the `operations-service` output and baked into that service's image; it is not independently deployable | ✅ | `…/baselines/ui-inventory.md` §11.2 |
| E4 | The existing browser asset hard-codes `http://localhost:5005/pacco` and bypasses the gateway | ✅ | `…/baselines/architecture-baseline.md` §7.2 |
| E5 | The existing browser asset has no bundler, transpiler, minifier or lockfile, and its SignalR client version is unknown | ✅ | `…/baselines/ui-inventory.md` §5.1; `…/baselines/architecture-baseline.md` §7.2 |
| E6 | Eleven repositories carry a pipeline; three do not, and `Pacco.Web` is one of the three | ✅ | `ADR-018` §6 |
| E7 | The edge forbids per-client shaping and names a separate backend-for-frontend as the route for a future client | ✅ | `ADR-004` §2 obligation 3; `ADR-004` §3 Alternative 1 |
| E8 | `POST /identity/sign-in` is a public synchronous route in the gateway's synchronous manifest | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:263-269`; `docs/architecture-inventory/component-internals/api-gateway.md` §6.1 |
| E9 | `identity`, `operations` and `pricing` have no asynchronous routes, so sign-in is synchronous in every manifest | ✅ | `…/component-internals/api-gateway.md` §5.2 |
| E10 | The capability ontology is `CAP-01` … `CAP-16` and contains no presentation capability | ✅ | `docs/architecture-inventory/baselines/capability-baseline.md` §1 |
| E11 | The one existing UI surface is capability-scoped to `CAP-11` and owned by `operations-service` | ✅ | `…/baselines/ui-inventory.md` §11.1, §12 |
| E12 | Every pipeline pins the same .NET SDK patch version and every image pins the same base images | ✅ | `ADR-020` §2, §6 |
| E13 | The pipelines build and push images; nothing deploys them | ✅ | `ADR-018` §6 |
| E14 | No `docs/standards/` directory exists in this repository, so no frontend rule family is catalogued | ✅ | Directory listing of `docs/` in this repository at the base ref |

### 11.1 Documentation-versus-code conflicts

| # | Conflict | Resolution |
| --- | --- | --- |
| X1 | `…/baselines/architecture-baseline.md` §7 opens with an input-gap note stating that `…/baselines/ui-inventory.md` "**does not exist** in this repository". It does exist, at 936 lines, and is the richer source on frontend deployment boundary, state management and auth handling | The file wins. This record cites `ui-inventory.md` directly, and `architecture-baseline.md` §7 is corrected in the same patch as this ADR |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | **This surface is baselined alone; no platform frontend standard is authored yet.** This was gate `AD-2`, option A, and it was taken by `auto_default` rather than chosen by a human, so it is recorded here as an assumption rather than as a decision | One surface is not evidence of a platform pattern, and the platform has no second browser surface to generalise from. Authoring a standard from a sample of one risks codifying accidents | The second surface inherits this one's framework, bundler, state model and shell contract by default rather than by decision, and the platform acquires a frontend standard nobody ratified. If a second surface is already planned, the deferral is wrong and the standard should be authored now | Confirm with the product owner whether a second browser surface is on the roadmap within two quarters. If it is, promote `AD-2` to a human decision before this surface's toolchain is fixed |
| A2 | A browser artifact can obtain per-environment configuration without a server process to inject it | Every other Pacco deployable is a process that reads configuration at start-up; a static bundle is not, and no mechanism for configuring one exists anywhere in the fourteen clones | Obligation 2 has no implementation, and the surface would either hard-code the gateway URL — failing `NFR-17` outright — or need a runtime host, which changes the deployable's shape and reopens `AD-1` | Follow-up F2. The `hls` stage names the mechanism concretely before the toolchain is fixed |
| A3 | Nothing outside the fourteen cloned repositories already provides a Pacco frontend | The absence is proven exhaustively inside the clone set, and the clone set is fixed by backlog issue 12998; what lies outside it was not observable | This decision would be creating a second frontend rather than the first, and the real one would carry conventions this record does not know about | Ask the platform owner whether a Pacco web client exists in any repository outside the fourteen |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no follow-up action in §10 has an accountable person and this record cannot leave `Proposed` | F1, F3, F4, F5, and this ADR's ratification | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | At the work item review that receives this record — it blocks ratification |
| B2 | **[ACTION NOW]** `Pacco.Web` has no pipeline, and the platform's eleven existing pipelines are .NET-specific copies that do not transfer to a browser bundle. Until one exists, "independently deployable" is an intention, not a property | Obligation 1 and `NFR-20`'s target | Platform owner | F1 — author a pipeline for a browser artifact rather than copying a service pipeline | 2026-10-17 |
| B3 | **[ACTION NOW]** Nothing deploys. `ADR-018` §6 records that the eleven pipelines build and push images and that no deployment step exists anywhere. A twelfth pipeline inherits that gap | Any claim that this surface reaches an environment, and the operational verification in `DO1` | Platform owner | Decide whether a deployment path is in scope for this work item or is a separate platform concern, and say which in the work item | Before any claim that the surface reaches an environment |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[handled later by the `hls` stage]** How does a static browser artifact receive its per-environment gateway base URL? | Obligation 2 and `NFR-17` depend on a mechanism nobody has named, and the platform has no precedent for configuring a non-process deployable | Supply it as a build-time environment substitution into a runtime-read configuration document rather than into module code, so one artifact is not rebuilt per environment. The `hls` stage picks and records the concrete mechanism | `hls` stage, with the platform owner |
| Q2 | **[ACTION NOW]** Is a static bundle a member of `compose/services.yml` and the PM2 manifests, or a different kind of artifact that neither set describes? | The two deployable sets already disagree with each other; adding a third kind of member without deciding this makes the platform's membership question worse | Record it in `compose/services.yml` behind a static-file server so the local topology stays complete, and state in the PM2 manifests why it is absent there | Platform owner |
| Q3 | **[handled later by the `lld` stage]** Which framework, bundler and component library does the surface use? | It is deliberately not decided here, but it must be decided before `NFR-23`'s baseline can be committed, and A1 means whatever is chosen becomes the platform's de facto standard | No recommendation is made by this record, and none by `ADR-024` either — but `ADR-024` bounds the choice: whatever is picked must be in vendor support on the day it is committed, declared once for the repository with a lockfile, and given a review dated before its support ends. The `lld` stage chooses within that, and records the choice against A1 so the de facto inheritance is visible | `lld` stage |
