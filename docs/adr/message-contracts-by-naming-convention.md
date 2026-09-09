# ADR-003: Message contracts by naming convention, with duplicated consumer DTOs

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-003` |
| **Backlog candidate** | `ADR-CANDIDATE-003` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Integration / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-002` (why there is no shared package to put contracts in), `ADR-001` (the topology these contracts travel over), `ADR-012` (the reliability layer that assumes the contract already matched) |

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

`ADR-001` fixed the message *topology* — who owns an exchange and how queues are bound. It left open what
travels on it. `ADR-002` established that there is no first-party shared library on this platform, so
there is nowhere obvious to put a shared contract type. This record states what the platform does
instead.

1. ✅ **A publisher marks its message type with a contract attribute and keeps it inside its own
   application project.** `customers-service` declares the customer-created event this way
   (`hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Application/Events/CustomerCreated.cs:6-15`),
   and `availability-service` declares its resource-reserved event the same way
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Application/Events/ResourceReserved.cs:6-14`).
2. ✅ **A consumer redeclares that message as its own class**, annotated with the *source exchange name*
   rather than with any reference to the publishing project. The customer-created event exists as four
   independent classes — in Customers, Availability, Orders and Parcels — and the resource-reserved
   event as three, in Availability, Orders and OrderMaker.
3. ✅ **The only binding link between the two is the message name**, derived from the type name under
   the snake-case convention each service configures for its broker client
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:117`). A publisher
   and a consumer that agree on the class name agree on the contract; nothing else is checked.
4. ✅ **The duplicated copies currently agree.** All four customer-created classes carry the same single
   identifier property, and all three resource-reserved classes carry the same two properties. The
   agreement is a fact about today's code, not a property the build enforces — the copies differ in
   incidental ways (constructor body style, namespace, attribute) precisely because nothing compares
   them.
