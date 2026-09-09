# ADR-002: Convey as the platform standard in place of a shared internal library

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-002` |
| **Backlog candidate** | `ADR-CANDIDATE-002` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Other (platform convention) / high |
| **Supersedes / Superseded by** | — (first ADR corpus on this platform; nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-001` (messaging), `ADR-008` (persistence), `ADR-004` (edge) — all three are composed through the mechanism this record fixes |

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

Pacco is eleven independently released deployables in eleven repositories, plus a platform repository
and this artifact repository. Every one of those deployables needs the same set of cross-cutting
capabilities: command and query dispatch, message publication and subscription, persistence, service
registration, load-balanced HTTP clients, secret loading, structured logging, tracing and metrics.

The usual answer on a platform this shape is a first-party shared library — a `Pacco.Common` package
that owns those concerns and is versioned once for everyone. Pacco does not have one. What it has
instead is:

1. ✅ **One external toolkit, `Convey`, referenced at a floating minor version by every deployable.**
   267 `Convey.*` package references appear across the 40 project files in the workspace, every one of
   them pinned to `0.4.*` rather than to an exact version, and no lock file exists anywhere
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Pacco.Services.Availability.Infrastructure.csproj`;
   `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Pacco.APIGateway.csproj:10-20`).
2. ✅ **No first-party shared library and no shared code repository.** No project in any repository
   holds a `ProjectReference` that crosses a repository boundary, and no package named for Pacco is
   published or consumed. The only artifact in the workspace with "Shared" in its name is a test
   fixture project internal to one service
   (`hianshul100_Pacco.Services.Availability/tests/Pacco.Services.Availability.Tests.Shared`).
3. ✅ **A composition root copied per repository rather than depended upon.** Each layered service
   carries an `Infrastructure/Extensions.cs` whose `AddInfrastructure` method is a chained list of
   toolkit registrations, and the same file names recur across services: `Contexts/`, `Decorators/`,
   `Exceptions/`, `Logging/`, `Mongo/`
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Extensions.cs:55-94`).

The uniformity of the platform is therefore real but unenforced: it is produced by copying a file into
each repository, not by a dependency that a build can check. That is the situation this record fixes in
writing, because it is the premise of almost every other decision in the backlog.

### 1.1 What this record does *not* cover

This record fixes *how* cross-cutting capability arrives in a service. It does not fix *what* each
capability does with it: the messaging topology is `ADR-001`, persistence is `ADR-008`, the edge is
`ADR-004`, and the authentication boundary is `ADR-006`. The runtime version those packages target is a
separate decision, carried in the backlog as `ADR-CANDIDATE-020`.

## 2. Decision

**Pacco takes its cross-cutting capabilities from the external `Convey` toolkit, composed per
repository in a hand-maintained composition root, and deliberately publishes no first-party shared
library.** Every deployable declares only the `Convey.*` packages it actually uses, registers them in
its own `Infrastructure/Extensions.cs`, and configures them through its own `appsettings.json`. No
Pacco-owned code is shared between repositories at build time, in either source or package form.

Two obligations are attached to that decision and are part of it:

1. 🎯 **Package versions must be pinned to exact versions with a lock file.** The floating `0.4.*`
   range means no build in this platform is reproducible and no environment's dependency set is
   knowable. This is target state, not current behaviour.
2. 🎯 **Semantics that define identity, roles or ownership must not be copied.** The copied files are
   not only plumbing — they include the caller-context and identity types on which every ownership
   check depends (`ADR-006`), and those copies have already begun to diverge. Those specific types
   should move into one versioned package while composition stays per repository.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **A first-party `Pacco.Common` shared library owning dispatch, messaging, persistence and context types.** | Rejected because it re-couples the release model. Every deployable releases independently from its own repository with its own pipeline (eleven `.travis.yml` files, no shared template), and a shared library reintroduces the coordinated release that model exists to avoid: one library change becomes an eleven-repository upgrade with an ordering constraint. It is also the alternative the platform is closest to needing — it is what obligation 2 above partially reverses — but as a whole-platform answer it buys consistency at the price of the autonomy that motivated repository-per-service in the first place. |
