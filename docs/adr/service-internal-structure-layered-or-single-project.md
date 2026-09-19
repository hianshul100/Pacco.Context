# ADR-019: Two internal service structures — layered domain split and single project

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-019` |
| **Backlog candidate** | `ADR-CANDIDATE-019` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Other / medium |
| **Supersedes / Superseded by** | — (nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-002` (the toolkit that makes either structure viable), `ADR-008` (the repository interface the layering exists to keep out of the domain), `ADR-018` (why a structure choice is per-repository and unreviewable elsewhere) |

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

`ADR-002` supplies every cross-cutting capability from one external toolkit, so a service's internal
project layout is genuinely free — nothing in the platform forces one shape. The eleven deployables
have settled into **three** shapes, not the two the backlog anticipated.

1. ✅ **Seven services use a four-project layout.** Availability, Customers, Deliveries, Identity,
   Orders, Parcels and Vehicles each hold exactly four projects under `src/`: an entry-point host, an
   application layer, a core domain layer and an infrastructure layer.
2. ✅ **In all seven, the core project references nothing.** Every one of the seven core project files
   declares zero project references and zero package references — no toolkit, no persistence, no
   serialization. The domain is genuinely isolated
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Core/Pacco.Services.Orders.Core.csproj`).
3. ✅ **Three deployables are single-project.** The gateway, `ordermaker-service` and `pricing-service`
   each hold one project under `src/`.
4. ✅ **`operations-service` is neither.** It holds two projects: an API host and a separate gRPC client
   marked as an executable. The second is a standalone console client for exercising the streaming
   surface, not an architectural layer
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.GrpcClient/Pacco.Services.Operations.GrpcClient.csproj:4-6`).
   Structurally the service itself is single-project with a client tool alongside.
5. ✅ **The single-project services still keep the same internal folders.** `pricing-service` organises
   its one project into `Core/Entities`, `Queries/Handlers`, `Services/Clients`, `DTO` and
   `Infrastructure` — the layered structure expressed as directories rather than as compilation units.
6. ✅ **The platform's own README frames the choice as optional.** It describes the services as built
   with clean architecture and domain-driven design "or another style that is the best fit", which
   sanctions divergence without bounding it.
7. ✅ **Nothing states when to choose which.** No repository holds a contributing guide, a template, a
   scaffold or an architecture test that would express or enforce the rule.

The layered services are not uniformly complex, either: `vehicles-service` uses four projects for a
catalog of vehicles, while single-project `ordermaker-service` runs the platform's only saga
(`ADR-011`). Project count does not track domain complexity today.

### 1.1 What this record does *not* cover

It does not decide the toolkit or the composition-root convention, which is `ADR-002`, nor the
route-table dispatch idiom, which the backlog excluded as an implementation detail. It does not cover
test-project layout, which follows the release model in `ADR-018`.

## 2. Decision

**A Pacco service uses the four-project layered structure — host, application, core, infrastructure —
by default, with the core project holding no project or package reference of any kind. A deployable may
be a single project only when it owns no persistent state and no domain invariant, in which case it
still organises itself into the same layers as directories. A deployable that needs a companion tool
ships it as an additional project rather than folding it into the host.**

The default is the four-project layout because that is what seven of eleven do and what every
state-owning service does. Three obligations follow:

1. 🎯 **The single-project exception must be re-tested when a service gains state.**
   `ordermaker-service` already fails this: it owns saga state (`ADR-011`) and is single-project.
2. 🎯 **A single-project service must still keep the layer boundaries as directories,** as
   `pricing-service` does, so that promoting it to four projects is a move rather than a rewrite.
3. 🎯 **The dependency direction must be verifiable, not merely observed.** Today it holds because
   seven developers kept it, with nothing to catch the eighth.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Mandate the four-project layout for every deployable**, with no exception. | Rejected because it is visibly wasteful at the small end. `pricing-service` persists nothing and its logic is one price calculation; four projects, four project files and a composition root split across them would be ceremony with no isolation benefit, since there is no domain invariant to protect. The gateway is more extreme still — it holds no domain code at all (`ADR-004`). |
| 2 | **Leave internal structure entirely to each team**, with no default and no rule. | Rejected because it is close to the current position and its cost is already visible: nothing states when a service is simple enough to stay single-project, so the choice is made by whoever scaffolds first and is never revisited. `ordermaker-service` is the result — a stateful coordinator in a single project because it started small. |
| 3 | **A shared scaffold or template repository** that generates the layout. | Rejected as inconsistent with `ADR-002` and `ADR-018`: the platform deliberately has no first-party shared artifact, and a template is a shared artifact with no versioning story. It would also drift silently, since eleven repositories have no mechanism to consume a template update. |
| 4 | **A vertical-slice layout** — folders per feature, each holding its own handler, entity and persistence — instead of horizontal layers. | Rejected implicitly; nothing in the workspace uses it. It would fit the dispatcher-bound endpoint style well, but it conflicts with the one property the current layout actually guarantees: a core project that cannot reference infrastructure because it references nothing at all. Under vertical slices that guarantee has no compilation unit to live in. |

