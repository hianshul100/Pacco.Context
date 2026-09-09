# ADR-006: Authentication enforced at the edge, with fail-open in-service authorization

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-006` |
| **Backlog candidate** | `ADR-CANDIDATE-006` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Security / high |
| **Supersedes / Superseded by** | — (first ADR corpus on this platform; nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-004` (the edge this boundary sits on), `ADR-001` (the exchanges that bypass it), `ADR-002` (why the faulty guard exists in four copies); premise of `ADR-CANDIDATE-007` |
| **Known defect** | This record describes current behaviour that contains a live authorization defect (§4.2). It ratifies the *placement* of the boundary, not the defect; supersession is expected once Q1 is answered |

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

Once one declarative edge fronts every service (`ADR-004`), the platform has to decide where it checks
who a caller is and what they may do. Pacco's answer has two halves, and the second half does not work
as written.

**At the edge** — this half is coherent and is the platform's strongest control:

1. ✅ Thirty-seven routes are marked as requiring authentication in the synchronous configuration; five
   are further gated on an administrator role claim
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:1-5, 137-246`).
2. ✅ Ten routes in that configuration, and thirteen in the asynchronous one, **bind the caller's own
   identity claim into the forwarded URL, query string or message payload**, so the customer identifier
   a service acts on comes from the validated token rather than from the request body
   (`…/ntrada.yml:103, 143, 167, 298, 316, 330, 348, 362, 386, 406`). On those routes impersonation is
   not possible regardless of what the caller sends.
3. ✅ Sign-up and sign-in are explicitly unauthenticated (`…/ntrada.yml:237-276`).

**Behind the edge** — this half is where the defect is:

1. ✅ Services re-check ownership in their command handlers, using a caller context populated either
   from the message envelope or from an HTTP correlation header
   (`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Contexts/AppContextFactory.cs:19-33`).
2. ✅ The guard is written so that **the check only runs when the caller is already authenticated**.
   An unauthenticated caller fails the first term of the condition, the guard is skipped, and the
   operation proceeds. The same shape appears independently in four handlers across two repositories
   (`…/Availability.Application/Commands/Handlers/ReserveResourceHandler.cs:29-33` and the three Orders
   handlers listed in §6).
3. ✅ When no caller context is present at all, the factory returns an empty context whose
   `IsAuthenticated` is `false` and whose identifier is the empty GUID — which is exactly the input
   that makes the guard skip
   (`…/Contexts/IdentityContext.cs:9-17, 33`; `…/Contexts/AppContextFactory.cs:26, 32`).
4. ✅ Two paths reach a handler without traversing the edge: a message published directly onto a
   service's exchange, and a direct call to a service's published container port. The order-creation
   saga demonstrates the first concretely — it processes every message with an empty saga context
   (`hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Handlers/AIOrderMakingHandler.cs:26-41`).
5. ✅ One handler on the order-creation path carries no identity check at all
   (`…/Orders.Application/Commands/Handlers/CreateOrderHandler.cs:29-40`).

### 1.1 What this record does *not* cover

This record fixes *where* authentication is enforced and what services may assume behind it. The split
token trust root — the edge validating with a symmetric key while services validate against a
certificate — is `ADR-CANDIDATE-007`. The certificate-based service-to-service authentication and its
single access list is `ADR-CANDIDATE-016`.

## 2. Decision

**Pacco enforces authentication at the gateway. The gateway validates the caller's token, gates routes
on claims, and binds the caller's identity into the forwarded request or published message. Services
behind the edge trust the caller context they are given and re-check only resource ownership; they do
not re-authenticate.**

Four obligations are attached and are part of the decision. The first is a defect fix, not a
refinement:

1. 🎯 **The ownership guard must deny an unauthenticated caller, not skip the check.** The condition
   must be "deny unless the caller is authenticated *and* is the owner or an administrator". Today it
   reads "deny only if the caller is authenticated and is not the owner", which passes for every
   unauthenticated caller.
2. 🎯 **Every route that takes a user identifier from the request must bind it from the token
   instead.** Binding is applied to ten of thirty-seven authenticated routes today.
3. 🎯 **The edge must be unavoidable, or the per-route rules are advisory.** Services publish their own
   container ports and their exchanges accept direct publication, so a caller inside the network
   reaches handlers directly.
4. 🎯 **The token-signing key must leave source control** and be loaded from the secret store the
   platform already runs.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Full authentication in every service** — each service validates the token itself and derives identity from it. | Rejected because it duplicates validation, key distribution and claim interpretation across ten deployables that share no library (`ADR-002`), so a change to claim handling becomes a ten-repository change. It is, however, the alternative that would close obligation 3 outright: with per-service validation, bypassing the edge would gain an attacker nothing. That is why it is the natural supersession path if Q1 is answered in favour of defence in depth. |
| 2 | **A service mesh or sidecar enforcing identity on every hop.** | Rejected because the platform has no orchestration, no mesh and no infrastructure-as-code in any of the fourteen repositories; it runs as compose stacks and a process manager. Adopting mesh-enforced identity means adopting a deployment platform first, which is a far larger decision than the one recorded here. |
| 3 | **Authorization at the edge only**, with no ownership re-check in handlers at all. | Rejected because the edge cannot know ownership: whether order X belongs to customer Y is a fact only the owning service holds. The handler check is therefore necessary — which makes the defect in §4.2 more serious, not less, because the check that exists is the only one there could be. |
| 4 | **A shared authorization service** that handlers call to ask whether a caller may act. | Rejected because it puts a synchronous dependency in the path of every command, including commands that arrive asynchronously from the broker where there is no caller to fail back to. It also contradicts the narrow synchronous-read rule the platform otherwise keeps (`ADR-CANDIDATE-010`). |

## 4. Consequences

### 4.1 Positive

1. ✅ There is one place to reason about authentication, and one place to change it.
2. ✅ On the ten routes that bind the identity claim, the caller cannot act as another customer no
   matter what the request body says — the strongest control the platform has.
3. ✅ No service holds token-validation code, so no service can get token validation subtly wrong.
4. ✅ Handlers are transport-agnostic: the same ownership check runs whether the command arrived over
   HTTP or over the broker, because both populate the same caller context.

### 4.2 Negative

1. ✅ **The ownership guard passes for unauthenticated callers.** The condition is written so that
   being unauthenticated skips it entirely. Any path that reaches a handler without a caller context —
   a direct broker publication or a direct call to a published container port — therefore performs the
   operation with no authorization at all. This is a live defect, present identically in four handlers
   across two repositories, and it is the reason this record is marked with a known defect.
2. ✅ **`CreateOrderHandler` has no identity check whatsoever,** so order creation relies entirely on
   the edge having bound the customer identifier.
3. ✅ **The whole design depends on every request having traversed the gateway,** and nothing enforces
   that. Services publish their own ports in the compose stack, and any client that can reach the
   broker can publish a command onto a service's exchange.
4. ✅ **The order-creation saga acts with no caller identity at all,** processing every message with an
   empty context. Under obligation 1 this becomes a correctness problem, not just a security one: a
   guard that denies unauthenticated callers would deny the saga too, so the saga needs an explicit
   service identity before the fix lands. This ordering is the reason obligation 1 is not a one-line
   change.
5. ✅ **The signing key the edge trusts is committed in source control** in all four gateway
   configuration files. Carried as blocker B2 and shared with `ADR-004`.

### 4.3 Neutral / follow-on

1. ✅ Certificate-based service-to-service authentication is provisioned platform-wide but enabled in
   exactly one service, `customers-service`, whose access list grants one read permission to
   `availability-service` and nothing else. It is therefore not a second enforcement layer today.
   Decided in `ADR-CANDIDATE-016`.
2. ✅ Administrator authorization is a role claim checked at the edge on five routes and re-derived in
   services from the same claim carried in the caller context.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/security/edge-enforced-authentication-with-identity-binding.md` (Status:
   `Candidate`, evidence Strong). Obligations 1–4 in §2 are that pattern's four stated conditions
   adopted as binding rather than advisory.
