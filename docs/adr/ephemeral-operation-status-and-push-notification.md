# ADR-014: Ephemeral operation status in a shared cache, pushed over a real-time channel

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-014` |
| **Backlog candidate** | `ADR-CANDIDATE-014` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Orchestration / medium |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-005` (the acknowledge-then-continue write path this completes, and the source of question Q3 answered here), `ADR-013` (how the observer learns what to watch), `ADR-006` (the authorization gap this surface reproduces), `ADR-011` (the saga whose state header this reads) |

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

`ADR-005` decided that in the asynchronous gateway configuration a write is acknowledged immediately
with a correlation id, before any service has processed it. That leaves the caller holding a receipt and
no result. This record decides how the caller gets the result.

1. ✅ **One service turns observed messages into caller-facing progress.** The three generic handlers
   introduced by `ADR-013` read the correlation id and the saga-state header from each observed message
   and record an operation against that id
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Handlers/GenericEventHandler.cs:28-61`,
   `…/Handlers/GenericRejectedEventHandler.cs:28-62`).
2. ✅ **State comes from a message header, defaulting by message kind.** A helper reads the header the
   coordinator writes (`ADR-011`); when it is absent, an event is treated as completed and a rejected
   event as rejected (`…/Handlers/Extensions.cs:9-26`).
3. ✅ **Operation state lives only in the shared cache.** A single service class reads and writes entries
   under a fixed key prefix, with a sliding expiry taken from settings
   (`…/Services/OperationsService.cs:24-62`), registered once for the process
   (`…/Infrastructure/Extensions.cs:58`). The configured expiry is three hundred seconds.
4. ✅ **A completed or rejected operation cannot be moved backwards.** The write path refuses to
   overwrite a terminal state (`…/Services/OperationsService.cs:39-42`).
5. ✅ **Every state change is pushed to the caller in real time.** The service raises a notification that
   a hub service turns into one of three client messages, addressed to a group named for the user
   (`…/Services/HubService.cs:15-45`, `…/Services/HubWrapper.cs:17-21`,
   `…/Infrastructure/Extensions.cs:32-33`).
6. ✅ **The push channel is horizontally scalable by configuration.** A shared backplane is wired only
   when the configured value names it; any other value starts the hub with no backplane and no warning
   (`…/Infrastructure/Extensions.cs:95-109`).
7. ✅ **There is also a pull endpoint and a streaming endpoint.** A read route returns an operation by id
   (`…/Program.cs:32-43`), and a streaming service offers a unary read and a server stream
   (`…/Operations.proto:5-8`, `…/Infrastructure/GrpcServiceHost.cs:15-41`).
8. ✅ **Neither the pull endpoint nor the stream checks who is asking.** The read route is exposed
   through the gateway with authentication switched off
   (`hianshul100_Pacco.APIGateway/ntrada-async.yml:321-334`) and performs no ownership check; the stream
   applies no identity filter at all.

### 1.1 What this record does *not* cover

This record fixes how a caller learns the outcome of an accepted write. It does not cover how the write
was accepted (`ADR-005`), how the observer knows which messages exist (`ADR-013`), or the platform's
authorization position (`ADR-006`) — although it reproduces that position's gap and says so. It does not
decide whether operation history should be retained; that is stated as a target, not a current
behaviour.

## 2. Decision

**Pacco reports the outcome of asynchronous writes through a dedicated observer service that keys every
operation by the caller's correlation id, holds that operation only in the shared cache under a short
sliding expiry, and pushes each state change to the caller over a real-time channel with a shared
backplane. The cache is the operation's only home: this is deliberately a progress signal, not an audit
trail.**

Six rules follow from the decision and are part of it:

1. ✅ **The correlation id is the operation's identity.** It is minted at the edge when the write is
   accepted and is the only handle the caller holds, so it is the only key the observer can use.
2. ✅ **Operation state is short-lived by design.** A caller learns an outcome within seconds of issuing
   a write; nothing on the platform reads an operation later.
3. ✅ **Terminal states are final.** A completed or rejected operation is never moved back to pending,
   because a later message on the same correlation id must not undo a reported result.
4. ✅ **Push is the primary channel; pull is a fallback.** The real-time channel is what the caller is
   expected to use, and the read route exists for callers that cannot hold a connection.
5. 🎯 **Every read of an operation must be restricted to the caller who created it.** The push channel
   already addresses one user's group; the read route and the stream must reach the same standard.
6. 🎯 **A message that arrives with no correlation id must be recorded, not dropped.** Silently
   returning means a real progress signal disappears with no trace anywhere.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Store operations in the observer's document database** — persist each operation and expire it on a schedule. | Rejected for the current shape because the operation has no readers after the caller sees it, and durable storage would add a write per observed message across eighty message types for data that is discarded minutes later. It remains the right answer if operation history is ever needed, and it is closer to hand than the code suggests: the observer already configures a document database it does not use (see §6.1). |
| 2 | **Let the caller poll the owning service** — no observer, each service answers for its own results. | Rejected because the caller does not know which service will complete the write. The gateway accepts the request and a chain of services may act on it, so there is no single service to poll; and polling would reintroduce the latency that `ADR-005`'s immediate acknowledgement was taken to avoid. |
| 3 | **Poll the observer instead of pushing** — keep the read route as the only channel. | Rejected because progress has several intermediate states and a poller either misses them or polls hard enough to cost more than the connection it replaced. The push channel also lets the observer address exactly one user, which polling cannot do without the ownership check that the read route currently lacks. |
| 4 | **Keep the push channel without a shared backplane** — one instance holds all connections. | Rejected because it pins every caller to one instance and makes the observer the platform's only non-scalable service. The backplane is configured, and rule 4's fallback route means a caller can still get a result if the connection is lost. |

## 4. Consequences

### 4.1 Positive

1. ✅ The asynchronous write path is complete end to end: accept, correlate, observe, report — with one
   service owning the last two steps.
2. ✅ Reporting is uniform across choreographed and orchestrated processes, because both are observed
   through the same subscription and the same state header.
3. ✅ Caller-facing progress costs no storage and no schema. Nothing has to be migrated when a message is
   added, renamed or removed.
4. ✅ The push channel is addressed per user, so one caller's progress is not broadcast to others on
   that channel.
5. ✅ The terminal-state guard means a late or duplicated message cannot walk a reported result
   backwards, which matters because delivery is at-least-once (`ADR-012` §2).

### 4.2 Negative

1. ✅ **Operation state is lost on cache eviction or loss.** With a sliding expiry of three hundred
   seconds and no other store, a caller who reconnects late has no way to learn what happened.
2. ✅ **Any caller can read any operation.** The read route performs no ownership check and is exposed
   through the gateway with authentication off, so knowing a correlation id is enough to read someone
   else's operation, including its message name and state.
3. ✅ **The stream has no identity filter and no natural end.** It hands out every operation the instance
   observes, it never terminates, and concurrent subscribers on one instance divide the stream between
   them rather than each receiving all of it — so a second subscriber silently degrades the first.
4. ✅ **Messages without a correlation id vanish.** Both generic handlers return silently in that case,
   so a progress signal is lost with nothing logged.
5. ✅ **A misconfigured backplane value degrades the service quietly.** Anything other than the expected
   value starts the hub with no shared backplane, which behaves correctly on one instance and drops
   notifications as soon as there are two.
6. ✅ **A blank token on the hub does not stop the connection.** The connection is disconnected but
   execution continues into the token-parsing path, and the failure is swallowed by the surrounding
   error handling — so a caller can reach the hub in an unexpected state.

### 4.3 Neutral / follow-on

1. ✅ Three ways to read the same data — push, pull and stream — exist for one consumer shape, and only
   the push channel is identity-aware. The catalog's position, which this record adopts, is that the
   push half is worth keeping and the stream half is not.
2. ✅ The single browser-facing asset in the workspace is a developer test page in the observer's static
   files that points at a fixed local address. It is a development aid, not a product surface.
3. ✅ A sample console client for the stream disables server certificate validation. It is a sample and
   not deployed, and it should not be used as a model for any client that is.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates `orchestration/acknowledge-then-notify-completion.md`.** This record is the decision
   that pattern describes: acknowledge at the edge, correlate, observe every message, report the
   outcome. The pattern is catalogued with confidence *Strong*, which is the highest in the catalog, and
   its recommendation is to adopt.
2. **Adopts the pattern's three corrections as decision rules.** The catalog asks for an ownership check
   on the pull endpoint, for missing correlation ids to be logged rather than dropped, and for the
   expiry to be a deliberate choice. Rules 5 and 6 carry the first two, and question Q2 carries the third.
3. **Adopts the pattern's framing that this is not an audit trail.** The decision statement says so
   directly, so that a future requirement for operation history is recognised as a change to this record
   rather than a tuning of the expiry.
4. **Instantiates `orchestration/real-time-push-with-shared-backplane.md`, partially.** The catalog's
   recommendation is to adopt the push half and not to inherit the streaming half, listing four defects
   in the latter. Consequence 4.2.3 and 4.3.1 record that position; this record adopts the push channel
   and marks the stream as not to be built on.
5. **Instantiates `data/prefix-partitioned-shared-cache.md`.** Operations are stored under a fixed key
   prefix in the shared cache, which is the pattern's shape. This is also the only place in the workspace
   where that cache is actually read and written, as the catalog notes.
6. **Depends on `integration/declarative-message-manifest-subscription.md`.** Every operation reported
   here originates in a subscription created by `ADR-013`, so a message missing from the manifest is an
   operation that is never reported — the two records share that failure mode.
7. **Pattern Drift: not applicable.** Every entry in `docs/architecture-inventory/patterns/index.md`
   currently carries status `Candidate`. Drift is reportable only against an `Approved` pattern, so no
   drift is recorded for this ADR.
8. **Pattern Update Proposal.** `orchestration/acknowledge-then-notify-completion.md` should record the
   hub's blank-token behaviour (consequence 4.2.6) and the silent fallback when the backplane value is
   unrecognised (consequence 4.2.5), neither of which the pattern file currently lists among its
   weaknesses. The **Related ADRs** entries for both orchestration patterns and for the shared-cache
   pattern should change from `None` to `ADR-014`.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| 1 | Observed events are turned into operations keyed by correlation id, and are dropped when it is missing | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Handlers/GenericEventHandler.cs:28-61` |
