# ADR-016: Centrally issued dynamic database credentials and per-service certificates

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-016` |
| **Backlog candidate** | `ADR-CANDIDATE-016` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Security / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-007` (the caller-facing trust split, and the source of question Q4 answered here), `ADR-006` (the authorization position this complements), `ADR-008` (the databases these credentials open), `ADR-017` (the environment this store runs in) |

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

`ADR-007` settled how a caller proves who they are at the edge and left open how one service proves who
it is to another, and how a service obtains the credentials for its own database. `ADR-008` gave every
service its own database, which multiplies the number of database credentials the platform has to manage
by the number of services.

1. ✅ **Nine of the eleven deployables obtain settings and credentials from a central store at startup.**
   Each declares a store section and calls the store extension in its composition root: Availability
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/Program.cs:48`),
   Customers (`…/Program.cs:43`), Deliveries (`…/Program.cs:41`), Identity (`…/Program.cs:74`),
   Operations (`…/Program.cs:50`), Orders (`…/Program.cs:44`), Parcels (`…/Program.cs:47`), Pricing
   (`…/Program.cs:33`) and Vehicles (`…/Program.cs:43`).
2. ✅ **The API gateway and the saga coordinator have no store configuration at all.** Neither declares a
   store section and neither calls the extension, so the platform's security posture has two exceptions
   that are not stated anywhere as exceptions.
3. ✅ **Each service pulls three things.** A settings path scoped to its own name, a certificate role
   with a common name of the form `<service>.pacco.io`, and — for the eight services that own a database —
   a dynamically issued database credential under a role named for the service, renewed automatically,
   with the connection string assembled from a template.
4. ✅ **The one service with no database has no credential lease.** Pricing carries the settings path and
   the certificate role but no lease, which is consistent with `ADR-008`: it owns no data.
5. ✅ **Every service authenticates to the store with the same static token.** All nine sections use
   token authentication against the same local address with an identical literal token value. The value
   is not reproduced in this record, but that is a convention of this document only and not a protection:
   the same literal is committed in plain text in all nine services' `appsettings.json` files, so anyone
   with read access to any one of those repositories already holds it. The file paths are given in §6.
6. ✅ **Exactly one certificate check is switched on, on exactly one route family.** Customers enables
   the check, restricts it to one domain and one host, and grants a single caller — Availability — a
   single named permission (`hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-181`).
7. ✅ **Availability is configured as the calling side only, and the certificate it presents is the one
   the store issued.** Its security section carries the header name used to transport the certificate and
   no enable flag
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:166-170`),
   and its client asks the certificate service for the certificate belonging to its configured
   certificate role, then attaches the raw certificate data under that header
   (`…/Infrastructure/Services/Clients/CustomersServiceClient.cs:21-34`). Nothing on that path reads a
   certificate from the filesystem.
8. ✅ **No other service has a security section.** Nine of the eleven deployables therefore accept
   internal calls with no service identity of any kind.
9. ✅ **The store itself runs in development mode.** It runs as the stock image with a well-known root
   token supplied through the environment, no transport encryption and no persistent volume
   (`hianshul100_Pacco/compose/consul-fabio-vault.yml:27-39`, `hianshul100_Pacco/compose/infrastructure.yml:115-127`).

### 1.1 What this record does *not* cover

This record fixes where services get database credentials and service certificates. It does not cover
caller-facing authentication (`ADR-007`), the authorization gap in the services' own handlers
(`ADR-006`), the database-per-service decision itself (`ADR-008`), or how the store is deployed and
operated (`ADR-017`). It states the current development-mode posture rather than endorsing it.

## 2. Decision

**Pacco obtains per-service database credentials and per-service certificate identities from one central
secrets store at startup. Each service pulls only its own settings path, its own certificate role and —
if it owns a database — its own dynamically issued, automatically renewed database credential. Service
identity between services is proved by a certificate presented on an internal call and checked against a
per-service allow list. Today this is configured in development mode with a shared static token, and one
service pair actually enforces it.**

Six rules follow from the decision and are part of it:

1. ✅ **A service pulls its own scope and nothing wider.** Settings path, certificate role and credential
   role are all named for the service, so a compromised service reaches its own database and no other.