| 2 | **Wire each capability directly against its own primitive library** (the RabbitMQ client, the MongoDB driver, the tracing client) with no toolkit in between. | Rejected because it multiplies the plumbing without removing the copying problem. The service would still need a composition root, still need the same configuration sections, and would now also own retry, convention, decorator and outbox behaviour that the toolkit currently supplies. The result is more first-party code to keep uniform across eleven repositories, not less. |
| 3 | **Source-generate or template the composition root** from a central definition, so the copies are produced rather than hand-maintained. | Rejected as a current-state option because nothing in the workspace generates anything: there is no template, no scaffolding tool and no code generator in any repository. It remains a reasonable evolution of obligation 2 and is noted as a pattern update proposal in §5, but it is not what the platform does today and this record does not claim it. |

## 4. Consequences

### 4.1 Positive

1. ✅ No coordinated release is required for a cross-cutting change: each repository upgrades its own
   packages on its own schedule, and no repository can block another.
2. ✅ A service's entire cross-cutting configuration is readable in one screen — one line per
   capability in `Extensions.cs`, one settings section per line in `appsettings.json`.
3. ✅ Each deployable declares only what it uses, so capability differences between services are
   visible in the project file rather than hidden behind a shared package that registers everything.

### 4.2 Negative

1. ✅ **Drift has no build-time signal.** Because uniformity comes from copies, a service that diverges
   from the platform convention still compiles. The divergence is only findable by reading eleven
   repositories.
2. ✅ **No build is reproducible.** 267 floating package references with no lock file mean two builds of
   the same commit can resolve different toolkit versions.
3. ✅ **The toolkit's version line is the platform's real architectural standard.** Anything `Convey
   0.4.*` cannot do, the platform cannot do without writing it eleven times.
4. `[INFERRED]` **A defect in a copied file is a defect in every copy.** The ownership guard analysed in
   `ADR-006` is the concrete instance: the same faulty condition appears independently in four handler
   files across two repositories, and fixing one fixes nothing else.

### 4.3 Neutral / follow-on

1. Upgrading the toolkit is a per-repository act with no platform-wide view of which repository is on
   which resolved version. ❓ No inventory of resolved versions per environment exists in the workspace.
2. `[INFERRED]` The toolkit is a single third-party dependency for the whole platform. Its maintenance
   status is a platform-level risk that no repository records; carried as question Q2.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/other/framework-supplied-platform-conventions.md` (Status: `Candidate`,
   evidence Strong). That pattern records exactly this approach and recommends adopting the composition
   style while fixing the versioning — the two obligations in §2 are that recommendation restated as
   part of the decision rather than as advice.
2. **Constrains** every other pattern in the catalog. Twenty-six of the twenty-seven catalogued patterns
   are implemented through toolkit registrations composed by this mechanism; a pattern the toolkit
   cannot express is a pattern this platform would have to hand-write per repository.
3. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` because no approval mechanism exists (`patterns/index.md`, *Governance*).
4. **Pattern Update Proposal:** obligation 2 in §2 — moving identity, role and ownership *semantics*
   into one versioned package while leaving composition per repository — is a change to the
   `framework-supplied-platform-conventions` pattern, not merely to this platform. It should be raised
   against that pattern file when this ADR is decided.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | The gateway takes logging, metrics and security from `Convey` at a floating `0.4.*` version | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Pacco.APIGateway.csproj:10-12` |
| E2 | 267 `Convey.*` package references exist across 40 project files, none pinned to an exact version, with no lock file present | ✅ | Workspace-wide count over all `*.csproj`; representative file `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Pacco.APIGateway.csproj:9-21` |
| E3 | A service's cross-cutting capability is one chained registration list in a per-repository composition root | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Extensions.cs:55-94` |
| E4 | The host itself is toolkit composition — `AddConvey`, `AddWebApi`, `AddApplication`, `AddInfrastructure`, then `UseLogging` and `UseVault` | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/Program.cs:28-48` |
| E5 | No project reference crosses a repository boundary; no first-party shared package exists | ✅ | Workspace-wide search over all `*.csproj` `ProjectReference` entries — zero cross-repository references |
| E6 | The only "Shared" artifact is a test fixture internal to one service, not a platform library | ✅ | `hianshul100_Pacco.Services.Availability/tests/Pacco.Services.Availability.Tests.Shared/Pacco.Services.Availability.Tests.Shared.csproj` |
| E7 | Identity and caller-context types are copied per repository rather than shared, so ownership semantics live in eleven places | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Contexts/IdentityContext.cs:24-33`; `…/Contexts/AppContextFactory.cs:19-33` |
| E8 | Eleven independent pipelines exist, one per deployable repository, with no shared template | ✅ | Eleven `.travis.yml` files, one per deployable repository (e.g. `hianshul100_Pacco.APIGateway/.travis.yml`) |
| E9 | The knowledge catalog for tenant `Q5SCXYFS` holds no ADR, Decision or Constraint node governing this platform | ✅ | Graph queries for ADR/Decision/Constraint nodes and for an unscoped node count both returned zero rows |

