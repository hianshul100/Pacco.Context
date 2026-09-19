# ADR-004: A declarative configuration-driven gateway as the single north-south edge

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-004` |
| **Backlog candidate** | `ADR-CANDIDATE-004` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Integration / high |
| **Supersedes / Superseded by** | — (first ADR corpus on this platform; nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-001` (the exchanges the edge can publish onto), `ADR-002` (the toolkit composed alongside the gateway library), `ADR-006` (the trust boundary this edge carries); premise of `ADR-CANDIDATE-005` |

## Notation

| Symbol | Meaning |
| --- | --- |
| ✅ | Confirmed current behaviour — observed in source at the cited path and line range |
| 🎯 | Target state — intended design, not in production today |
| ❓ | Needs validation — assumed or inferred, not observed in source |
| `[INFERRED]` | Conclusion drawn from observed evidence rather than stated by it |

## Contents

1. [Context](#1-context)
2. [Decision](#2-decision)
3. [Alternatives Considered](#3-alternatives-considered)
4. [Consequences](#4-consequences)
5. [Relationship to the implementation pattern catalog](#5-relationship-to-the-implementation-pattern-catalog)
6. [Evidence](#6-evidence)
7. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Context

Ten deployables sit behind one public entry point. Everything a caller can do to Pacco — routing,
authentication, claim gates, payload binding, downstream forwarding, CORS, error shaping, Swagger and
tracing — has to be expressed somewhere, and Pacco expresses it as a YAML document rather than as
gateway code:

1. ✅ **The gateway repository contains no controller and no route code.** Its entire executable is a
   host that reads a YAML file, registers the gateway library, and lets it serve
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Program.cs:24-49`). The only first-party code in
   the project is four small infrastructure classes for correlation, span context and a request hook.
2. ✅ **Which YAML file is loaded is an environment variable.** The host takes the first command-line
   argument, falls back to the `NTRADA_CONFIG` environment variable, and falls back again to a default
   file name (`…/Program.cs:28-39`).
3. ✅ **Four configuration files ship in the image**, all copied to the output directory: a local and a
   Docker variant of a synchronous edge, and a local and a Docker variant of an asynchronous one
   (`…/Pacco.APIGateway.csproj:24-37`).
4. ✅ **The route table covers nine service modules** — availability, customers, deliveries, identity,
   operations, orders, parcels, pricing and vehicles — plus a home route
   (`…/ntrada.yml:66-455`).
5. ✅ **The edge library and its six extensions are pinned to a floating `0.4.*` version line**, the
   same version line as the platform toolkit (`…/Pacco.APIGateway.csproj:14-20`).

The consequence is that the public API of Pacco is not defined by any compiled artifact. It is defined
by a YAML document in a dialect specific to one library.

### 1.1 What this record does *not* cover

This record fixes *how* the edge is expressed and that there is exactly one of it. It does not decide
what the edge does with a write — proxy it downstream or publish it as a command — which is
`ADR-CANDIDATE-005`, nor the authentication semantics it enforces, which are `ADR-006`.

## 2. Decision

**Pacco exposes exactly one north-south edge, implemented as a thin host embedding a declarative
gateway library, whose entire behaviour — routes, authentication flags, claim gates, payload binding,
downstream targets and extensions — is expressed in a YAML configuration document selected at startup
by the `NTRADA_CONFIG` environment variable. No routing or forwarding logic is written as gateway
code.**

Three obligations are attached and are part of the decision:

1. 🎯 **The routing configuration is a reviewed architectural artifact,** not an operations file. A
   change to it changes the platform's public contract and must be reviewed as such.
2. 🎯 **Each environment must record which configuration file it loads.** Nothing in the workspace
   states this today, and the four files differ in contract, not merely in hostname.
3. 🎯 **Response aggregation or per-client shaping must not be added to this configuration.** If a
   future client needs either, introduce a separate backend-for-frontend service and record that
   decision rather than stretching this one.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **A hand-written backend-for-frontend service** — a first-party ASP.NET project with controllers that call downstream services. | Rejected because it would make the edge a tenth codebase to maintain, test and release, and every new downstream route would become a code change plus a deployment. It is the right answer when the edge must *compose* several downstream calls into one response, and Pacco's edge does not: every route forwards to exactly one downstream target or publishes one message. Obligation 3 in §2 preserves this alternative for the day composition is actually needed. |
| 2 | **A code-configured gateway** — the same library, or a middleware pipeline, wired up in C# rather than YAML. | Rejected because it gives up the property that motivated the choice: an edge change becomes a build, an image and a release rather than a configuration change. It would have bought compile-time checking of route definitions, which is exactly what the platform now lacks — so this is the alternative whose absence is felt in §4.2. |
| 3 | **Exposing each service directly**, with no gateway, and letting callers address services by name. | Rejected because it removes the only place where authentication, claim gates and identity binding are enforced (`ADR-006`), and it would publish ten independent public surfaces with no shared error, CORS or documentation behaviour. Note that this alternative is *partially in force anyway*: services publish their own container ports, so a caller inside the network can bypass the edge — a consequence recorded in `ADR-006`, not a property of this decision. |
| 4 | **A service mesh ingress** with routing policy as mesh configuration. | Rejected because no mesh, no container orchestration and no infrastructure-as-code exists in any of the fourteen repositories; the platform runs as compose stacks and a process manager. Adopting mesh ingress would have been adopting a whole deployment platform at the same time. |

## 4. Consequences

### 4.1 Positive

1. ✅ The platform's entire public surface is readable in one document — 455 lines for the synchronous
   edge — without reading any service.
2. ✅ An edge change is a configuration change: a route can be added, gated on a claim, or repointed
   without a build.
3. ✅ Cross-cutting edge behaviour is uniform by construction, because CORS, error shaping, JWT
   validation, Swagger and tracing are extensions configured once rather than per route.
4. ✅ The gateway repository stays tiny — four infrastructure classes and a host — so there is very
   little first-party edge code that can rot.

### 4.2 Negative

1. ✅ **The edge is invisible to the compiler.** A typo in a downstream target, a route that points at
   a service that no longer exposes it, or a missing authentication flag produces no build error.
2. ✅ **No test in the workspace exercises the edge.** The service test suites test services; the
   gateway repository has no test project. `[INFERRED]` The route table's correctness is verified only
   by using it.
3. ✅ **The edge's behaviour is expressed in one library's dialect,** and the platform is pinned to a
   floating version of it. A change in how that dialect is interpreted changes the public API with no
   first-party code involved.
4. ✅ **Four configuration files exist and no document states which one production loads.** Because two
   of them convert twenty write routes from downstream calls to message publications, this is not a
   cosmetic ambiguity — it changes what callers get back. Carried as blocker B2 and decided in
   `ADR-CANDIDATE-005`.
5. ✅ **Credential material is committed inside the configuration.** The JWT extension section of every
   gateway configuration carries a symmetric signing key inline (cited by path and line only, value not
   reproduced). Anyone with repository access holds the key the edge trusts. Carried as blocker B3.

### 4.3 Neutral / follow-on

1. ✅ Registry-driven load balancing is switched off in the gateway configuration, so the edge
   addresses services directly rather than through the router. The discovery registry therefore
   mediates east-west traffic only — analysed in `ADR-CANDIDATE-015`.
2. ✅ The route table is where external contract review has to happen, because it is the only complete
   description of the public surface. That makes obligation 1 in §2 structural rather than procedural.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/integration/declarative-configuration-driven-api-gateway.md` (Status:
   `Candidate`, evidence Strong). Obligations 1–3 in §2 are that pattern's recommendation adopted as
   binding: treat the configuration as a reviewed artifact, record which file each environment loads,
   and do not stretch the pattern to response aggregation.
2. **Enables** `patterns/integration/dual-mode-edge-write.md` — the dual-mode write is only expressible
   because the edge is configuration, and it is the reason obligation 2 matters.
3. **Constrains** `patterns/security/edge-enforced-authentication-with-identity-binding.md`: the
   authentication boundary in `ADR-006` exists inside this configuration and nowhere else.
4. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*).
5. **Pattern Update Proposal:** the pattern file does not state how a declarative edge should be
   *tested*. Since the compiler cannot see this configuration and no test in the workspace covers it, a
   contract-level smoke check per route belongs in the pattern rather than only in this platform's
   backlog.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | The gateway is a host that loads a YAML configuration and delegates everything to the embedded library | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Program.cs:24-49` |
| E2 | The configuration file is selected by command-line argument, then `NTRADA_CONFIG`, then a default file name | ✅ | `…/Program.cs:28-39` |
| E3 | The gateway library and six extension packages are referenced at a floating `0.4.*` version, alongside three platform-toolkit packages | ✅ | `…/Pacco.APIGateway.csproj:10-20` |
| E4 | Four configuration files ship with the image — synchronous and asynchronous, local and Docker variants | ✅ | `…/Pacco.APIGateway.csproj:24-37`; files `ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` |
| E5 | The synchronous configuration is 455 lines and the asynchronous one 534, so the two differ structurally rather than by hostname | ✅ | Line counts of `…/ntrada.yml` and `…/ntrada-async.yml` |
| E6 | The route table declares nine service modules plus a home route, each route naming its upstream path, method, mode and downstream target | ✅ | `…/ntrada.yml:66-455` (module list begins at :66; `availability` at :75, `customers` :129, `deliveries` :189, `identity` :237, `operations` :277, `orders` :292, `parcels` :356, `pricing` :400, `vehicles` :415) |
| E7 | Cross-cutting edge behaviour is configured once as extensions — custom errors, CORS, JWT, Swagger and tracing | ✅ | `…/ntrada.yml:23-64` |
| E8 | Registry-driven load balancing is disabled in the gateway configuration | ✅ | `…/ntrada.yml:19-21` |
| E9 | A symmetric JWT signing key is committed inline in the gateway configuration (value not reproduced here) | ✅ | `…/ntrada.yml:43-48` |
| E10 | The gateway repository contains no controller, no route code and no test project — only a host and four infrastructure classes | ✅ | Full file listing of `hianshul100_Pacco.APIGateway` |
| E11 | Twenty write routes across six services publish to a broker exchange instead of forwarding downstream in the asynchronous configuration | ✅ | `…/ntrada-async.yml:118-528` |

### 6.1 Documentation-versus-code conflicts

1. ✅ **The registry is provisioned platform-wide but does not mediate edge traffic.** All ten services
   register with the discovery registry, while the gateway's load balancing is disabled in all four
   configurations. The code is followed: the registry mediates east-west traffic only. Same conflict as
   recorded in `patterns/deployment/registry-mediated-discovery-and-routing.md`.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The gateway library interprets the configuration as its keys read — an `auth` flag enforces authentication, a `claims` entry gates on a claim, a `downstream` value forwards | The library is a NuGet reference with no source in this workspace, so only the configuration's vocabulary is visible | The public contract described by the route table would differ from what the edge actually enforces, which would also invalidate part of `ADR-006` | Send authenticated, unauthenticated and wrong-role requests to representative routes in a running environment and record the responses |
| A2 | The four configuration files are the complete set the platform can load | All four are copied into the image by the project file, and the selection mechanism accepts any path | An environment could be loading a fifth file held outside the repository, so the reviewed artifact would not be the deployed one | Read `NTRADA_CONFIG` and the mounted files on each running gateway instance |
| A3 | One gateway instance type serves all callers — there is no second edge for internal or partner traffic | Only one gateway repository exists, and only one gateway entry appears in the compose stacks and the process manifests | The claim that this is "the single north-south edge" would be false, and edge-enforced authentication would have a second, undescribed door | Ask the platform owner whether any other public entry point exists, including a load balancer with its own routing rules |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so the routing configuration — the platform's public contract — has no reviewer | Obligation 1 in §2, which requires the configuration to be reviewed as an architectural artifact, and this ADR leaving `Proposed` | Platform owner | Name an owner for the gateway repository and make them a required reviewer on any change to a `ntrada*.yml` file | TBD |
| B2 | **[ACTION NOW]** Nobody has said which gateway configuration each environment loads. The synchronous and asynchronous files give the same twenty write URLs different contracts — one returns a result, the other an acknowledgement | Obligation 2 in §2, any impact analysis on the write path, and the decisions in `ADR-CANDIDATE-005` and `ADR-CANDIDATE-014` | Platform owner | Read `NTRADA_CONFIG` in each environment, record the answer here, and delete or clearly mark the files that are not used | TBD |
| B3 | **[ACTION NOW]** The token-signing key the edge trusts is committed in the repository, in all four configuration files. Anyone holding it can mint a token the gateway accepts, including an administrator one | Any statement that the edge authenticates callers — until this is answered, `ADR-006` records an exposure rather than a control | Platform security owner | Confirm whether the committed value matches anything in a running environment. If it does, rotate first, then move the key to the secret store the platform already runs | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** How should the route table be tested, given that nothing compiles or exercises it? | A typo in a downstream target or a missing authentication flag reaches production silently. This is the direct cost of choosing configuration over code, and no mitigation exists today | Add a smoke suite that calls every declared route once — authenticated, unauthenticated, and with the wrong role where a claim gate exists — and run it against each environment after a configuration change | Platform owner |
| Q2 | **[ACTION NOW]** Should the four configuration files be reduced to one per environment? | Four files that differ in contract, with no record of which is live, is the root of blocker B2. Keeping all four means the ambiguity returns after every deployment | Keep one file per environment plus the deliberate transport choice from `ADR-CANDIDATE-005`, and generate the environment variants rather than maintaining parallel copies by hand | Platform owner |
| Q3 | **[handled later by the design stage]** Should the edge remain purely declarative if a browser client is built? | The current edge assumes a machine caller: no response aggregation, no per-client shaping, and asynchronous writes that return an acknowledgement rather than a result. A browser client would push against all three | Keep the declarative edge and add a separate backend-for-frontend service if aggregation is needed, as obligation 3 in §2 requires — do not extend the YAML dialect to cover it | Platform owner |
