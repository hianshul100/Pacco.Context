# ADR-020: .NET Core 3.1 as the platform runtime baseline

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-020` |
| **Backlog candidate** | `ADR-CANDIDATE-020` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Other / high |
| **Supersedes / Superseded by** | — (nothing to supersede). 🎯 This record is written to be superseded: see §2 |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-002` (the toolkit whose version line is coupled to this pin), `ADR-018` (why moving the pin is an eleven-repository change with no coordinated release), `ADR-017` (the images this runtime ships in) |

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

One runtime version is pinned across the whole platform, in four independent places per deployable, with
no exception anywhere.

1. ✅ **Every project targets the same framework.** All **40** project files in the workspace — across
   eleven deployable repositories, covering source and test projects alike — declare
   `netcoreapp3.1`. Not one targets anything else.
2. ✅ **Every pipeline pins the same SDK patch version.** All eleven Travis configurations request SDK
   `3.1.100` (`hianshul100_Pacco.Services.Availability/.travis.yml:5`).
3. ✅ **Every container image pins the same base images.** All eleven Dockerfiles build on the .NET Core
   3.1 SDK image and run on the 3.1 ASP.NET runtime image
   (`hianshul100_Pacco.Services.Orders/Dockerfile:1, 6`).
4. ✅ **The pin is fourfold and mutually reinforcing.** Changing the runtime for one service requires
   editing its project files, its pipeline, and both stages of its Dockerfile — and, because no
   coordinated release exists (`ADR-018`), doing that eleven times independently.

The decisive fact is not in the repositories:

5. ✅ **.NET Core 3.1 reached end of support on 13 December 2022.** The platform has therefore been on
   an unsupported runtime for over three and a half years as of this record's date. It receives no
   security patches, and the base images it builds on are no longer updated.
6. ✅ **The SDK pin is to an exact patch, `3.1.100`** — the first 3.1 SDK release — rather than to a
   floating 3.1 line, so builds do not pick up even the patches that were published before support
   ended.
7. ❓ **Nothing in the workspace acknowledges this.** No repository holds an upgrade branch, a
   dependency-update configuration, a deprecation note or a tracked work item referencing the runtime.

The pin is also coupled outward: `ADR-002` records the platform's real standard as the external toolkit
composed into every service, and that toolkit's version line is constrained by the runtime it targets.
Moving one implies moving the other.

### 1.1 What this record does *not* cover

It does not cover the toolkit version itself, which is `ADR-002`, nor the release mechanism that makes a
platform-wide change expensive, which is `ADR-018`. It does not choose the target runtime version —
that is question Q1, and deliberately not decided here.

## 2. Decision

**Every Pacco deployable targets one runtime version, pinned identically in its project files, its
pipeline SDK version and both stages of its container image. No service selects its own runtime.**

That is the decision as it stands, and it is recorded as a current-state fact rather than a
ratification. **This record does not endorse remaining on .NET Core 3.1.** The runtime named in the
pin has been out of support since 13 December 2022, so the decision as implemented now delivers the
opposite of what it was taken for: instead of guaranteeing that every service is on a known-good
runtime, it guarantees that every service is on an unsupported one simultaneously.

Four obligations are attached, and the first is the point of the record:

1. 🎯 **The platform must move to a supported runtime, and this ADR must then be superseded** by one
   naming the new version and the date it was adopted.
2. 🎯 **The single-version rule survives the move.** Uniformity is the part worth keeping; the version
   is the part that must change. A migration may run mixed versions temporarily, but not
   indefinitely, and the end state is one version again.
3. 🎯 **The pin must be visible in one place per repository, not four.** Four independent pins are four
   chances to migrate a service partially — for example a project file moved while its Dockerfile base
   image is not.
