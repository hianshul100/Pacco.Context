# ADR-005: Dual-mode edge writes selected by configuration rather than by code

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-005` |
| **Backlog candidate** | `ADR-CANDIDATE-005` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Integration / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-004` (the declarative edge this switch lives in), `ADR-001` (the exchanges the async mode publishes onto), `ADR-006` (the identity binding both modes depend on), `ADR-CANDIDATE-014` (where the deferred outcome is reported) |

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

`ADR-004` established that the whole north-south edge is a YAML document interpreted by an embedded
gateway library rather than gateway code. That decision has a consequence nobody has yet recorded: it
made it possible to change the platform's *write semantics* by editing configuration.

1. ✅ **Four gateway configuration files exist,** and they differ architecturally rather than by
   hostname. The two synchronous files route every one of forty routes to a downstream service. The two
   asynchronous files route twenty of those forty to a service exchange instead, leaving twenty reads
   as downstream calls
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada.docker.yml`,
   `ntrada-async.yml`, `ntrada-async.docker.yml`).
2. ✅ **The same URL means two different things depending on which file is loaded.** Reserving a
   resource is a downstream call in the synchronous configuration
   (`ntrada.yml:96-104`) and a publication of a named command onto the availability exchange in the
   asynchronous one (`ntrada-async.yml:123-133`). The path, the method and the identity binding are
   identical; only the transport changes.
3. ✅ **Which file loads is an environment variable read at startup,** falling back to a command-line
   argument and then to the synchronous default
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Program.cs:30-38`).
4. ✅ **The workspace disagrees with itself about which mode is intended.** The gateway's own container
   image defaults to the *synchronous* Docker configuration
   (`hianshul100_Pacco.APIGateway/Dockerfile:11`), the platform's main Compose service stack overrides
   it to the *asynchronous* one (`hianshul100_Pacco/compose/services.yml:9`), the local Compose stack
   selects the *synchronous* one (`hianshul100_Pacco/compose/services-local.yml:9`), and a start script
   in the gateway repository selects the asynchronous one
   (`hianshul100_Pacco.APIGateway/scripts/start-async.sh:3`).
5. ✅ **Nothing in the workspace states which mode production uses.** No environment document, no
   deployment manifest outside Compose, and no README statement names one.

This is a decision about the platform's public write contract that is currently expressed only as a
default and three overrides.

### 1.1 What this record does *not* cover

This record fixes *that* the edge offers two write modes and how the choice is made. It does not fix the
route table itself (`ADR-004`), the authentication and claim gating applied to both modes (`ADR-006`),
the exchange ownership the asynchronous mode publishes onto (`ADR-001`), or the mechanism by which a
caller learns the outcome of an accepted write (`ADR-CANDIDATE-014`). It also does not choose which mode
production should run — that is blocker B2 below.

## 2. Decision

**Pacco keeps both a synchronous and an asynchronous edge write mode, expresses the choice between them
as a whole-gateway configuration selection rather than as a per-route or per-code decision, and treats
the two modes as two different public API contracts for the same URLs.**

Four rules follow from the decision and are part of it:

1. ✅ **The choice is whole-gateway, not per route.** A configuration file is either the synchronous
   pair or the asynchronous pair; there is no file mixing modes for the same resource. Twenty write
   routes move together.
2. ✅ **Read routes never change mode.** All twenty read routes are downstream calls in all four files.
   Only writes are dual-mode.
3. 🎯 **Each environment must declare its mode explicitly.** Relying on the image default is not
   acceptable, because the image default is the synchronous configuration while the platform's own
   service stack overrides it to the asynchronous one — an environment that forgets the override
   silently changes the contract.
4. 🎯 **A caller-facing description of both contracts must exist before either mode is exposed to a
   client that is not the platform team.** Today the only description of either is the gateway
   configuration itself.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Pick one mode and commit the edge to it** — either all writes proxy synchronously, or all writes publish. | Rejected, and this is the alternative most worth revisiting. Committing to synchronous would give every write a real status code and a result, at the cost of coupling caller latency to the slowest service in the chain and losing the buffering the broker provides. Committing to asynchronous would give uniform acknowledgement semantics, at the cost that every caller must handle a deferred outcome and the platform must keep the notification channel (`ADR-CANDIDATE-014`) permanently available. Keeping both defers that trade-off rather than resolving it, which is why this record exists at all. |
| 2 | **Decide the mode per route** — publish the writes that are genuinely long-running, proxy the ones that are not. | Rejected implicitly: no configuration file mixes modes for a resource, so nothing in the workspace shows this being attempted. It is the design most likely to be right — resource reservation and order approval have very different latency profiles — but it multiplies the contract surface, because a client would then need to know each route's mode rather than the gateway's mode. Revisit together with rule 4 in §2. |
| 3 | **Expose only the asynchronous mode publicly and keep the synchronous mode as a development convenience.** | Rejected because nothing marks either pair as development-only. Both pairs have a local and a Docker variant, both are complete, and both are selected by real launch paths in the workspace. Declaring one a convenience after the fact would be documentation, not a decision. |
| 4 | **Two separate gateways** — one synchronous edge and one asynchronous edge, deployed side by side on different hostnames. | Rejected because it doubles the operational surface for a platform that has no container orchestration at all (`ADR-CANDIDATE-017`), and because it would still leave the same twenty URLs with two contracts, only now simultaneously rather than alternately. |

