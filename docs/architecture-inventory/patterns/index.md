# Implementation Pattern Catalog

Reusable implementation approaches observed in the Pacco source repositories, written so they can be
applied to new work rather than only describing what exists.

**This catalog is not a set of ADRs.** An ADR records one project's decision, with its context and its
consequences. A pattern here records a *reusable approach* — the problem it solves, when to reach for it,
when not to, and what it costs. Where a decision record would be the right place for something, this
catalog references it rather than restating it. No ADR, decision record, or RFC exists anywhere in the
cloned repositories, so every pattern's **Related ADRs** entry is empty; that absence is itself recorded
below.

Every pattern was derived from source code. Where documentation and code disagreed, the code was taken
as authoritative and the disagreement was written into the affected pattern's **Evidence → Notes**
rather than resolved silently.

## How to read this catalog

- **Status** is `Candidate`, `Approved`, or `Deprecated`. **Every pattern here is `Candidate`** — see
  *Governance* below for why, and what that means for you.
- **Evidence strength** describes how firmly the source supports the pattern, not how good the pattern
  is:
  - **Strong** — applied consistently across several services, with the mechanism visible in code and
    configuration, and no material contradiction.
  - **Moderate** — clearly implemented, but in one or two places only, or with a significant part of it
    configured rather than exercised.
  - **Weak** — the shape is present and the intent is legible, but something essential is missing,
    unused, or unverifiable. Read the pattern's open questions before relying on it.
- Cross-references between patterns use `[[pattern-file-name]]`, matching the file name in this
  directory tree.
- Source citations name repository-relative paths and line numbers. Committed credential material is
  cited **by path only** — no secret value is reproduced anywhere in this catalog.

## Catalog