4. 🎯 **The runtime version must have a scheduled review before its support end date,** rather than
   being discovered years afterwards by an architecture review.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Per-service runtime choice**, letting each team upgrade on its own schedule. | Rejected, and this is the alternative microservices are usually adopted to enable. It was given up for uniformity: one runtime means one set of build images, one toolkit version line (`ADR-002`), and no per-service compatibility matrix. The cost has now inverted — with independent release and no coordinated mechanism (`ADR-018`), per-service runtimes would have let services migrate one at a time instead of stranding all eleven together. This is the alternative most worth revisiting during the upgrade, and it is question Q2. |
| 2 | **Track the latest LTS continuously**, upgrading the whole platform on each LTS release. | Rejected in practice rather than on merit — no upgrade has happened. It is the alternative obligations 1 and 4 in §2 effectively adopt going forward. Its cost is a recurring eleven-repository change every two years; its benefit is that the platform is never unsupported, which is precisely what has been lost. |
| 3 | **Pin a floating 3.1 SDK line** rather than the exact patch `3.1.100`. | Rejected by the current configuration, apparently for build reproducibility. It is the weaker version of the same trade-off: it would have collected 3.1 security patches automatically until support ended, at the cost of builds that change without a commit. It does not solve the end-of-support problem, only delays part of it. |
| 4 | **Publish self-contained deployments**, bundling the runtime with each service so the base image no longer determines it. | Rejected implicitly; every Dockerfile uses a framework-dependent publish onto a runtime base image. It would remove one of the four pin locations but not the underlying problem — a bundled unsupported runtime is still unsupported, and image size and patch cadence both get worse. |

## 4. Consequences

### 4.1 Positive

1. ✅ No compatibility matrix is needed. Any service can consume any other's contracts and any shared
   package version without a runtime check, because there is only one runtime.
2. ✅ Build and base images are identical across eleven repositories, so image layers cache well and a
   developer's local SDK works for every repository.
3. ✅ The exact SDK patch pin makes builds reproducible — the same source produces the same output
   regardless of when the pipeline runs.
4. ✅ Because the pin is uniform, the *scope* of an upgrade is unambiguous: forty project files, eleven
   pipelines and eleven Dockerfiles, with no service needing individual assessment.

### 4.2 Negative

1. ✅ **The entire platform runs on a runtime that has received no security patches since December
   2022.** Every deployable is affected identically; there is no partially-exposed subset and no
   service that could be treated as safe.
2. ✅ **The base images are equally stale.** The SDK and ASP.NET 3.1 images are no longer rebuilt, so
   operating-system-level vulnerabilities in those layers also go unpatched.
3. ✅ **The upgrade is an eleven-repository change with no mechanism to perform it as one.**
   `ADR-018` records that no coordinated multi-service release exists, so the migration must proceed
   service by service, in an order nobody has defined.
4. ✅ **The toolkit constrains the move.** `ADR-002` pins an external toolkit at a version line
   targeting this runtime; ❓ whether a version of it supports a modern runtime is not observable from
   this workspace, and if none does, the runtime upgrade becomes a toolkit migration as well.
5. ✅ **Four pin locations per repository invite a partial migration.** Nothing checks that a project
   file's target framework and its Dockerfile's base image agree, so a half-migrated service would
   build and might not be noticed until runtime.
6. ✅ **Nothing tracks the problem.** No upgrade branch, dependency-update configuration or
   deprecation note exists in any repository, so the end-of-support date passed with no recorded
   response.

### 4.3 Neutral / follow-on

1. ✅ Test projects share the pin, so the test suites will need migrating alongside the source — which
   makes `ADR-018`'s finding that `availability-service` never runs its tests materially worse: the
   platform's richest suite would not be exercised during the one change that most needs it.
2. ✅ The pin is genuinely uniform, with no exception anywhere in forty project files. Whatever else
   is true, the migration has no special cases to discover.

## 5. Relationship to the implementation pattern catalog

1. **Constrains** `patterns/other/framework-supplied-platform-conventions.md` (Status: `Candidate`,
   evidence Strong). That pattern names the external toolkit as the platform's real standard; this
   record fixes the runtime that bounds which toolkit versions are available, so the two must move
   together.
2. **Constrained by** `patterns/deployment/independent-per-repository-release.md`, which is what makes
   the upgrade expensive: eleven independent releases with no way to ship them as one (§4.2 item 3).
