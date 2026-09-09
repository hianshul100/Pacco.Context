# ADR-013: Contract-blind universal message subscription from a manifest

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-013` |
| **Backlog candidate** | `ADR-CANDIDATE-013` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Integration / medium |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-003` (the naming convention this depends on, and the source of question Q3 answered here), `ADR-001` (the exchanges the manifest enumerates), `ADR-014` (the only consumer of what this subscription produces) |

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

`ADR-003` decided that message contracts are matched by name rather than by a shared assembly: a
publisher and a subscriber agree because the message class name maps to the same routing key. That
decision works for a service that consumes five specific messages. `operations-service` needs to
observe *every* message on the platform in order to report progress to callers, and it has no interest
in any payload — only in the fact that a named message arrived and what state header it carried.

1. ✅ **One service subscribes to the whole platform.** `operations-service` reads a manifest file
   listing message names grouped by exchange and by kind, and subscribes to all of them
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs:18-152`).
2. ✅ **The subscriber has no compiled contract for any of those messages.** It builds the types at
   startup with the runtime type-emission facilities of the framework — a dynamic assembly, a dynamic
   module, and one emitted type per manifest entry (`…/Infrastructure/Subscriptions.cs:42-83`).
3. ✅ **The emitted types carry no fields.** Each is defined with a base type of command, event or
   rejected event and a message attribute naming its exchange, and nothing else. The subscriber
   therefore cannot read a single value out of any message it receives.
4. ✅ **Three generic handlers serve all of them.** One handler per kind is registered against the base
   interfaces (`…/Infrastructure/Extensions.cs:53-55`), and subscription is wired by a single startup
   call (`…/Infrastructure/Extensions.cs:90`).
5. ✅ **The manifest holds eighty message names across eight exchanges.** Counted directly from
   `…/Pacco.Services.Operations.Api/messages.json`: twenty-four commands, twenty-nine events and
   twenty-seven rejected events, grouped under the availability, customers, deliveries, identity,
   ordermaker, orders, parcels and vehicles exchanges. There is no entry for the operations exchange
   itself.
6. ✅ **A missing, blank or empty manifest is silently tolerated.** Each of the three cases returns the
   subscriber unchanged (`…/Infrastructure/Subscriptions.cs:22-37`), so the service starts, reports
   healthy, subscribes to nothing, and reports no operation to any caller.
7. ✅ **The manifest is hand-maintained.** No build step, script, test or tool in any of the fourteen
   clones writes or checks it, so it drifts whenever a service adds or renames a message.

### 1.1 What this record does *not* cover

This record fixes how the platform-wide observer learns what to subscribe to. It does not cover the
naming convention that makes name-only matching work (`ADR-003`), the exchange topology the manifest
enumerates (`ADR-001`), or what the observer does with a received message (`ADR-014`). It does not
extend to any component that needs a payload — that exclusion is part of the decision.

## 2. Decision

**Pacco lets one service — the platform-wide operation observer — subscribe to every message on every
exchange without holding a compiled contract for any of them, by reading a checked-in manifest of
message names and emitting a field-less type per name at startup. This technique is confined to
components that need only the arrival of a message and its headers, never its payload.**

Five rules follow from the decision and are part of it:

1. ✅ **Only the observer may do this.** Every other subscriber on the platform declares the message
   types it consumes as ordinary classes. A second contract-blind subscriber would be a change to this
   record, not an application of it.
2. ✅ **The emitted types are deliberately field-less.** They exist to name a routing key and a base
   kind. Any requirement to read a payload value ends the technique for that component and requires a
   real contract.
3. ✅ **The manifest is the platform's only enumerated list of messages.** Nothing else in the workspace
   lists what messages exist, so the manifest doubles as documentation whether or not that was intended.
4. 🎯 **The manifest must be generated at build time rather than hand-maintained.** This closes the
   question `ADR-003` carried forward: message names should be produced from the publishing services'
   compiled contracts, not typed by hand into a file in one consumer.
5. 🎯 **A missing or empty manifest must fail startup.** Silently observing nothing is the worst
   available failure, because the platform's only progress-reporting surface goes quiet while every
   health signal stays green.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **A shared contracts package** — publish every service's message classes as a library and reference it from the observer. | Rejected because it contradicts `ADR-002` and `ADR-003`: the platform deliberately has no first-party shared library, and adding one for the observer's benefit would make eleven services depend on a package version to release independently. It would also give the observer eighty compiled types it does not use a single field from. |
| 2 | **Subscribe with a wildcard routing key** — bind one queue to each exchange with a match-everything pattern and handle raw payloads. | Rejected because the subscription layer in use binds a queue per message type using the type's name, so there is no supported wildcard binding; taking this route would mean bypassing the messaging abstraction the other ten deployables share and hand-writing broker code in one service. It would also lose the command/event/rejected-event distinction, which is exactly what the observer uses to decide the reported state. |
| 3 | **Have each service report its own operations** — every service publishes a progress message to the observer's exchange. | Rejected because it puts progress reporting into eleven services instead of one, requires every publisher to know the observer exists, and would need the same eighty call sites to be maintained by hand — the manifest's maintenance burden, multiplied and distributed. |
| 4 | **Do without a platform-wide observer** — let each caller poll the owning service for its own result. | Rejected because the platform's write path is asynchronous by design (`ADR-005`): the caller is handed a correlation id and has no service to poll, since the accepting service is the gateway and the completing service varies by message. Some component has to watch all exchanges; this record decides how it does so. |

## 4. Consequences

### 4.1 Positive

1. ✅ Adding a message to the platform requires no code change in the observer — one manifest line, and
   progress reporting works for it.
2. ✅ The observer stays independent of every other repository. It compiles and releases with no
   reference to any service, which is what keeps `ADR-002`'s no-shared-library position intact.
3. ✅ The technique costs one file of roughly one hundred and sixty lines and three small handlers, in
   exchange for coverage of eighty message types.
4. ✅ The manifest is a readable, sorted, single-file inventory of the platform's messages — the closest
   thing the workspace has to a message catalogue.

### 4.2 Negative

1. ✅ **The manifest drifts silently.** It is hand-maintained (context point 7) with nothing verifying
   it, so a newly published message is simply not observed and no error is raised anywhere.
2. ✅ **A missing or empty manifest disables the platform's progress reporting without a signal.** The
   service starts and answers health checks while subscribing to nothing.
3. ✅ **The observer can never enrich what it reports.** Field-less types mean the report can say a
   named message completed, and nothing about what it did.
4. ✅ **Type emission is opaque to tooling.** No editor, compiler or static analyser can tell what the
   observer subscribes to; the answer exists only at runtime, derived from a data file.
5. ✅ **The technique conceals broken contracts.** Because the observer builds a type for any name it is
   given, a manifest entry for a message that nothing publishes — the coordinator's rejection message is
   one such entry — produces a live subscription that will never fire, with no warning.

### 4.3 Neutral / follow-on

1. ❓ The manifest has no group for the observer's own exchange (evidence 11). `[INFERRED]` that this is
   deliberate rather than an omission, because subscribing to it would make the observer consume its own
   progress notifications; nothing in any clone states the intent, so the reason is assumed (assumption
   A1).
2. ✅ The three generic handlers are what make the technique useful rather than merely clever; the
   handlers, not the emitted types, are where the observer's behaviour lives (`ADR-014` §1).
3. ✅ The observer subscribes to the coordinator's exchange (`ADR-011`) like any other, so orchestrated
   and choreographed processes are reported through one mechanism.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates `integration/declarative-message-manifest-subscription.md`.** This record is the
   decision behind that pattern: a data file drives subscription, types are emitted at runtime, and
   generic handlers absorb everything. The pattern is catalogued with confidence *Moderate* and one
   deployable using it, which matches the evidence here.
2. **Adopts the pattern's two recommendations as decision rules.** The catalog recommends generating the
   manifest at build time and failing startup on a missing or empty manifest. Rules 4 and 5 make both
   part of the decision rather than advice attached to it.
3. **Constrains the pattern's scope.** The catalog warns against extending the technique to any
   component that consumes payloads. Rule 2 states that limit as a boundary of the decision, and rule 1
   confines the technique to one named service.
4. **Depends on the decision recorded in `ADR-003`** (`docs/adr/message-contracts-by-naming-convention.md`).
   Name-only matching is the precondition: a field-less emitted type can subscribe correctly only because
   the routing key is derived from the type's name and nothing else. If contract matching ever became
   structural, this record would be invalidated in full. The catalog holds no separate pattern for
   name-based contract matching, so the dependency is on the ADR rather than on a pattern file.
5. **Depends on `integration/service-owned-topic-exchange-messaging.md`.** The manifest's top-level
   grouping is one entry per service exchange, so the manifest is a direct restatement of that topology
   and drifts with it.
6. **Pattern Drift: not applicable.** Every entry in `docs/architecture-inventory/patterns/index.md`
   currently carries status `Candidate`. Drift is reportable only against an `Approved` pattern, so no
   drift is recorded for this ADR.
7. **Pattern Update Proposal.** `integration/declarative-message-manifest-subscription.md` states the
   manifest holds "roughly 80 types"; the exact figures are twenty-four commands, twenty-nine events and
   twenty-seven rejected events across eight exchanges, and the pattern file should carry the exact
   counts and the counting method. Its **Related ADRs** entry should change from `None` to `ADR-013`.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| 1 | The manifest is read from the working directory at startup | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs:18-24` |
