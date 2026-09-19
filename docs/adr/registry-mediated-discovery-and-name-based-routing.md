# ADR-015: Registry-mediated service discovery and name-based routing

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-09 |
| **ADR id** | `ADR-015` |
| **Backlog candidate** | `ADR-CANDIDATE-015` (`docs/architecture-inventory/adr-candidates.md`) |
| **Category / Impact** | Networking / medium |
| **Supersedes / Superseded by** | — |
| **Deciders** | Unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-002` (the toolkit that supplies the registration and routing clients), `ADR-001` (why east-west traffic is mostly messaging, not routing), `ADR-004` (the edge whose route table addresses services directly), `ADR-CANDIDATE-016` (the secret store that shares the registry's storage), `ADR-CANDIDATE-017` (the deployment model this discovery layer substitutes for) |

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

`ADR-001` made messaging the platform's primary integration mechanism, and `ADR-002` made the toolkit
the supplier of every cross-cutting capability. A residual question remains: when one service *does*
call another over HTTP, or when the edge forwards a request, how is the target's address found?

1. ✅ **Every one of the ten deployables self-registers with a discovery registry at startup**, giving
   its own logical service name, its address, its port, a health-ping endpoint, a ping interval and a
   deregistration interval
   (`hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:7-17`, and the
   equivalent section in the other nine).
2. ✅ **Every deployable also declares a registry-driven router** under its own service name
   (`…Orders.Api/appsettings.json:18-22`), and nine of the ten set their HTTP client to route through
   that router rather than to call hosts directly (`…Orders.Api/appsettings.json:23-35`).
3. ✅ **Callers address each other by logical name.** A caller's HTTP client configuration maps a short
   key to a registered service name — `orders-service` maps three keys to `parcels-service`,
   `pricing-service` and `vehicles-service` — and the typed client classes use the short key rather than
   a host.
4. ✅ **`ordermaker-service` is the one deployable that opts out of routing while still registering.**
   It registers itself and declares the router, but leaves its HTTP client type empty, so its two
   outbound calls do not go through the router
   (`hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json:7-34`).
5. ✅ **The gateway does not use the registry at all.** Its router integration is disabled in all four
   of its configurations
   (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:19-21`), and it resolves each of nine
   downstream services from a static per-service entry giving a local address and a container address,
   choosing between them with a single flag (`ntrada.yml:18`, `ntrada.yml:123-126`).
6. ✅ **The registry and the router run as single containers alongside the secret store**
   (`hianshul100_Pacco/compose/infrastructure.yml:4-21`, `compose/consul-fabio-vault.yml:4-6`), with the
   router configured to read its routing table from the registry.

So the registry is provisioned platform-wide and used for east-west traffic only, by nine of ten
deployables, over a synchronous surface that `ADR-CANDIDATE-010` describes as eight point-reads. That
gap between provisioning and use is what makes this a decision worth recording rather than a
configuration note.

### 1.1 What this record does *not* cover

This record fixes how a caller finds a callee and what the registry is responsible for. It does not fix
which calls are allowed to be synchronous at all (`ADR-CANDIDATE-010`), the deployment model that runs
these components (`ADR-CANDIDATE-017`), the secret store that shares the registry's storage
(`ADR-CANDIDATE-016`), or the gateway's route table (`ADR-004`).

## 2. Decision

**Pacco discovers services through a registry that every deployable self-registers with at startup, and
addresses them by logical service name through a registry-driven router. The registry is responsible for
east-west traffic only: the north-south edge resolves its downstream services from static configuration
and does not consult the registry.**

Four rules follow from the decision and are part of it:

1. ✅ **A new deployable registers itself** with its logical name, its port and a health-ping endpoint,
   matching the ten that already do.
2. ✅ **A caller names its callee by logical service name**, never by host and port. Hosts appear in
   configuration only as the registry's and router's own addresses.
3. 🎯 **The router is the only east-west HTTP path.** One deployable bypasses it today; that is an
   exception to be closed, not a precedent.
