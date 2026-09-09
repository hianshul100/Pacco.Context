# ADR-008: Database per service on MongoDB, hand-mapped, with no migration tooling

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-008` |
| **Backlog candidate** | `ADR-CANDIDATE-008` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Storage / high |
| **Supersedes / Superseded by** | — (first ADR corpus on this platform; nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-001` (why data cannot be joined and must instead be messaged), `ADR-002` (the toolkit that supplies the repository abstraction); premise of `ADR-CANDIDATE-009`, `ADR-CANDIDATE-010` and `ADR-CANDIDATE-012` |

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

Once services integrate asynchronously and release independently (`ADR-001`, `ADR-002`), the next
question is who owns what data. Pacco's answer is visible in every persisting service and is uniform:

1. ✅ **One logical MongoDB database per service, named after the service.** Eight exist —
   `availability-service`, `customers-service`, `deliveries-service`, `identity-service`,
   `operations-service`, `orders-service`, `parcels-service`, `vehicles-service` — each declared only
   in its owning service's settings
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:97-101`).
2. ✅ **No service holds another's database name or connection string.** Every `mongo` section names
   only the owning service's database.
3. ✅ **Aggregates are mapped to documents by hand.** A document class per aggregate lives under
   `Infrastructure/Mongo/Documents/`, and a static extension file converts document → entity, entity →
   document and document → DTO explicitly
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Mongo/Documents/Extensions.cs:11-38`).
4. ✅ **The domain never sees the database.** The core project declares a repository interface in
   domain terms; the infrastructure project implements it over a generic toolkit repository
   (`…Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:11-36`).
5. ✅ **No migration tooling of any kind exists in the workspace.** A search across all fourteen clones
   for migration, Flyway, Liquibase, Alembic or EF Core migration artifacts returns nothing. The only
   schema management found anywhere is a single unique index created at startup in one service
   (`hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs:13-28`).

That last point is what makes this a decision rather than a technology note: schema evolution on this
platform rests entirely on document-model tolerance and on whoever edits a mapper next.

### 1.1 What this record does *not* cover

This record fixes ownership, the store, the mapping style, and the absence of migrations. It does not
decide how a service obtains *another* service's data — replication (`ADR-CANDIDATE-009`) and narrow
synchronous point-reads (`ADR-CANDIDATE-010`) are separate records, and the platform currently uses
both for the same customer data. Reliability of publication from a database write is
`ADR-CANDIDATE-012`.

## 2. Decision

**Each Pacco service owns exactly one logical MongoDB database, named after the service, reachable only
by that service. Aggregates are mapped to documents by hand inside the infrastructure layer, behind a
domain-shaped repository interface declared in the core project. No shared schema, no ORM and no
migration framework is used; schema evolution is handled by writing tolerant mappers.**

Three obligations are attached and are part of the decision:

1. 🎯 **Every list query must page.** Read-path queries today return whole collections.
2. 🎯 **Every predicate a repository filters on must have a declared index.** One index exists on the
   whole platform.
3. 🎯 **A service that stores nothing must not configure a database.** `operations-service` configures
   one and registers no repository for it.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **One shared relational database** for all services, with per-service schemas or tables. | Rejected because it undoes the independence that the rest of the platform is built for. A shared database is a shared deployment unit: a schema change becomes a cross-team change, a slow query becomes everyone's outage, and any service can read any other's tables, which erodes ownership faster than any convention can defend it. It would also have made `ADR-001`'s messaging largely pointless — services would simply join. |
| 2 | **A relational store per service with a migration framework** (for example PostgreSQL with Flyway or EF Core migrations). | Rejected in practice, and this is the alternative the platform most visibly gave up. It would have supplied exactly what is missing today: a versioned, reviewable, replayable record of every schema change, and a deploy-time failure when a change is incompatible. It was traded away for schema-less write flexibility and for one storage technology across the platform. The cost is stated plainly in §4.2 rather than hidden. |
| 3 | **Event sourcing** — the aggregate's event stream as the system of record, with projections for reads. | Rejected because the platform already publishes domain events for integration (`ADR-001`) and would have had to distinguish integration events from a durable internal stream, plus own snapshotting, replay and projection rebuild. Nothing in the workspace holds an event store or a projection rebuild path, so this was not taken and would be a significant new decision. |
| 4 | **A shared MongoDB database with per-service collections.** | Rejected because it retains the operational coupling of alternative 1 while giving up relational integrity as well — the worst of both. The current design already shares a server (a single container) but keeps logical ownership separate, which is what makes per-service credentials and per-service leases possible. |

