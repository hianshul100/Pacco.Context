# ADR-017: Composable container stacks and process manifests as the deployment path

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-017` |
| **Backlog candidate** | `ADR-CANDIDATE-017` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Deployment / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-015` (the discovery and routing this environment provides), `ADR-005` (the write mode the two stacks disagree about), `ADR-011` (the coordinator whose deployment status this record cannot settle), `ADR-016` (the credential store this environment runs in development mode) |

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

Eleven independently released deployables and nine infrastructure components have to be startable
together by one developer on one machine, and the same definitions are the only description the platform
has of how it runs anywhere else.

1. ✅ **The environment is described by seven container-compose files split by concern.** One aggregate
   for all infrastructure (`hianshul100_Pacco/compose/infrastructure.yml`), a host-networking variant of
   it (`…/compose/host-infrastructure.yml`), three per-concern files covering discovery and routing and
   the credential store, the data stores and broker, and the observability tools, plus two application
   stacks — one running published images and one building locally.
2. ✅ **The two application stacks disagree about the platform's write mode.** The published-image stack
   points the gateway at the asynchronous configuration (`…/compose/services.yml:9`) and the local-build
   stack points it at the synchronous one, which is also the image's own default
   (`hianshul100_Pacco.APIGateway/Dockerfile:11`). The choice `ADR-005` describes therefore depends on
   which file was used to start the platform.
3. ✅ **There is a second, container-free deployment path.** Two process-manager manifests define ten
   applications each, capped at three restarts; the production one runs published output on fixed ports
   (`hianshul100_Pacco/services.yml`, `hianshul100_Pacco/prod-services.yml`).
4. ✅ **The saga coordinator appears in one path and not the other.** It has a container definition and a
   host port in both application stacks (`…/compose/services.yml:78-85`) and is scraped for metrics
   (`…/compose/prometheus/prometheus.yml:38-40`), and it is absent from both process manifests and from
   all four gateway configurations.
5. ✅ **Nothing describes more than one host.** No orchestration manifest, no infrastructure-as-code, no
   package chart and no cluster definition exists anywhere in the fourteen clones.
6. ✅ **Nothing describes health or scale.** No container definition and no process manifest declares a
   health check, a replica count or any placement constraint.
7. ✅ **Continuous integration stops before deployment.** Eleven repositories each run the same
   three-step pipeline — build, test, then produce an image on the two long-lived branches — and none of
   them deploys anything.
8. ✅ **Infrastructure state is mostly not persisted.** Only the data store and the cache declare named
   volumes; the declarations for discovery, the broker, and the three observability tools are present but
   commented out, and the data store runs with authentication disabled
   (`…/compose/infrastructure.yml:57-59,128-147`).

### 1.1 What this record does *not* cover

This record fixes how the platform is started and what the deployment path is today. It does not cover
discovery and routing (`ADR-015`), the write mode itself (`ADR-005`), the credential store's design
(`ADR-016`), or whether the saga coordinator should exist (`ADR-011`). It does not choose a production
platform; it records that none is described.

## 2. Decision

**Pacco is defined as a set of composable per-concern container stacks plus an optional container-free
process-manager path, both describing a single host. A developer starts the infrastructure aggregate and
then either the application stack or the per-repository run scripts. This is a development environment
that also serves as the platform's only deployment description; it is adopted as such, and explicitly not
as a production deployment model.**

Six rules follow from the decision and are part of it:

1. ✅ **Infrastructure and applications are started separately.** Infrastructure is long-lived and
   rarely restarted; applications are restarted constantly. Keeping them in different files is what makes
   the developer loop fast.
2. ✅ **Infrastructure is grouped by concern, and the aggregate is the entry point.** A developer who
   needs only the broker and the stores can start that file alone.
3. ✅ **Every deployable is independently startable.** A service can be run from its own repository
   against the shared infrastructure, which is what `ADR-002`'s no-shared-library position and the
   per-repository pipeline are for.
4. ✅ **The pipeline stops at a built image.** Releasing is a separate act from building, and no
   repository's pipeline can change a running environment.
5. 🎯 **Exactly one file must decide the platform's write mode.** Two application stacks that disagree
   (context point 2) mean the platform's externally visible write semantics depend on a start command.
6. 🎯 **A production deployment path must be described before the platform runs anywhere shared.**
   Single-host definitions with no health checks, no replicas and unpersisted infrastructure state
   describe a developer machine, and nothing else in the workspace describes anything more.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **One container-compose file for everything** — infrastructure and all eleven applications together. | Rejected because it forces a developer working on one service to start twenty containers, and it couples the restart cadence of long-lived infrastructure to that of applications. The per-concern split is what makes rule 1 possible. It would, however, have prevented the disagreement in rule 5, which is the cost of the split. |