3. **Affects** `patterns/deployment/composable-per-concern-environment-stacks.md`, since every image
   those stacks pull is built on the pinned base images.
4. **Affects** `patterns/testing/layered-service-test-suite.md`, whose five projects share the pin and
   would need migrating with the source (§4.3 item 1).
5. **Pattern Drift:** none reportable. Drift requires an `Approved` pattern to violate, and every entry
   in `patterns/index.md` carries status `Candidate` (*Governance*).
6. **Pattern Update Proposal:** the framework-conventions pattern should record the runtime version
   alongside the toolkit version as a single coupled platform baseline, and should carry the support
   end date of both. A pattern that names a standard without naming when that standard expires is how
   a platform arrives at three and a half years past end of support with nothing tracking it.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | All 40 project files in the workspace target `netcoreapp3.1`, with no exception | ✅ | `<TargetFramework>` across every `.csproj` in the eleven deployable repositories — 40 files, 40 matches, one distinct value |
| E2 | All eleven pipelines pin SDK `3.1.100` | ✅ | `dotnet:` in each of the eleven `.travis.yml` files, e.g. `hianshul100_Pacco.Services.Availability/.travis.yml:5` |
| E3 | Every container image builds on the .NET Core 3.1 SDK image | ✅ | `hianshul100_Pacco.Services.Orders/Dockerfile:1`, and line 1 of the other ten deployable Dockerfiles |
| E4 | Every container image runs on the ASP.NET 3.1 runtime image | ✅ | `hianshul100_Pacco.Services.Orders/Dockerfile:6`, and line 6 of the other ten |
| E5 | Test projects share the same pin as source projects | ✅ | `<TargetFramework>` in the five test projects of `hianshul100_Pacco.Services.Availability` and the pact projects of Orders and Parcels |
| E6 | The pin appears in four independent places per repository | ✅ | Project files (E1), pipeline SDK (E2), image build stage (E3), image runtime stage (E4) |
| E7 | .NET Core 3.1 reached end of support on 13 December 2022 | ✅ | Microsoft's published .NET support lifecycle. External to this workspace, cited as a dated public fact rather than inferred from source |
| E8 | No upgrade branch, dependency-update configuration or deprecation note exists in any repository | ✅ | Workspace-wide search across all fourteen clones for dependency-update configuration and runtime deprecation notes — zero matches |
| E9 | The toolkit is pinned at a version line contemporaneous with this runtime | ✅ | Toolkit package references in every deployable `.csproj`, recorded in `ADR-002` |
| E10 | Every deployable publishes framework-dependent output onto a runtime base image rather than self-contained | ✅ | The publish command in each of the eleven Dockerfiles, e.g. `hianshul100_Pacco.Services.Orders/Dockerfile:4` |

### 6.1 Documentation-versus-code conflicts

1. ✅ **No conflict was found for this record.** `docs/architecture-inventory/repo-inventory.md` §2.1
   and §2.3, `docs/architecture-inventory/baselines/architecture-baseline.md` §1.3 and constraint C9
   all state the runtime pin and its end-of-support date, and the source agrees in all four pin
   locations (E1–E4). This is recorded explicitly because the agreement is itself informative: the
   platform's own documentation already names the problem, and no repository has acted on it (E8).
2. ✅ **One count is stated more precisely here than in the baselines.** Those documents say the
   runtime is "targeted by every project file" without a number; the count is **40** project files
   (E1). The figure is given so the upgrade's scope is unambiguous.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The committed source reflects what is deployed, so the unsupported runtime is running and not merely committed | Every pipeline builds an image from these Dockerfiles and pushes it under a branch tag, and no other build path exists in any repository (`ADR-018`) | The exposure in §4.2 items 1 and 2 would be about dormant code rather than running services, which changes the urgency entirely but not the need to upgrade | Inspect the running images in each environment and report the .NET version each container actually reports |