| Pattern name | Category | Status | Short description | Related ADRs | Related services/repos | Evidence strength | Last updated |
|---|---|---|---|---|---|---|---|
| [Service-Owned Topic Exchange Messaging](integration/service-owned-topic-exchange-messaging.md) | integration | Candidate | Each service owns one topic exchange named after it and publishes only its own messages; consumers bind queues named for the consuming service | None | All 10 messaging deployables | Strong | 2026-09-04 |
| [Declarative Configuration-Driven API Gateway](integration/declarative-configuration-driven-api-gateway.md) | integration | Candidate | The entire north-south edge — routes, auth, transforms, forwarding — is a YAML document rather than gateway code | None | `Pacco.APIGateway` | Strong | 2026-09-04 |
| [Dual-Mode Edge Write](integration/dual-mode-edge-write.md) | integration | Candidate | The same edge route can either proxy a write synchronously or publish it as a command, chosen by configuration rather than by a code path | None | `Pacco.APIGateway`, all write-side services | Strong | 2026-09-04 |
| [Narrow Synchronous Point-Read Between Services](integration/narrow-synchronous-point-read.md) | integration | Candidate | Service-to-service HTTP is confined to single-resource reads that gate a write, with everything else asynchronous | None | Orders, Availability, Parcels, Pricing, Vehicles, Customers | Strong | 2026-09-04 |
| [Rejected-Event Failure Contract](integration/rejected-event-failure-contract.md) | integration | Candidate | Every command failure becomes a typed rejection message with a stable reason code, so failure is a first-class part of the message contract | None | All 7 layered services | Strong | 2026-09-04 |
| [Declarative Message Manifest with Runtime Type Emission](integration/declarative-message-manifest-subscription.md) | integration | Candidate | A service subscribes to messages it has no compile-time types for by declaring them in configuration and emitting the types at startup | None | `Pacco.Services.Operations` | Moderate | 2026-09-04 |
| [Saga Process Manager for Multi-Service Workflows](orchestration/saga-process-manager.md) | orchestration | Candidate | A long-running multi-service workflow is coordinated by an explicit state machine reacting to events, with compensation on failure | None | `Pacco.Services.OrderMaker` | Moderate | 2026-09-04 |
| [Acknowledge Then Notify Completion](orchestration/acknowledge-then-notify-completion.md) | orchestration | Candidate | Asynchronous writes return an immediate acknowledgement carrying a correlation id, and completion is reported later over a separate channel | None | `Pacco.Services.Operations`, `Pacco.APIGateway`, all write-side services | Strong | 2026-09-04 |
| [Real-Time Push With a Shared Backplane](orchestration/real-time-push-with-shared-backplane.md) | orchestration | Candidate | Server-to-client push over a hub, with a shared backplane so any instance can reach any connected client | None | `Pacco.Services.Operations` | Moderate | 2026-09-04 |
| [Database Per Service With Document Mapping](data/database-per-service-with-document-mapping.md) | data | Candidate | Each service owns its own database and maps aggregates to documents by hand, with no shared schema and no ORM | None | All 8 persisting services | Strong | 2026-09-04 |
| [Event-Carried Reference Replica](data/event-carried-reference-replica.md) | data | Candidate | A service keeps a minimal local replica of another service's data, updated by events, so it can decide without a synchronous call | None | Orders, Availability, Deliveries | Moderate | 2026-09-04 |
| [Transactional Outbox and Inbox Behind a Handler Decorator](data/transactional-outbox-handler-decorator.md) | data | Candidate | Message deduplication and reliable publication are applied by decorating every handler, so no handler contains reliability code | None | All 7 layered services | Strong | 2026-09-04 |
| [Prefix-Partitioned Shared Cache](data/prefix-partitioned-shared-cache.md) | data | Candidate | Services share one cache server and stay isolated by a per-service key prefix rather than by separate instances | None | 9 services provision it; 1 uses it | Weak | 2026-09-04 |
| [Edge-Enforced Authentication With Identity Binding](security/edge-enforced-authentication-with-identity-binding.md) | security | Candidate | The gateway authenticates, gates routes on claims, and binds the caller's identity into the forwarded payload so services need not trust the body | None | `Pacco.APIGateway`, `Pacco.Services.Identity`, all protected services | Strong | 2026-09-04 |
| [Vault-Issued Dynamic Credentials And Service PKI](security/vault-issued-dynamic-credentials-and-service-pki.md) | security | Candidate | Services load settings from a secret store at startup and take short-lived database credentials from it instead of holding static ones | None | 9 of 11 deployables | Moderate | 2026-09-04 |
| [Transport-Agnostic Caller Context](security/transport-agnostic-caller-context.md) | security | Candidate | One caller-context abstraction is populated from a message envelope or an HTTP request, so handlers never know which transport they were reached by | None | All 7 layered services | Strong | 2026-09-04 |
| [Correlation And Span Propagation](observability/correlation-and-span-propagation.md) | observability | Candidate | A correlation envelope and a span context ride every message and HTTP hop, so one user action is one trace across services | None | All 10 traced deployables | Strong | 2026-09-04 |
| [Structured Logging With Property Redaction](observability/structured-logging-with-property-redaction.md) | observability | Candidate | Handler logging is declared as per-message templates, and a named property list is redacted from every sink | None | 6 services declare templates; 10 redact | Strong | 2026-09-04 |
| [Registry-Mediated Discovery And Routing](deployment/registry-mediated-discovery-and-routing.md) | deployment | Candidate | Services register with a registry at startup and address each other by logical name through a registry-driven router | None | All 10 registering deployables | Strong | 2026-09-04 |
| [Composable Per-Concern Environment Stacks](deployment/composable-per-concern-environment-stacks.md) | deployment | Candidate | The local environment is assembled from small per-concern compose files rather than one large one, so a developer runs only what they need | None | `Pacco` (platform repo) | Strong | 2026-09-04 |
| [Independent Per-Repository Release](deployment/independent-per-repository-release.md) | deployment | Candidate | Every deployable has its own repository and its own identical three-script pipeline, and releases without reference to the others | None | All 11 deployable repositories | Strong | 2026-09-04 |
| [Inward-Dependency Service Skeleton](domain/inward-dependency-service-skeleton.md) | domain | Candidate | A four-project layout in which every dependency points inward and the innermost project references nothing at all | None | All 7 layered services | Strong | 2026-09-04 |
| [Dispatcher-Bound CQRS Endpoints](domain/dispatcher-bound-cqrs-endpoints.md) | domain | Candidate | HTTP routes are declared as a table mapping paths directly to commands and queries, with no controllers | None | All 10 HTTP deployables | Strong | 2026-09-04 |
| [Aggregate-Buffered Domain Events](domain/aggregate-buffered-domain-events.md) | domain | Candidate | Aggregates record what happened into an internal buffer, and the events are published only after the change is persisted | None | All 7 layered services | Strong | 2026-09-04 |
| [Layered Service Test Suite](testing/layered-service-test-suite.md) | testing | Candidate | Unit, integration, end-to-end and performance levels over one service, with shared fixtures, testing both entry paths against the same assertion | None | `Pacco.Services.Availability` | Moderate | 2026-09-04 |
| [Consumer-Driven Contract Test Pair](testing/consumer-driven-contract-test-pair.md) | testing | Candidate | The consumer declares what it expects of a provider's response, and the provider verifies that expectation in its own build | None | `Pacco.Services.Orders`, `Pacco.Services.Parcels` | Weak | 2026-09-04 |
| [Framework-Supplied Platform Conventions](other/framework-supplied-platform-conventions.md) | other | Candidate | Cross-cutting capabilities come from one external toolkit composed per service, with no shared internal library | None | All 11 deployable repositories | Strong | 2026-09-04 |

