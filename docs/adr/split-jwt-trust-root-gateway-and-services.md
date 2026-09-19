# ADR-007: Split JWT trust root between the gateway and the domain services

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-007` |
| **Backlog candidate** | `ADR-CANDIDATE-007` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Security / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-006` (the edge that this trust root authenticates for), `ADR-004` (the declarative configuration the gateway's key lives in), `ADR-CANDIDATE-016` (the secret store that issues the service certificates) |

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

`ADR-006` placed the platform's authentication boundary at the gateway. That record assumed a single
notion of "a valid token". The source does not support that assumption: **three different token
validation configurations exist across eleven deployables**, and they do not agree on what makes a token
acceptable.

1. ✅ **The gateway validates with a symmetric shared secret.** Its configuration carries an issuer
   signing key, requires the issuer to match, and validates lifetime
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:43-48`, and identically in the other
   three gateway configurations). It references no certificate, even though a certificate file sits
   unused in its own project folder.
2. ✅ **Eight domain services validate against a certificate on disk.** Availability, Customers,
   Deliveries, OrderMaker, Orders, Parcels, Pricing and Vehicles each point at the same public
   certificate path, require the issuer to match, and validate lifetime
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:72-80`). None of
   them holds the symmetric key at all.
3. ✅ **Two deployables hold both credentials and validate neither issuer.** `identity-service`, which
   issues the tokens, carries a private-key certificate *and* the symmetric key, and turns issuer
   validation off (`hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/appsettings.json:32-45`).
   `operations-service` carries the public certificate *and* the symmetric key, and also turns issuer
   validation off
   (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json:32-43`).
4. ✅ **The two groups even use different configuration keys for the issuer.** The eight domain services
   and the gateway name it `validIssuer`; `identity-service` and `operations-service` name it `issuer`
   while setting `validateIssuer` to false.
5. ✅ **Token revocation exists in exactly one place.** The issuing service exposes revoke commands and
   validates access tokens against a cache-backed store on its own requests
   (`hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Infrastructure/Extensions.cs:82,97`,
   `…Identity.Api/Program.cs:57-68`). Neither the gateway nor any domain service consults it.
6. ✅ **The signing material is committed to the repositories.** The symmetric key appears in nine
   configuration files across three repositories, and the private-key certificate is committed with a
   development password. Both are cited by path only in this record; no value is reproduced.

`ADR-006` is therefore true about *where* authentication happens and incomplete about *what* it trusts.
This record fills that gap.

### 1.1 What this record does *not* cover

This record fixes the trust root and the revocation position. It does not fix where authentication is
enforced or the fail-open authorization guard behind it (`ADR-006`), the certificate authority that
issues per-service identity certificates for transport authentication (`ADR-CANDIDATE-016` — a
different use of certificates from the token signing discussed here), or the edge's claim gating
(`ADR-004`).

## 2. Decision

**Pacco today validates the same token against two different trust roots — a symmetric shared secret at
the gateway and an asymmetric certificate in the domain services — with two deployables holding both
and validating the issuer of neither. This record ratifies nothing: it records the split as the current
state, states that it is not a design the platform chose deliberately, and commits the platform to
converging on a single asymmetric root.**

Four rules follow from the decision and are part of it:

1. ✅ **A new domain service adopts the certificate root**, matching the eight that already do. Nothing
   may be added to the symmetric group.
2. 🎯 **The gateway converges onto the same certificate root as the domain services.** Until it does,
   the gateway is the only component that can accept a token the services would reject, and vice versa.
3. 🎯 **Issuer validation is on everywhere.** Two deployables have it off today with no stated reason,
   and one of them is the token issuer itself.
4. 🎯 **Revocation is consulted at the boundary that enforces authentication.** Today a revoked token is
   still accepted by the gateway and by all eight domain services, because only the issuing service
   checks the revocation store.

This is a record of a state with a known defect, written per Q1 of
`docs/architecture-inventory/adr-candidates.md`: it describes today truthfully and names what must
change, rather than describing a corrected design the platform is not running.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **One asymmetric key pair everywhere** — the issuer signs with a private key, every validator (gateway included) holds only the public certificate. | Not rejected on merit; this is the target rule 2 in §2 commits to. It is not the current state, and recording it as the decision would make this ADR describe a system nobody is running. Its cost is real but small: the gateway must load a certificate instead of a string, and the certificate must be distributed to it. Its benefit is that no validator holds signing material, so a compromised gateway cannot mint tokens. |
| 2 | **One shared symmetric secret everywhere** — the gateway's current model extended to all eleven deployables. | Rejected because it puts signing capability in every validator. Any of eleven deployables, or anyone with read access to any of the repositories, could then mint a token the whole platform accepts — including one carrying the admin role claim that gates five gateway routes (`ADR-006`). It is simpler than the split that exists today, which is the only argument for it, and it is strictly worse than alternative 1. |
| 3 | **An external identity provider** with published verification metadata, replacing the in-house issuing service. | Rejected as out of scope rather than on merit. It would resolve the trust root, revocation and key rotation in one move, but it removes a service the platform already owns and operates, and nothing in the workspace shows it being considered. Revisit if the platform gains external or federated users. |
| 4 | **Keep the split and document it** — accept that the gateway and the services trust different roots, and specify which components a token is valid for. | Rejected because the split has no stated benefit anywhere in the code, and because it makes "is this token valid" a question with no single answer. A component added tomorrow would have to guess which root applies to it, which is exactly the condition rule 1 in §2 removes. |

## 4. Consequences

### 4.1 Positive

1. ✅ The eight domain services hold no signing material — only a public certificate — so a compromise
   of any one of them does not yield the ability to mint tokens.
2. ✅ Every validator requires token lifetime to be checked, so an expired token is rejected everywhere.
3. ✅ The issuing service is a single, identifiable place where tokens are minted and where refresh and
   revoke commands live.

### 4.2 Negative

1. ✅ **A token accepted by one boundary is not guaranteed to be accepted by the other.** The gateway
   and the domain services derive validity from different keys. `[INFERRED]` the platform works today
   only because the issuing service happens to produce tokens both roots accept, and nothing in the
   workspace establishes how.
2. ✅ **Signing material is committed to source control.** The symmetric key appears in nine files
   across the gateway, `identity-service` and `operations-service`, and the private-key certificate is
   committed with a development password. Anyone with repository read access can mint a token the
   gateway accepts, including one carrying the admin role claim.
3. ✅ **A revoked token still works.** Revocation is checked only on the issuing service's own requests.
   The gateway and all eight domain services accept a revoked token until it expires, which is up to the
   configured token lifetime.
4. ✅ **The token issuer does not validate issuers.** `identity-service` and `operations-service` both
   have issuer validation off, so a token signed by the right key but issued by something else would
   pass at those two deployables.
5. `[INFERRED]` **Key rotation is an eleven-repository change with no mechanism to coordinate it.**
   Rotating the symmetric key means editing nine files in three repositories; rotating the certificate
   means replacing a file in eleven. The platform has no coordinated release mechanism
   (`ADR-CANDIDATE-018`).

### 4.3 Neutral / follow-on

1. ✅ `operations-service` sitting in the symmetric group is consistent with it being the one service
   the gateway's asynchronous mode does not front — but nothing states that as the reason, and its
   certificate is a public one rather than a signing one, so it is not obviously a second issuer.
2. ✅ The gateway's own project carries an unused public certificate file. `[INFERRED]` this is the
   residue of an earlier attempt at alternative 1, or a copy taken with the project template.

## 5. Relationship to the implementation pattern catalog

1. **Constrains** `patterns/security/edge-enforced-authentication-with-identity-binding.md` (Status:
   `Candidate`, evidence Strong). That pattern describes the enforcement point and assumes one notion of
   a valid token; this record supplies the missing half and narrows the pattern to the certificate root
   in rules 1 and 2 of §2.
2. **Relates to** `patterns/security/vault-issued-dynamic-credentials-and-service-pki.md` without
   instantiating it. The certificates that store issues are *service identity* certificates for
   transport authentication, and the certificate discussed here is the *token signing* root loaded from
   disk. Conflating the two would be wrong, and the pattern's own note that services load certificates
   from disk while a certificate authority is configured is the same observation from the other side.
3. **Deliberately diverges** — and this is the one place in the batch where the divergence is not
   defensible. The split trust root is a divergence from any coherent pattern, has no recorded
   rationale, and is recorded here so that it stops being invisible.
4. **Pattern Drift:** none formally, because every pattern in the catalog is `Candidate`
   (`patterns/index.md`, *Governance*). If the edge-authentication pattern is approved as written, the
   symmetric gateway root becomes drift immediately, and rule 2 in §2 is the fix.
5. **Pattern Update Proposal:** amend
   `patterns/security/edge-enforced-authentication-with-identity-binding.md` to state the trust root
   explicitly — one asymmetric key pair, validators hold the public certificate only, issuer validation
   on, revocation consulted at the enforcement point. The pattern currently says nothing about any of
   the four, which is why the split went unrecorded.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | The gateway validates tokens with a symmetric issuer signing key, requires the issuer to match, and validates lifetime | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:43-48` (key value not reproduced) |
| E2 | All four gateway configurations carry the same symmetric key and the same validation settings | ✅ | `ntrada.yml:43-48`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` |
| E3 | The gateway configurations reference no certificate for token validation, although a certificate file exists in the project | ✅ | Search of all four gateway configurations — no certificate key; file present at `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/certs/localhost.cer` |
| E4 | Eight domain services validate against a public certificate on disk, with issuer and lifetime validation on and no symmetric key | ✅ | `…Orders.Api/appsettings.json:72-80`; same shape in Availability (`:79-87`), Customers, Deliveries, OrderMaker, Parcels, Pricing and Vehicles |
| E5 | The issuing service holds a private-key certificate and the symmetric key together, and has issuer validation off | ✅ | `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/appsettings.json:32-45` (values not reproduced) |
| E6 | `operations-service` holds a public certificate and the symmetric key together, and has issuer validation off | ✅ | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json:32-43` |
| E7 | The two groups use different configuration key names for the issuer | ✅ | `validIssuer` in the gateway and the eight domain services; `issuer` in `identity-service` and `operations-service` at the lines cited in E1, E4, E5 and E6 |
| E8 | The symmetric key is present in nine configuration files across three repositories | ✅ | Four gateway configurations; `…Identity.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json`; `…Operations.Api/appsettings.json`, `appsettings.docker.json` — cited by path only |
| E9 | The issuing service registers token signing, a cache-backed access-token store, and an access-token validation step on its own pipeline | ✅ | `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Infrastructure/Extensions.cs:56,74,82,88,97` |
| E10 | Revoke commands exist only on the issuing service's route table | ✅ | `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/Program.cs:57-68` |
| E11 | No other deployable registers an access-token validation step, so none consults the revocation store | ✅ | Workspace-wide search for the access-token validation registration — present only in `identity-service` |
| E12 | A private-key certificate with a development password is committed to the issuing service | ✅ | `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/certs/localhost.pfx`, referenced at `appsettings.json:33-37` — cited by path only |

### 6.1 Documentation-versus-code conflicts

1. `docs/architecture-inventory/baselines/architecture-baseline.md` §8.6 and
   `docs/architecture-inventory/repo-inventory.md` §6 gap G9 describe the split as **gateway symmetric
   versus services certificate**, with the same key value also appearing in `operations-service`. The
   code shows a **third configuration**: `identity-service` and `operations-service` hold *both*
   credentials and both have issuer validation off, and the two groups use different configuration key
   names. The code is followed here and the fuller picture is stated in §1 items 3 and 4.
2. `docs/architecture-inventory/adr-candidates.md` blocker B1 states the symmetric key "appears in five
   files". The code shows **nine** — four gateway configurations, three `identity-service` settings
   files and two `operations-service` settings files. The code is followed; the wider spread makes the
   rotation consequence in §4.2 item 5 larger than the backlog implies.
3. `docs/architecture-inventory/baselines/architecture-baseline.md` §11.3 conflict X6 records revocation
   as living only in the issuing service. The code agrees, and E11 confirms no other deployable consults
   it.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The issuing service currently signs tokens in a way both trust roots accept, because the platform demonstrably works end to end | Its configuration carries both a private-key certificate and the symmetric key, and the toolkit that decides which one is used for signing is a package reference with no source in this workspace | If it signs with only one of them, then either the gateway or all eight domain services are rejecting every token today, and the platform's authentication is broken rather than merely inconsistent | Issue a token in a running environment and verify it independently against the certificate and against the symmetric key; record which succeeds |
| A2 | The certificate the eight domain services load is the public half of the certificate the issuing service holds | Both are named for the same host, the issuing service holds the private-key variant and every other repository holds the public one, and issuer validation passes on the domain services | Domain services would be validating against an unrelated key and would reject every token, which contradicts A1 | Compare the public key in the services' certificate with the one in the issuing service's private-key certificate |
| A3 | The certificate committed to the gateway's project is unused rather than loaded by some path not visible in configuration | No gateway configuration references a certificate, and the gateway's key-based settings are complete on their own | The gateway might already validate asymmetrically in some environment, and rule 2 in §2 would be describing work already done | Search the gateway's runtime configuration sources, including environment variables, for a certificate reference in a deployed environment |
| A4 | Turning issuer validation off in the two deployables that do so was not deliberate | Nothing in either repository states a reason, both use a different configuration key name for the issuer than the other nine components, and both accept the same issuer name the others require | It could be intentional — for example if those two accept tokens from a second issuer — and rule 3 in §2 would break that | Ask whoever configured those two services whether a second token issuer exists or was planned |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so nobody can approve this record or own the platform's trust root | This ADR leaving `Proposed`, and every rule in §2 — each needs a person accountable for a credential change across three or eleven repositories | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has confirmed whether the committed signing material is live or throwaway. The symmetric key is in nine files across three repositories and the private-key certificate is committed with a development password. Anyone holding the key can mint a token the gateway accepts, including an administrator one | This record cannot be treated as an architecture decision until it is answered — if the material is live, this is an incident and rotation comes before any ADR | Platform security owner | Check whether the committed values match anything in a running environment. If they do, rotate first, then converge on the certificate root per rule 2 in §2 | TBD |
| B3 | **[ACTION NOW]** A revoked token is still accepted by the gateway and by all eight domain services, because revocation is consulted only by the issuing service. There is no way to cut off a stolen or misused token before it expires | Rule 4 in §2, and any statement that the platform can revoke access. It also blocks any answer to Q2 that relies on revocation as a mitigation | Platform security owner | Decide where revocation is checked — the gateway is the natural point — and implement it there before the platform is exposed to callers outside the team | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Is there a reason for the split trust root that is not visible in the code? | Every alternative in §3 is better than the split, and rule 2 in §2 commits to removing it. If a reason exists — an external constraint, a library limitation, a migration in progress — the plan changes | No reason was found in any of the fourteen repositories. Treat it as unintentional and converge, unless the gateway's owner names a constraint | Platform security owner |
| Q2 | **[ACTION NOW]** How is the token signing key rotated, and how long does a rotation take? | Rotation touches nine files in three repositories for the symmetric key and eleven for the certificate, and the platform has no coordinated release mechanism. Without an answer, the response to a leaked key is undefined | Move both roots out of source control into the secret store every service already loads settings from (`ADR-CANDIDATE-016`), so rotation is a store update rather than an eleven-repository change | Platform security owner |
| Q3 | **[ACTION NOW]** Do the two deployables with issuer validation off need to accept tokens from a second issuer? | Rule 3 in §2 turns issuer validation on everywhere. If a second issuer is planned or exists, that rule breaks it | No second issuer was found. Turn validation on in both and make the configuration key name consistent with the other nine components at the same time | Platform security owner |
| Q4 | **[handled later by adr_generation]** Should the token signing root and the service identity certificates be issued by the same authority? | Both are certificates and both are per-platform concerns, but they answer different questions — who signed this token, versus which service is calling. Conflating them would put token signing capability into the per-service certificate issuance path | Keep them separate. Record the service identity half in `ADR-CANDIDATE-016` and cross-reference this record so the distinction is explicit rather than assumed | `adr_generation` stage, in `ADR-CANDIDATE-016` |