## 4. Consequences

### 4.1 Positive

1. ✅ Switching the platform's write semantics requires no build and no code change — a different
   configuration file and a restart.
2. ✅ The asynchronous mode removes the gateway's dependency on the write-side services being reachable
   at request time: a write is accepted as long as the broker is available.
3. ✅ Both modes share one identity binding, so the caller's identity claim reaches the handler the same
   way whether it arrives on a forwarded URL or in a published message body (`ADR-006`).
4. ✅ The synchronous mode remains available as a debugging path — a developer can see a service's real
   status code and error body without a message round trip.

### 4.2 Negative

1. ✅ **No client can be written against the edge without knowing which configuration is deployed.**
   In the synchronous mode a write returns the service's result; in the asynchronous mode it returns an
   acknowledgement and defers the outcome. These are different contracts for the same URL and method.
2. ✅ **The default disagrees with the platform's own stack.** The container image defaults to
   synchronous and the main Compose stack overrides to asynchronous. Anyone running the image without
   the override gets the other contract, and nothing warns them.
3. ✅ **Error reporting differs fundamentally between the modes.** A synchronous write can return a
   validation failure directly; an asynchronous one cannot, and the failure surfaces later as a rejected
   event on the message topology (`ADR-001`). `[INFERRED]` a client that only handles HTTP status codes
   will treat a failed asynchronous write as a success.
4. ✅ **The asynchronous mode has a hard dependency on a channel this record does not own.** Without the
   status projection and push channel in `ADR-CANDIDATE-014`, an accepted write has no outcome path at
   all — and that channel holds status in a cache with a five-minute expiry.
5. `[INFERRED]` **Four files that must stay in step.** The two pairs differ only by mode and by whether
   local or container addresses are used, so a route added to one must be added to the other three by
   hand. Nothing checks that they agree.

### 4.3 Neutral / follow-on

1. ✅ Read routes are unaffected in all four configurations, so the read half of the platform's public
   API is stable across the switch.
2. ✅ The asynchronous mode is what makes the edge a publisher onto six services' exchanges, which is
   only possible because exchanges are per service and publicly named (`ADR-001`). It is a consequence
   of that topology, not an independent capability.
3. ✅ `pricing-service` has no exchange (`ADR-001`), so its routes are downstream calls in all four
   configurations regardless of mode.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/integration/dual-mode-edge-write.md` (Status: `Candidate`, evidence
   Strong). This record is that pattern adopted as the platform's edge decision, with rules 3 and 4 in
   §2 added as binding conditions the pattern states only as advice.
2. **Constrains** `patterns/integration/declarative-configuration-driven-api-gateway.md`: the
   declarative edge is what makes a mode switch a configuration edit, so the risk this record carries is
   the price of that pattern rather than a separate defect.
3. **Depends on** `patterns/orchestration/acknowledge-then-notify-completion.md`: the asynchronous half
   of this decision is incomplete without it, and that pattern's own weakness — a status store with a
   five-minute expiry — is inherited here as consequence §4.2 item 4.
4. **Deliberately diverges** from nothing in the catalog.
5. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*). If the dual-mode pattern is ever
   approved, the disagreement between the image default and the Compose override in §4.2 item 2 becomes
   drift and needs an owner.
6. **Pattern Update Proposal:** add a consistency check across the four gateway configurations to
   `patterns/integration/declarative-configuration-driven-api-gateway.md` — a build step asserting that
   the four files expose the same upstream paths and methods, differing only in transport and in
   address style. That closes §4.2 item 5 without changing any runtime component.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | Four gateway configuration files exist, in a synchronous and an asynchronous pair, each with a local and a container variant | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/{ntrada.yml, ntrada.docker.yml, ntrada-async.yml, ntrada-async.docker.yml}` |
