# Pattern: Vault-Issued Dynamic Credentials And Service PKI

## Status

**Candidate.** No approval metadata, ADR, or documented human sign-off exists in the workspace, and
the CAKE catalog for tenant `Q5SCXYFS` holds no governing decision.

## Category

security

## Problem

A microservice needs three kinds of secret at startup: settings that differ per environment, a database
credential, and a certificate to identify itself. The default answer is to put all three in the
configuration file, which means they are in source control, shared by every environment, identical for
every service, and effectively permanent — nobody rotates a password that thirty files depend on.

## Context

Applies where each service owns its own datastore and its own configuration, and where the number of
services makes hand-managed credentials unworkable. Pacco has nine services that read settings, a
database credential, and a certificate from HashiCorp Vault at startup, each under a path named for
itself.

## Architecture Summary

Each service names itself to Vault and gets back three things, from three different Vault engines:

1. **Settings**, from the KV v2 engine at a path named for the service. These are merged into the
   application configuration, overriding the values shipped in `appsettings.json`.
2. **A database credential**, from the database secrets engine under a role named for the service.
   Vault creates a short-lived MongoDB user; the connection string is assembled from a template with
   `{{username}}` and `{{password}}` placeholders. The lease is renewed automatically while the service
   runs.
3. **A certificate**, from the PKI engine under a role named for the service, issued for a common name
   of the form `<service>.pacco.io`.

All three are configured in one `vault` block per service and activated by a single call in the host
builder. The service code never sees a credential literal — it sees a connection string that Vault
filled in.

## Structure / Flow

```mermaid
flowchart LR
    subgraph Startup
        S[Service host builder]
    end
    S -->|authenticate: token| V[(Vault)]
    V -->|kv/orders-service/settings| CFG[Merged configuration]
    V -->|database role orders-service| CRED["mongodb://user:pass@host"]
    V -->|pki role orders-service<br/>CN orders-service.pacco.io| CERT[Service certificate]
    CFG --> APP[Running service]
    CRED --> APP
    CERT --> APP
    APP -.->|auto-renew lease| V
```

## When to Use

- Each service owns its own datastore, so each can hold a distinct credential without coordination.
- Credentials should be short-lived and revocable centrally rather than rotated by editing files.
- Services need identity certificates and issuing them by hand does not scale past a handful.
- Per-environment configuration differences are real and should not live in the repository.

## When Not to Use

- The datastore does not support programmatic user creation, so dynamic credentials degrade into a
  static secret with extra machinery.
- The service must start when the secret store is unreachable. Fetching secrets at startup makes the
  store a hard dependency of every service coming up.
- One or two services with one operator. The overhead of running, unsealing, and backing up a secret
  store exceeds the benefit.

## Key Components

| Component | Responsibility |
|-----------|----------------|
| Vault server | Holds settings, issues database credentials and certificates |
| `vault` configuration block | Per-service: URL, auth type, KV path, PKI role, database lease |
| `UseVault()` in the host builder | The single activation point; without it the block is inert |
| KV v2 engine (`mountPoint: kv`) | Per-service settings at `<service>-service/settings` |
| Database engine role | Creates a MongoDB user per lease, named for the service |
| PKI engine role | Issues a certificate for `<service>-service.pacco.io` |
| Connection-string template | `mongodb://{{username}}:{{password}}@localhost:27017` — filled at lease time |

## Data / Event / API Contracts

- **KV path:** `kv/<service>-service/settings`, engine version 2. Nine services, nine paths, identical
  shape.
- **Database lease:** `type: database`, `roleName: <service>-service`, `autoRenewal: true`. Present in
  eight services; **`pricing-service` configures `kv` and `pki` but no `lease` block**, consistent with
  it being the one service that owns no database.
- **PKI:** `roleName: <service>-service`, `commonName: <service>-service.pacco.io`. Present in all nine.
- **Auth:** `authType: token` with a `token` value in configuration. A `username`/`password` pair is
  also present in every block but unused while `authType` is `token`.
- **Absent entirely:** `ordermaker-service` and `api-gateway` have no `vault` block. Neither owns a
  database; the gateway nonetheless holds a committed signing key that Vault would be the natural place
  for.