| 2 | **A container orchestrator** — declarative workloads, health checks, replicas and rolling updates. | Rejected for the current stage, not on merit. It answers rule 6 directly and would remove the single-host limit, the missing health checks and the missing replica counts in one step. Nothing in the workspace references one, so adopting it is a project rather than a configuration change, and it should be the subject of its own record. |
| 3 | **Only the process-manager path, with no containers** — run published output directly on hosts. | Rejected because it still needs the nine infrastructure components running somewhere, and containers are how the platform provides them. It also drifts: the manifests define ten applications where the container stacks define eleven, which is the drift this record records as a defect rather than a model. |
| 4 | **Extend the pipeline to deploy** — have each repository's pipeline push to an environment. | Rejected because there is no environment for it to push to (context point 5), and eleven independent pipelines deploying into one shared single-host environment would give eleven repositories the ability to restart each other's dependencies. Rule 4 keeps building and releasing separate until rule 6 is satisfied. |

## 4. Consequences

### 4.1 Positive

1. ✅ A developer starts the whole platform with two commands and no bespoke tooling.
2. ✅ Working on one service costs one process, not twenty containers, because infrastructure and
   applications are separate and each deployable runs standalone.
3. ✅ The per-concern split means an infrastructure subset can be started for a focused task.
4. ✅ Every repository's pipeline is identical, so a new service inherits the build path by copying three
   scripts and one pipeline file.
5. ✅ The environment definitions double as the platform's component inventory: reading the aggregate
   tells you what the platform depends on.

### 4.2 Negative

1. ✅ **The platform's write semantics depend on which stack was started.** The published-image stack and
   the local-build stack point the gateway at different configurations, so `ADR-005`'s decision has two
   different answers in the same repository.
2. ✅ **The eleven applications are restated across four files with no include mechanism**, so any change
   to a deployable has to be applied in several places, and they have already drifted: the process
   manifests are missing the deployable the container stacks include.
3. ✅ **It is unresolved whether the saga coordinator runs.** It is in both container stacks and the
   metrics scrape list, and absent from both process manifests and every gateway configuration. Whether
   it is part of the platform depends entirely on which path is used.
4. ✅ **There is no production deployment description at all.** Everything present describes one host,
   with no health checks, no replicas and no placement.
5. ❓ **Most infrastructure state is discarded on restart.** What is observed is that only the data store
   and the cache declare named volumes, and that five other declarations are present but commented out
   (evidence 13). `[INFERRED]` that discovery registrations, broker definitions, metrics, traces and logs
   are therefore lost on a restart of the infrastructure stack; whether each of those components keeps
   anything worth losing has not been verified against a running environment (assumption A3).
6. ✅ **The data store runs with authentication disabled**, which means the dynamically issued database
   credentials of `ADR-016` are being issued against a store that would accept an unauthenticated
   connection.
7. ✅ **Nothing verifies that a started platform is healthy.** With no health checks and no deployment
   step, the only signal that the environment came up is that requests succeed.

### 4.3 Neutral / follow-on

1. ✅ The host-networking variant of the infrastructure aggregate exists for developers whose container
   networking differs, and is a second copy of the same nine components — the same restatement cost as
   consequence 4.2.2, in a different place.
2. ✅ Metrics are scraped every five seconds across twelve jobs, which is a development-appropriate
   interval and a broader scrape list than either process manifest's application set.
3. ✅ The platform's own guidance describes the two-command developer path and treats each service's
   internal structure as the service's own choice, which is consistent with rules 1 and 3 and says
   nothing about anything beyond a developer machine.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates `deployment/composable-per-concern-environment-stacks.md`.** This record is the
   decision that pattern describes: infrastructure split by concern, applications in separate stacks, one
   aggregate as the entry point. The pattern is catalogued with confidence *Strong*.
2. **Adopts the pattern's recommendation as a boundary on the decision.** The catalog recommends adopting
   the pattern for development only, noting that eight files restate the same components with no include
   mechanism and have already drifted, and that these definitions describe a single host. The decision
   statement carries that boundary explicitly, and rule 6 makes the missing production path a condition
   rather than an observation.
3. **Instantiates `deployment/independent-per-repository-release.md`.** Eleven identical three-step
   pipelines that stop at a built image are exactly that pattern, and rules 3 and 4 state it.
4. **Provides the environment `deployment/registry-mediated-discovery-and-routing.md` assumes.**
   Discovery, the load balancer and the services that register with them are all defined in these files
   (`ADR-015`), so a change to the stacks changes what that pattern can rely on.