| 2 | A missing file returns the subscriber unchanged | ✅ | `…/Infrastructure/Subscriptions.cs:22-24` |
| 3 | A blank file returns the subscriber unchanged | ✅ | `…/Infrastructure/Subscriptions.cs:28-31` |
| 4 | A manifest with no entries returns the subscriber unchanged | ✅ | `…/Infrastructure/Subscriptions.cs:34-37` |
| 5 | A dynamic assembly and module are created at startup | ✅ | `…/Infrastructure/Subscriptions.cs:42-44` |
| 6 | One field-less type is emitted per manifest entry, with a base kind and an exchange attribute | ✅ | `…/Infrastructure/Subscriptions.cs:61-83` |
| 7 | Subscriptions are established by reflection over the emitted types | ✅ | `…/Infrastructure/Subscriptions.cs:85-152` |
| 8 | Three generic handlers are registered, one per message kind | ✅ | `…/Infrastructure/Extensions.cs:53-55` |
| 9 | Subscription is triggered by one startup call | ✅ | `…/Infrastructure/Extensions.cs:90` |
| 10 | The manifest lists eighty message names across eight exchanges (24 commands, 29 events, 27 rejected events) | ✅ | `…/Pacco.Services.Operations.Api/messages.json` (counted across all groups) |
| 11 | The manifest contains no group for the observer's own exchange | ✅ | `…/Pacco.Services.Operations.Api/messages.json` |
| 12 | The manifest lists a coordinator rejection message that nothing publishes | ✅ | `…/Pacco.Services.Operations.Api/messages.json`; `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Events/Rejected/MakeOrderRejected.cs` (workspace-wide search finds no publisher) |
| 13 | No build script, test or tool in any clone generates or validates the manifest | ✅ | Workspace-wide search across the fourteen clones' `scripts/`, `*.csproj` and continuous-integration files |