## Categories

| Category | Patterns | Notes |
|---|---|---|
| ingestion | 0 | No ingestion pipeline, batch loader, or file-intake path exists in any repository. |
| integration | 6 | The platform's richest area; messaging and the edge dominate it. |
| orchestration | 3 | One saga, one status-tracking mechanism, one push channel. |
| data | 4 | Ownership, replication, reliability, and caching. |
| security | 3 | Edge enforcement, secret handling, and the caller-context abstraction. |
| observability | 2 | Tracing and logging. No pattern for metrics: they are enabled by a single call with no local conventions on top. |
| deployment | 3 | Discovery, local environments, and release. |
| domain | 3 | Service layering, the HTTP surface, and aggregate event handling. |
| ui | 0 | **No UI pattern is recorded.** The only browser-facing asset is a static developer test page in one service, and the web repository tracks nothing but a one-line README. There is no reusable UI approach to extract, and inventing one would not be grounded in source. See *Q3* below. |
| testing | 2 | A layered suite in one repository and a contract pair between two. |
| other | 1 | The composition convention that underlies all of the above. |

## Governance

### Status: every pattern is a Candidate

`Approved` requires explicit approval metadata or documented human approval in the repository. None
exists: there is no ADR directory, no decision record, no RFC, no `CODEOWNERS` file, and no sign-off
recorded in any repository. The one tracked work item found in the workspace attachments is still open.
So every pattern in this catalog is `Candidate`, and this catalog is a proposal, not a standard.

### Pattern drift

**Not applicable at present.** Drift means an implementation violating an *approved* pattern. With no
approved pattern, nothing can drift from one, and no drift is flagged anywhere in this catalog.

What individual patterns *do* record is **inconsistency** — a convention followed by nine services and
not the tenth, a capability registered everywhere and used once, a pipeline step present in ten
repositories and missing from one. Those are recorded as open questions against the specific pattern,
not as governance findings, precisely because there is no approved baseline to call them violations of.
Once any pattern here is approved, those same inconsistencies become drift, and that is the point at
which they need owners.

### Pattern update proposals

**Every entry in this catalog is a pattern update proposal.** The catalog is new; nothing in it has been
reviewed. Each pattern file is a self-contained proposal for a reusable approach observed in the code,
and each carries its own **Recommendation** section stating whether it should be adopted as written,
adopted with changes, or reconsidered.

Review is by pull request against this repository. A reviewer approving a pattern should change its
`## Status` from `Candidate` to `Approved` in the pattern file and in the table above, and record who
approved it and when — after which drift against that pattern becomes meaningful and enforceable.

Several patterns recommend adoption *with a named change first*. Those changes are listed as blockers in
the pattern's final section and should be settled before the pattern is approved, so that approval does
not bless a known defect. **Fourteen of the twenty-seven patterns carry at least one blocker.** The three
security patterns are the ones to read first — [[edge-enforced-authentication-with-identity-binding]],
[[vault-issued-dynamic-credentials-and-service-pki]] and [[transport-agnostic-caller-context]] — because
their blockers describe credential material and authorization behaviour that is a live exposure rather
than a decision waiting to be made.