## 4. Consequences

### 4.1 Positive

1. ✅ **The domain layer cannot take an infrastructure dependency, because it has no references to take
   one through.** This is enforced by the compiler in all seven layered services, not by convention.
2. ✅ Domain rules are unit-testable without any host, database or broker — `availability-service`'s
   unit tests exercise handlers against test doubles for exactly this reason.
3. ✅ The uniform layout makes seven repositories navigable by anyone who has read one: the same four
   projects and the same internal folders appear in each.
4. ✅ The single-project exception keeps genuinely trivial deployables trivial. The gateway is a host
   and a configuration file (`ADR-004`), and forcing four projects on it would communicate a structure
   it does not have.

### 4.2 Negative

1. ✅ **Nothing enforces the dependency direction.** It holds today in all seven services, but no
   architecture test, lint rule or build check would fail if a core project gained an infrastructure
   reference tomorrow.
2. ✅ **The choosing rule does not exist in any repository,** so the next service's structure is
   decided by whoever creates it, with the README explicitly permitting "another style".
3. ✅ **`ordermaker-service` is on the wrong side of the rule this record states.** It is
   single-project and owns saga state, which §2 forbids and which `ADR-011` records as an unresolved
   durability problem. The structure choice and the durability gap have the same root: it was built as
   a small thing and became a stateful one.
4. ✅ **Four projects per service multiplies the composition root.** Registration is split across the
   host and the infrastructure project in each of the seven, so wiring one service means reading two
   files — and `ADR-002` already notes that this composition root is copied rather than shared.
5. ✅ **`operations-service`'s second project is undescribed by any rule.** It is a client tool inside
   a service repository, built and published by the same pipeline, with no statement of whether it is
   a deliverable or a development aid.

### 4.3 Neutral / follow-on

1. ✅ `pricing-service` demonstrates that the layers survive without the projects: the same directory
   names appear inside its single project. That is what makes obligation 2 in §2 realistic.