| A2 | A version of the external toolkit exists that supports a currently-supported runtime | The toolkit is actively used and this is the ordinary lifecycle for a .NET library, but its source and release history are not in this workspace | The runtime upgrade becomes a toolkit replacement as well, which is a far larger change touching the composition root of all eleven deployables (`ADR-002`) | Check the toolkit's published package versions and their target frameworks before planning the upgrade — this is the first task, not a later one |
| A3 | No compliance or contractual obligation is already being breached by running an unsupported runtime | Nothing in the workspace names a compliance regime, an audit scope or a customer commitment | The upgrade is not a backlog item but an obligation with a deadline someone else has already set, and blocker B2 becomes urgent rather than important | Ask the platform owner whether this platform is in scope for any security audit, certification or customer commitment |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no one can accept the upgrade obligation in §2 or sequence a migration across eleven repositories | This ADR leaving `Proposed`, and every obligation in §2 — an upgrade spanning eleven independently released repositories cannot start without someone accountable for the order | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Every deployable is running a runtime that stopped receiving security patches on 13 December 2022, and nothing in any repository tracks it. This is a live exposure, not a design preference | Any claim that the platform is patchable. It also blocks obligation 1 in §2 from being scheduled, because no one has assessed what the exposure currently is | Platform security owner | Confirm what is deployed (assumption A1), review published advisories affecting .NET Core 3.1 and the 3.1 base images, and decide whether the upgrade is scheduled work or incident response | TBD |
| B3 | **[ACTION NOW]** Nobody has confirmed that the external toolkit has a version supporting a modern runtime. If it does not, the runtime upgrade is also a framework migration touching every service's composition root | Planning the upgrade at all — the scope differs by an order of magnitude between the two cases, so no estimate or sequence can be produced until this is known | Platform owner | Check the toolkit's package versions and target frameworks, and record the finding against `ADR-002` before any migration work is scoped | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Which runtime version should the platform move to? | This record deliberately does not choose one, because the choice depends on the toolkit's support (B3) and on how long the platform wants before the next forced move. Choosing wrongly means doing an eleven-repository migration twice | Move to the newest LTS release that the toolkit supports, so the next scheduled review (§2 obligation 4) is as far out as possible. Confirm the toolkit's support first — B3 gates this question | Platform owner |
| Q2 | **[ACTION NOW]** Should the single-version rule be kept after the upgrade, or should services be allowed to move independently? | Uniformity is why all eleven are stranded together today. Allowing per-service versions would let the migration proceed incrementally and would prevent a repeat, at the cost of a compatibility matrix the platform has never needed | Keep the single-version rule as the steady state (§2 obligation 2), but permit a mixed state explicitly during a migration window with a stated end date. That gets the incremental migration without permanently accepting eleven runtimes | Platform owner |
| Q3 | **[ACTION NOW]** In what order should the eleven repositories be migrated? | `ADR-018` provides no coordinated release, so the order is the plan. A wrong order can leave a publisher and its consumers on different runtimes mid-flight with no build-time signal (`ADR-003`) | Migrate leaf services with no synchronous dependents first — `deliveries-service`, `vehicles-service`, `parcels-service` — then their callers, then the gateway last, since it fronts everything. Derive the exact order from the seven synchronous edges in `ADR-010` | Platform owner |
| Q4 | **[handled later by the devops stage]** How should the four pin locations per repository be reduced to one? | Four independent pins are four chances to migrate a service partially, and nothing checks that they agree (§4.2 item 5) | Put the target framework in a single build properties file per repository, and derive the pipeline SDK and the image base tags from it or check them against it. Add the check to the pipeline while the shared-definition work in `ADR-018` question Q4 is being done | Platform owner |
| Q5 | **[handled later by the devops stage]** What triggers the runtime review before the next end-of-support date? | The current position was reached because nothing watched the date. Fixing the version without fixing the watch reproduces the problem in a few years (§2 obligation 4) | Record the chosen runtime's support end date in this repository when this ADR is superseded, and schedule a review at least twelve months before it. Add automated dependency-update configuration to all eleven repositories at the same time | Platform owner |