### 6.1 Documentation-versus-code conflicts

1. **The recorded message counts are wrong.** `docs/architecture-inventory/adr-candidates.md` states the
   manifest holds twenty-six commands, thirty events and thirty-one rejected events — eighty-seven
   names — and its assumption A3 repeats "roughly 87 message names". The file actually holds twenty-four,
   twenty-nine and twenty-seven, for eighty. The code is authoritative: the counts in this record come
   from counting the manifest's entries directly. The lower figure agrees with two other inventory
   documents, which describe "roughly 80" messages, so the eighty-seven figure appears to be a single
   miscount that propagated into one assumption. The backlog is corrected as part of this batch; the
   conflict is recorded here rather than silently overwritten.
2. **A subscription for a message that is never published.** The manifest declares a rejection message
   under the coordinator's exchange and the coordinator declares the matching class, but no publisher
   exists (evidence 12). The observer therefore holds a live subscription that can never fire, and a
   caller waiting for a rejected order creation waits forever. Recorded as **Future/Intended State (Not
   Implemented)** — the manifest describes the intended contract set, the code shows one member of it
   missing.
3. **No stated intent for manifest maintenance.** Nothing in any clone says whether the manifest is
   meant to be hand-edited permanently or was a first step toward generation. Decision rule 4 states the
   target; the current position is recorded as **Unverifiable — Missing Source Evidence** because no
   design note either way exists.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The manifest is meant to list every message on the platform, not a chosen subset | It groups by every service exchange except the observer's own, and covers all three message kinds for each | If it is a curated subset, the drift risk in negative consequence 1 is a design choice rather than a defect, and decision rule 4 becomes unnecessary | Compare the manifest's names against the message classes declared in the ten publishing repositories and list every name that exists in code but not in the manifest |
