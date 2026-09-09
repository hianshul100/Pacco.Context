# ADR-010: Narrow synchronous point-reads as the bounded exception to messaging

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-010` |
| **Backlog candidate** | `ADR-CANDIDATE-010` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Integration / medium |
| **Supersedes / Superseded by** | — (nothing to supersede) |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-001` (the asynchronous default this is the exception to), `ADR-008` (the ownership rule that forces one of these two paths), `ADR-015` (the name-based router these calls use); paired with `ADR-009`, which records the *other* answer to the same question |

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

`ADR-001` makes messaging the default integration transport and `ADR-008` forbids reaching into another
service's database. Service-to-service HTTP nevertheless exists, and it exists in a recognisable,
deliberately small shape. This record fixes the rule that shape implies, and — importantly — records
the two places where the code already breaks it.

1. ✅ **Seven synchronous edges exist, and no more.** Every one is declared twice: as a logical service
   key in the caller's `httpClient` service map, and as a typed client class in the caller's
   infrastructure. The complete set is `availability-service` → `customers-service`;
   `pricing-service` → `customers-service`; `orders-service` → `parcels-service`, `pricing-service`
   and `vehicles-service`; `ordermaker-service` → `availability-service` and `vehicles-service`
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:23-35` and the
   corresponding sections in the other three callers).
2. ✅ **Every edge is a read.** No typed client on the platform issues a POST, PUT, PATCH or DELETE.
   All seven client methods call the toolkit's GET helper and deserialise a DTO.
3. ✅ **Five of the seven are single-entity point-reads by id.** `ParcelsServiceClient.GetAsync`,
   `VehiclesServiceClient.GetAsync`, `CustomersServiceClient.GetAsync`,
   `CustomersServiceClient.GetStateAsync` and `AvailabilityServiceClient.GetResourceReservationsAsync`
   each fetch one resource by identifier
   (`…Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs:20-21`).
4. ✅ **Two are not point-reads, and the backlog's rule does not survive contact with them.**
   `PricingServiceClient.GetOrderPriceAsync` is a *computation* invoked over HTTP with query-string
   arguments, not a fetch of a stored entity
   (`…Orders.Infrastructure/Services/Clients/PricingServiceClient.cs:20-21`).
   `VehiclesServiceClient.GetBestAsync` fetches a **paged collection of every vehicle** and selects the
   first item client-side, throwing if the page is empty
   (`hianshul100_Pacco.Services.OrderMaker/src/…/Services/Clients/VehiclesServiceClient.cs:21-31`).
   That is an unbounded list read, and the selection logic that gives it meaning lives in the caller.
5. ✅ **Every edge gates a write.** Each client is injected into a command handler that calls it before
   deciding, and none serves a read-only query path except the pricing computation, which is itself
   reached from a query handler in `pricing-service`.
6. ✅ **There is no synchronous composition and no synchronous fan-out.** No handler calls two services
   in sequence and combines the results, and no service calls a service that calls a third.
7. ✅ **Six of the seven edges resolve through the name-based router; one does not.** The three layered
   callers set their HTTP client type to the router, while `ordermaker-service` leaves the type empty
   and so addresses its two dependencies directly by name
   (`hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json:23-34`).
8. ✅ **Only one caller presents a credential.** `availability-service`'s customers client attaches its
   Vault-issued certificate to the request when PKI is enabled; the other six callers attach nothing
   (`…Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs:21-34`).

### 1.1 What this record does *not* cover

It does not cover replication, which is `ADR-009`; the two are written to be read together. It does not
cover the north-south edge (`ADR-004`), the router itself (`ADR-015`), or the certificate material the
one authenticated call uses (`ADR-016`).

## 2. Decision

**Service-to-service HTTP is permitted only as a read that gates a write, and only in one of two
shapes: a fetch of a single resource by its identifier, or a call to a service whose entire purpose is
to compute an answer from arguments. A synchronous call must never write, must never return an
unbounded collection, must never be chained through a third service, and must resolve through the
name-based router. Anything else is asynchronous and belongs on the owning service's exchange.**

The second permitted shape is added because the code requires it: `pricing-service` exists to compute,
and calling it is not a data read at all. Four obligations follow:

1. 🎯 **`ordermaker-service`'s vehicle selection must move off the list read.** Fetching every vehicle
   and taking the first is the one call in the platform this rule forbids. Either `vehicles-service`
   exposes a selection endpoint, or the choice becomes an asynchronous request.
2. 🎯 **Every synchronous caller must route through the name-based router.**
   `ordermaker-service` is the one exception and has no stated reason to be.
3. 🎯 **Every synchronous call must present a service credential.** One of seven does today.
4. 🎯 **A caller must state what it does when the callee is unavailable.** No client declares a
   timeout, a circuit breaker or a fallback; the only resilience anywhere is a retry count in
   configuration.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **Forbid synchronous service-to-service calls entirely**, forcing every cross-service fact into a replica (`ADR-009`). | Rejected because the data these calls fetch is either large and rarely needed (a full parcel or vehicle) or genuinely computed rather than stored (order pricing). Replicating all of it would multiply the reconciliation duty that `ADR-009` already struggles to discharge for a single identifier, and a computation cannot be replicated at all. |
| 2 | **Allow synchronous calls generally**, with no rule about shape. | Rejected because it is the road to a synchronous call graph with cascading failure and shared latency. With no timeouts, no circuit breakers and no bulkheads anywhere in the workspace (§4.2), a general permission would turn any slow service into a platform-wide outage. The narrowness *is* the control. |
| 3 | **A read-model or API-composition service** that aggregates cross-service reads on behalf of callers. | Rejected because it creates a component every write path depends on, owned by nobody, and it would need its own copy of every contract — an especially poor bet given `ADR-003` provides no build-time contract signal. It also solves a fan-out problem the platform does not have: no handler needs more than one other service. |
| 4 | **gRPC instead of HTTP** for these edges, reusing the streaming surface `operations-service` already exposes. | Rejected in practice, though the platform demonstrably can do it. It would have given typed, generated contracts — the one thing `ADR-003` says is missing — but at the cost of a second east-west transport alongside HTTP and the router. It remains the most defensible future change and is carried as question Q3. |

## 4. Consequences

### 4.1 Positive

1. ✅ The synchronous surface is small enough to hold in the head: seven edges, seven methods, all
   reads, all documented in two places per edge.
2. ✅ Because no call writes, a failed synchronous call can be retried freely — the toolkit's retry
   count is safe by construction.
3. ✅ No cycle exists in the synchronous call graph, so no distributed deadlock or recursive timeout is
   possible `[INFERRED]` from the seven edges enumerated in §1.
4. ✅ Callers depend on a domain-shaped interface declared in their own application layer, so the HTTP
   client is swappable and unit-testable — `availability-service` does exactly that in its handler
   tests.

### 4.2 Negative

1. ✅ **Every synchronous edge puts the callee's availability on the caller's write path.** Reserving a
   resource fails if `customers-service` is down; creating an order priced through
   `pricing-service` fails if pricing is down.
2. ✅ **No client declares a timeout, circuit breaker, bulkhead or fallback.** The only resilience
   setting present is a retry count, which without a timeout can extend a slow call rather than bound
   it.
3. ✅ **`ordermaker-service`'s vehicle call scales with the fleet.** It requests a paged collection and
   discards all but the first item, so the cost of choosing a vehicle grows with the number of
   vehicles, and the selection rule is invisible to `vehicles-service`.
4. ✅ **Six of seven calls are unauthenticated.** The one enforcement point on the platform is
   `customers-service`'s certificate access list, and `pricing-service` calls it without a certificate
   and does not appear on that list (`ADR-016`, question Q3 there).
5. ✅ **The rule is enforced by nothing.** There is no lint, no architecture test and no review gate
   that would stop the eighth edge from being a synchronous write. Until §2 has an owner it is a
   description, not a constraint.

### 4.3 Neutral / follow-on

1. ✅ The two shapes in §2 are genuinely different decisions wearing the same clothes. A point-read is
   a data access choice constrained by `ADR-008`; a call to `pricing-service` is a service invocation
   that has no asynchronous alternative because the caller needs the answer to proceed.
2. ✅ `customers-service` is read by two callers over two different endpoints — a full customer for
   pricing and a state projection for availability — which is evidence the platform already prefers
   narrow, purpose-shaped endpoints over one general read.

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/integration/narrow-synchronous-point-read.md` (Status: `Candidate`,
   evidence Strong), and **constrains** it by adding the four obligations in §2, which the pattern does
   not state.
