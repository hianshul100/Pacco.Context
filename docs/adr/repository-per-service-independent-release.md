# ADR-018: Repository per service, with independent per-repository release

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-018` |
| **Backlog candidate** | `ADR-CANDIDATE-018` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Deployment / medium |
| **Supersedes / Superseded by** | — (nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-002` (no shared library, which this makes possible), `ADR-017` (what receives these images), `ADR-003` (the contract exposure this is the release-side half of), `ADR-020` (the runtime pin this model makes expensive to move) |

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

Eleven deployables live in eleven repositories, plus a platform repository holding environment
definitions and an artifact repository holding this documentation. Each deployable releases on its own.

1. ✅ **Eleven repositories carry a pipeline; three do not.** The gateway and the ten services each hold
   a Travis configuration. `hianshul100_Pacco` (platform), `hianshul100_Pacco.Web` and
   `hianshul100_Pacco.Context` hold none.
2. ✅ **The pipeline is copied, not shared.** All eleven configurations are byte-identical apart from a
   single line, pin the same SDK version, and trigger on the same two branches. There is no template,
   no reusable workflow and no shared pipeline definition anywhere.
3. ✅ **Each pipeline runs the same three scripts,** each a thin wrapper: build compiles in release
   configuration, test runs the solution's tests, and the dockerize step builds and pushes an image
   (`hianshul100_Pacco.Services.Orders/scripts/build.sh`, `test.sh`, `dockerize.sh`).
4. ✅ **`availability-service` is the one repository whose pipeline never runs its tests.** Its
   configuration invokes only the build script; the other ten invoke build and then test
   (`hianshul100_Pacco.Services.Availability/.travis.yml:12-13` against
   `hianshul100_Pacco.Services.Orders/.travis.yml:12-14`). The script itself exists and is executable
   — it is present in that repository's `scripts/` directory and is simply never called. This is the
   repository with **five test projects**, by far the platform's richest suite; every other repository
   has one or none.
5. ✅ **Images are tagged from the branch and the build number, and pushed to a personal namespace.**
   The main branch produces a `latest` tag plus a numeric build tag; the development branch produces a
   `dev` tag. The image repository is composed from a Docker Hub username supplied as a pipeline
   environment variable, not from an organisation or a private registry
   (`hianshul100_Pacco.Services.Orders/scripts/dockerize.sh:17-22`).
6. ✅ **Nothing deploys.** Every pipeline ends at the image push. No repository contains a deployment
   step, an environment promotion, or a release manifest update.
7. ✅ **An aggregate solution stitches the repositories together by relative path.** The platform
   repository holds a solution referencing 41 project files across sibling directories, each addressed
   as `..\<repo>\src\…`. It only resolves if every clone sits in one parent directory under its exact
   default name (`hianshul100_Pacco/Pacco.sln`).
8. ✅ **That aggregate solution references a repository that does not exist.** It declares a
   `Pacco.APIGateway.Ocelot` project at `..\Pacco.APIGateway.Ocelot\src\…`, and no such repository is
   in this workspace or in the request's repository list
   (`hianshul100_Pacco/Pacco.sln:152-154`). The solution cannot load as committed.

### 1.1 What this record does *not* cover

It does not cover what happens to an image after it is pushed — that is `ADR-017`, which records that
no orchestration exists. It does not cover the message contracts that a coordinated release would exist
to protect, which is `ADR-003`, nor the runtime version every pipeline pins, which is `ADR-020`.

## 2. Decision

**Every deployable owns its own repository, its own solution, its own container image and its own
release. A repository builds, tests and publishes without reference to any other, on push to its main
or development branch, and no mechanism exists — or is intended to exist — for releasing several
services together. Cross-service compatibility is the responsibility of the change author, not of the
release process.**

Four obligations follow, and each closes a gap the evidence above exposes:

1. 🎯 **Every pipeline must run its own tests.** One of eleven does not, and it is the one with the
   most tests to run.
2. 🎯 **Images must be published to a namespace the platform owns.** The registry namespace is
   currently a personal account name injected at build time.
3. 🎯 **The aggregate solution must either be repaired or deleted.** As committed it references a
   repository that does not exist, so it is a convenience that does not work.