| E2 | The synchronous configurations route all forty routes downstream and publish nothing | ✅ | `ntrada.yml` and `ntrada.docker.yml` — 40 downstream routes, no broker-routed route in either |
| E3 | The asynchronous configurations convert twenty write routes to broker publications and keep twenty reads downstream | ✅ | `ntrada-async.yml` — 20 broker-routed routes at lines 118, 126, 138, 149, 198, 210, 241, 249, 259, 269, 357, 367, 378, 390, 402, 444, 454, 506, 514, 524; same in `ntrada-async.docker.yml` |
| E4 | The same upstream path and method is a downstream call in one pair and a named command publication in the other | ✅ | `ntrada.yml:96-104` versus `ntrada-async.yml:123-133` (resource reservation) |
| E5 | The configuration file is chosen at startup from an environment variable, then a command-line argument, then a synchronous default | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/Program.cs:30-38` |
| E6 | The gateway container image defaults to the synchronous container configuration | ✅ | `hianshul100_Pacco.APIGateway/Dockerfile:11` |
| E7 | The platform's main Compose service stack overrides the image default to the asynchronous container configuration | ✅ | `hianshul100_Pacco/compose/services.yml:9` |
| E8 | The local Compose service stack selects the synchronous container configuration | ✅ | `hianshul100_Pacco/compose/services-local.yml:9` |
| E9 | A launch script in the gateway repository selects the asynchronous configuration | ✅ | `hianshul100_Pacco.APIGateway/scripts/start-async.sh:3` |
| E10 | The asynchronous mode publishes onto six services' exchanges by name, which is only possible because exchanges are per service | ✅ | `ntrada-async.yml:118-528` — exchanges `availability`, `customers`, `deliveries`, `orders`, `parcels`, `vehicles` |
| E11 | No document in the workspace states which configuration production uses | ✅ | Workspace-wide search of all Markdown, Compose and pipeline files — no production environment statement |

### 6.1 Documentation-versus-code conflicts

1. `docs/architecture-inventory/repo-inventory.md` §6 records gap G6 as "the Compose service stack
   selects the asynchronous Docker variant, and no document states production intent". The code
   confirms that and adds a detail the inventory does not: the **image default is the synchronous
   configuration**, and a second Compose stack (`services-local.yml`) selects it. So the workspace
   contains two selections of each mode, not one asynchronous selection against a silent default. The
   code is followed here, and the fuller picture is stated in §1 item 4 and §4.2 item 2.
2. No other conflict found. `docs/architecture-inventory/baselines/api-inventory.md` §2 and §3 describe
   the same forty routes and the same twenty-route transport difference the configurations show.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | A broker-routed write returns an acknowledgement rather than the service's result, and the caller must obtain the outcome elsewhere | The route has no downstream target to return a result from, and a separate service exists solely to project message outcomes into a client-visible status. The gateway library's response behaviour is not in this workspace, so the exact status code and body are not observable here | The asynchronous contract described in §4.2 would be wrong, and rule 4 in §2 would be aimed at the wrong problem | Send one write to a gateway running the asynchronous configuration and record the status code, headers and body it returns |
| A2 | A failed asynchronous write surfaces only as a rejected event on the message topology, with nothing returned to the original caller | Every layered service maps handler exceptions to a typed rejection message, and the accepted write has already been acknowledged by the time the handler runs | Callers might in fact receive a failure signal, and §4.2 item 3 would overstate the risk | Send a write that fails validation through the asynchronous edge and observe what, if anything, reaches the caller |
| A3 | The two configuration pairs are intended to be kept identical apart from transport and address style | Both pairs expose the same nine resource groups and the same upstream paths, and differ only in the twenty write routes and in local versus container addresses | A pair might be deliberately narrower than the other, and the consistency check proposed in §5 would fail on intended differences | Diff the four files route by route and have the gateway's owner confirm each difference is intentional |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so nobody can approve this record or own the four configuration files it describes | This ADR leaving `Proposed`, and rules 3 and 4 in §2, which both need someone to be accountable for an environment's declared mode | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has said which gateway configuration production runs. The image default says synchronous, the platform's main service stack says asynchronous, and the same twenty write URLs behave differently under each | The consequences in §4.2 cannot be stated as facts about the running platform, and `ADR-CANDIDATE-014` cannot decide whether the outcome-notification channel is on the critical path or optional | Platform owner | Confirm which configuration is loaded in each environment, record it in this repository, then delete or clearly mark the configurations that are not used | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should the platform keep both modes at all, or commit to one? | Keeping both means every client integration starts with "which mode is this environment", and means four configuration files must be maintained in step forever. Committing to one halves the contract surface | Commit to the asynchronous mode for writes if the notification channel in `ADR-CANDIDATE-014` is made durable; commit to the synchronous mode if it is not. The two decisions should be taken together, because an asynchronous write with an expiring status store has no reliable outcome path | Platform owner |
| Q2 | **[ACTION NOW]** What should a client do when it never receives an outcome for an accepted asynchronous write? | The status projection expires after five minutes. A write whose processing outruns that window has no recorded result, and no retry or reconciliation path exists anywhere in the workspace | Give the caller a way to re-query the write by correlation id against durable state, or extend the status retention beyond the longest saga. Until one of those exists, treat the asynchronous mode as unsuitable for writes that can take minutes | Platform owner |
| Q3 | **[handled later by adr_generation]** Should the mode be selectable per route rather than per gateway? | Alternative 2 in §3 is the design most likely to fit the platform's actual latency profile, but it changes what a client must know from one fact to forty | Decide after B2 answers what production runs today. If both modes survive, per-route selection with the mode published in the API description is better than a whole-gateway switch nobody can see from outside | `adr_generation` stage, in `ADR-CANDIDATE-014` |