5. ✅ **The platform-wide list of what exists is a hand-maintained file inside one service**
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:1-151`),
   enumerating eight exchanges and their commands, events and rejected events. No step generates it and
   no step validates it against the publishing services.
6. ✅ **One verification mechanism exists, and it covers an HTTP contract rather than a message one.**
   A consumer-driven pact is declared between `orders-service` and `parcels-service` for a single
   synchronous parcel read
   (`hianshul100_Pacco.Services.Orders/tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs:23-38`).

### 1.1 What this record does *not* cover

This record fixes how a *payload contract* is expressed and verified. It does not fix the topology
(`ADR-001`), the delivery guarantee around a handler (`ADR-012`), or the runtime-emitted subscription
used by the one service that consumes everything (`ADR-CANDIDATE-013`). It also does not decide the
narrow synchronous read surface (`ADR-CANDIDATE-010`), although the single pact pair sits on it.

## 2. Decision

**Pacco expresses cross-service message contracts as a shared *naming convention* rather than as shared
code: the publisher owns the type, every consumer redeclares an independent copy of it inside its own
application project, and the snake-case message name derived from the class name is the whole contract.
No shared contract package, schema registry, or generated client exists or will be introduced without a
superseding record.**

Three rules follow from the decision and are part of it:

1. ✅ **A consumer never references a publisher's assembly.** The consumer's copy is annotated with the
   source exchange name and nothing more, which is what keeps the eleven repositories independently
   buildable.
2. 🎯 **A change to a published message's shape must be additive.** Renaming a message type or removing
   a property is a breaking change with no build-time or CI signal today, so it must be treated as a
   two-phase change: publish the new name alongside the old, migrate consumers, then retire the old.
   Nothing enforces this today.
3. 🎯 **Every cross-service contract that matters must have a verifying test.** One pact pair exists and
   it is not wired end to end (§4.2). Until a contract has a test that runs in both repositories'
   pipelines, the contract is a convention, not a guarantee.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **A shared `Pacco.Contracts` package** referenced by every publisher and consumer. | Rejected because it reintroduces exactly the coupling `ADR-002` removed. A shared package needs a coordinated release, and the platform has no mechanism to perform one — eleven repositories, eleven independent pipelines, no release train (`ADR-CANDIDATE-018`). A contract change would become a version bump every consumer must take before the publisher can deploy, which inverts the independence the repository split was chosen for. The cost accepted instead is that nothing catches a mismatch. |
| 2 | **A schema registry with a compatibility policy** — publish a schema per message, reject incompatible changes at publish time. | Rejected because it adds a runtime component that must be available for a service to start or publish, and the platform already has four such components (broker, registry, secret store, cache) each running as a single instance with no replication. It is the technically strongest option and is the one to revisit first if the platform gains an operations team; it was not taken because nothing in the workspace shows a registry being run or considered. |
| 3 | **Generated clients from a published schema** — keep `messages.json` as the source and generate consumer types from it at build time. | Rejected implicitly: the manifest exists and generation does not. Generating from it would make the manifest authoritative, and the manifest is a hand-maintained copy inside `operations-service` of names owned by eight other repositories. Generating from a file with no owner would convert a documentation risk into a build risk. This is the cheapest improvement available and is proposed as a pattern update in §5. |
| 4 | **Consumer-driven contract tests for every cross-service message,** extending the one pact pair that exists. | Not rejected on merit — it is the direction rule 3 in §2 points at. It is not the current state because the single existing pact covers an HTTP read rather than a message, and because it is not actually exchanged between the two repositories (§4.2). Recording it as an alternative rather than as the decision keeps this ADR honest about what runs today. |

## 4. Consequences

### 4.1 Positive

1. ✅ A publisher and a consumer never block each other at build time. Each of the eleven repositories
   compiles, tests and releases with no reference to any other.
2. ✅ A consumer takes only the fields it needs. The consumer's copy is its own projection of the
   message, so a publisher adding a property does not force a consumer change.
3. ✅ The contract surface is readable per service: everything a service consumes is under its own
   `Events/External/` folder, and everything it publishes is under `Events/`.
4. ✅ Adding a consumer requires no change anywhere except in the new consumer.

### 4.2 Negative

1. ✅ **A rename or a field change breaks consumers silently at runtime.** There is no compile-time
   check, no schema check and no CI check anywhere in the workspace. This is recorded as constraint C2
   in `docs/architecture-inventory/baselines/architecture-baseline.md` §11.2, whose "what it prevents"
   column reads *nothing*.
2. ✅ **The one verification mechanism is not connected.** The consumer test writes the pact file to a
   path six directory levels above the test binary, and the provider test reads it from the same
   relative path in a *different* repository
   (`hianshul100_Pacco.Services.Parcels/tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs:21-25`).
   That path only resolves to the same directory if both clones sit side by side in one parent folder,
   which no pipeline arranges: each repository's Travis file runs build, test and dockerize in isolation
   and neither publishes nor fetches a pact. `[INFERRED]` the provider verification therefore either
   finds no pact file in CI or verifies a stale local one.
3. ✅ **The pact ignores values.** The consumer test sets its comparison options to ignore contract
   values, so it asserts response *shape* only. A provider that returns structurally valid but
   semantically wrong data passes.
4. ✅ **The manifest can drift without anyone noticing.** It is a copy of names owned by eight other
   repositories with no generation and no validation step, and it is the input to the one service that
   subscribes to everything (`ADR-CANDIDATE-013`).
5. `[INFERRED]` **Duplication scales with consumers.** The customer-created event already exists four
   times. Every new consumer of an existing message adds another copy that must be kept in agreement by
   hand.

### 4.3 Neutral / follow-on

1. ✅ The copies agreeing today is evidence the convention is being followed carefully, not evidence
   that it is safe. The next contract change is the test of the convention, and no such change is
   visible in the workspace history examined here.
2. ✅ `pricing-service` consumes no messages at all, so it is unaffected by this decision and is
   coupled to `customers-service` only through the synchronous read surface
   (`ADR-CANDIDATE-010`).

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/integration/service-owned-topic-exchange-messaging.md` (Status:
   `Candidate`, evidence Strong) at the payload level: that pattern names the message convention as the
   binding contract, and this record adopts it as the platform rule together with the two obligations in
   §2.
2. **Constrains** `patterns/testing/consumer-driven-contract-test-pair.md` (Status: `Candidate`,
   evidence Weak). Rule 3 in §2 turns that pattern from an optional practice into the condition under
   which a cross-service contract may be relied on. The pattern's own Weak rating is for the same reason
   this record gives in §4.2 — the pact is not exchanged between the repositories.
3. **Constrains** `patterns/integration/declarative-message-manifest-subscription.md`: the manifest that
   pattern depends on has no validation step, which is a direct consequence of this decision rather than
   a defect of that pattern.
4. **Deliberately diverges** from nothing in the catalog. No catalogued pattern recommends a shared
   contract package or a schema registry, so no divergence exists to justify.
5. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*).
6. **Pattern Update Proposal:** add a generation step to
   `patterns/testing/consumer-driven-contract-test-pair.md` and to the messaging pattern — specifically,
   derive `messages.json` from the publishing services' contract-attributed types and fail the build on
   a mismatch. That closes §4.2 item 4 with a build step rather than a new runtime component, and it is
   the one improvement here that costs no infrastructure.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | The publisher declares its event inside its own application project, marked with a contract attribute, with no shared package | ✅ | `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Application/Events/CustomerCreated.cs:6-15` |