2. **Instantiates** `patterns/security/transport-agnostic-caller-context.md`, which is the mechanism
   that lets one ownership check serve both HTTP and broker entry paths — and also the mechanism that
   supplies the empty context the defective guard mishandles.
3. **Depends on** `patterns/integration/declarative-configuration-driven-api-gateway.md`: every rule in
   the first half of §1 lives in that configuration and nowhere else.
4. **Pattern Drift:** none *yet*, because every pattern in the catalog is `Candidate`
   (`patterns/index.md`, *Governance*). This is the clearest case in the corpus of an inconsistency
   that becomes drift the moment the security pattern is approved: approving it as implemented would
   make the fail-open guard the platform standard. It should be approved describing the corrected
   behaviour, with obligation 1 as a precondition.
5. **Pattern Update Proposal:** the pattern's *Anti-patterns* section should name the exact guard shape
   found here — an ownership check conditioned on the caller being authenticated — because it reads as
   correct at a glance and has already been copied four times.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | The edge declares authentication as a per-route flag with a configured role claim type, not as a global gate | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:1-5` |
| E2 | Thirty-seven routes require authentication in the synchronous configuration | ✅ | `…/ntrada.yml` — 37 occurrences of the per-route authentication flag |
| E3 | Five routes are additionally gated on an administrator role claim | ✅ | `…/ntrada.yml:137-138, 151-152, 159-160, 180-181, 245-246` |
| E4 | Ten routes bind the caller's identity claim into the forwarded URL, query string or payload (thirteen in the asynchronous configuration) | ✅ | `…/ntrada.yml:103, 143, 167, 298, 316, 330, 348, 362, 386, 406`; `…/ntrada-async.yml` — 13 occurrences |
| E5 | Sign-up and sign-in are explicitly unauthenticated, and user administration is admin-gated | ✅ | `…/ntrada.yml:237-276` |
| E6 | The caller context is built from the broker correlation context when present, and from an HTTP correlation header otherwise, returning an empty context when neither exists | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Contexts/AppContextFactory.cs:19-33` |
| E7 | An empty caller context reports `IsAuthenticated == false`, an empty identifier and no role | ✅ | `…/Infrastructure/Contexts/IdentityContext.cs:9-17, 33` |
| E8 | The ownership guard denies only when the caller *is* authenticated, so an unauthenticated caller skips it — `availability-service` | ✅ | `…/Availability.Application/Commands/Handlers/ReserveResourceHandler.cs:29-33` |
| E9 | The identical guard shape appears in three `orders-service` handlers | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Commands/Handlers/AddParcelToOrderHandler.cs:54-57`; `…/ApproveOrderHandler.cs:35-38`; `…/AssignVehicleToOrderHandler.cs:39-42` |
| E10 | `CreateOrderHandler` performs no identity check at all | ✅ | `…/Orders.Application/Commands/Handlers/CreateOrderHandler.cs:29-40` |
| E11 | The order-creation saga processes every command and event with an empty saga context | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Handlers/AIOrderMakingHandler.cs:26-41` |
| E12 | Services publish their own container ports in the platform compose stack, so the edge is avoidable inside the network | ✅ | `hianshul100_Pacco/compose/services.yml:10, 19, 28, 37, 46, 55, 73, 82, 91, 100, 109` |
| E13 | Services accept commands directly from their own exchange, so a broker client can invoke a handler without passing the edge | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Extensions.cs:106-112` |
| E14 | A symmetric token-signing key is committed in the gateway configuration (value not reproduced here) | ✅ | `…/ntrada.yml:43-48` |
| E15 | Certificate-based service authentication is enabled in exactly one service, granting one read permission to one caller | ✅ | `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-182` |
| E16 | Services validate tokens against a certificate on disk rather than the edge's symmetric key | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:79-87` |