2. **Deliberately diverges** from that pattern in one place, with rationale: the pattern's name and
   body describe single-resource reads, but `PricingServiceClient` and `VehiclesServiceClient.GetBestAsync`
   are not that (§1 item 4). Rather than pretend the platform is uniform, this record admits a second
   permitted shape for computation and forbids the list read outright as obligation 1.
3. **Paired with** `patterns/data/event-carried-reference-replica.md`. Section 2 of this record and §2
   of `ADR-009` together supply the choosing rule neither pattern contains.
4. **Constrained by** `patterns/deployment/registry-mediated-discovery-and-routing.md`: six of seven
   edges resolve logical names through the router, which is why no client holds a host or port.
5. **Interacts with** `patterns/security/vault-issued-dynamic-credentials-and-service-pki.md`, whose
   certificate is attached by exactly one of the seven clients.
6. **Pattern Drift:** none reportable. Drift requires an `Approved` pattern to violate, and every entry
   in `patterns/index.md` carries status `Candidate` (*Governance*).
7. **Pattern Update Proposal:** the pattern should be renamed or widened to cover computation calls,
   should state explicitly that a collection read is out of scope, and should add the resilience
   requirement from §2 obligation 4 — a synchronous edge with no timeout is the pattern's largest
   unstated cost.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | `orders-service` declares three synchronous dependencies as logical service keys | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:23-35` |
| E2 | `availability-service` declares one, and sets its client type to the name-based router | ✅ | `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/appsettings.json:23-33` |
| E3 | `pricing-service` declares one, the same way | ✅ | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/appsettings.json:23-33` |
| E4 | `ordermaker-service` declares two and leaves the client type empty, bypassing the router | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json:23-34` |
| E5 | The remaining five deployables declare an empty service map — no synchronous dependency | ✅ | `httpClient.services` in the `…Api/appsettings.json` of Customers, Deliveries, Identity, Operations, Parcels and Vehicles |
| E6 | A point-read client fetches one resource by id and nothing else | ✅ | `…Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs:20-21`; `…/VehiclesServiceClient.cs:20-21` |
| E7 | `availability-service` reads a purpose-shaped customer state projection rather than the whole customer | ✅ | `…Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs:37-38` |
| E8 | `pricing-service` is called as a computation with query-string arguments, not as an entity read | ✅ | `…Orders.Infrastructure/Services/Clients/PricingServiceClient.cs:20-21` |
| E9 | `ordermaker-service` fetches a paged collection of all vehicles and selects the first client-side | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs:21-31` |
| E10 | Exactly one of the seven clients attaches a service certificate, and only when PKI is enabled | ✅ | `…Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs:21-34` |
| E11 | No typed client anywhere issues a write verb | ✅ | All seven client classes under `…/Services/Clients/` — every method calls the toolkit GET helper |
| E12 | Synchronous reads gate writes: the reserve-resource handler consumes the customers client | ✅ | `…Availability.Application/Commands/Handlers/ReserveResourceHandler.cs` |
| E13 | No timeout, circuit-breaker or fallback configuration exists for any client | ✅ | `httpClient` sections declare a type, a retry count, a service map and request masking only (E1–E4) |

