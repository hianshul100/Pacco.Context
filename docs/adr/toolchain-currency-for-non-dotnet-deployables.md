# ADR-024: Toolchain currency for non-.NET deployables, governed alongside the runtime baseline

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-19 |
| **ADR id** | `ADR-024` |
| **Backlog candidate** | — (no candidate exists; this record is written to answer risk `R9` in `docs/architecture-inventory/risk-constraint-gap-register.md` §5.2, whose `adr_recommendation` names it and whose `owner_stage` is `architecture`) |
| **Source** | `R9` — the browser toolchain drifts with no governing rule (RPN 280); `GAP-13155-07`; `ADR-021` §7 `ARCHITECTURE_ALIGNMENT_EXCEPTION` and F4 |
| **Originating NFRs** | `NFR-23` (decision `needs_adr`); `NFR-2` (constraint honoured) |
| **Category / Impact** | Other (platform governance) / high |
| **Supersedes / Superseded by** | — (nothing to supersede). This record does not supersede `ADR-020`; it is its companion — see §4 rule 1 |
| **Deciders** | Authored by the `architecture` stage of work item 13155 to close `R9`. Repository ownership remains unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-020` (the .NET half of the same rule, and the record whose scope this one completes), `ADR-021` (the first non-.NET deployable, and the record that raised the exception), `ADR-018` (the per-repository release rule that makes a pin a per-repository obligation), `ADR-002` (the toolkit version line coupled to `ADR-020`'s pin, and the precedent for a version line nobody reviews) |
| **Known defect** | This record does not make the platform current. `ADR-020` records that every existing deployable runs on a runtime that reached end of support on 13 December 2022, and nothing here changes that. It governs the deployables `ADR-020` does not reach, and it states the review obligation `ADR-020` §2 obligation 4 already carries and that nobody has yet performed |

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

`ADR-021` adds the platform's twelfth deployable and its first that is not a .NET host. In doing so
it declares an `ARCHITECTURE_ALIGNMENT_EXCEPTION` against `ADR-020`, and that exception is the whole
subject of this record.

1. ✅ **One rule governs toolchain version on this platform, and it is written for .NET.**
   `ADR-020` §2: "Every Pacco deployable targets one runtime version, pinned identically in its
   project files, its pipeline SDK version and both stages of its container image. No service selects
   its own runtime."
2. ✅ **That rule cannot reach a browser bundle.** Its three pin locations are a `.csproj` target
   framework, a Travis `dotnet:` SDK version and a pair of Dockerfile base images. A static browser
   artifact has none of the three (`ADR-021` §3, §6.2 item 1).
3. ✅ **`ADR-021` records the exception rather than claiming conformance.** §7: "The rule's scope is
   the .NET runtime of a Pacco service, and this deployable has none. The gap it leaves — nothing
   governs the surface's Node/toolchain version — is carried as `GAP-13155-07`."
4. ✅ **Nothing replaces it.** No frontend toolchain rule, version policy, scaffold, contributing
   guide or architecture test exists in any of the fourteen clones, and `docs/standards/` does not
   exist in this repository (`ADR-021` §7, E14; `GAP-13155-08`). The catalogued constraints that
   mention Node tooling all record its **absence** — the existing browser asset was authored with no
   bundler, no transpiler, no minifier and no lockfile (`…/baselines/ui-inventory.md` §5.1;
   `…/baselines/architecture-baseline.md` §7.2).
5. ✅ **The platform has already run this failure once, at full scale.** Constraint C9 in
   `…/baselines/architecture-baseline.md` §11.2 pins all eleven deployables to `.NET Core 3.1`, which
   reached end of support on 13 December 2022 — discovered years afterwards by an architecture review
   rather than by any scheduled trigger (`ADR-020` §1 item 5, §4.2 item 1).
6. ✅ **`ADR-020` already names the missing mechanism.** Its §2 obligation 4 requires "a scheduled
   review before its support end date, rather than being discovered years afterwards by an
   architecture review." No such review exists for any deployable, and none would exist for a twelfth.

The question this record answers is narrow: **what governs the toolchain version of a Pacco
deployable that has no .NET runtime to pin?** It is scored in the register as `R9` — severity 5,
occurrence 7, detection 8, RPN 280, `mitigation_type: adr`, `owner_stage: architecture`. An
`adr` mitigation owned by the `architecture` stage is a record this stage writes, not a decision it
delegates, which is why the deferral originally recorded against `R9` is replaced by this record.

### 1.1 What this record does *not* cover

It does not choose a framework, bundler, package manager or component library — that is `ADR-021` Q3
and it belongs to the `lld` stage. It does not name a concrete version number for anything, because
no toolchain has been chosen yet and inventing a value would be exactly the error the register
forbids in `GAP-13155-03`. It does not author a platform frontend standard covering state ownership,
the host/shell contract or accessibility — that was deferred at gate `AD-2` and is carried as
`GAP-13155-09`. It does not change `ADR-020`'s pin, its scope or its status, and it does not move the
platform off `.NET Core 3.1`.

## 2. Decision Drivers

| # | Driver | Source |
| --- | --- | --- |
| D1 | No Pacco deployable may be governed by no toolchain rule at all, whatever its runtime | `R9`; `GAP-13155-07`; `ADR-020` §2 |
| D2 | The surface must carry a committed dependency manifest, build and lint baseline, and that baseline must be more than a snapshot of whatever was current the week it was written | `NFR-23`; `ADR-021` obligation 3; `ASM-17` |
| D3 | Drift must have an event. The platform's existing failure mode is not a bad decision but a silent one — no trigger fired for three and a half years | `ADR-020` §2 obligation 4, §4.2 item 6; `architecture-baseline.md` §11.2 C9 |
| D4 | A version pin must be readable in one place per repository, because four independent pins are four chances to migrate a deployable partially | `ADR-020` §2 obligation 3, §4.2 item 5 |
| D5 | The rule must not require a coordinated multi-repository release, because no mechanism for one exists | `ADR-018` §2, §6; `ADR-020` §4.2 item 3 |
| D6 | No version number may be invented by this record, because no toolchain has been selected | `ADR-021` Q3; register §2.2 `GAP-13155-03` note |
| D7 | The rule must survive `ADR-020` being superseded by a runtime move, which `ADR-020` §2 obligation 1 requires to happen | `ADR-020` §2 obligation 1 |

## 3. Architecture Fit Evaluation

**No existing record governs a non-.NET deployable's toolchain, and one of them says so in its own
text.** Each candidate was evaluated before proposing a companion record.

| # | Existing record or extension point | Evaluated as | Primary reason for rejection |
| --- | --- | --- | --- |
| 1 | `ADR-020`, amended in place to cover every deployable | The obvious move: one record, one rule, no second document to keep in step | `ADR-020` is written as a record of observed current state — forty project files, eleven pipelines and eleven Dockerfiles carrying an identical fourfold pin. Widening its text would make it assert a scope that no source implements, which is the one thing a current-state record may not do. It is also explicitly written to be superseded when the runtime moves (§2 obligation 1), and a browser-toolchain rule buried inside it would be superseded with it for reasons that have nothing to do with browsers |
| 2 | `ADR-018`, the per-repository release rule | It already reaches `Pacco.Web`, and it is where pipelines are governed | It governs *release identity* — one repository, one deployable, one independent release — and says nothing about versions. Adding version currency to it would merge two concerns that have different review cadences and different owners |
| 3 | `ADR-002`, the toolkit-as-platform-standard record | It is the platform's other version-line record, and `ADR-020` couples to it | It governs one external .NET toolkit's version line. A browser toolchain is neither that toolkit nor .NET, and the coupling runs the wrong way: `ADR-002`'s line is constrained by the runtime, not by a bundler |
| 4 | `ADR-021` itself, by adding a seventh obligation | The exception is declared there, so the fix could be attached there | `ADR-021` governs one surface. A rule that only binds `Pacco.Web` is not a platform rule, and the second non-.NET deployable — a CLI, a worker, a second surface — would inherit nothing. `R15` records exactly this failure mode: one feature's record becoming a de-facto platform standard nobody authored |
| 5 | A `docs/standards/` toolchain rule family | Where a rule of this kind would normally live | `docs/standards/` does not exist and eleven rule families are uncovered (`GAP-13155-08`). Creating the directory for one rule would either be a token gesture or a commitment to author the other ten, and neither is this work item's to make. The record is placed in `docs/adr/` where every other platform rule on this repository lives |

**Why a new record is required.** The gap is not that a rule is hard to apply — it is that no rule
exists to apply. `ADR-020`'s scope statement is correct and this record does not dispute it; what it
leaves behind is a class of deployable with no governing constraint at all, and that class stopped
being hypothetical when `ADR-021` was decided.

**Why the boundary stays stable.** This record constrains three things only — where a version is
declared, that it is in support when declared, and that a review is booked before it stops being in
support. None names a technology, so nothing here changes if the surface is built with one framework
or another, or if a second non-.NET deployable is a worker rather than a browser bundle.

**Why extending an existing record would violate cohesion or ownership.** Rows 1 and 3 fold a
non-.NET concern into records whose evidence base is entirely .NET, so their next review would have
to reason about two runtimes to change one sentence. Row 4 gives a platform rule a feature's owner.

## 4. Decision

**Every Pacco deployable is governed by exactly one toolchain-currency record. `ADR-020` governs
deployables with a .NET runtime and keeps its scope unchanged. This record governs every deployable
that has none: its toolchain version is declared once per repository in a committed manifest with a
lockfile, mirrored by the pipeline rather than restated by it, chosen from versions that are in
vendor support on the day they are committed, and given a scheduled review dated before that
version's support ends. No version number is named here.**

Six rules follow from the decision and are part of it:

1. 🎯 **No deployable is governed by neither record.** A Pacco deployable with a .NET runtime is
   governed by `ADR-020`; one without is governed by this record. When a deployable's kind is
   arguable — a bundle that is also served by a .NET host, say — the owning repository names which
   record applies, in the repository, before its first release.
2. 🎯 **One declaration per repository.** The toolchain version is stated in exactly one committed
   file, and every other consumer of it — the pipeline, a container image, a contributor's local
   environment — reads that file rather than repeating the value. This is `ADR-020` §2 obligation 3
   generalised: four independent pins were four chances to migrate a deployable halfway, and the same
   arithmetic holds for two.
3. 🎯 **A lockfile is committed and the pipeline installs from it.** A manifest without a lockfile
   describes an intention; a lockfile is what makes the build reproducible and what makes a
   dependency review possible at all. This rule is also the precondition for the dependency scanning
   that `ADR-022` F2 owes `GAP-13155-04` — nothing can be scanned that is not first pinned.
4. 🎯 **The declared version is in vendor support on the day it is committed, and the record says
   when that support ends.** A version whose support-end date is unknown or already past is not
   adoptable under this rule. The platform's existing condition — every deployable on a runtime
   unsupported since December 2022 — is the outcome this rule exists to prevent repeating, not a
   precedent it inherits.
5. 🎯 **A review is scheduled before the support-end date, and the review is the obligation, not the
   upgrade.** Booking a dated review is what converts drift from a condition into an event. Whether
   the upgrade is taken at that review is a decision with its own trade-offs; whether the review
   happens is not.
6. 🎯 **Uniformity is per toolchain family, not per deployable.** A second deployable in the same
   family joins the baseline the first one declared or changes it for both, following `ADR-020` §2
   obligation 2 — "uniformity is the part worth keeping; the version is the part that must change."
   It does not silently select its own. This rule constrains versions only; it decides nothing about
   frameworks, libraries or shell contracts, which remain deferred to `GAP-13155-09`.

**What this binds today.** Exactly one deployable: the `Pacco.Web` browser surface established by
`ADR-021`. Rules 2, 3 and 4 are satisfied when its dependency manifest is first committed, which
`ADR-021` obligation 3 already requires; rule 5 is satisfied by a dated entry, and it is the only one
of the six that needs a named person rather than a file.

## 5. Options Considered

| # | Option | Why it was rejected |
| --- | --- | --- |
| 1 | **A companion record governing non-.NET deployables, naming no version** | **Chosen.** It closes `R9` at the stage that owns it, leaves `ADR-020` factually intact, and binds the one deployable in scope without inventing a value that nobody has chosen. It is the first of the two courses `R9`'s `adr_recommendation` names |
| 2 | **Amend `ADR-020` to cover every deployable** | Rejected by fit-evaluation row 1. `ADR-020` records an observed fourfold .NET pin; widening its scope in place would make a current-state record assert a scope no source implements, and would tie a browser rule to a record written to be superseded by a .NET runtime move. It is the second course `R9` names, and it is the weaker of the two |
| 3 | **Leave `R9` to the platform owner as an open gap** | Rejected. This was the disposition this record replaces. `R9` scores RPN 280 with `mitigation_type: adr` and `owner_stage: architecture`, so the mitigation is a record this stage writes; deferring it leaves the platform's only currency rule inapplicable at precisely the moment a second toolchain arrives, and leaves `ADR-021`'s alignment exception with nothing on the other side of it. The deferral also mis-read the constraint: naming *a rule* needs no product input, whereas naming *a version* does |
| 4 | **Author one platform-wide toolchain standard covering runtime, frontend and everything else** | Rejected as a different artifact with a different owner. That is the standards catalogue `GAP-13155-08` records as absent across eleven rule families; producing one family of it inside a feature's architecture run would set ten precedents nobody asked for. `R15` warns against exactly this |
| 5 | **Pin a concrete Node and bundler version in this record** | Rejected. No toolchain has been selected — `ADR-021` Q3 leaves the choice to the `lld` stage — so any version named here would be invented rather than decided, and would be stale before it was applied. Rule 4 constrains *how* the version is chosen and leaves *which* to the stage that chooses the toolchain |
| 6 | **Require automated dependency updates instead of a scheduled review** | Rejected as premature and out of scope. No repository on the platform has dependency-update configuration of any kind (`ADR-020` §1 item 7), no pipeline performs dependency or image scanning (`…/baselines/architecture-baseline.md` §9.5), and automation without an owner produces pull requests nobody merges. Rule 5 requires the review; whether it is fed by automation is a `devops` decision, recorded as Q2 |

## 6. Consequences

### 6.1 Positive

1. 🎯 **`ADR-021`'s alignment exception now has something on the other side of it.** The exception
   remains correct — `ADR-020`'s scope genuinely does not reach a browser bundle — but it stops being
   a hole. A reader following the exception arrives at a rule rather than at a gap entry.
2. 🎯 **Drift acquires an event.** Rule 5 is the single change that distinguishes this platform's
   next toolchain from its current runtime, which nothing has ever triggered a review of.
3. 🎯 **The dependency-scanning follow-up becomes possible.** `ADR-022` F2 owes `GAP-13155-04` a
   content-security policy and dependency scanning in the `Pacco.Web` pipeline. Rule 3's lockfile is
   the artifact a scanner reads; without it the follow-up has no input.
4. 🎯 **The second non-.NET deployable inherits a rule instead of a precedent.** Rule 6 gives it a
   baseline to join, which is a narrow, version-only answer to the failure mode `R15` describes —
   without pre-empting the frontend standard that `GAP-13155-09` still owns.
5. 🎯 **`ADR-020` is left factually intact.** Its evidence, its scope and its supersession path are
   unchanged, so the runtime move it calls for is no harder to make than it was.

### 6.2 Negative

1. 🎯 **Two records now govern one concern, and they can drift apart.** If the platform moves to a
   supported .NET runtime and supersedes `ADR-020`, the successor must restate the split in rule 1 or
   the browser surface silently falls outside both. Carried as follow-up F3.
2. 🎯 **This record binds exactly one deployable today, and that deployable does not exist yet.**
   Nothing in the workspace satisfies any of the six rules, because `Pacco.Web` holds one `README.md`
   (`ADR-021` E1). Every rule here is 🎯 and none is ✅, so the record's value is entirely in
   constraining work that has not been written.
3. 🎯 **Rule 5 needs a named person and there is not one.** A scheduled review with no owner is a
   date in a document. This is `B1`/`GAP-13155-14` again, and this record cannot escape it any more
   than the other three can.
4. 🎯 **Rule 4 will be uncomfortable at the first application.** The platform's own .NET runtime
   fails it. A rule that the existing eleven deployables would not pass is being applied to the
   twelfth, and whoever applies it should read `ADR-020` §4.2 rather than treating the asymmetry as
   an oversight — it is the point.
5. ❓ **"In vendor support" is not uniformly defined across the ecosystems this record could reach.**
   For a Node runtime it is a published LTS schedule; for a bundler or a lint tool it may be nothing
   more than a major-version convention. Rule 4 is therefore sharper for runtimes than for tools, and
   Q1 records the question rather than pretending it is settled.

### 6.3 Neutral

1. No pipeline, manifest or image changes as a result of this record. Its first application is the
   `Pacco.Web` dependency manifest that `ADR-021` obligation 3 already requires.
2. The record names no technology, so it neither endorses nor forbids any framework, bundler or
   package manager. `ADR-021` Q3 is untouched.
3. `.NET Core 3.1` remains the platform runtime and remains unsupported. Nothing here accelerates or
   delays the migration `ADR-020` §2 obligation 1 calls for.

## 7. Compliance Considerations

| Governing source | Rule as it applies here | How this decision conforms |
| --- | --- | --- |
| `docs/adr/dotnet-core-31-as-platform-runtime-baseline.md` §2 — "Every Pacco deployable targets one runtime version… No service selects its own runtime." | The rule's scope is the .NET runtime of a Pacco deployable | Conforms, and completes. This record neither widens nor narrows that scope; §4 rule 1 states the split explicitly so no deployable falls between the two records. `ADR-021`'s `ARCHITECTURE_ALIGNMENT_EXCEPTION` remains an accurate description of `ADR-020`'s reach |
| `docs/adr/dotnet-core-31-as-platform-runtime-baseline.md` §2 obligations 2, 3 and 4 — uniformity survives the move; one pin location per repository; a scheduled review before support ends | Three obligations stated for .NET and worth generalising | Conforms. §4 rules 6, 2 and 5 are those three obligations restated for a deployable with no .NET runtime, deliberately in the same order and the same terms so a reader can see they are the same rule |
| `docs/adr/repository-per-service-independent-release.md` §2 — every deployable owns its own repository, image and release | A per-repository rule must not require a coordinated multi-repository change | Conforms. Every obligation here is satisfiable inside one repository, and rule 6's family uniformity is a review obligation at the point a second deployable joins, not a synchronised release |
| `docs/adr/standalone-browser-surface-in-pacco-web.md` §4 obligation 3 — the surface ships a committed dependency manifest, build and lint/format baseline | The obligation says the baseline exists; it does not say what governs its version | Conforms and extends. §4 rules 2, 3 and 4 say what that manifest must contain and how its version is chosen, which is the part `ADR-021` left to `GAP-13155-07` |
| `docs/architecture-inventory/patterns/index.md` — "**Every pattern here is `Candidate`**" | No pattern in the catalog is binding | Honoured. §9 relates this decision to the catalog without claiming approval the catalog does not grant |
| Deployment and infrastructure standards (toolchain currency, dependency policy, supply-chain scanning) | No `docs/standards/` directory exists in this repository and no covering rule family is catalogued | No silent default is taken. The absence is `GAP-13155-08`, and this record states that it supplies one rule for one class of deployable rather than the family that gap describes |

## 8. Non-Functional Requirements & Testing

| NFR | Target | How this decision addresses it | How it is verified |
| --- | --- | --- | --- |
| `NFR-23` [maintainability] | A committed frontend build and lint baseline, governed rather than merely present | §4 rules 2, 3 and 4 — one declaration, a committed lockfile, and a supported version at the point of commit | The repository contains exactly one file declaring the toolchain version, a lockfile is committed, the pipeline installs from the lockfile rather than resolving afresh, and the declared version's support-end date is recorded and in the future |
| `NFR-2` [security] | Zero hard-coded credentials, and a dependency set that can be assessed | §4 rule 3 — a lockfile is the artifact a dependency or secret scan reads | A dependency scan runs against the committed lockfile and produces a result; an unpinned transitive set would produce none |

**Testing obligations this decision creates.** `OQ-4` records that the platform has no house standard
to inherit — the build is not broken by a failing test suite and the suite is not run in CI
(`…/baselines/architecture-baseline.md` §9.5). Three checks therefore have to be specified rather
than assumed, and they belong to the `devops` stage that builds the `Pacco.Web` pipeline:

1. A build-time assertion that the pipeline's toolchain version and the repository's single
   declaration agree. This is the only mechanical guard on rule 2, and its absence is precisely how
   `ADR-020` §4.2 item 5 describes a half-migrated deployable arising.
2. An install step that fails when the lockfile is absent, out of date with the manifest, or bypassed.
   A lockfile that the pipeline does not install from satisfies rule 3 on paper and nothing in fact.
3. A check that the recorded support-end date is in the future at build time, so rule 4 degrades into
   a failing build rather than into a stale line in a document.

## 9. Relationship to the implementation pattern catalog

| Pattern | Relationship |
| --- | --- |
| [`deployment/independent-per-repository-release.md`](../architecture-inventory/patterns/deployment/independent-per-repository-release.md) | **Relied on, not changed.** The pattern establishes that each repository releases alone, which is what makes a per-repository version declaration sufficient. This record adds no cross-repository step to it |
| [`other/framework-supplied-platform-conventions.md`](../architecture-inventory/patterns/other/framework-supplied-platform-conventions.md) | **Bounded.** The pattern describes conventions arriving from an external toolkit rather than from platform code — the same shape as a version line nobody on the platform reviews. §4 rule 5 is the review this record attaches to that shape for non-.NET deployables |
| [`deployment/composable-per-concern-environment-stacks.md`](../architecture-inventory/patterns/deployment/composable-per-concern-environment-stacks.md) | **Unaffected.** A toolchain version is a build-time property; nothing in the environment stacks reads it |

Every pattern in the catalog is `Candidate`, so none of these relationships constitutes approval.

## 10. Follow-Up Actions

| # | Action | Owner | Due | Blocks |
| --- | --- | --- | --- | --- |
| F1 | Apply rules 2, 3 and 4 when the `Pacco.Web` dependency manifest is first committed: one declaration, a committed lockfile, and a version whose support-end date is recorded and in the future | `lld` stage, with the `devops` stage for the pipeline | With `ADR-021` obligation 3 | `NFR-23`; `ADR-021` F4 |
| F2 | Book the rule 5 review for the `Pacco.Web` toolchain, dated before the declared version's support ends, against a named person | Platform owner (unassigned — see B1) | Within one month of the first release | `R9`'s residual; `GAP-13155-07` |
| F3 | When `ADR-020` is superseded by a supported-runtime record, restate the §4 rule 1 split in the successor so no deployable falls outside both records | Architecture owner (unassigned — see B1) | At the runtime migration | `ADR-020` §2 obligation 1; §6.2.1 |
| F4 | Fold the lockfile requirement into the dependency-scanning work `ADR-022` F2 owes `GAP-13155-04`, rather than specifying a second scanning mechanism | Platform security owner (unassigned — see B1) | 2026-10-17 | `GAP-13155-04`; `R3` |
| F5 | Decide whether this record's rules are absorbed into a deployment-and-infrastructure standards family if `docs/standards/` is ever authored | Architecture owner (unassigned — see B1) | At the standards catalogue's authoring | `GAP-13155-08` |

## 11. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | `ADR-020` states the single-runtime rule and confines it to a runtime pinned in project files, pipeline SDK version and both container image stages | ✅ | `docs/adr/dotnet-core-31-as-platform-runtime-baseline.md` §2 |
| E2 | `ADR-020` carries obligations for uniformity surviving the move, one pin location per repository, and a scheduled review before the support end date | ✅ | `…/dotnet-core-31-as-platform-runtime-baseline.md` §2 obligations 2, 3, 4 |
| E3 | `.NET Core 3.1` reached end of support on 13 December 2022 and every deployable still targets it | ✅ | `…/dotnet-core-31-as-platform-runtime-baseline.md` §1 items 1–5; `docs/architecture-inventory/baselines/architecture-baseline.md` §11.2 C9 |
| E4 | No repository holds an upgrade branch, dependency-update configuration or deprecation note referencing the runtime | ✅ | `…/dotnet-core-31-as-platform-runtime-baseline.md` §1 item 7, §4.2 item 6 |
| E5 | `ADR-021` records an `ARCHITECTURE_ALIGNMENT_EXCEPTION` against `ADR-020` and carries the resulting gap as `GAP-13155-07` | ✅ | `docs/adr/standalone-browser-surface-in-pacco-web.md` §7; F4 |
| E6 | `ADR-021` obligation 3 requires a committed dependency manifest, build and lint/format baseline, and says nothing about its version | ✅ | `…/standalone-browser-surface-in-pacco-web.md` §4 obligation 3 |
| E7 | The existing browser asset has no bundler, transpiler, minifier or lockfile, and its vendored SignalR client's version cannot be established | ✅ | `docs/architecture-inventory/baselines/ui-inventory.md` §5.1; `…/baselines/architecture-baseline.md` §7.2 |
| E8 | No `docs/standards/` directory exists in this repository, and eleven rule families — including deployment and infrastructure — have no covering content | ✅ | `docs/architecture-inventory/risk-constraint-gap-register.md` `GAP-13155-08`; `ADR-021` E14 |
| E9 | No pipeline on the platform performs dependency or image scanning | ✅ | `…/baselines/architecture-baseline.md` §9.5 |
| E10 | The eleven existing pipelines are `language: csharp` with `dotnet: 3.1.100` and do not transfer to a browser artifact | ✅ | `ADR-018` §6; `ADR-020` §1 item 2 |
| E11 | `R9` scores S 5, O 7, D 8, RPN 280, with `mitigation_type: adr` and `owner_stage: architecture`, and its `adr_recommendation` names extending `ADR-020` or authoring a companion record | ✅ | `docs/architecture-inventory/risk-constraint-gap-register.md` §5.1, §5.2 `R9` |
| E12 | The platform's catalogued knowledge holds no toolchain-governance constraint for Node or any non-.NET toolchain — every catalogued mention records the existing frontend's *absence* of tooling | ✅ | Tenant knowledge-graph query over `Technology`, `Pipeline`, `Constraint` and `QualityGate` entities matching `node`, `npm`, `bundler`, `toolchain`, `frontend` and `lint`; corroborated by E7 |

### 11.1 Documentation-versus-code conflicts

| # | Conflict | Resolution |
| --- | --- | --- |
| X1 | `ADR-020` §2 states the rule as "**Every** Pacco deployable targets one runtime version", while its §1 evidence covers only the eleven .NET deployables and its pin locations are all .NET artifacts. Read literally, the rule claims a scope its evidence does not support | The evidence wins, and `ADR-021` §7 already read it that way in declaring the exception. This record does not amend `ADR-020`'s wording; §4 rule 1 states the operative split so that the literal reading cannot be used to claim a browser bundle is already governed |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The toolchain the `lld` stage selects publishes a support lifecycle that a support-end date can be read from | Every mainstream browser-tooling runtime publishes a release schedule, and `ADR-020` assumed the same of .NET without stating it | Rule 4 has no date to check and rule 5 no date to book against, so both degrade into a convention rather than a rule | The `lld` stage records the support-end date alongside the version at the point of selection; if none is published, Q1's answer applies instead |
| A2 | One repository per non-.NET deployable remains true, so a per-repository declaration is a per-deployable declaration | `ADR-018` §2 makes it the platform's rule, and `ADR-021` §4 keeps the browser surface inside it | A repository holding two deployables with different toolchains would satisfy rule 2 while leaving one of them ungoverned | Confirm at the point any repository is proposed to hold more than one deployable |
| A3 | No non-.NET deployable exists outside the fourteen cloned repositories | The absence is proven exhaustively inside the clone set, which is fixed by backlog issue 12998; what lies outside it was not observable | This record would be governing one deployable while another already runs ungoverned, with conventions it does not know about | The same question `ADR-021` A3 asks: whether any Pacco artifact exists outside the fourteen |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so F2's scheduled review has nobody to schedule it and this record cannot leave `Proposed` | F2 through F5, and this ADR's ratification | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | At the work item review that receives this record — it blocks ratification |
| B2 | **[ACTION NOW]** The platform's eleven existing deployables would fail §4 rule 4 today, because `.NET Core 3.1` has been out of support since December 2022. Applying a currency rule to the twelfth while the other eleven fail it is defensible only if the migration `ADR-020` §2 obligation 1 requires is actually funded | Any claim that the platform as a whole is toolchain-current; the credibility of rule 4 at its first application | Platform owner | `ADR-020` §2 obligation 1 — move to a supported runtime and supersede that record | Not set by this record; `ADR-020` leaves it open as its Q1 |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[handled later by the `lld` stage]** What counts as "in vendor support" for a build tool that publishes no support lifecycle, as distinct from a runtime that does? | Rule 4 is sharp for a runtime and vague for a linter, and a rule that is vague where it is most often applied will be applied inconsistently. See §6.2.5 | Apply rule 4 strictly to the runtime and the package manager, where lifecycles are published, and to everything else apply the weaker test that the declared version is the current major line at the point of commit. Record which test was used | `lld` stage |
| Q2 | **[handled later by the `devops` stage]** Is the rule 5 review fed by automated dependency updates, or performed by hand? | No repository on the platform has dependency-update configuration, and automation with no owner produces pull requests nobody merges (§5 option 6) | Perform the first review by hand, and introduce automation only once F2 has a named owner to receive its output | `devops` stage |
| Q3 | **[ACTION NOW]** Does this record bind a future non-.NET deployable that is not a browser artifact — a CLI, a worker, a build tool shipped as a deployable? | §4 rule 1 says it does, and nobody has confirmed that the platform wants one rule spanning every non-.NET kind rather than one per kind | Yes, as written. The three rules constrain declaration, reproducibility and currency, none of which is browser-specific; splitting the record per kind would recreate `R15` on a different axis | Architecture owner |