## 4. Consequences

### 4.1 Positive

1. ✅ Each service changes its own schema without coordinating with anyone.
2. ✅ Ownership is enforceable at the credential level, not only by convention: each service takes a
   database credential scoped to its own role from the secret store.
3. ✅ The domain layer has no persistence dependency at all — the core project's repository interface
   is expressed in aggregates, and mapping lives entirely in infrastructure.
4. ✅ The read path is deliberately simplified: queries go straight from document to DTO without
   rebuilding an entity, which keeps read code short.

### 4.2 Negative

1. ✅ **No cross-service join and no cross-service transaction exists.** Any question spanning two
   services is answered by replication or by a synchronous read, each with its own trade-off.
2. ✅ **Every document-shape change is an unguarded compatibility decision.** With no migration tool
   and no schema registry, nothing at build or deploy time can tell a developer that a rename will
   orphan existing documents. `[INFERRED]` The failure mode is a field silently reading as its default.
3. ✅ **Optimistic concurrency is enforced by a hand-written predicate,** not by the store: the update
   path replaces a document only when the stored version is lower than the incoming one. A repository
   that forgets that predicate loses the guarantee, and nothing detects it.
4. ✅ **Only one index exists across eight databases.** Every other repository predicate is an
   unindexed scan. ❓ Whether that matters at current data volumes has not been measured.
5. ✅ **Separate databases on one shared server buy design isolation, not availability isolation.**
   Every infrastructure component runs as a single container with no replication or quorum, so all
   eight databases share one failure domain.

### 4.3 Neutral / follow-on

1. ✅ `pricing-service` persists nothing and configures no database — evidence the rule is "one
   database per *persisting* service", not per deployable.
2. ✅ `operations-service` configures a database but registers no repository over it, so its
   configuration describes an intent the code does not implement. Carried as question Q2.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/data/database-per-service-with-document-mapping.md` (Status:
   `Candidate`, evidence Strong). The three obligations in §2 — paging, declared indexes, and no
   database for a service that stores nothing — are that pattern's recommendation adopted as binding.
2. **Constrains** `patterns/data/event-carried-reference-replica.md` and
   `patterns/integration/narrow-synchronous-point-read.md`: both exist only because this record forbids
   reaching into another service's database, and both are the two permitted ways around it.
3. **Constrains** `patterns/data/transactional-outbox-handler-decorator.md`, whose inbox and outbox
   collections live inside each service's own database precisely because that database is private to
   the service.
4. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*).
5. **Pattern Update Proposal:** the pattern file describes hand-mapping without stating a rule for
   *backward-compatible* mapper changes. Since this platform has no migrations, that rule is the whole
   safety story and should be added to the pattern: a mapper may add an optional field or tolerate a
   missing one, but may never rename or retype a persisted field without a written backfill.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | A service declares exactly one database, named after itself, in its own settings | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:97-101` |
| E2 | Eight per-service databases exist, one per persisting service, each declared only in its owner's settings | ✅ | `mongo.database` in the `…Api/appsettings.json` of Availability, Customers, Deliveries, Identity, Operations, Orders, Parcels and Vehicles |
| E3 | Aggregates are mapped to and from documents by hand, including a document → DTO read path that bypasses the entity | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Mongo/Documents/Extensions.cs:11-38` |
| E4 | The repository implementation adapts a generic toolkit repository to a domain-shaped interface declared in the core project | ✅ | `…Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:11-36` |
| E5 | Optimistic concurrency is a hand-written version predicate on the replace operation, not a store guarantee | ✅ | `…Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:30-32` |
| E6 | The repository and its document type are registered per collection in the composition root | ✅ | `…Infrastructure/Extensions.cs:59, 84, 90` |
| E7 | The only schema management on the platform is one unique index created at startup in `identity-service` | ✅ | `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs:13-28` |
| E8 | No migration tooling of any kind exists in any of the fourteen clones | ✅ | Workspace-wide search for migration, Flyway, Liquibase and Alembic artifacts — zero matches |
| E9 | Database credentials are per-service leases from the secret store, scoped to a per-service role, rather than static passwords | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:189-198` |
| E10 | `pricing-service` declares no database at all | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json` — no `mongo` section |
| E11 | Inbox and outbox collections live inside each service's own database, alongside its domain collection | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:102-110` |