### 6.1 Documentation-versus-code conflicts

1. ✅ **The edge count is seven, not eight.**
   `docs/architecture-inventory/adr-candidates.md` (candidate 010) and
   `docs/architecture-inventory/repo-inventory.md` §3.1 both state "eight synchronous edges", while
   enumerating seven: Availability→Customers, Pricing→Customers, Orders→Parcels, Orders→Pricing,
   Orders→Vehicles, OrderMaker→Availability, OrderMaker→Vehicles. Counting the service keys in the four
   callers' configuration gives 1 + 1 + 3 + 2 = **seven** (E1–E4). The code is followed here.
2. ✅ **"Every one is a single-entity read" is not true.** The same sources state that every edge is a
   single-entity read gating a write. Two are not: the pricing call is a computation (E8) and the
   vehicle call is an unbounded collection read (E9). This record follows the code, admits the
   computation shape in §2, and forbids the collection read as an explicit obligation rather than
   describing it as compliant.
3. ✅ **The router is not universal.** `docs/architecture-inventory/baselines/architecture-baseline.md`
   §4.1 records `ordermaker-service` as the one caller outside the router, and the code agrees (E4).
   Recorded here as obligation 2 rather than left as a note.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The toolkit's HTTP client applies no default timeout that the configuration does not show | Only a retry count is configurable in the observed sections, and the toolkit's source is not in this workspace | §4.2 item 2 would overstate the exposure, and obligation 4 in §2 would be partly satisfied already | Read the pinned toolkit version's HTTP source, or issue a call against a deliberately stalled callee and measure when it gives up |