5. **Constrains `security/vault-issued-dynamic-credentials-and-service-pki.md`.** The credential store's
   development-mode posture is a property of these definitions, not of `ADR-016`'s design, which is why
   that record's rule 6 cannot be satisfied without a change here.
6. **Pattern Drift: not applicable.** Every entry in `docs/architecture-inventory/patterns/index.md`
   currently carries status `Candidate`. Drift is reportable only against an `Approved` pattern, so no
   drift is recorded for this ADR.
7. **Pattern Update Proposal.** `deployment/composable-per-concern-environment-stacks.md` should record
   two facts established here that it does not currently carry: that the two application stacks configure
   different gateway modes (consequence 4.2.1), and that most infrastructure components have their volume
   declarations commented out (consequence 4.2.5). Its **Related ADRs** entry, and that of
   `deployment/independent-per-repository-release.md`, should change from `None` to `ADR-017`.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| 1 | The infrastructure aggregate defines the platform's nine supporting components | ✅ | `hianshul100_Pacco/compose/infrastructure.yml` (147 lines) |
| 2 | A host-networking variant of the same aggregate exists | ✅ | `hianshul100_Pacco/compose/host-infrastructure.yml` (104 lines) |
| 3 | Infrastructure is also split into three per-concern files | ✅ | `hianshul100_Pacco/compose/consul-fabio-vault.yml`; `hianshul100_Pacco/compose/mongo-rabbit-redis.yml`; `hianshul100_Pacco/compose/grafana-seq-jaeger-prometheus.yml` |
| 4 | The published-image stack defines eleven applications, each restarted unless stopped, on fixed host ports | ✅ | `hianshul100_Pacco/compose/services.yml` |
| 5 | That stack points the gateway at the asynchronous configuration | ✅ | `hianshul100_Pacco/compose/services.yml:9` |
| 6 | The local-build stack points the gateway at the synchronous configuration | ✅ | `hianshul100_Pacco/compose/services-local.yml` (gateway environment) |
| 7 | The gateway image's own default is the synchronous configuration | ✅ | `hianshul100_Pacco.APIGateway/Dockerfile:11` |
| 8 | The saga coordinator has a container definition and a host port | ✅ | `hianshul100_Pacco/compose/services.yml:78-85` |
| 9 | The observer waits on eight services including the coordinator | ✅ | `hianshul100_Pacco/compose/services.yml:59-67` |
| 10 | Two process manifests define ten applications each, capped at three restarts | ✅ | `hianshul100_Pacco/services.yml` (40 lines); `hianshul100_Pacco/prod-services.yml` (61 lines) |
| 11 | The production manifest runs published output on fixed ports and omits the saga coordinator | ✅ | `hianshul100_Pacco/prod-services.yml` |
| 12 | Metrics are scraped every five seconds across twelve jobs, including the coordinator | ✅ | `hianshul100_Pacco/compose/prometheus/prometheus.yml:38-40` |
| 13 | Only the data store and the cache declare named volumes; five other declarations are commented out | ✅ | `hianshul100_Pacco/compose/infrastructure.yml:128-147` |
| 14 | The data store's root credentials are commented out, leaving authentication disabled | ✅ | `hianshul100_Pacco/compose/infrastructure.yml:57-59` |
| 15 | Eleven repositories each run the same three-step pipeline, producing an image on the two long-lived branches and deploying nothing | ✅ | `.travis.yml` in `hianshul100_Pacco.APIGateway` and the ten service repositories |
| 16 | No orchestration manifest, infrastructure-as-code file, package chart or cluster definition exists in any clone | ✅ | Workspace-wide search across the fourteen clones |
| 17 | No health check, replica count or placement key appears in any container definition or process manifest | ✅ | Workspace-wide search across `hianshul100_Pacco/compose/` and the two process manifests |
| 18 | The platform guidance describes the two-command developer path | ✅ | `hianshul100_Pacco/README.md` |

### 6.1 Documentation-versus-code conflicts

1. **The two application stacks contradict each other on the gateway's mode.** One selects the
   asynchronous configuration and the other the synchronous one (evidence 5–7). `ADR-005` describes both
   modes as available by configuration, but neither the guidance nor any file states which is intended
   for a real environment. The code shows both, so this record does not pick one: it states the conflict,
   makes single ownership a target in rule 5, and carries the choice as blocker B2.
2. **The two deployment paths disagree about how many applications the platform has.** The container
   stacks define eleven and both process manifests define ten, the difference being the saga coordinator
   (evidence 4, 8, 10, 11), which is additionally scraped for metrics (evidence 12) and absent from every
   gateway configuration. Recorded as drift between two descriptions of one platform. The code is
   authoritative in the sense that both files exist and disagree; nothing resolves them, so this is
   carried as blocker B3 rather than reconciled.