2. ✅ **Database credentials are issued per process, not checked in.** They are minted on startup and
   renewed while the process lives, so no long-lived database password exists in any repository.
3. ✅ **A service with no data has no credential lease.** Pricing demonstrates the rule.
4. ✅ **Where a service-to-service call is restricted, the restriction is an explicit allow list of
   caller and permission, held by the callee.** Customers holds the only such list today.
5. 🎯 **Each service must authenticate to the store as itself.** A shared static token means every
   service can request every other service's scope, which defeats rule 1 entirely.
6. 🎯 **The store must run outside development mode before it holds anything real** — persistent
   storage, transport encryption, sealed at rest, and no bootstrap material in any repository.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Static credentials in configuration or environment variables** — one connection string per service, set at deploy time. | Rejected because it puts a long-lived database password into eight deployment descriptors, gives no rotation path short of a coordinated restart, and makes a leaked configuration file a permanent compromise. It is also what the platform would fall back to if the store were removed, which is why the store's development-mode posture (rule 6) matters rather than being cosmetic. |
| 2 | **The platform's caller token for service-to-service calls too** — forward the caller's bearer token on internal calls. | Rejected because it conflates two questions: who the end caller is, and which service is calling. A forwarded caller token cannot express "only Availability may read customers", which is exactly the restriction the one enabled allow list encodes. `ADR-007` already splits the trust root at the edge; extending that root inward would make every service a token validator. |
| 3 | **A service mesh with mutual transport authentication** — identity issued and enforced by the platform, invisible to application code. | Rejected on cost and fit, not on merit: the platform runs as containers on a single host with no orchestrator (`ADR-017`), so there is nothing for a mesh to attach to. It would also remove the per-permission allow list, which is finer-grained than transport identity alone. It becomes the right answer if the platform ever moves to an orchestrator. |
| 4 | **No service identity at all** — treat the internal network as trusted. | Rejected because it is the posture nine of the eleven deployables already have by omission (context point 8), and stating it as a decision would make an accident into a policy. The one enforced pair shows the intended direction. |

## 4. Consequences

### 4.1 Positive

1. ✅ No database password exists in any repository. The credential a service uses did not exist before
   the process started and stops working after it ends.
2. ✅ Blast radius is bounded by service. A credential opens one database, matching the boundary
   `ADR-008` drew.
3. ✅ Settings and credentials arrive through one mechanism, so a service's composition root has one
   place where external configuration enters.
4. ✅ Automatic renewal means a long-running service does not need a restart to keep working, and a short
   lease is affordable.
5. ✅ The one enforced allow list is specific: named caller, named permission, restricted domain and
   host. It is a small, readable model that would scale to more pairs unchanged.

### 4.2 Negative

1. ✅ **One static token opens every service's scope.** All nine sections authenticate with the same
   value, so per-service scoping (rule 1) is descriptive rather than enforced: any process holding that
   token can request any other service's settings, certificate and database credential.
2. ✅ **Bootstrap material is committed to the repository.** A file in the platform repository contains
   the store's initialisation output — unseal material and an initial root token — alongside the command
   recipes that create the database and certificate roles. It is cited by path only in §6; no value from
   it is reproduced here. Anyone with repository access can unseal and take over the store.
3. ✅ **The store has no persistent storage and no transport encryption.** In development mode its
   contents are lost on restart, and every credential it issues travels unencrypted on the host network.
4. ✅ **Service identity is enforced on one call path out of many.** One callee has an allow list; nine
   services accept internal calls with no service identity at all.
5. ✅ **Two deployables are outside the mechanism entirely.** The gateway and the saga coordinator pull
   nothing from the store, so whatever they need is configured some other way — and the coordinator
   already acts with no caller identity (`ADR-011` §4.2.4).
6. ✅ **Seven of the nine services request a certificate role that does not exist.** Nine services declare
   a certificate role and a common name of the form `<service>.pacco.io`, while the store's setup recipes
   create roles for `availability-service` and `customers-service` only. Exactly one call path both
   presents and checks an issued certificate — Availability to Customers — and on that path the
   certificate is the store-issued one, fetched by role name at construction time. The other seven
   services request an identity from an authority that has nothing to issue for them, and no call path
   would check it if it did.

### 4.3 Neutral / follow-on