4. 🎯 **The registry's health checks must be acted on.** Registration declares a ping endpoint and a
   deregistration interval, but nothing in the workspace consumes the resulting health state — no alert,
   no dashboard, no gate.

## 3. Alternatives Considered

| # | Alternative | Why it was rejected |
| --- | --- | --- |
| 1 | **DNS-based discovery** — address each service by a stable DNS name resolved by the platform. | Rejected because the platform has no container orchestration and therefore no platform-supplied DNS (`ADR-CANDIDATE-017`). The Compose network provides container-name resolution, but only inside one Compose project on one host, and the process-manager deployment path has no equivalent at all. A registry works identically under both paths, which is the property that decided it. |
| 2 | **Static configuration** — each caller holds the host and port of each callee. | Rejected because it makes every address change a change in every caller, and because it cannot express health: a statically-addressed service that is down is indistinguishable from one that is up. It is nonetheless what the gateway actually does for its nine downstream services, which is the inconsistency §4.2 item 1 records. |
| 3 | **A service mesh** with sidecar proxies handling discovery, routing, retries and mutual TLS. | Rejected because it presupposes an orchestrator to inject sidecars into, and because the operational cost is large relative to a platform whose synchronous surface is eight point-reads. It would have subsumed this decision and part of `ADR-CANDIDATE-016`; revisit it only if the platform adopts orchestration. |
| 4 | **No discovery layer at all** — route everything through messaging and remove the eight synchronous edges. | Rejected because four of the eight point-reads gate a write on data the caller does not own, and replacing them with replication would widen the staleness problem `ADR-CANDIDATE-009` already records for customer data. It remains the design that would delete the most infrastructure, which is why it is recorded rather than assumed away. |

## 4. Consequences

### 4.1 Positive

1. ✅ No caller holds a callee's host or port. Adding an instance or moving a service changes nothing in
   any caller's configuration.
2. ✅ Registration is uniform across all ten deployables and is one configuration section plus one
   toolkit registration, so a new service inherits it by copying the composition root.
3. ✅ Health pings and a deregistration interval are declared everywhere, so the registry has the
   information it needs to remove a dead instance from routing.
4. ✅ The same mechanism works under both deployment paths — Compose containers and bare processes under
   a process manager — because it does not depend on the platform providing name resolution.

### 4.2 Negative

1. ✅ **The registry mediates east-west traffic only.** The gateway's router integration is disabled in
   all four configurations and it resolves nine services from static entries instead, so the platform's
   busiest HTTP path is exactly the one the discovery layer does not serve. Health-based removal
   therefore does not protect any north-south request.
2. ✅ **One deployable registers but does not route.** `ordermaker-service` leaves its HTTP client type
   empty while still declaring the router, so its two outbound calls resolve some other way.
   `[INFERRED]` they fall back to the toolkit's direct-address behaviour, which is not observable in
   this workspace.
3. ✅ **Two more single-instance infrastructure components to run.** Both the registry and the router are
   single containers with no replication and, in the platform's own Compose stack, with persistent
   storage commented out. `[INFERRED]` a registry restart empties the routing table until every service
   re-registers.
4. ✅ **The routing layer has no test coverage.** No test in any of the thirteen repositories exercises
   name-based resolution; the typed service clients are tested, where they are tested at all, against
   direct addresses.
5. ✅ **Health state is collected and never used.** Rule 4 in §2 is a target precisely because the ping
   configuration exists everywhere and nothing consumes its result.
6. ✅ **The committed registry address is a developer-machine alias.** Seven services register with a
   Windows Docker Desktop host alias as their advertised address, which resolves on one kind of
   developer machine and nowhere else.

### 4.3 Neutral / follow-on

1. ✅ Because integration is primarily asynchronous (`ADR-001`), the discovery layer carries a small
   fraction of the platform's inter-service traffic — eight point-reads against roughly 87 message
   names. Its blast radius if it fails is correspondingly narrow.
2. ✅ The registry shares a Compose file with the secret store (`ADR-CANDIDATE-016`), so in the
   platform's own environment definitions the two components' availability is coupled even though the
   decisions are independent.