- **Disabled deliberately:** the Pact provider test project sets `vault.enabled: false` and uses an
  older flat `key` field rather than the `kv` block — so tests run without the secret store.

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| KV settings path | `<service>-service/settings` under mount `kv` | `orders-service/settings` |
| Database role | `<service>-service` | `parcels-service` |
| PKI role | `<service>-service` | `vehicles-service` |
| Certificate common name | `<service>-service.pacco.io` | `deliveries-service.pacco.io` |
| Connection template placeholders | Double-brace, lower case | `{{username}}`, `{{password}}` |
| Activation | `.UseVault()` on the host builder, after configuration | — |

The one name that does not follow the platform's own convention is the JWT certificate: it is
`certs/localhost.cer`, a file on disk, not a Vault-issued `<service>.pacco.io` certificate. The PKI
engine is configured but nothing observed in source consumes its output.

## Service / Boundary Guidance

- **One Vault path per service, named for the service.** No shared path, no shared role. This is what
  makes a compromised service a bounded problem rather than a total one.
- **Bind the database role to that service's database only.** The configuration names a role; what the
  role can reach is defined inside Vault and is not visible from these repositories. If the role grants
  server-wide access, the per-service split is cosmetic.
- **Keep `UseVault()` in the host builder, not in service code.** Secrets arrive as ordinary
  configuration, so handlers and repositories stay unaware of where they came from.
- **A service with no datastore configures no lease.** `pricing-service` demonstrates this; the pattern
  is composable per concern rather than all-or-nothing.
- **The gateway is outside this pattern and should not be.** It holds the most sensitive value on the
  platform in a committed file.

## Security / Compliance Considerations

- **Vault runs in development mode in every compose file.** `VAULT_DEV_ROOT_TOKEN_ID=secret` means an
  in-memory store, auto-unsealed, over plain HTTP, with a known root token. Nothing persists across a
  restart, and there is no seal to protect anything.
- **Services authenticate with that root token.** `authType: token` with `token: "secret"` in each
  `appsettings.json` is the Vault root credential, committed to source, shared by nine services. A
  service compromised at runtime has full administrative access to the secret store — including every
  other service's secrets. This inverts the isolation the per-service paths were meant to provide.
- **Vault is reached over `http://localhost:8200`**, so tokens and issued credentials cross the network
  unencrypted.
- **Vault unseal keys and a root token are recorded in a text file in the deployment repository.** They
  are referenced here by path only and no value is reproduced.
- **Committed defaults remain as fallbacks.** Every value Vault is meant to supply also exists in
  `appsettings.json`. Where the KV path is empty or unreachable, the service runs on the committed
  value rather than failing, and nothing distinguishes the two cases at runtime.
- **The PKI engine is configured but its output is not observed in use.** Services present
  `certs/localhost.cer` from disk. Whether PKI is aspirational or wired up outside these repositories
  is unresolved.
- **No TLS is configured between services or to MongoDB, Redis, or RabbitMQ**, so a dynamically issued
  database credential travels in clear text to a datastore that accepts clear-text connections.

## Observability Considerations

- **No metric or log field reports secret-store health.** A Vault failure surfaces as a startup
  exception or, worse, as a silent fallback to committed configuration.
- **Lease renewal is automatic and unobserved.** If renewal stops, the database credential expires and
  the service begins failing on data access with no leading indicator.
- The `/ping` endpoint every service exposes reports process liveness only; it does not check that the
  database credential is still valid ([[registry-mediated-discovery-and-routing]]).
- `ConnectionString` and `Password` are on the logger's excluded-property list, so an assembled
  connection string is not written to logs by the structured logging path
  ([[structured-logging-with-property-redaction]]).

## Failure Handling

- **Vault unreachable at startup:** behaviour depends on the toolkit's handling; the committed defaults
  are present and the observable outcome in this repository set is that a service can start on them.
  This is the single most important thing to test deliberately.
- **Vault restarted:** in development mode all stored data is lost. Every KV path becomes empty and
  every service falls back to committed values on its next start.
- **Lease renewal failure:** the MongoDB user is revoked when the lease expires and data access fails.
  No compensating behaviour is configured.
- **PKI issuance failure:** not observable, because nothing in source consumes the issued certificate.
- **Test environment:** the Pact provider tests disable Vault outright, which is correct — a contract
  test should not depend on a secret store.

## Trade-offs