| 2 | Rejected events follow the same path and default to a rejected state | ✅ | `…/Handlers/GenericRejectedEventHandler.cs:28-62` |
| 3 | State is read from the saga header, defaulting by message kind | ✅ | `…/Handlers/Extensions.cs:9-26` |
| 4 | Operations are read from the shared cache | ✅ | `…/Services/OperationsService.cs:24-29` |
| 5 | A terminal state is never overwritten | ✅ | `…/Services/OperationsService.cs:39-42` |
| 6 | Entries are written with a sliding expiry from settings, and a state-change notification is raised | ✅ | `…/Services/OperationsService.cs:50-57` |
| 7 | The cache key carries a fixed prefix | ✅ | `…/Services/OperationsService.cs:62` |
| 8 | The operations service is registered once for the process | ✅ | `…/Infrastructure/Extensions.cs:58` |
| 9 | The configured expiry is three hundred seconds | ✅ | `…/Pacco.Services.Operations.Api/appsettings.json:149-151` |
| 10 | The shared backplane is wired only when the configured value names it | ✅ | `…/Infrastructure/Extensions.cs:95-109`; `…/appsettings.json:152-154` |
| 11 | Three client messages are emitted, one per state | ✅ | `…/Services/HubService.cs:15-45` |
| 12 | Notifications are addressed to a per-user group | ✅ | `…/Services/HubWrapper.cs:17-21`; `…/Infrastructure/Extensions.cs:32-33` |
| 13 | A blank token disconnects but does not stop execution, and the parse failure is swallowed | ✅ | `…/Hubs/PaccoHub.cs:18-42` |
| 14 | The read route returns an operation with no ownership check | ✅ | `…/Pacco.Services.Operations.Api/Program.cs:32-43` |
| 15 | The hub and the streaming service are mapped alongside it | ✅ | `…/Pacco.Services.Operations.Api/Program.cs:46-47` |
| 16 | The read route is published through the gateway with authentication off | ✅ | `hianshul100_Pacco.APIGateway/ntrada-async.yml:321-334`; `hianshul100_Pacco.APIGateway/ntrada.yml:277-290` |
| 17 | The stream is fed by a per-instance queue, loops without cancellation, and applies no identity filter | ✅ | `…/Infrastructure/GrpcServiceHost.cs:15-41` |
| 18 | The service contract offers a unary read and a server stream | ✅ | `…/Pacco.Services.Operations.Api/Operations.proto:5-8` |
| 19 | The sample stream client disables server certificate validation | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.GrpcClient/Program.cs:35-39` |

### 6.1 Documentation-versus-code conflicts

1. **A document database is configured and never used.** The observer's settings declare a database named
   for the service (`…/Pacco.Services.Operations.Api/appsettings.json:98-102`) and the composition root
   registers the database client (`…/Infrastructure/Extensions.cs:72`), but the repository contains no
   repository class and no document type, and no code path writes an operation anywhere but the cache
   (evidence 4–7). The configuration says operations are persisted; the code says they are not. The code
   is authoritative, and this record states the cache as the only home. The configuration is recorded as
   **Future/Intended State (Not Implemented)** and is also the reason alternative 1 is closer to hand
   than it appears.
2. **The gateway publishes an unauthenticated read of another caller's operation.** The gateway
   configuration marks the route as open (evidence 16) while the platform's stated position is that
   reads are authenticated at the edge (`ADR-006`). The code confirms the route is open and unchecked.
   Recorded as a conflict rather than reconciled: rule 5 states the target, and the current behaviour is
   consequence 4.2.2.
3. **No stated intent for the expiry value.** Three hundred seconds appears once, in settings, with no
   comment, note or document anywhere explaining it. Whether it is a considered bound on how long a
   caller may take to reconnect, or a default that was never revisited, is **Unverifiable — Missing
   Source Evidence**, and is carried as question Q2.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section records what this document assumes, what is blocking it, and what still needs an answer.
> Items here are not decided. Treat every entry as open until an owner closes it.

### Assumptions

| ID | Assumption | Why we made it | Impact if wrong |
| --- | --- | --- | --- |
| A1 | Nothing on the platform needs an operation after the caller has seen it. | No component in any of the fourteen clones reads an operation other than the caller-facing routes and the push channel. | If reporting, billing or support needs operation history, the cache-only decision is wrong and alternative 1 becomes the correct answer. |
| A2 | Callers of the asynchronous write path can hold a real-time connection. | The push channel is the primary channel by rule 4, and the only client asset in the workspace connects to it. | If important callers cannot, the read route becomes primary — and it is the surface with no ownership check, which would make consequence 4.2.2 far more serious. |
| A3 | The three hundred second expiry is long enough for a normal write to complete and be reported. | The write path is a handful of message hops within one broker, so completion is expected in seconds. | If a process can legitimately run longer — order creation waits on a reservation, see `ADR-011` — its operation expires before it finishes and the caller is told nothing. |

### Blockers

| ID | Blocker | Who is affected | Owner |
| --- | --- | --- | --- |
| B1 | No repository in the workspace names an owner, a team or a review group, so this record cannot list deciders. | Every ADR in the set. | **[handled later by the architecture PR review stage]** — the reviewer assigns deciders when the batch is reviewed as a whole. |
| B2 | Anyone holding a correlation id can read the matching operation through an unauthenticated gateway route with no ownership check. | Every caller of the asynchronous write path. | **[ACTION NOW]** — recorded as consequence 4.2.2 with its evidence and stated as a target in decision rule 5, so the gap is visible in this batch rather than carried silently. |
| B3 | It is unresolved which gateway configuration production uses, so it cannot be stated whether the asynchronous write path this record completes is the live one. | Anyone reasoning about how writes behave in production. | **[handled later by the batch 4 authoring stage, in the record covering the deployment path]** — the two container stacks disagree, as recorded in `ADR-017`. |

### Open Questions

| ID | Question | Why it matters | Owner |
| --- | --- | --- | --- |
| Q1 | Should the read route and the stream restrict results to the caller who created the operation? | Rule 5 says they should. Doing it needs a caller identity on both surfaces, which the read route does not currently receive because the gateway route is open. | **[ACTION NOW]** — stated as decision rule 5 with the evidence for the current behaviour, for the deciders assigned under B1 to schedule. |
| Q2 | Is three hundred seconds a deliberate bound, and does it hold for long-running processes? | If a process can outlive its operation entry, the caller is silently left without a result — which is assumption A3's failure mode. | **[handled later by the architecture PR review stage]** |
| Q3 | Should the streaming surface be removed rather than repaired? | It duplicates the push channel, has no identity filter, never terminates, and splits its output between concurrent subscribers. Removing it is less work than fixing it, but something may depend on it. | **[ACTION NOW]** — both options are stated here with the defects that motivate the question, for the deciders assigned under B1 to choose between. |
| Q4 | Should the edge write mode be selectable per route rather than per gateway configuration? | This is the question `ADR-005` handed to this record. A per-route choice would let reads and simple writes answer directly while long processes use the observer, but it splits one clear rule into a per-route decision that has to be maintained across four configuration files. | **[handled later by the batch 4 authoring stage, in the record covering the gateway configuration set]** |