4. 🎯 **A change that alters a message contract must name every repository that consumes it, in the
   change description.** This is the only compensating control available: `ADR-003` provides no
   build-time signal and this record provides no coordinated release.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **A monorepo** holding all deployables with one pipeline and one dependency graph. | Rejected because it couples release cadence to repository membership: every team waits on one build, and a change anywhere can turn every service's pipeline red. It would also have undercut `ADR-002` — with everything in one tree, a shared `Pacco.Common` library becomes the obvious move, and the platform deliberately has none. The cost of rejecting it is stated plainly in §4.2: the eleven-fold duplication and the absent coordinated release are both direct consequences. |
| 2 | **A coordinated multi-service release train** — repositories stay separate, but a release orchestrator promotes a versioned set together. | Rejected in practice, and this is the alternative whose absence hurts most. It would have supplied exactly what `ADR-003` needs: a way to ship a publisher and its consumers as one unit when a contract changes. It requires a release orchestrator and a compatibility matrix, neither of which exists. Constraint C3 in the architecture baseline states the resulting position: coordinated releases are prevented because no mechanism exists to perform one. |
| 3 | **A shared pipeline template** — separate repositories, but one reusable workflow definition consumed by all eleven. | Rejected, or more likely never considered: Travis's support for shared configuration is limited, and the eleven files are short enough that copying was cheaper than abstracting. The consequence is that improving the pipeline — adding a security scan, a coverage gate, or the missing test step from obligation 1 — is an eleven-repository change. |
| 4 | **Publishing versioned contract packages** per service, so consumers resolve message types by dependency rather than by convention. | Rejected as a consequence of `ADR-002` and `ADR-003` rather than on its own merits. It would have made a contract change a visible, versioned, build-time event. It was traded for the freedom to change a publisher without waiting on any consumer. |

## 4. Consequences

### 4.1 Positive

1. ✅ No team blocks another. A change to one service builds, tests and publishes without any other
   repository being green.
2. ✅ Each repository has a small, comprehensible history scoped to one deployable, and a build that
   completes in the time it takes to compile one solution.
3. ✅ The blast radius of a bad pipeline change is one repository.
4. ✅ Image tags carry the build number alongside the moving tag, so a specific build remains
   addressable after `latest` moves on.

### 4.2 Negative

1. ✅ **There is no way to release two services together, even when a contract change requires it.**
   Combined with `ADR-003`, which provides no build-time contract signal, a breaking change is
   detectable only at runtime and repairable only by two independent releases in the right order —
   an order nothing records or enforces.
2. ✅ **The pipeline is duplicated eleven times with no template**, so any improvement is an
   eleven-repository change, applied by hand, with no signal if one is missed. Finding 4 in §1 is that
   failure mode already realised: ten files gained a test step and one did not.
3. ✅ **The platform's best-tested service is the one whose tests never run in CI.** Five test projects
   in `availability-service` — unit, integration, end-to-end, performance and shared fixtures — are
   compiled by the build and then never executed by the pipeline.
4. ✅ **Release provenance depends on a personal registry account.** The namespace comes from a build
   environment variable naming a Docker Hub user; ❓ whether that account is individual or shared is
   not observable from the repositories.
5. ✅ **The aggregate solution is broken as committed** and would fail to load for anyone following it,
   because one referenced repository does not exist (§1 item 8).
6. ✅ **No repository records which versions of its peers it was tested against.** There is no
   compatibility matrix, no lock file across services, and no environment manifest pinning image tags.

### 4.3 Neutral / follow-on

1. ✅ The pipeline triggers on both a main and a development branch, producing distinguishable tags —
   the only environment-shaped distinction the release process makes.
2. ✅ `hianshul100_Pacco.Web` holds no pipeline because it holds no code, consistent with its being an
   empty placeholder rather than an omission.