| A2 | The observer never needs a payload value from any message | The emitted types are field-less by construction (evidence 6), and the reporting behaviour uses only the message name and headers (`ADR-014` §1) | If a reported operation ever needs a value from the message, the technique cannot supply it, and that component needs real contracts — which rule 2 already anticipates | Review the caller-facing progress surface with the platform owner and confirm no field beyond message name, correlation id and state is required |
| A3 | The eighty names in the manifest correspond to real published messages, apart from the one identified exception | Spot checks matched manifest names to publishing services for each exchange, and only the coordinator's rejection message had no publisher | If more names are stale, the observer holds more subscriptions that can never fire, and the manifest is less trustworthy as the platform's message inventory than context point 3 claims | Run A1's comparison in the other direction: list every manifest name with no publisher in any of the fourteen clones |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[handled later by the architecture PR review stage]** No repository in the workspace names an owner, a team or a review group, so this record cannot list deciders | This ADR leaving `Proposed`, and rules 4 and 5 in §2, which both need someone accountable for the manifest | Platform owner (unassigned) | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository; the reviewer assigns deciders when the batch is reviewed as a whole | TBD |
| B2 | **[ACTION NOW]** The published inventory's message counts disagree with the manifest, so any planning that used the higher figure is off by seven messages | Any sizing done against the message inventory, and the two other inventory documents that carry approximate figures | Platform owner | The counts in `docs/architecture-inventory/adr-candidates.md` are corrected to 24/29/27 in this change; confirm the corrected figures, then run A1's comparison so the inventory is verified rather than recounted by hand next time | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[handled later by the batch 4 authoring stage, in the record covering the build and release path]** Should the manifest be generated from the publishing services' compiled contracts at build time? | This is the question `ADR-003` handed to this record. Generation removes the drift in negative consequence 1, but it needs a build step that can see eleven repositories, which the current independent-release model (`ADR-017`) does not provide | Have each publishing repository emit its own message-name list as a build artifact, and have the observer's build concatenate them — so no build needs to see more than one repository | Platform owner |
| Q2 | **[ACTION NOW]** Should a missing or empty manifest stop the service from starting? | Rule 5 says it should. The counter-argument is that a degraded observer is better than a service that will not boot, and nobody has stated which failure the platform prefers | Fail startup. A silently empty observer takes the platform's only progress-reporting surface offline while every health signal stays green, which is harder to detect than a service that does not start | Platform owner |
| Q3 | **[handled later by the architecture PR review stage]** Should the manifest be moved out of the observer's source tree, given that it describes the whole platform rather than one service? | It is the platform's only message inventory (context point 3) but lives in, and is deployed with, one consumer. Moving it changes who is responsible for keeping it current | Leave it where it is until Q1 is answered; if the manifest becomes generated, ownership moves to the publishing repositories and the question resolves itself | Platform owner |