| Gain | Cost |
|------|------|
| Database credentials are short-lived and revocable centrally | Every service now depends on the secret store being up to start |
| Per-service paths and roles bound a compromise to one service | Undone here by every service holding the root token |
| Configuration differences move out of the repository | The repository still carries a full set of working defaults, so nothing forces the move |
| Certificates can be issued per service without manual work | The issued certificates are not observed in use; the configured PKI does no work today |
| One block and one call per service; no code changes | Also means no code notices when it silently does nothing |
| Composable — a service with no database configures no lease | Composability makes partial adoption easy to leave permanently partial |

## Variants

- **KV only** (settings), **KV + PKI** (`pricing-service` — no datastore), or **KV + PKI + lease**
  (the other eight). One block, three independently switchable concerns.
- **Token auth** versus the `username`/`password` fields also present in every block. Only token auth
  is active; the other is configured and unused.
- **Vault disabled** for tests, with a flat `key` field from an earlier configuration shape.
- **No Vault at all** — `ordermaker-service` and `api-gateway`.

## Anti-patterns

- **Authenticating with the root token.** The per-service paths, per-service roles, and per-service
  certificates all assume a service can only read its own; the root token means every service can read
  everything.
- **A development-mode secret store in the platform's deployment files.** In-memory, auto-unsealed,
  known root token, plain HTTP — every property that makes a secret store a secret store is off.
- **Unseal keys and a root token committed as a text file.**
- **Keeping working defaults in the repository for every value the secret store supplies.** It removes
  the forcing function and turns a store outage into a silent downgrade instead of a loud failure.
- **Configuring an engine nothing consumes.** The PKI block reads as though services present issued
  certificates; they present a file called `localhost.cer`.
- **Plain HTTP to the secret store**, which exposes in transit exactly what the store exists to protect.

## Evidence

- **ADR:** none — no ADR exists in any cloned repository or in the CAKE catalog for tenant `Q5SCXYFS`.
- **Repo:** nine service repositories carrying a `vault` block; `hianshul100_Pacco` (deployment).
- **Service:** `identity`, `availability`, `vehicles`, `operations`, `deliveries`, `customers`,
  `orders`, `parcels`, `pricing`. Not `ordermaker`, not `api-gateway`.
- **File:**
  `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json` — `vault` block
  with `authType: token`, a committed `token`, `kv.path: orders-service/settings`,
  `pki.roleName: orders-service` / `commonName: orders-service.pacco.io`, and
  `lease.mongo.templates.connectionString: mongodb://{{username}}:{{password}}@localhost:27017`;
  the same block, differing only in the service name, in
  `Identity.Api`, `Availability.Api`, `Vehicles.Api`, `Operations.Api`, `Deliveries.Api`,
  `Customers.Api`, `Parcels.Api`;
  `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json` — `kv` and `pki`
  present, **no `lease` block**;
  `hianshul100_Pacco.Services.Parcels/tests/Pacco.Services.Parcels.PactProviderTests/appsettings.json`
  — `vault.enabled: false` with the older flat `key` field;
  `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json` and
  `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/appsettings.json` — no `vault` block;
  `UseVault()` at `Identity.Api/Program.cs:74`, `Availability.Api/Program.cs:48`,
  `Vehicles.Api/Program.cs:43`, `Operations.Api/Program.cs:50`, `Deliveries.Api/Program.cs:41`,
  `Customers.Api/Program.cs:43`, `Orders.Api/Program.cs:44`, `Parcels.Api/Program.cs:47`,
  `Pricing.Api/Program.cs:33`;
  certificates on disk: `localhost.pfx` only under `Identity.Api/certs/`, `localhost.cer` under nine
  other service `certs/` directories and under `Pacco.APIGateway/certs/`.
- **API/Event:** none — this pattern has no runtime API or message contract.
- **Deployment/Config:** `hianshul100_Pacco/compose/consul-fabio-vault.yml:27-37`,
  `infrastructure.yml:115-125`, `host-infrastructure.yml:79-88` — `image: vault` with
  `VAULT_DEV_ROOT_TOKEN_ID=secret` and `VAULT_ADDR=http://127.0.0.1:8200` in all three;
  `hianshul100_Pacco/docker-images.txt` — unseal keys and root token, recorded by path only.
- **Notes:** `architecture-baseline.md` §8.5, §11.3. **Conflict — configured versus consumed:** the
  `pki` block implies services present Vault-issued certificates, while source shows every service
  loading `certs/localhost.cer` from disk. Source is treated as authoritative: PKI is configured, and
  its output is **not observed** in use.