3. **No production deployment is described anywhere, and nothing says one is intended.** The manifest
   named for production runs ten processes on one host with no health check and no replicas
   (evidence 11, 17), which is a naming, not a production model. Whether a real deployment target exists
   outside the workspace is **Unverifiable — Missing Source Evidence**, and rule 6 states the target on
   that basis.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | These definitions are the whole of the platform's deployment description | A search across all fourteen clones found no orchestration, infrastructure-as-code, chart or cluster definition of any kind (evidence 16) | If a deployment description exists outside the workspace, rule 6 may already be satisfied and several negative consequences apply only to development | Ask the platform owner whether any deployment description is held outside these repositories, and inspect it if one exists |
| A2 | The container path is the primary one and the process-manager path is a secondary option | The container stacks are more complete — eleven applications against ten — and are what the platform guidance describes | If the process-manager path is primary, the saga coordinator is not part of the platform at all, which would settle `ADR-011`'s deployment question by removal | Ask the platform owner which path is used to start the platform outside a developer machine, if either is |
| A3 | The commented-out volume declarations were disabled for developer convenience rather than by mistake | They are present, complete and adjacent to the two that are enabled, which reads as a deliberate toggle | If they were disabled accidentally, restarting infrastructure loses state that someone expects to survive, and the fix is one uncommenting rather than a design change | Ask whoever commented them out; failing that, restart the infrastructure stack and record which components come back empty — which also settles the ❓ on consequence 4.2.5 |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[handled later by the architecture PR review stage]** No repository in the workspace names an owner, a team or a review group, so this record cannot list deciders | This ADR leaving `Proposed`, and rules 5 and 6 in §2, which both need someone accountable for the environment definitions | Platform owner (unassigned) | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository; the reviewer assigns deciders when the batch is reviewed as a whole | TBD |
| B2 | **[ACTION NOW]** It cannot be stated which gateway configuration the platform runs, because the two application stacks select different ones | Every record that depends on the write mode, including `ADR-004` B2, `ADR-005` and `ADR-014` B3 | Platform owner | Decide which mode the platform runs (question Q1), set it in one file, make the other stack read that file or delete it, and record the answer here so the dependent records can cite one place | TBD |
| B3 | **[ACTION NOW]** It cannot be stated whether the saga coordinator is deployed, because the container stacks include it and both process manifests omit it | `ADR-011` B3, which carries the same blocker, and anyone trying to exercise order creation | Platform owner | Decide whether `ordermaker-service` is part of the platform. If it is, add it to both process manifests and give it a gateway route; if it is not, remove it from the container stacks and the metrics scrape list | TBD |
| B4 | **[handled later by the batch 4 authoring stage, in the record covering the build and release path]** There is no production deployment description, so no record in this set can state how the platform behaves outside a developer machine | Every record whose consequences depend on scale, restart behaviour or failure handling | Platform owner (unassigned) | Decide a production platform in its own record (question Q3), then describe health checks, replicas and persistence there rather than stretching these definitions | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Which file should own the gateway's mode, and which mode should it select? | Rule 5 requires one owner. Selecting the synchronous mode makes `ADR-014`'s observer largely unused; selecting the asynchronous one makes it load-bearing | Let the published-image stack own it and keep the asynchronous mode: the observer, the correlation id and the push channel are all built for it, and the synchronous mode leaves three services with no purpose | Platform owner |
| Q2 | **[handled later by the architecture PR review stage]** Should the eleven applications be defined once and included, rather than restated across four files? | The restatement has already drifted once (consequence 4.2.2), and the drift is what makes B3 unanswerable | Define the applications once and have each stack extend it, so the container and process paths cannot disagree about how many deployables exist | Platform owner |
| Q3 | **[handled later by the batch 4 authoring stage, in the record covering the build and release path]** Should the platform move to a container orchestrator, and when? | It is the direct answer to rule 6 and removes the single-host limit, the missing health checks and the missing replica counts together — but it is a project, not a configuration change | Not before there is a shared environment to run; when there is, adopt one and record it separately, because it also answers `ADR-016` rule 6 and question Q1 there | Platform owner |
| Q4 | **[ACTION NOW]** Should the data store run with authentication enabled in the developer environment? | Leaving it open means the dynamically issued credentials of `ADR-016` are never actually required, so a service that failed to obtain one would still connect and nobody would notice | Enable it. It is one uncommented pair of lines (evidence 14) and it is the only way the credential mechanism `ADR-016` describes is exercised at all before a shared environment exists | Platform owner |