### 6.1 Documentation-versus-code conflicts

1. ✅ **`operations-service` configures a database it does not use.** Its settings declare a `mongo`
   section, but no repository is registered over it and the only observed state path in that service is
   the distributed cache. The code is followed here: `operations-service` is counted as configuring a
   database, not as persisting one. This is the same conflict recorded as X5 in
   `docs/architecture-inventory/baselines/architecture-baseline.md` §11.3 and as gap G4 in
   `docs/architecture-inventory/repo-inventory.md` §6.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | Each service's declared database is genuinely private, because no other service holds its name or connection string | Every `mongo` section names only its own service's database, and every credential lease is scoped to a per-service role | The ownership rule this record states would be an aspiration rather than a fact, and a service could be reading another's collections in a way this document does not describe | Inspect the database users and their grants on a running MongoDB instance and confirm each role reaches only its own database |
| A2 | The generic repository supplied by the toolkit performs no hidden schema management, since the toolkit's source is not in this workspace | Only the registration call and the domain-shaped wrapper are visible; the toolkit is a package reference with no source here | A claim central to this record — that there is no migration or schema mechanism at all — would be incomplete | Read the pinned toolkit version's persistence source, or start a service against an empty database and inspect what it creates |
| A3 | The eight databases share one MongoDB server, so they share a failure domain | Every service's connection string points at one host, and the compose stacks run one MongoDB container with no replication | The availability-isolation consequence in §4.2 would be wrong, and the "no transactions" configuration choice in the outbox would read differently | Confirm the deployed topology with whoever operates the platform — one instance, a replica set, or per-service instances |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no one can approve this record or own the schema-compatibility rule it depends on | This ADR leaving `Proposed`, and the mapper-compatibility rule proposed in §5 — a rule with no owner will not be applied | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has confirmed whether any deployed database holds real data. With no migration tooling, the cost of the missing safety net depends entirely on whether existing documents must survive a mapper change | Obligations 1 and 2 in §2 cannot be prioritised, and the mapper-compatibility rule cannot be scoped, without knowing whether data is disposable | Platform owner | Confirm which environments hold non-disposable data, and record a backup and restore procedure for those before the next mapper change | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** How does a developer safely rename or retype a persisted field with no migration tool? | This is the single largest exposure created by this decision. Today the answer is "hope the mapper tolerates it", and a wrong guess silently reads existing documents as defaults | Forbid renames and type changes outright. Add a new optional field, write both for one release, backfill with a one-off script kept in the service repository, then remove the old field in a later release | Platform owner |
| Q2 | **[ACTION NOW]** Should `operations-service` keep its database configuration? | It configures a database and registers no repository, so the configuration promises durability the code does not provide. Anyone reading the settings would reasonably conclude operation status is stored | Remove the unused configuration, or implement the repository — but do not leave settings that describe behaviour the code does not have. This is coupled to the operation-status decision in `ADR-CANDIDATE-014` | Platform owner |
| Q3 | **[handled later by adr_generation]** Should replication and synchronous point-reads be one record or two? | Both are ways around the ownership rule this record fixes, and the platform uses both for the same customer data. Two records risk contradicting each other about which applies when | Keep `ADR-CANDIDATE-009` and `ADR-CANDIDATE-010` separate and have each state explicitly when the other applies, as the backlog's own merge recommendation proposes | `adr_generation` stage |
| Q4 | **[handled later by the design stage]** Which indexes does each service actually need? | One index exists across eight databases; every other repository predicate is an unindexed scan. This is invisible until data volume makes it an incident | Derive the index set from the predicates in each repository class, declare them at startup the way `identity-service` already does, and add paging at the same time | Owners named per service once B1 is resolved |