| E2 | A consumer redeclares the same event as an independent class annotated with the source exchange name | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Events/External/CustomerCreated.cs:7-16` |
| E3 | The customer-created event exists as four independent classes across four repositories | ✅ | `…Customers.Application/Events/CustomerCreated.cs`, `…Availability.Application/Events/External/CustomerCreated.cs`, `…Orders.Application/Events/External/CustomerCreated.cs`, `…Parcels.Application/Events/External/CustomerCreated.cs` |
| E4 | The resource-reserved event exists as three independent classes, all carrying the same two properties | ✅ | `…Availability.Application/Events/ResourceReserved.cs:6-14`, `…Orders.Application/Events/External/ResourceReserved.cs:7-18`, `…OrderMaker/Events/External/ResourceReserved.cs:7-18` |
| E5 | The message name is derived from the type name under a snake-case convention set per service | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:117` |
| E6 | The platform's message catalogue is a hand-maintained file inside one service, listing eight exchanges with their commands, events and rejected events | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:1-151` |
| E7 | The one consumer-driven contract test covers an HTTP read, ignores contract values, and writes the pact to a path six levels above the test binary | ✅ | `hianshul100_Pacco.Services.Orders/tests/Pacco.Services.Orders.PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs:20-38` |
| E8 | The provider test reads the pact from the same relative path, in a different repository | ✅ | `hianshul100_Pacco.Services.Parcels/tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs:21-25` |
| E9 | Neither repository's pipeline publishes or fetches a pact; each runs the same three isolated scripts | ✅ | `hianshul100_Pacco.Services.Orders/.travis.yml` and `hianshul100_Pacco.Services.Parcels/.travis.yml` — `build.sh`, `test.sh`, `dockerize.sh` only |
| E10 | No shared contract package, schema file, or code-generation step exists in any of the thirteen repositories | ✅ | Workspace-wide search: no `*.Contracts` project, no schema definition, no generator invocation in any `.csproj`, script, or pipeline file |

### 6.1 Documentation-versus-code conflicts

None found for this decision. `docs/architecture-inventory/repo-inventory.md` §3.3 and
`docs/architecture-inventory/baselines/architecture-baseline.md` §11.2 describe the same duplication and
the same absent verification the source shows. The code adds one detail the inventory does not state:
the pact comparison is configured to ignore contract values, which further narrows what the single
existing test proves. That detail is recorded in §4.2 rather than treated as a conflict.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The toolkit derives the wire message name from the class name under the configured snake-case convention, so two identically-named classes in two repositories are the same message | Every service sets the same casing convention, and every consumer copy carries the source exchange name and the same class name as the publisher's type. The toolkit itself is a package reference with no source in this workspace, so the derivation is not directly observable | If the name were derived from the full type name including namespace, no consumer would receive anything at all, and the platform's integration would not work as described | Publish one event in a running environment and read the routing key off the broker, comparing it to the class name |
| A2 | The provider pact verification does not currently verify anything meaningful in CI | The pact path resolves six levels above the test binary — outside the repository — and neither pipeline publishes or fetches the file, so the provider job would find no pact where it looks | The contract pair might actually be protecting the orders-to-parcels read, and §4.2 item 2 would overstate the exposure | Run the provider test in a clean CI checkout and record whether it finds a pact file or passes vacuously |
| A3 | The manifest in `operations-service` is a faithful list of the platform's messages | Its eight exchanges and message names match the publisher and consumer classes found independently in the other repositories | The contract surface described here would be incomplete, and the drift risk in §4.2 item 4 could be understated | Generate the message list from the contract-attributed types in each repository and compare it to the manifest |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so nobody can approve this record or be accountable for a contract change made under it | This ADR leaving `Proposed`, and rule 2 in §2 — a two-phase contract change needs someone to own both phases across two repositories | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** How should a breaking message change actually be rolled out, given that no coordinated release mechanism exists? | Rule 2 in §2 requires a two-phase change, but the platform has eleven independent pipelines and no way to sequence them. Without an answer, the rule is advice nobody can follow | Publish the new message name alongside the old from the publisher, deploy that first, migrate each consumer at its own pace, then remove the old name in a later publisher release. Write the retirement date into the publisher's own backlog so it does not linger | Platform owner |
| Q2 | **[ACTION NOW]** Should the pact pair be completed, replaced, or removed? | Right now it reads as protection that does not exist, which is worse than no test: a reviewer seeing a pact project assumes the orders-to-parcels contract is covered | Complete it — publish the pact as a build artifact from the consumer's pipeline and fetch it in the provider's — or delete both projects. Do not leave it in the middle | Owner of `orders-service` and `parcels-service` (unassigned — see B1) |
| Q3 | **[handled later by adr_generation]** Should `messages.json` be generated from the publishing services rather than hand-maintained? | It is the input to the contract-blind subscriber and the only platform-wide view of the message surface, and nothing keeps it current | Yes — generate it in each publisher's build and fail on mismatch. Record the mechanism in `ADR-CANDIDATE-013`, which is where the manifest's consumer is decided | `adr_generation` stage, in `ADR-CANDIDATE-013` |