1. ❓ Pricing calls Customers over HTTP and does not appear on the Customers allow list (evidence 16).
   `[INFERRED]` that either the call does not reach the restricted route family or the allow list is
   incomplete; which of the two holds has not been observed, because it depends on runtime behaviour of
   the certificate check rather than on anything visible in the configuration. Carried as question Q5.
2. ✅ Nothing in the workspace terminates transport encryption anywhere: not the gateway, not any
   service, not the store. Certificate identity between services is therefore carried over unencrypted
   connections.
3. ✅ The mechanism is available to any service that adds a section and a call; nothing about it is
   specific to the nine that use it, so extending it to the gateway and the coordinator is configuration
   rather than design.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates `security/vault-issued-dynamic-credentials-and-service-pki.md`.** This record is the
   decision that pattern describes: a central store, per-service scopes, dynamically issued database
   credentials with automatic renewal, and a certificate identity per service. The pattern is catalogued
   with confidence *Moderate*, which matches the gap between the shape and its enforcement.
2. **Adopts the pattern's recommendation to keep the shape and fix the posture.** The catalog recommends
   adopting the design while removing bootstrap material from the repository and moving the store out of
   development mode. Rules 5 and 6 make both part of the decision.
3. **Narrows the pattern's third recommendation to what the evidence supports.** The catalog says the
   platform should either consume the issued certificates or stop configuring the authority. The
   certificates *are* consumed, on the one call path where both sides are configured; what is unresolved
   is the other seven services, which request a role the authority does not have. Consequence 4.2.6
   records that state and question Q2 carries the choice, because the two options lead to different
   services changing.
4. **Constrains `security/edge-enforced-authentication-with-identity-binding.md`.** That pattern places
   the whole authentication boundary at the edge and leaves the services' own handlers open (`ADR-006`).
   The one enabled allow list here is the platform's only working example of a check made inside a
   service, so this record narrows that pattern's scope by one call path rather than contradicting it.
5. **Depends on `data/database-per-service-with-document-mapping.md`.** The credential lease is per
   service precisely because the database is; the two decisions scale together, and a shared database
   would make rule 1 meaningless.
6. **Constrained by `deployment/composable-per-concern-environment-stacks.md`.** The store's
   development-mode posture is a property of the environment stacks, not of this decision, so rule 6
   cannot be satisfied inside this record — it needs the deployment path `ADR-017` describes.
7. **Pattern Drift: not applicable.** Every entry in `docs/architecture-inventory/patterns/index.md`
   currently carries status `Candidate`. Drift is reportable only against an `Approved` pattern, so no
   drift is recorded for this ADR.