### 6.1 Documentation-versus-code conflicts

None found for this decision. `docs/architecture-inventory/repo-inventory.md` §3.3 and
`docs/architecture-inventory/baselines/architecture-baseline.md` §1.2 and §3.4 both record the absence
of a shared library, and the source confirms it.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The fourteen cloned repositories are the whole platform, so no shared Pacco library exists outside this workspace | Every project reference, compose entry and process manifest entry resolves inside this set, and nothing points outside it | The central claim of this record — that there is no first-party shared library — would be false, and the whole decision would need rewriting | Ask the platform owner whether any Pacco-owned package is published to a private feed, and check the NuGet sources configured on the build agents |
| A2 | `Convey 0.4.*` behaves as its registration names suggest, because its source is not in this workspace | Only the package reference and the registration call are visible; the toolkit is a NuGet dependency with no source here | Consequences that depend on toolkit semantics (retry behaviour, convention casing, decorator ordering) could be wrong in detail | Read the pinned toolkit version's source or its release notes for the resolved version, and record the resolved version per repository |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner. There is no `CODEOWNERS`, contributing guide or team metadata in any of the fourteen clones, so this ADR has no one who can approve it | This record and every other ADR in the corpus — an ADR nobody owns cannot leave `Proposed` | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has confirmed which resolved `Convey` version each deployed service is actually running. With floating versions and no lock file, the deployed dependency set is unknown | Obligation 1 in §2 cannot be executed as a pin until someone knows what to pin to, and no upgrade can be planned | Platform owner | Capture the resolved package versions from a current build of each repository, record them, then convert the `0.4.*` ranges to those exact versions | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should the identity, role and caller-context types move into a versioned Pacco package, leaving only composition per repository? | These copied types decide who owns an order and whether a caller may act. Today a fix to one copy reaches one service, and the copies have already diverged — this is the mechanism behind the authorization defect recorded in `ADR-006` | Yes for those types specifically. Keep composition per repository; move ownership semantics into one small package so a security fix is one release, not eleven | Platform owner |
| Q2 | **[ACTION NOW]** Is the platform's dependence on a single external toolkit acceptable, and who watches its maintenance status? | Every cross-cutting capability on the platform comes from one third-party package line. If it stops being maintained, the platform has no fallback and no one is currently responsible for noticing | Name one owner responsible for tracking the toolkit's releases and for deciding when a capability should be taken in-house instead | Platform owner |
| Q3 | **[handled later by the design stage]** Should the copied composition root be generated from a central template rather than hand-copied? | It is the only way the platform gets a signal when a service diverges, short of adopting a shared library. Today divergence is invisible until someone reads eleven repositories | Generate the composition root and the shared folder layout from one template, keep the output committed per repository so it stays readable, and fail the build when a repository's generated section differs from the template | Platform owner |