## Related ADRs

None. No ADR, decision record, or RFC exists in the workspace, and the CAKE graph for tenant
`Q5SCXYFS` returned zero nodes.

## Related Patterns

- [[edge-enforced-authentication-with-identity-binding]] — the committed signing key this pattern was
  meant to eliminate.
- [[database-per-service-with-document-mapping]] — the per-service databases the dynamic credentials
  are scoped to.
- [[framework-supplied-platform-conventions]] — `UseVault()` is one of the toolkit's one-line
  capabilities.
- [[registry-mediated-discovery-and-routing]] — the other startup-time infrastructure dependency.
- [[composable-per-concern-environment-stacks]] — where the Vault container is declared.
- [[structured-logging-with-property-redaction]] — keeps issued credentials out of log output.

## Recommendation

**Adopt the shape; do not mistake the current configuration for a working secret store.** Per-service
KV paths, per-service database roles, per-service certificate common names, and a single `UseVault()`
activation point are all the right structure, and `pricing-service` omitting the lease block shows the
composition works. But as configured, Vault runs in development mode with a known root token that every
service uses to authenticate — which means the isolation the per-service naming implies does not exist,
and a restart loses every secret. Before this can be relied on: run Vault sealed and persistent behind
TLS, give each service its own least-privilege auth credential instead of the root token, remove the
unseal keys and root token from the repository, remove the committed fallback values so a store outage
fails loudly, and either consume the PKI-issued certificates or stop configuring the engine.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The development-mode Vault settings in the compose files are for local work only, and a real deployment uses a sealed, persistent, TLS-protected server | The compose files are named for local infrastructure and use `localhost` addresses throughout; dev mode is not something anyone chooses deliberately for a real environment | The platform would be storing every secret in a process that loses them on restart, unsealed, behind a published root token | Ask the operator how Vault is run outside these compose files; if there is no such deployment, dev mode is the deployment |
| A2 | The Vault database role for each service grants access only to that service's own MongoDB database | The roles are named per service and the databases are named per service, which is what per-service scoping looks like from the outside | A compromised service could read and write every other service's data, and the database-per-service boundary would be naming only | Read the Vault database role definitions and check the MongoDB creation statement each one issues |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Every service authenticates to Vault with the root token, and that token is committed in nine `appsettings.json` files | Any isolation benefit from per-service paths and roles; safe use of Vault in a shared environment | Platform security owner, with the platform owner | Create a least-privilege auth method per service (AppRole or Kubernetes auth) with a policy limited to that service's KV path, database role, and PKI role; remove the root token from all configuration | TBD |
| B2 | **[ACTION NOW]** Vault unseal keys and a root token are stored as a text file in the deployment repository | Treating any Vault-held secret as secret; safe sharing of the deployment repository | Platform security owner | Rotate the Vault instance, remove the file from the repository and its history, and hold unseal material out of band under a documented recovery procedure | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should a service start when Vault is unavailable? | Every value Vault supplies also exists as a committed default, so an outage may silently downgrade the platform to repository values instead of failing | It should fail to start. Silent fallback is worse than downtime because nothing reports it, and the fallback values are the ones already known to anyone with repository access | Platform owner, with the platform security owner |
| Q2 | **[ACTION NOW]** Is the PKI engine actually used, or is it configured and inert? | All nine services configure a PKI role and common name, yet every service loads `certs/localhost.cer` from disk. Configuration that does nothing is misleading in a security review | Not used, based on source. Either wire the issued certificate into service-to-service TLS, or remove the block so it stops implying a control that is not there | Platform security owner, with the owner of `hianshul100_Pacco` |
| Q3 | **[handled later by the design stage]** Should `api-gateway` and `ordermaker-service` read from Vault too? | The gateway holds the JWT signing key — the most sensitive value on the platform — in a committed file, and it is the one component with no `vault` block | Yes for the gateway; it is the clearest single win. `ordermaker-service` owns no datastore and may genuinely need nothing | Platform security owner |
| Q4 | **[handled later by the design stage]** Should traffic to Vault, MongoDB, Redis, and RabbitMQ use TLS? | Dynamically issued credentials currently travel in clear text to datastores that accept clear-text connections, which undoes much of the value of rotating them | Yes, at least for Vault, since it carries the credentials for everything else | Platform security owner, with the operator |