8. **Pattern Update Proposal.** `security/vault-issued-dynamic-credentials-and-service-pki.md` should
   record that two of the eleven deployables are outside the mechanism entirely (consequence 4.2.5), that
   certificate roles exist for only two services while nine request a certificate role
   (consequence 4.2.6), and that the one enforced call path presents the store-issued certificate rather
   than a file-based one (evidence 11a). Its **Related ADRs** entry should change from `None` to
   `ADR-016`.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| 1 | Nine services call the store extension in their composition roots | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/Program.cs:48`; `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/Program.cs:43`; `hianshul100_Pacco.Services.Deliveries/src/Pacco.Services.Deliveries.Api/Program.cs:41`; `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/Program.cs:74`; `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Program.cs:50`; `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/Program.cs:44`; `hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Api/Program.cs:47`; `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Program.cs:33`; `hianshul100_Pacco.Services.Vehicles/src/Pacco.Services.Vehicles.Api/Program.cs:43` |
| 2 | Each of the nine declares a store section with a settings path, a certificate role and a common name of the form `<service>.pacco.io` | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json:164-193` (representative; the other eight follow the same shape in their own `appsettings.json`) |
| 3 | Eight of the nine also declare an automatically renewed database credential lease with a templated connection string | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json:182-192` |
| 4 | Pricing declares a settings path and certificate role but no credential lease | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json` (store section) |
| 5 | All nine sections use token authentication against the same address with an identical token value, committed in plain text in each file (value not reproduced here, but readable in all nine) | ✅ | The nine services' `src/*.Api/appsettings.json` store sections |
| 6 | The API gateway declares no store section and calls no store extension | ✅ | `hianshul100_Pacco.APIGateway/` (no store section in any configuration file; no extension call in the host) |
| 7 | The saga coordinator declares no store section | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json` |
| 8 | Customers is the only service with the certificate check enabled, restricted to one domain and host, with a one-caller one-permission allow list | ✅ | `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-181` |
| 9 | Availability carries only the transport header on its security section | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:166-170` |
| 10 | Both services register and enable the certificate check in their infrastructure setup | ✅ | `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Infrastructure/Extensions.cs:79,91`; `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Extensions.cs:92,105` |
| 11 | The calling client attaches the certificate under the configured header | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs:32-34` |
| 11a | The certificate it attaches is obtained from the store's certificate service by the configured certificate role name, guarded by the store and certificate-authority enable flags — not read from the filesystem | ✅ | `…/Services/Clients/CustomersServiceClient.cs:21-34`; role name `availability-service` at `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:184-189` |
| 11b | The only certificate file referenced on disk belongs to token validation, not to service identity | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:79-81` (inside the `jwt` section) |
| 11c | The checking side reads no certificate from the filesystem; it registers and applies the framework's certificate check against the configured allow list | ✅ | `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Infrastructure/Extensions.cs:79,91`; allow list at `…/Pacco.Services.Customers.Api/appsettings.json:163-181` |
| 12 | The store runs as the stock image in development mode with a root token from the environment and elevated memory-locking capability | ✅ | `hianshul100_Pacco/compose/consul-fabio-vault.yml:27-39`; `hianshul100_Pacco/compose/infrastructure.yml:115-127` |
| 13 | The store has no persistent volume declared | ✅ | `hianshul100_Pacco/compose/infrastructure.yml:128-147` |
| 14 | Store bootstrap material and the role-creation recipes are committed to the platform repository (cited by path only; no value reproduced) | ✅ | `hianshul100_Pacco/docker-images.txt` |
| 15 | Certificate roles are created for two services only — `availability-service` and `customers-service` — while nine request a certificate role | ✅ | `hianshul100_Pacco/docker-images.txt:305,314` compared with evidence 2 |
| 16 | Pricing calls Customers over HTTP and does not appear on the Customers allow list | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/` service client; `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-181` |
| 17 | No transport encryption is terminated anywhere in the workspace | ✅ | Workspace-wide search across the fourteen clones' configuration files and container definitions |

### 6.1 Documentation-versus-code conflicts

1. **A certificate identity is configured for nine services and issuable for two.** Nine services request
   a certificate role and common name from the store (evidence 2), while the store's setup creates roles
   for `availability-service` and `customers-service` only (evidence 15). On the one call path where both
   sides are configured, the certificate presented is the store-issued one, fetched by role name
   (evidence 11a); the checking side reads nothing from disk (evidence 11c), and the only certificate
   file on disk belongs to token validation (evidence 11b). So the mechanism works where it is
   provisioned, and seven services declare an identity that the authority cannot issue and that no callee
   would check. The configuration describes platform-wide service identity; the code implements it for
   one pair. Recorded as **Future/Intended State (Not Implemented)** for the remaining seven, and carried
   as question Q2.
2. **Per-service scoping is configured but not enforced.** Each store section names a scope belonging to
   one service, which reads as isolation, while all nine authenticate with the same token (evidence 5).
   The configuration's structure implies a guarantee the authentication does not provide. Recorded as a
   conflict rather than reconciled; rule 5 states the target and consequence 4.2.1 states the current
   truth.
3. **No stated intent for the two deployables outside the mechanism.** Nothing in any clone says whether
   the gateway and the coordinator are deliberately exempt from the store or simply were never wired to
   it. Recorded as **Unverifiable — Missing Source Evidence** and carried as question Q3.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The store's development-mode configuration exists to make local development easy, not as the intended production posture | It uses the stock image's built-in development mode, a well-known token and no storage — a combination nothing would choose deliberately for real data | If it is the intended posture, this record's rule 6 is a change of direction rather than a correction, and the security position of the whole platform is much weaker than it reads | Ask the platform owner whether any environment beyond a developer machine runs this store, and inspect that environment's store configuration if one exists |
| A2 | The one enabled allow list is the intended model for restricted internal calls, not a leftover experiment | It is complete and coherent — caller, permission, domain, host — and the calling side is wired to match, fetching the store-issued certificate for its own role (evidence 11a) | If it is an experiment, rule 4 has no basis and internal calls have no intended identity model at all, which would leave `ADR-006`'s gap unaddressed on every path | Ask whoever configured the Customers allow list whether it was meant as the platform's model; failing that, call Customers' restricted routes from Availability and from Pricing and compare the responses |
| A3 | Every service that owns a database uses the issued credential rather than a fallback connection string | Each of the eight declares the lease with a connection-string template, which is the store's mechanism for supplying the credential | If a service silently falls back to a checked-in connection string when the store is unreachable, positive consequence 1 is false for that service and a long-lived password exists after all | Start a service with the store unavailable and observe whether it fails to start or connects to the database anyway |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[handled later by the architecture PR review stage]** No repository in the workspace names an owner, a team or a review group, so this record cannot list deciders | This ADR leaving `Proposed`, and rules 5 and 6 in §2, which both need someone accountable for the store | Platform owner (unassigned) | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository; the reviewer assigns deciders when the batch is reviewed as a whole | TBD |
| B2 | **[ACTION NOW]** Store bootstrap material — unseal material and an initial root token — is committed to the platform repository, so repository access is equivalent to control of the store | Any statement that the store protects anything, and decision rule 6 | Platform security owner | Confirm whether the committed material matches any running store. If it does, re-initialise that store and rotate first, then remove the material from the repository and its history and move the recipes to a runbook that reads values from an operator, not from a file | TBD |
| B3 | **[ACTION NOW]** One static token, committed in plain text in all nine services' settings files, is used by every service, so the per-service scoping this record describes is not enforced | Decision rule 1, which is descriptive until this is fixed, and decision rule 5 | Platform security owner | Rotate the shared token, issue one token or authentication method per service, and set each service's settings from the environment rather than from a committed file | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[handled later by the batch 4 authoring stage, in the record covering the deployment path]** Should every service authenticate to the store as itself, and by what method? | Rule 5 requires it. The method matters: a per-service token has the same distribution problem one level down, and the alternatives depend on how the platform is deployed | Decide it with the deployment path: an identity supplied by the environment the service runs in is the only method that does not move the distribution problem, and that environment does not exist yet (`ADR-017` rule 6) | Platform security owner |
| Q2 | **[ACTION NOW]** Should the seven services that request a certificate role with no matching role in the store be given roles, or should the certificate block be removed from their settings? | Today those seven declare a service identity the authority cannot issue and no callee would check, which reads as platform-wide service identity while exactly one call path has it (consequence 4.2.6, §6.1 item 1). Either direction is defensible; they change different files | Create roles only for pairs that will actually enforce a check, and remove the certificate block from the rest — configuration that implies a guarantee nothing delivers is the more expensive of the two states | Platform security owner |
| Q3 | **[handled later by the batch 4 authoring stage, in the record covering service-to-service identity]** Should the API gateway and the saga coordinator be brought inside the store mechanism? | They are two of eleven deployables with no central credential source, and the coordinator publishes commands into four services' exchanges with no identity at all (`ADR-011`) | Bring both in: the gateway holds a committed signing key (`ADR-004` B3) and the coordinator holds none at all, and both problems are the store's shape of problem | Platform security owner |
| Q4 | **[ACTION NOW]** Should services identify themselves to each other with per-service certificates on every internal call, rather than on one call path? | This is the question `ADR-007` handed to this record. Nine services currently accept internal calls with no service identity, so the allow-list model is a working example rather than a policy | Extend it to every synchronous internal call, since there are few of them (`patterns/integration/narrow-synchronous-point-read.md`) and each already has a named caller and callee — but settle Q2 first, because the answers must not contradict | Platform security owner |
| Q5 | **[handled later by the architecture PR review stage]** Is Pricing's call to Customers meant to be on the Customers allow list? | If it is, the allow list is incomplete and the call works only because the restricted route family does not cover it. If it is not, the two callers of Customers are governed by different rules for no recorded reason | Run assumption A2's validation path first; the answer decides whether this is a missing entry or a deliberate scope, and it is a one-line change either way | Platform security owner |