3. ✅ `pricing-service` routes through the router for its one call to `customers-service`, so it
   participates in discovery even though it participates in no messaging (`ADR-001`).

## 5. Relationship to the implementation pattern catalog

1. **Instantiates** `patterns/deployment/registry-mediated-discovery-and-routing.md` (Status:
   `Candidate`, evidence Strong). Rules 1 and 2 in §2 are that pattern adopted as binding; rules 3 and 4
   are the two conditions this record adds before it should be approved.
2. **Constrains** `patterns/integration/narrow-synchronous-point-read.md`: every point-read that pattern
   describes resolves its target through the router, so the discovery decision is a precondition of that
   pattern rather than an independent concern.
3. **Constrains** `patterns/deployment/composable-per-concern-environment-stacks.md`: the registry and
   the router are one of the per-concern stacks, and their single-instance, storage-disabled shape is
   inherited from that pattern rather than chosen here.
4. **Deliberately diverges** in one place — the gateway does not use the registry. The divergence is
   real, is not justified anywhere in the source, and is bounded by §2 making the registry's scope
   explicitly east-west rather than pretending it is platform-wide.
5. **Pattern Drift:** none. Drift requires an `Approved` pattern to violate, and every pattern in the
   catalog is `Candidate` (`patterns/index.md`, *Governance*). The catalog already lists the disabled
   gateway routing as one of its three documentation-versus-code conflicts; on approval it becomes drift
   with `ordermaker-service`'s bypass alongside it.
6. **Pattern Update Proposal:** amend
   `patterns/deployment/registry-mediated-discovery-and-routing.md` to say explicitly that the pattern
   covers east-west traffic and that the north-south edge resolves statically. The pattern currently
   reads as platform-wide, which is what made the gap between provisioning and use easy to miss.

## 6. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | A service registers with the discovery registry under its own logical name, with address, port, health-ping endpoint, ping interval and deregistration interval | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/appsettings.json:7-17` |
| E2 | All ten deployables carry the same registration section under their own service name | ✅ | The `consul` section of each `…Api/appsettings.json` in Availability, Customers, Deliveries, Identity, Operations, OrderMaker, Orders, Parcels, Pricing and Vehicles |
| E3 | Every deployable declares the registry-driven router under its own service name | ✅ | The `fabio` section of the same ten settings files, e.g. `…Orders.Api/appsettings.json:18-22` |
| E4 | Nine of ten deployables set their HTTP client to route through the router; callees are named by logical service name | ✅ | `…Orders.Api/appsettings.json:23-35` (three logical service keys); the same client type in Availability, Customers, Deliveries, Identity, Operations, Parcels, Pricing and Vehicles |
| E5 | `ordermaker-service` registers and declares the router but leaves its HTTP client type empty, so its two outbound calls bypass the router | ✅ | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json:7-34` |
| E6 | Registration and routing are toolkit registrations composed per service, not hand-written code | ✅ | `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Extensions.cs:70-71` |
| E7 | The gateway's router integration is disabled in all four of its configurations | ✅ | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:19-21`, and the same in `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` |
| E8 | The gateway resolves each downstream service from a static per-service entry carrying a local address and a container address, chosen by one flag | ✅ | `ntrada.yml:18` and `ntrada.yml:123-126`; nine such entries in each of the four configurations |
| E9 | The registry and the router run as single containers, with the router reading its routing table from the registry and persistent storage commented out | ✅ | `hianshul100_Pacco/compose/infrastructure.yml:4-21` and `:134`; `hianshul100_Pacco/compose/consul-fabio-vault.yml:4-6` |
| E10 | Seven services advertise a Windows Docker Desktop host alias as their registered address | ✅ | The registration `address` value in the settings files of Availability, Customers, Deliveries, Identity, Operations, Orders and Parcels |
| E11 | No test in any repository exercises name-based routing or registry resolution | ✅ | Workspace-wide search of all test projects — no test references the router, the registry, or a logical service key |

### 6.1 Documentation-versus-code conflicts