### Source of truth

Source code was treated as authoritative throughout. Three documentation-versus-code conflicts were
found and are recorded in the affected patterns' **Evidence → Notes** rather than reconciled:

- A cache is provisioned across nine services while only one reads or writes it
  ([[prefix-partitioned-shared-cache]]).
- A certificate authority is configured while services load certificates from disk instead
  ([[vault-issued-dynamic-credentials-and-service-pki]]).
- All ten services register with the discovery registry while the gateway's load balancing is disabled
  in all four of its configurations, so the registry mediates east-west traffic only
  ([[registry-mediated-discovery-and-routing]]).

Where configuration describes intent that the code does not implement, the pattern says so in those
words. Where a capability could not be verified from source, the pattern says **not observed** rather
than assuming.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The eleven cloned deployable repositories plus the platform repository are the whole system | The gateway's route table, the compose files, and the process manifests all resolve to services within this set, and no reference points outside it | Patterns derived from what is visible would be incomplete, and a service outside the set could follow conventions this catalog contradicts | Ask the platform owner whether any deployable exists outside these repositories |
| A2 | The default branch of each cloned repository reflects the current deployed state | Each pipeline builds and pushes an image from its default branch, and no other release path exists in any repository | Patterns would describe a state nobody is running, and the recommendations would target the wrong code | Compare the running image tags in each environment against the pipeline's build numbers |
| A3 | Absence of a pattern in this catalog means absence in the source, not an oversight in the review | Every repository was enumerated and every project and configuration file was walked; the two empty categories were confirmed by search, not inferred | A real reusable approach would go unrecorded, and a team applying this catalog would rebuild it from scratch | Have each service owner read the catalog and name anything reusable they use that is not here |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Who approves a pattern, and what does approval commit a team to? | Everything here is a Candidate because no approval mechanism exists. Until one does, the catalog is a description and no pattern can be drifted from, so none of the inconsistencies it records has an owner | Name one or two reviewers per category, and state plainly that approval means new work follows the pattern while existing code is not required to change | Platform owner |
| Q2 | **[ACTION NOW]** Should the security patterns' blockers be resolved before those patterns are approved? | Three security patterns describe approaches that are sound in shape but currently implemented with known defects — committed signing material, a shared root credential, and authorization guards that pass when the caller is unauthenticated. Approving them as they stand would make the defects the standard | Yes. Approve the shape and record the fix as a condition, so the approved pattern describes the corrected behaviour rather than the current one | Platform security owner |
| Q3 | **[ACTION NOW]** Is a user interface planned, and should a UI pattern exist? | The `ui` category is empty because no application frontend exists — the web repository tracks only a README. If a frontend is planned, its conventions should be decided rather than discovered later from whatever gets built first | If a frontend is planned, add a UI pattern before the first one is written, and derive its edge assumptions from [[edge-enforced-authentication-with-identity-binding]] and [[acknowledge-then-notify-completion]], since both currently assume a machine caller | Platform owner |
| Q4 | **[handled later by the design stage]** Should this catalog be kept current, and by what trigger? | A pattern catalog that is not maintained becomes misleading faster than no catalog, because it is read as current. Nothing today updates it when a service changes | Require a catalog check in the pull-request template for changes touching messaging, the edge, or authorization — the three areas where a change most often invalidates a pattern | Platform owner |
| Q5 | **[handled later by the design stage]** Should the inconsistencies recorded across patterns be triaged as one list? | Each is currently an open question inside its own pattern, so no single view exists of how many services deviate from how many conventions, and the cheap fixes are not distinguishable from the expensive ones | Yes — extract them into one list once Q1 gives them owners, and sort by whether the fix is a configuration line or a code change | Platform owner |
| Q6 | **[handled later by the design stage]** Should evidence strength be re-assessed after the weakly evidenced patterns are acted on? | Two patterns are marked Weak because something essential is missing rather than because the approach is wrong. Once the cache has a second consumer and the contract pact is actually exchanged, both would be Moderate or better | Yes, and re-check the strength column whenever a pattern's blockers are closed | Owners named in the affected patterns |