2. ✅ Test-project counts vary independently of structure — five in `availability-service`, one each in
   `orders-service` and `parcels-service`, none in the other eight — so structure and test discipline
   are not correlated on this platform.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/domain/inward-dependency-service-skeleton.md` (Status: `Candidate`,
   evidence Strong) and **constrains** it by supplying the choosing rule in §2, which the pattern does
   not contain.
2. **Enabled by** `patterns/other/framework-supplied-platform-conventions.md`: the layout is free to
   vary precisely because every cross-cutting capability comes from the toolkit rather than from a
   shared internal project (`ADR-002`).
3. **Depends on** `patterns/data/database-per-service-with-document-mapping.md` for the reason the
   layering exists at all — the repository interface belongs to the core project and its Mongo
   implementation to infrastructure.
4. **Compatible with** `patterns/domain/dispatcher-bound-cqrs-endpoints.md` and
   `patterns/domain/aggregate-buffered-domain-events.md`, both of which are expressed identically in
   the layered and single-project shapes.
5. **Pattern Drift:** none reportable. Drift requires an `Approved` pattern to violate, and every entry
   in `patterns/index.md` carries status `Candidate` (*Governance*).
6. **Pattern Update Proposal (correction):** the pattern's related-services entry reads "All 7 layered
   services", which matches the source; but the ADR backlog and the architecture baseline both count
   eight by including `operations-service`. Source shows `operations-service` holds two projects, one
   of which is a console client (§1 item 4). The conflict is stated in §6.1 and the baseline's count
   should be corrected to seven.
7. **Pattern Update Proposal (addition):** the pattern should add the promotion trigger from §2
   obligation 1 — a single-project deployable that acquires persistent state or a domain invariant
   moves to the layered layout — and should require the layer-as-directory convention that makes that
   promotion cheap.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | Seven services hold exactly four projects under `src/`: Api, Application, Core, Infrastructure | ✅ | `src/` project enumeration in Availability, Customers, Deliveries, Identity, Orders, Parcels and Vehicles |
| E2 | Every one of the seven core projects declares no project reference and no package reference | ✅ | `Pacco.Services.<Service>.Core.csproj` in each of the seven — zero `ProjectReference` and zero `PackageReference` elements |
| E3 | The gateway holds one project under `src/` | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Pacco.APIGateway.csproj` — the only project in that tree |
| E4 | `ordermaker-service` holds one project under `src/` | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj` |
| E5 | `pricing-service` holds one project under `src/` | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj` |
| E6 | `operations-service` holds two projects, the second an executable gRPC client | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.GrpcClient/Pacco.Services.Operations.GrpcClient.csproj:4-6` |
| E7 | A single-project service keeps the layer names as directories | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/` — `Core/Entities`, `Queries/Handlers`, `Services/Clients`, `DTO`, `Infrastructure` |
| E8 | The platform README sanctions an alternative style without bounding it | ✅ | `hianshul100_Pacco/README.md`, quoted in `docs/architecture-inventory/repo-inventory.md` §2.3 |
| E9 | No contributing guide, scaffold, template or architecture test exists in any of the fourteen clones | ✅ | Workspace-wide search for `CONTRIBUTING`, template and scaffold artifacts — zero matches |
| E10 | Structure does not track complexity: a four-project catalog service and a single-project saga coordinator coexist | ✅ | `hianshul100_Pacco.Services.Vehicles/src/` (four projects) against `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` (one project, stateful) |
| E11 | Registration is split between the host and the infrastructure project in the layered services | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:54` alongside the host's own composition root |
| E12 | Test-project counts vary independently of structure | ✅ | `tests/` directories: five projects in Availability, one each in Orders and Parcels, absent in the other eight |

### 6.1 Documentation-versus-code conflicts

1. ✅ **The layered-service count is seven, not eight.**
   `docs/architecture-inventory/adr-candidates.md` (candidate 019) states "seven services use a
   four-project layout" in its rationale but then lists eight in its evidence by including
   `operations-service`, and
   `docs/architecture-inventory/baselines/architecture-baseline.md` §3.1 repeats the eight. Source
   shows `operations-service` holds an API host and an executable console client — two projects, no
   application or core layer (E6). The code is followed: **seven** layered services, and
   `operations-service` is recorded here as a third shape rather than folded into either group.
2. ✅ **"Three deployables are single-project" is right for the wrong reason.** The backlog's three are
   the gateway, `ordermaker-service` and `pricing-service`, which the source confirms (E3–E5). But the
   backlog reaches eleven by counting `operations-service` among the layered eight; the correct
   partition is seven layered, three single-project and one two-project outlier.
3. ✅ **The README framing is recorded as intent, and the code agrees with it.** The README's "or
   another style" is honoured rather than contradicted — three deployables genuinely use another
   style. This is one of the few places where documented intent and code do not conflict, and it is
   noted because the *absence* of a bound on that permission is the gap this record fills.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The four-project layout was chosen deliberately rather than inherited from a sample or a template | It is applied identically across seven independently released repositories, and the README names clean architecture and domain-driven design explicitly (E8) | The default in §2 would be ratifying a copy-paste rather than a decision, and the alternatives in §3 would never have been weighed by anyone | Ask the platform owner whether the layout came from a starter template, and if so whether that template is still maintained anywhere |
| A2 | `operations-service`'s gRPC client project is a development aid rather than a shipped deliverable | It is marked as an executable and consumes the same streaming contract the service exposes, and no environment stack references it | The consequence in §4.2 item 5 would be wrong, and the project would need a release and versioning story that no repository currently gives it | Ask the owner of `operations-service` whether anything outside the repository runs that client |
| A3 | The single-project deployables have no domain invariant to protect, which is what makes the exception in §2 safe | `pricing-service` persists nothing and the gateway holds no domain code; both are stateless by construction | The exception in §2 would be permitting unprotected domain logic, and two more services would need promoting rather than one | Read each single-project deployable for state that outlives a request, and confirm `ordermaker-service` is the only one that has it |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so the default in §2 cannot be accepted and no one can decide whether `ordermaker-service` is promoted | This ADR leaving `Proposed`, and obligation 1 in §2, which asks a specific service to change structure | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should `ordermaker-service` be promoted to the layered structure? | It is the platform's only orchestrator, it owns saga state, and `ADR-011` records that the state has no configured durability. It is also the deployable this record's own rule says should not be single-project (§4.2 item 3) | Resolve the durability question in `ADR-011` first — the answer determines what the core layer would hold. If saga state becomes persistent, promote it; if the saga is redesigned to be stateless, leave it as one project | Platform owner, with the owner of `Pacco.Services.OrderMaker` |
| Q2 | **[ACTION NOW]** Is `operations-service`'s gRPC client a deliverable or a development tool? | It is built and published by the same pipeline as the service, so it is released either way, but nothing states who is expected to run it or whether its contract is supported (§4.2 item 5) | Treat it as a development tool and move it out of `src/` into a tools directory excluded from the published image, unless someone names an external consumer | Owner of `Pacco.Services.Operations` once B1 is resolved |
| Q3 | **[handled later by the design stage]** How should the inward dependency direction be enforced rather than observed? | It is the one property the layering actually guarantees, and it is guaranteed today only because seven core projects happen to have no references (§4.2 item 1). Nothing would catch the eighth | Add an architecture test per service asserting the core project has no references. It is duplicated per repository, which `ADR-018` establishes as the accepted cost of the release model — the same shape as `ADR-010`'s question Q4 | Owners named per service once B1 is resolved |
| Q4 | **[handled later by the design stage]** Should the layered services' split composition root be consolidated? | Wiring one service means reading a registration file in the host and another in infrastructure, and `ADR-002` records that this pair is copied into every repository rather than shared, so the duplication is sevenfold | Leave it. The split follows the layering and consolidating it would push infrastructure knowledge into the host, which is the coupling the structure exists to prevent. Revisit only if the composition roots start diverging between services | Owners named per service once B1 is resolved |