1. `docs/architecture-inventory/patterns/deployment/registry-mediated-discovery-and-routing.md` records
   the disabled gateway load balancing as a documentation-versus-code conflict. The code confirms it in
   all four configurations, and adds what the gateway does *instead* — static per-service entries with a
   local and a container address (E8). That mechanism is what makes §4.2 item 1 a design property rather
   than a misconfiguration, and it is stated in §1 item 5.
2. `docs/architecture-inventory/baselines/architecture-baseline.md` §4.1 records `ordermaker-service` as
   the one caller not routed through the router. The code agrees, and shows the mechanism: an empty HTTP
   client type alongside a router declaration that is therefore inert (E5).

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | An empty HTTP client type makes the toolkit call the configured address directly rather than through the router | It is the only deployable with an empty type, its two calls demonstrably work in the platform's own environment definitions, and the toolkit is a package reference with no source in this workspace | `ordermaker-service`'s calls might fail entirely, or might route in some third way, and §4.2 item 2 would describe the wrong behaviour | Start `ordermaker-service` against a running registry, trigger one of its two outbound calls, and record the target address from the request log |
| A2 | The registry holds no state across a restart, because persistent storage is commented out in the platform's own environment definitions | The volume mount for the registry is present but commented out in both stacks that define it, and the container carries no other storage | A registry restart might preserve the routing table, and §4.2 item 3 would overstate the impact | Restart the registry container in a running environment and check whether registrations survive before services re-register |
| A3 | Nothing consumes the health state the registry collects | No alert rule, dashboard definition, or gate anywhere in the fourteen repositories references registry health; the metrics scrape configuration targets the services directly | A health-driven behaviour might exist outside the repositories, and rule 4 in §2 would be asking for something that already works | Ask whoever operates the platform whether registry health drives anything — paging, routing decisions, or deployment gates |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so nobody can approve this record or own the registry and router as shared components | This ADR leaving `Proposed`, and rules 3 and 4 in §2, which both need someone accountable across repositories | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | TBD |
| B2 | **[ACTION NOW]** The registered addresses in the committed settings are developer-machine aliases, so nobody can say what these services advertise in a real environment. Until that is known, it cannot be said whether discovery works outside a developer laptop at all | Any claim that this decision describes a deployed platform, and the operational consequences in §4.2 items 3 and 6 | Platform owner | Confirm how the advertised address is supplied in each environment — an override, an environment variable, or the committed value — and record it | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Should the gateway route through the registry, or should the registry's scope stay east-west? | Today the platform runs a discovery layer that does not serve its busiest HTTP path, and pays for two components to serve eight point-reads. Either the gateway should use it or the platform should know it is paying for less than it thinks | Enable the gateway's router integration, so a failed instance is removed from north-south routing too. If that is not wanted, say so in this record and drop the router from the gateway's environment entirely rather than leaving it configured and off | Platform owner |
| Q2 | **[ACTION NOW]** Should `ordermaker-service` be brought onto the router, or is its bypass deliberate? | It is the single exception to rule 2 in §2. Left unrecorded it becomes the precedent that dissolves the rule, and it is also the service that acts with no caller identity (`ADR-006`) | Set its HTTP client type to match the other nine. Nothing in its code needs to change — it already names its two callees by logical service name | Owner of `Pacco.Services.OrderMaker` (unassigned — see B1) |
| Q3 | **[ACTION NOW]** What is supposed to happen when the registry is unavailable? | Nine services resolve every synchronous call through it, and it runs as a single instance with storage disabled. No caller in the workspace has a fallback address or a documented degraded mode | Decide explicitly: either accept that the eight point-reads fail during a registry outage — which is survivable because they gate writes rather than reads — or give the router a static fallback table. Write the answer into the operational runbook the platform does not yet have | Platform owner |
| Q4 | **[handled later by adr_generation]** Should discovery survive if the platform adopts container orchestration? | An orchestrator supplies name resolution and health-based removal itself, which would make the registry redundant for everything except the router's routing table | Revisit this record when `ADR-CANDIDATE-017` decides the deployment model. If orchestration is adopted, alternatives 1 and 3 in §3 both become cheap and this decision should be superseded rather than amended | `adr_generation` stage, in `ADR-CANDIDATE-017` |