| A2 | The seven edges are the complete synchronous surface, because a call not declared in the service map cannot be made | Every observed client resolves its base address from the service map by key, and a missing key would throw at construction | An undeclared call built from a literal address would be invisible to this record, and the "no fan-out, no chaining" claims would be unproven | Search the running services' egress connections, or grep for literal `http://` addresses in handler code across all eleven deployables |
| A3 | `pricing-service` genuinely has no asynchronous alternative, justifying the second permitted shape in §2 | It persists nothing (`ADR-008` §4.3) and exists to return a computed price the caller needs before it can proceed | The second shape in §2 would be an unnecessary widening of the rule, and the pricing call should have been an obligation to fix rather than a permission | Confirm with the owner of `orders-service` whether order creation can proceed without a price and reconcile later |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so the rule in §2 has nobody to enforce it and the four obligations have nobody to accept them | This ADR leaving `Proposed`, and every obligation in §2 — a rule with no owner is a description (§4.2 item 5) | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** Nobody has confirmed whether `pricing-service`'s uncredentialed call to `customers-service` currently succeeds. `customers-service` runs the platform's only certificate access list, and `pricing-service` is not on it | Whether §4.2 item 4 is a latent gap or a live broken call path, and therefore whether obligation 3 is urgent. It is the same unknown `ADR-016` carries as its question Q3 | Platform security owner | Call `customers-service` from `pricing-service` in a running environment and record whether the certificate check rejects it | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** How should `ordermaker-service` choose a vehicle? | It currently fetches every vehicle and takes the first, so the selection rule lives in the caller, the cost grows with the fleet, and `vehicles-service` cannot optimise or index for it. This is the one call §2 forbids | Add a purpose-shaped endpoint on `vehicles-service` that returns one vehicle for a stated requirement — the same move `availability-service` already made when it read a customer *state* rather than a whole customer | Platform owner, with the owner of `vehicles-service` |
| Q2 | **[ACTION NOW]** What should a caller do when a synchronous callee is unavailable? | Every one of the seven edges is on a write path with no timeout and no fallback, so a slow callee stalls a write rather than failing it (§4.2 items 1 and 2) | Set an explicit timeout on every client and fail the command with a typed rejection, which the platform already has a contract for. Add a circuit breaker only where a fallback answer is actually meaningful | Platform owner |
| Q3 | **[handled later by the design stage]** Should east-west calls move to gRPC? | It would supply the generated, typed contracts `ADR-003` records as missing, and `operations-service` proves the platform can already host a gRPC surface. Against that, it adds a second east-west transport alongside HTTP and the router | Not yet. Resolve Q1 and Q2 first — a typed transport does not fix an unbounded read or a missing timeout. Revisit once the seven edges are stable and credentialed | Owners named per service once B1 is resolved |
| Q4 | **[handled later by the design stage]** Should the rule in §2 be enforced mechanically rather than by review? | Nothing today would stop a synchronous write or an eighth edge from being added (§4.2 item 5), and this platform has no shared library where such a check could live (`ADR-002`) | Add an architecture test to each service's own test project asserting that no typed client declares a write method. It is duplicated per repository, which `ADR-018` shows is the accepted cost of the release model | Owners named per service once B1 is resolved |