### 6.1 Documentation-versus-code conflicts

1. ✅ **"The platform authenticates every request" is not supportable from source.** The edge
   authenticates every request that reaches it, but nothing makes the edge unavoidable, and the
   in-service guard passes when the caller is unauthenticated. The code is followed: this record states
   the boundary as *placed at the edge and currently bypassable*, not as enforced end to end.
2. ✅ **"The platform uses mutual TLS between services" would be false.** Certificate checking is
   provisioned platform-wide by the secret store's PKI role but enabled in one service only, and no
   TLS termination is configured anywhere in the workspace.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The gateway library enforces the per-route authentication flag and the claim gate as their names read | The library is a NuGet reference with no source in this workspace, so only the configuration vocabulary is visible | The one half of this design that currently works might not, which would make the platform unauthenticated rather than merely bypassable behind the edge | Call an authentication-flagged route without a token, and an admin-gated route with a non-admin token, in a running environment and record the responses |
| A2 | The identity binding replaces any caller-supplied value rather than merely adding one | The binding entries name the same parameter the route would otherwise take from the request, and the pattern catalog reaches the same reading | Impersonation would be possible on routes this record calls safe, which would remove the platform's strongest control | Send a request to a bound route with a conflicting customer identifier in the body and observe which value the service acts on |
| A3 | No network policy prevents a caller inside the deployment network from reaching a service port or the broker directly | The compose stack publishes every service port, and no firewall, network policy or mesh configuration exists in any repository | Consequence 3 in §4.2 would be overstated and obligation 3 less urgent — though the guard defect would still stand on its own | Ask whoever operates the platform whether the service ports and the broker are reachable from outside the host, and confirm on a running environment |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so this record — which describes a live authorization defect — has no one who can accept it or schedule the fix | Obligation 1 in §2 and this ADR leaving `Proposed`. A defect nobody owns does not get fixed | Platform owner | Name a security owner for the platform and an owner for `orders-service` and `availability-service`, and record them in this repository | TBD |
| B2 | **[ACTION NOW]** The token-signing key the edge trusts is committed in the repository, in all four gateway configuration files. Anyone holding it can mint a token the gateway accepts, including an administrator one | Any claim that edge authentication is a control rather than a formality; also blocks `ADR-CANDIDATE-007` and `ADR-CANDIDATE-016` | Platform security owner | Confirm whether the committed value matches anything running. If it does, rotate first, then load the key from the secret store the platform already operates | TBD |
| B3 | **[ACTION NOW]** The order-creation saga acts with no caller identity. Fixing the guard as obligation 1 requires would deny the saga's own commands, so someone must decide what identity the saga acts as before the fix ships | Obligation 1 cannot be applied safely to `orders-service` until the saga has an identity, or the fix breaks order creation | Owner of `Pacco.Services.OrderMaker` (unassigned — see B1) | Decide whether the saga carries the originating customer's identity forward or acts as a named service principal, then apply the guard fix and the saga change together | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should this ADR ratify the current boundary, or should it require per-service token validation as well? | The whole design rests on the edge being unavoidable, and it is not. Ratifying as-is accepts that any caller inside the network is unauthenticated and, today, unauthorized-but-permitted | Ratify the placement and fix the guard first (obligation 1), then decide separately whether to add per-service validation. Do not do both at once — the guard fix is small, urgent, and independent | Platform security owner |
| Q2 | **[ACTION NOW]** Should the remaining twenty-seven authenticated routes bind the caller identity from the token? | Ten of thirty-seven bind it today. On the rest, a service acts on a customer identifier taken from the request, and the only thing preventing impersonation is the handler guard that currently fails open | Yes — bind on every route that names a user identifier. It is a configuration change per route and it makes the guard defect far less exploitable even before the code fix lands | Platform security owner |
| Q3 | **[ACTION NOW]** Is a caller inside the deployment network trusted? | Every service port is published and the broker accepts publications, so "inside the network" is currently equivalent to "authorized". Nobody has stated whether that is intended | State plainly that it is not trusted, then stop publishing service ports outside the host and require credentials on the broker. Otherwise the edge is documentation, not a boundary | Platform owner |
| Q4 | **[handled later by adr_generation]** Should the split token trust root be converged as part of fixing this boundary? | The edge validates with a symmetric key and services validate against a certificate, so a token accepted by one is not automatically accepted by the other. If per-service validation is ever added (Q1), this must be settled first | Record it separately in `ADR-CANDIDATE-007` and converge on one asymmetric key pair with a published verification key, so the same token means the same thing everywhere | `adr_generation` stage, in `ADR-CANDIDATE-007` |