3. ✅ The platform repository has no pipeline of its own, so the environment definitions in
   `ADR-017` are never validated by any automated step.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/deployment/independent-per-repository-release.md` (Status: `Candidate`,
   evidence Strong), and **constrains** it with the four obligations in §2.
2. **Enabled by** `patterns/other/framework-supplied-platform-conventions.md`: independent release is
   only affordable because no first-party shared library needs coordinated versioning (`ADR-002`).
3. **Constrains** `patterns/testing/layered-service-test-suite.md` in a way that pattern does not
   anticipate. The pattern is derived entirely from `availability-service`, which is the one repository
   whose pipeline does not run tests — so the platform's model test suite has no enforcement behind it.
4. **Aggravates** `patterns/testing/consumer-driven-contract-test-pair.md`: the pact pair exists to
   catch exactly the breakage this release model permits, and neither repository configures a broker to
   carry the pact between them, so it catches nothing across the boundary.
5. **Feeds** `patterns/deployment/composable-per-concern-environment-stacks.md`, which consumes the
   images these pipelines publish.
6. **Pattern Drift:** none reportable. Drift requires an `Approved` pattern to violate, and every entry
   in `patterns/index.md` carries status `Candidate` (*Governance*).
7. **Pattern Update Proposal:** the release pattern should state that an independent-release model
   requires two compensating controls it currently omits — every pipeline runs its own tests, and every
   contract-affecting change enumerates its consumers. It should also record the duplication cost
   explicitly, with the missing test step as the worked example of how copied pipelines drift.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | Eleven repositories carry a pipeline; the platform, web and artifact repositories carry none | ✅ | `.travis.yml` present in `hianshul100_Pacco.APIGateway` and the ten `hianshul100_Pacco.Services.*` clones; absent from `hianshul100_Pacco`, `hianshul100_Pacco.Web`, `hianshul100_Pacco.Context` |
| E2 | All eleven pipeline files are identical except for one line, and pin the same SDK version | ✅ | Line-by-line comparison of the eleven `.travis.yml` files — the only difference is the presence of the test script |
| E3 | `availability-service`'s pipeline invokes the build script and stops | ✅ | `hianshul100_Pacco.Services.Availability/.travis.yml:12-13` |
| E4 | Every other pipeline invokes build and then test | ✅ | `hianshul100_Pacco.Services.Orders/.travis.yml:12-14`, and the same two lines in the other nine |
| E5 | `availability-service` does hold the test script it never runs | ✅ | `hianshul100_Pacco.Services.Availability/scripts/` — contains `build.sh`, `test.sh`, `dockerize.sh`, `start.sh` |
| E6 | `availability-service` holds five test projects; no other repository holds more than one | ✅ | `hianshul100_Pacco.Services.Availability/tests/` — Unit, Integration, EndToEnd, Performance, Shared; `Orders` and `Parcels` hold one pact project each; the remaining eight hold none |
| E7 | The build and test scripts are one-line wrappers over the SDK | ✅ | `hianshul100_Pacco.Services.Orders/scripts/build.sh`; `…/scripts/test.sh` |
| E8 | Image tags derive from the branch and the build number | ✅ | `hianshul100_Pacco.Services.Orders/scripts/dockerize.sh:4-15` |
| E9 | The image namespace is a Docker Hub username supplied as a build environment variable | ✅ | `hianshul100_Pacco.Services.Orders/scripts/dockerize.sh:17-22` |
| E10 | No pipeline contains a deployment or promotion step; each ends at the image push | ✅ | `after_success` in all eleven files invokes the dockerize script and nothing further |
| E11 | The aggregate solution references 41 project files across sibling repositories by relative path | ✅ | `hianshul100_Pacco/Pacco.sln` |
| E12 | The aggregate solution references a `Pacco.APIGateway.Ocelot` repository that is not in the workspace | ✅ | `hianshul100_Pacco/Pacco.sln:152-154`; no matching directory exists and no such repository appears in the request's repository list |
| E13 | Each deployable builds its own image from its own repository root | ✅ | One `Dockerfile` at the root of each of the eleven deployable repositories, each publishing that repository's own entry-point project |

### 6.1 Documentation-versus-code conflicts

1. ✅ **The claim that no CI pipeline exists is wrong.**
   `docs/architecture-inventory/architecture-views.md` §4.5 states that no CI pipeline definition
   exists; eleven Travis configurations show otherwise (E1). The code is followed. The CD half of that
   claim stands and is confirmed here: no pipeline deploys anything (E10). This is conflict X1 in
   `docs/architecture-inventory/baselines/architecture-baseline.md` §11.3.
2. ✅ **The pipelines are not identical.**
   `docs/architecture-inventory/adr-candidates.md` (candidate 018) and
   `docs/architecture-inventory/repo-inventory.md` §2.3 describe "the same identical three-script
   pipeline" in all eleven repositories. Ten run three scripts; `availability-service` runs two (E3,
   E4). The exception matters more than the rule, so this record states it in §1 and §4.2 rather than
   averaging it away.
3. ✅ **The aggregate solution's relative-path fragility is recorded, but not its broken reference.**
   `docs/architecture-inventory/repo-inventory.md` §7 notes that the solution resolves only if clones
   share a parent directory. It does not record that one referenced repository does not exist at all
   (E12), which makes the solution unloadable regardless of layout.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The committed pipeline files describe the release process actually in use | They are the only build definition in any repository, and each produces the image name that the environment stacks in `ADR-017` consume | The release model recorded here would describe something nobody runs, and every obligation in §2 would target the wrong system | Ask the platform owner which CI service builds these repositories today, and compare a running image's tag against a build number |
| A2 | `availability-service`'s missing test step is an oversight rather than a deliberate exclusion | The test script is present and executable in that repository (E5), and the same two lines appear in all ten other files (E4) — the shape of a copy that was not updated | Obligation 1 in §2 would be arguing against a decision rather than fixing a defect, and there would be a reason nobody has recorded — most likely a slow or flaky suite | Ask the platform owner, or run that repository's test suite locally and observe whether it passes and how long it takes |
| A3 | The Docker Hub namespace is reachable by whoever needs to deploy, whatever account it names | Images are pushed there by every pipeline and pulled by the environment stacks, so the path evidently works today | Release provenance would depend on an individual's account, and losing that access would break every deployment path at once | Ask the platform owner who owns the registry account named by the build variable, and whether it is individual or organisational |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no obligation in §2 can be assigned and no reviewer can accept the contract-change discipline that obligation 4 depends on | This ADR leaving `Proposed`, and obligation 4 in particular — it is a review practice, and review practices need reviewers | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has confirmed who controls the container registry account the pipelines push to. Every deployable image on the platform lands in a namespace named by a build-time variable | Obligation 2 in §2, and any statement about release provenance. If that account belongs to an individual who leaves, every pipeline's publish step breaks and no image can be pulled | Platform owner | Identify the account, move the images to an organisational namespace, and rotate the registry credential held in the CI configuration | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** How does a breaking message-contract change get shipped safely, given that no coordinated release is possible? | This is the central cost of the decision. `ADR-003` gives no build-time signal and this record gives no coordinated release, so a rename in a publisher breaks its consumers silently at runtime with nothing in between | Require additive-only contract changes: publish the new message alongside the old, migrate consumers one release at a time, and retire the old one only after every consumer is confirmed off it. Record the consumer list in the change description, per §2 obligation 4 | Platform owner |
| Q2 | **[ACTION NOW]** Should `availability-service`'s pipeline start running its tests? | The platform's only substantial test suite — five projects covering both entry paths — is compiled and discarded on every build. Every other repository runs what it has | Yes. Add the test step, and if the suite proves slow or flaky, split the fast levels into the pipeline and schedule the rest, rather than running none | Platform owner, with the owner of `availability-service` |
| Q3 | **[ACTION NOW]** Should the aggregate solution be repaired or deleted? | As committed it references a repository that does not exist, so it cannot load for anyone (§1 item 8). A convenience that fails on first use is worse than no convenience, and it implies a twelfth repository that may or may not still be intended | Delete it. Its only function is opening every service at once, which contradicts the per-repository model this record fixes; if a gateway alternative is genuinely planned, record that as its own decision instead | Platform owner |
| Q4 | **[handled later by the devops stage]** Should the eleven pipelines be replaced by one shared, reusable definition? | The duplication has already produced one divergence that costs the platform its best test coverage (§4.2 items 2 and 3), and every future pipeline improvement pays the same eleven-fold cost | Yes, once a CI platform with reusable workflows is chosen. Keep the per-repository *trigger* and independent release, which is the part this record decides, and share only the definition | Platform owner |
