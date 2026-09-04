# Pacco — Capability Baseline (Discovery)

**Project Key:** Common Architecture
**Stage:** `architecture_discovery` — observed current-state capability landscape.
**No ADRs, no recommendations, no target state, no KG JSON, no test inventory.**
**Date of analysis:** 2026-09-04
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`

This document is the **single authoritative capability reference** for the Pacco platform. It
replaces both a standalone capability list and a separate service-capability mapping: capability
definitions live in §1, and all service ownership and lookup information lives in §2.

**Inputs used**

- `docs/architecture-inventory/repo-inventory.md` — repository inventory, 14 dimensions per repo.
- `docs/architecture-inventory/architecture-views.md` — C1 contexts, dependency graph, runtime
  flows, deployment topology, ER views.
- `docs/architecture-inventory/baselines/service-summaries.md` — **observed** on this branch
  (commit `f66b23d`); service and bounded-context baseline.
- `docs/architecture-inventory/repo-summary/*.md` — 13 per-repo summaries.
- The thirteen cloned source repositories (read-only), which are the **source of truth** for every
  statement below. Where a prior document and the code disagree, the code wins and the
  disagreement is stated explicitly in §6.
- `.attachments/01_product_backlog_20260903_170135_37cf143b.xlsx` — backlog issue **12998**
  "Pacco - Discovery - Attempt-2", which fixes the thirteen-repository scope.
- No external capability catalog entry for Pacco, its capabilities, their maturity, or their owning
  domains was available for this baseline. Every capability below is derived from the cloned source
  and the three prior discovery documents in this repository. **Capability maturity is therefore
  `[unknown]` platform-wide** and is not asserted anywhere in this document.

**Evidence taxonomy used throughout.** *Observed fact* — read directly from a runtime manifest,
source file, or config file, path cited. *Inferred observation* — a conclusion drawn from two or
more observed facts, labelled as such. *Assumption* — stated belief beyond what evidence shows,
labelled `[assumption]` and rolled into the ABQ section. *Unknown* — labelled `[unknown]` or
**not observed** / **not evidenced**, never filled with a guess.

**Capability granularity.** A capability here is a **functional boundary that owns behaviour** —
either a business ability the platform delivers or an operational responsibility a component
carries. Framework plumbing (Convey composition roots, `Decorators/`, DTO mappers, document
mappers) is deliberately **not** promoted to a capability; it appears only as a quality signal in
§7 or as evidence of a capability it serves.

**Cross-reference convention.** Two registers are referenced in this document and they are numbered
independently:

- **`A#` / `B#` / `Q#`** always refer to the *Assumptions, Blockers & Open Questions* tables at the
  end of **this** file. Every such reference in §§1–8 resolves there and nowhere else.
- **`G#`** always refers to the **gap register in `service-summaries.md` §5** (`G1`–`G14`), which
  this file does not restate. A `G#` is cited only to point at the prior document's evidence; the
  actionable item for this baseline is always the accompanying `B#` or `Q#`. `G#` ids are written
  as `service-summaries.md G7` at first use in each section and as plain `G7` thereafter.

## Table of contents

1. [Capability-first view](#1--capability-first-view)
2. [Service ownership mapping](#2--service-ownership-mapping)
3. [Requirement / feature traceability](#3--requirement--feature-traceability)
4. [Code evidence](#4--code-evidence)
5. [Confidence assessment](#5--confidence-assessment)
6. [Gaps, unknowns, and assumptions](#6--gaps-unknowns-and-assumptions)
7. [Architectural characteristics](#7--architectural-characteristics)
8. [Behavioral constraints](#8--behavioral-constraints)
9. [Service Lookup Index](#service-lookup-index)
10. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

**Capability index.** Sixteen capabilities are evidenced: eleven business/domain capabilities
(CAP-01 … CAP-11) and five platform/technical capabilities (CAP-12 … CAP-16).

| ID | Capability | Primary owner |
|---|---|---|
| CAP-01 | Identity & Access Management | `identity-service` |
| CAP-02 | Edge Routing & Access Enforcement | `api-gateway` |
| CAP-03 | Customer Profile & Lifecycle Management | `customers-service` |
| CAP-04 | Resource Availability & Reservation | `availability-service` |
| CAP-05 | Vehicle Fleet Catalogue | `vehicles-service` |
| CAP-06 | Parcel Catalogue & Volume Calculation | `parcels-service` |
| CAP-07 | Order Lifecycle Management | `orders-service` |
| CAP-08 | Order Pricing & Discounting | `pricing-service` |
| CAP-09 | Delivery Execution & Tracking | `deliveries-service` |
| CAP-10 | Automated Order Orchestration | `ordermaker-service` |
| CAP-11 | Operation Status Projection & Real-Time Notification | `operations-service` |
| CAP-12 | Asynchronous Messaging & Event Distribution | no single owner — platform-wide |
| CAP-13 | Service Discovery & Load Balancing | `Pacco` (definition) + every service (participation) |
| CAP-14 | Platform Observability | `Pacco` (definition) + every service (participation) |
| CAP-15 | Secrets & Service-Identity Management | `Pacco` (definition) + `customers-service` (enforcement point) |
| CAP-16 | Environment & Deployment Definition | `Pacco` |

---

## 1 — Capability-first view

### Business and domain capabilities

**Capability: CAP-01 — Identity & Access Management**

- **Description:** Registers users against a unique lower-cased email, hashes and verifies
  passwords, issues signed JWTs carrying a role and optional `permissions` claims, issues and
  rotates opaque refresh tokens, and revokes both refresh tokens and issued access tokens. It is
  the platform's only issuer of credentials; every other capability consumes the resulting bearer
  token but none can mint one.
- **Purpose / business value:** **not observed.** No repository states a business driver for
  authentication; `Pacco/README.md` does not mention identity at all.
- **Confidence:** high
- **Evidence:** `Pacco.Services.Identity/src/Pacco.Services.Identity.Application/Services/Identity/IdentityService.cs`
  (email regex, `EmailInUseException`, `InvalidCredentialsException`, role defaulting to `user`);
  `.../Services/Identity/RefreshTokenService.cs`; `.../Identity.Infrastructure/Auth/JwtProvider.cs`,
  `PasswordService.cs`, `Rng.cs`; `Core/Entities/User.cs`, `RefreshToken.cs`, `Role.cs`;
  `repo-inventory.md` §2.2 row `Pacco.Services.Identity`; `service-summaries.md` §2.1.

**Capability: CAP-02 — Edge Routing & Access Enforcement**

- **Description:** The single north-south entry point. Terminates untrusted traffic, validates a
  JWT it did not issue against `validIssuer: pacco`, enforces `role: admin` claim gates on five
  routes, binds the caller's `@user_id` claim into downstream URLs and message payloads, and then
  either proxies the request over HTTP (`use: downstream`) or converts it into a RabbitMQ publish
  (`use: rabbitmq`) onto a named exchange and routing key. The sync/async choice is a whole-config
  swap selected by the `NTRADA_CONFIG` environment variable, not a per-route flag.
- **Purpose / business value:** **not observed** beyond the routing table itself.
- **Confidence:** high
- **Evidence:** `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` (40 × `use: downstream`, 5 ×
  `role: admin`), `ntrada-async.yml` (20 × `use: downstream` + 20 × `use: rabbitmq`, same 5 ×
  `role: admin`), `ntrada.docker.yml`, `ntrada-async.docker.yml`;
  `src/Pacco.APIGateway/Program.cs`; `Infrastructure/CorrelationContextBuilder.cs`,
  `SpanContextBuilder.cs`, `HttpRequestHook.cs`; `Pacco/compose/services.yml`
  (`NTRADA_CONFIG=ntrada-async.docker.yml`).

**Capability: CAP-03 — Customer Profile & Lifecycle Management**

- **Description:** Owns the customer record created from an identity sign-up, drives it through a
  five-value state machine (`Unknown`, `Valid`, `Incomplete`, `Suspicious`, `Locked`), accepts a
  one-time registration completion that supplies full name and address, accumulates the set of
  completed order ids, and applies a VIP policy off that count. It is the platform's authority on
  whether a customer may transact: `availability-service` calls it synchronously before allowing a
  reservation.
- **Purpose / business value:** **not observed.** The VIP and state concepts appear only in code;
  no repository document explains what "Suspicious" or "Locked" mean commercially.
- **Confidence:** high
- **Evidence:** `Pacco.Services.Customers/src/Pacco.Services.Customers.Core/Entities/Customer.cs`,
  `State.cs`; `Core/Services/VipPolicy.cs`, `IVipPolicy.cs`;
  `Application/Events/External/Handlers/OrderCompletedHandler.cs`, `SignedUpHandler.cs`;
  `src/Pacco.Services.Customers.Api/appsettings.json` (`mongo.database: customers-service`,
  exchange `customers`).

**Capability: CAP-04 — Resource Availability & Reservation**

- **Description:** Maintains a catalogue of tagged resources and reserves them against a **calendar
  date** with an integer priority. A resource holds its reservations inside its own aggregate; a
  new reservation whose date collides with an existing one succeeds only if it carries a strictly
  higher priority, in which case the incumbent reservation is expropriated and cancelled. This is
  the mechanism behind the platform's "limited resources" concept.
- **Purpose / business value:** `Pacco/README.md` describes Pacco as a platform for "exclusive
  parcel delivery" built around "limited resources availability" — the only documented business
  framing of any capability in this baseline. It does not describe the priority/expropriation rule.
- **Confidence:** high
- **Evidence:** `Pacco.Services.Availability/src/Pacco.Services.Availability.Core/Entities/Resource.cs`
  (`AddReservation`, `CannotExpropriateReservationException`); `Core/ValueObjects/Reservation.cs`
  (equality and hash on `DateTime.Date`); `Application/Commands/Handlers/ReserveResourceHandler.cs`;
  `Infrastructure/Services/Clients/CustomersServiceClient.cs`; `Pacco/README.md`.

**Capability: CAP-05 — Vehicle Fleet Catalogue**

- **Description:** A CRUD catalogue of vehicles carrying brand, model, description, payload and
  loading capacities, a price per service, and a flags-style `Variants` set. It is a reference data
  source: `orders-service` reads a vehicle's `PricePerService` to price an order and
  `ordermaker-service` reads the vehicle list to pick one. It performs no reservation of its own —
  a vehicle's date-level availability is CAP-04's concern.
- **Purpose / business value:** **not observed.**
- **Confidence:** high
- **Evidence:** `Pacco.Services.Vehicles/src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs`
  (`InvalidVehicleCapacity`, `InvalidVehiclePricePerServiceException`), `Variants.cs`;
  `src/Pacco.Services.Vehicles.Api/appsettings.json` (`mongo.database: vehicles-service`, exchange
  `vehicles`); `repo-inventory.md` §2.2 row `Pacco.Services.Vehicles`.

**Capability: CAP-06 — Parcel Catalogue & Volume Calculation**

- **Description:** Owns parcels as customer-scoped items with a name, description, `Size` and
  `Variant`, and computes an aggregate volume for a supplied set of parcel ids. It also tracks
  which order (if any) a parcel currently belongs to, keeping that link in step by subscribing to
  four `orders` events.
- **Purpose / business value:** **not observed.**
- **Confidence:** high
- **Evidence:** `Pacco.Services.Parcels/src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`
  (`AddToOrder`, `DeleteFromOrder`, `InvalidParcelNameException`), `Size.cs`, `Variant.cs`;
  `Infrastructure/Mongo/Queries/Handlers/GetParcelsVolumeHandler.cs`;
  `Application/Queries/GetParcelsVolume.cs`; `Core/Services/IParcelsService.cs`;
  `Application/Events/External/Handlers/` (4 `orders` event handlers).

**Capability: CAP-07 — Order Lifecycle Management**

- **Description:** The platform's hub capability. Owns the order aggregate and its five-state
  lifecycle (`New` → `Approved` → `Delivering` → `Completed`, with `Canceled` reachable from any
  non-terminal state), the set of parcels attached to an order, the assigned vehicle and delivery
  date, and the order's total price. It calls out synchronously to three capabilities to validate
  and price an order, and it advances its own state in response to reservation and delivery events
  it does not control. It carries the largest contract surface on the platform: 7 commands,
  9 events and 10 rejected events.
- **Purpose / business value:** **not observed.** No repository document describes the order
  states or what business meaning `Approved` carries.
- **Confidence:** high
- **Evidence:** `Pacco.Services.Orders/src/Pacco.Services.Orders.Core/Entities/Order.cs`,
  `OrderStatus.cs`, `Parcel.cs`, `Customer.cs`;
  `Application/Commands/Handlers/{AddParcelToOrder,AssignVehicleToOrder,ApproveOrder,CancelOrder,CreateOrder,DeleteOrder,DeleteParcelFromOrder}Handler.cs`;
  `Application/Events/External/Handlers/{ResourceReserved,ResourceReservationCanceled,DeliveryStarted,DeliveryCompleted,DeliveryFailed,ParcelDeleted,CustomerCreated}Handler.cs`;
  `Infrastructure/Services/Clients/{Parcels,Pricing,Vehicles}ServiceClient.cs`.

**Capability: CAP-08 — Order Pricing & Discounting**

- **Description:** A stateless calculation that turns a customer id and a base order price into a
  discount rate and a final price. The rate is a step function of the customer's completed-order
  count (>0 → 2%, >3 → 5%, ≥10 → 10%) with a flat +10 percentage points if the customer is VIP,
  and the final price falls back to the undiscounted price if the computed result is not positive.
  It owns no data and emits no events; it reads customer facts from CAP-03 on every call.
- **Purpose / business value:** **not observed.** The discount tiers are hard-coded constants with
  no accompanying documentation, ADR, or configuration key.
- **Confidence:** high
- **Evidence:** `Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs`;
  `Queries/Handlers/GetOrderPricingHandler.cs`; `Services/Clients/CustomersServiceClient.cs`;
  `src/Pacco.Services.Pricing.Api/appsettings.json` (no `mongo`, no `redis`, no `rabbitMq`).

**Capability: CAP-09 — Delivery Execution & Tracking**

- **Description:** Creates a delivery against an order, accumulates timestamped tracking
  registrations (scans) while the delivery is in progress, and drives it to a terminal `Completed`
  or `Failed` state carrying a failure reason. Its three events are what move an order from
  `Approved` through `Delivering` to `Completed`, so it is the capability that closes the order
  lifecycle.
- **Purpose / business value:** **not observed.**
- **Confidence:** high for the capability; **medium** for how it is initiated — see Q1 (evidence
  first recorded as `service-summaries.md` G7).
- **Evidence:** `Pacco.Services.Deliveries/src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs`,
  `DeliveryStatus.cs`; `Core/ValueObjects/DeliveryRegistration.cs`;
  `Application/Commands/Handlers/StartDeliveryHandler.cs` (`DeliveryAlreadyStartedException`);
  `src/Pacco.Services.Deliveries.Api/appsettings.json` (exchange `deliveries`).

**Capability: CAP-10 — Automated Order Orchestration**

- **Description:** A Chronicle saga that assembles a complete order from a single `MakeOrder`
  trigger: it creates the order, attaches every requested parcel, waits until all parcels are
  confirmed attached, selects a vehicle, selects a reservation date, assigns the vehicle, reserves
  the resource, and completes when the order is approved. It owns no aggregate — it issues commands
  onto four other capabilities' exchanges and is keyed entirely by `OrderId`. Its only compensation
  path cancels the order when parcel attachment fails.
- **Purpose / business value:** **not observed.** `repo-inventory.md` records the repository's own
  "uber AI order maker" framing; no document explains the business need.
- **Confidence:** medium — the choreography is unambiguous, but the service has no gateway route
  and appears in neither PM2 manifest, so whether this capability is a live path is undetermined
  (B1; evidence first recorded as `service-summaries.md` G2).
- **Evidence:** `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`,
  `Sagas/AIMakingOrderData.cs`; `Handlers/AIOrderMakingHandler.cs`;
  `Services/Clients/{Availability,Vehicles}ServiceClient.cs`; `Services/ResourceReservationsService.cs`;
  `Extensions.cs` (`AddChronicle()`); `Pacco/compose/services.yml` (port `5015`).

**Capability: CAP-11 — Operation Status Projection & Real-Time Notification**

- **Description:** Subscribes to every command, event and rejected event declared on the platform,
  keys each one by the incoming message's correlation id, projects a single `Pending` /
  `Completed` / `Rejected` operation state per correlation id, and pushes each transition to
  browsers over the SignalR hub `/pacco` and to subscribers over a server-streaming gRPC method.
  Subscription types are emitted at runtime with `System.Reflection.Emit` from bare names in a
  hand-maintained catalogue file, so the capability's contract surface is data-driven rather than
  compiled. It is read-only: it writes no domain state and issues no commands.
- **Purpose / business value:** **not observed** in any document. Functionally it is the only way a
  caller learns the outcome of a request made through the gateway's asynchronous mode, since those
  20 routes return no result.
- **Confidence:** high
- **Evidence:** `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Services/OperationsService.cs`
  (`IDistributedCache`, key `requests:{id}`, `SlidingExpiration`);
  `Handlers/Generic{Command,Event,RejectedEvent}Handler.cs`; `Infrastructure/Subscriptions.cs`;
  `messages.json`; `Hubs/PaccoHub.cs`; `Operations.proto`; `wwwroot/ui/index.html`,
  `wwwroot/ui/js/app.js`; `Infrastructure/Extensions.cs` (`AddRedis()`, SignalR Redis backplane).

### Platform and technical capabilities

These five are included because each is an **operational responsibility with an identifiable
owner, a configuration surface, and a platform-wide blast radius** — not because they are
infrastructure. Framework plumbing that carries none of those properties is excluded.

**Capability: CAP-12 — Asynchronous Messaging & Event Distribution**

- **Description:** The platform's east-west integration substrate: one durable RabbitMQ **topic
  exchange per service** (`availability`, `customers`, `deliveries`, `identity`, `ordermaker`,
  `orders`, `parcels`, `vehicles`, `operations`), `snakeCase` message naming, queue template
  `<service>/{{exchange}}.{{message}}`, correlation propagated in a `message_context` header and
  trace context in a `span_context` header, and a transactional inbox/outbox enabled in the seven
  aggregate-owning services. The catalogue of every message name is a single hand-maintained JSON
  file inside `operations-service`. There is **no schema registry and no generated contract** — the
  `snake_case` name on the wire is the only binding contract, and each consumer redeclares the
  payload class itself.
- **Purpose / business value:** `Pacco/README.md` names "event-driven" integration as a governing
  choice for the platform. It does not describe the exchange-per-service topology, the outbox, or
  the catalogue.
- **Confidence:** high
- **Evidence:** every service's `src/**/appsettings.json` → `rabbitMq` section (`outbox` present in
  Availability, Customers, Deliveries, Identity, Orders, Parcels, Vehicles; absent in Operations,
  OrderMaker; **no `rabbitMq` section at all** in Pricing);
  `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`;
  `Pacco/compose/rabbitmq/Dockerfile`; each service's `Events/External/Handlers/`;
  `repo-inventory.md` §3.2, §3.3; `service-summaries.md` §4 (C2, C3).

**Capability: CAP-13 — Service Discovery & Load Balancing**

- **Description:** Every service registers itself with Consul at startup and resolves its outbound
  HTTP dependencies through a Fabio load-balancer URL, so the logical keys in each service's
  `httpClient.services` map — not hostnames — are what bind caller to callee. This is the mechanism
  that makes the eight synchronous service-to-service edges relocatable.
- **Purpose / business value:** **not observed.** `Pacco/README.md` does not mention Consul or
  Fabio.
- **Confidence:** high
- **Evidence:** `consul` and `fabio` sections present in all ten service `appsettings.json` files
  (verified across `hianshul100_Pacco.Services.*/src/**/appsettings.json`);
  `Convey.Discovery.Consul` and `Convey.LoadBalancing.Fabio` package references in every service
  `*.csproj`; `Pacco/compose/consul-fabio-vault.yml`, `compose/infrastructure.yml`;
  `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.docker.yml` (`loadBalancer.url: fabio:9999`).

**Capability: CAP-14 — Platform Observability**

- **Description:** Three separate signal paths, configured per service and aggregated centrally:
  distributed traces to Jaeger, structured logs to Seq with per-service property masking and
  secret-bearing properties excluded, and metrics exposed through App.Metrics and scraped by
  Prometheus into Grafana. The gateway additionally generates and exposes `Request-ID` and
  `Trace-ID` headers, which is how a client-side request is correlated to the operation state that
  CAP-11 projects.
- **Purpose / business value:** **not observed.**
- **Confidence:** high for the configured estate; **medium** for coverage completeness, because it
  is demonstrably uneven — see the dependency-surface note in §7.
- **Evidence:** `jaeger`, `seq`/`logger`, `metrics` sections in each service `appsettings.json`
  (`jaeger` present in nine of ten services; **absent in `ordermaker-service`**, which also
  references no `Convey.Tracing.Jaeger` package); `Pacco/compose/prometheus/prometheus.yml`;
  `Pacco/compose/grafana-seq-jaeger-prometheus.yml`;
  `Pacco.APIGateway/src/Pacco.APIGateway/Infrastructure/SpanContextBuilder.cs`,
  `CorrelationContextBuilder.cs`.

**Capability: CAP-15 — Secrets & Service-Identity Management**

- **Description:** Vault is the configured secret and PKI source for **nine of the ten services**
  (`vault.enabled: true`); `ordermaker-service` has no `vault` section at all and references no
  `Convey.Secrets.Vault` package, so it is the one host outside this capability. On top of it, two
  services implement a second,
  certificate-based trust layer: callers present a client certificate in a `Certificate` header,
  and `customers-service` carries an **access-control list** granting `availability-service` the
  `customers:read` permission, restricted to `allowedDomains: ['pacco.io']`. This is the only
  service-to-service authorization boundary on the platform that is finer-grained than "holds a
  valid JWT".
- **Purpose / business value:** **not observed.** `Pacco/docker-images.txt` documents the Vault
  init, unseal, `userpass` policy and PKI role setup operationally, but states no rationale.
- **Confidence:** high for the configuration; **medium** for runtime enforcement — see Q7.
- **Evidence:** `vault` section with `enabled: true` in nine of the ten service `appsettings.json`
  files (including `pricing-service`; **absent** in
  `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json`);
  `Convey.Secrets.Vault` package reference in those same nine service `*.csproj` files (**absent**
  in `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj`);
  `Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json`
  (`security.certificate.acl.availability-service.permissions`, `allowedDomains: ['pacco.io']`);
  `Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs`
  (certificate attached in the constructor); `Convey.WebApi.Security` referenced by Availability
  and Customers only; `Pacco/docker-images.txt`.

**Capability: CAP-16 — Environment & Deployment Definition**

- **Description:** One repository owns the definition of every environment the platform runs in:
  six Docker Compose stacks (infrastructure, services, services-local, host-infrastructure,
  consul-fabio-vault, grafana-seq-jaeger-prometheus), two PM2-style process manifests, the
  Prometheus scrape configuration, the RabbitMQ image build, the aggregate solution file, and the
  clone/pull scripts. Per-service build and release is separately owned by each repository's
  `Dockerfile`, `.travis.yml` and `scripts/`. **There is no Kubernetes, Helm, or Terraform anywhere
  in the workspace.**
- **Purpose / business value:** **not observed.**
- **Confidence:** high
- **Evidence:** `Pacco/compose/infrastructure.yml`, `compose/services.yml`,
  `compose/services-local.yml`, `compose/host-infrastructure.yml`,
  `compose/consul-fabio-vault.yml`, `compose/grafana-seq-jaeger-prometheus.yml`,
  `compose/mongo-rabbit-redis.yml`, `compose/prometheus/prometheus.yml`,
  `compose/rabbitmq/Dockerfile`; `Pacco/services.yml`, `Pacco/prod-services.yml`; `Pacco/Pacco.sln`;
  `Pacco/scripts/git-clone.sh`; each service repo's `Dockerfile`, `.travis.yml`, `scripts/*.sh`.

**Capabilities considered and deliberately not recorded.** Contract testing between
`orders-service` and `parcels-service` (Pactify 1.1.0, one consumer suite and one provider suite)
is a **testing practice attached to CAP-06/CAP-07**, not a capability with its own runtime
boundary; it is recorded as a quality signal in §7. The `Pacco.Services.Operations.GrpcClient`
console project is a client of CAP-11, not a capability. Clean-architecture layering, Convey
composition roots, document mappers and CQRS decorators are framework plumbing and are excluded
per the granularity rule stated above.

---

## 2 — Service ownership mapping

Ownership below means **runtime ownership of the capability's behaviour and its write model**, as
observed in code — **not** organizational or team ownership. **Team ownership is `[unknown]` for
every capability on this platform**: no `CODEOWNERS`, `CONTRIBUTING.md`, or team metadata exists in
any of the thirteen clones. Capability descriptions are not repeated here; see §1.

| Capability | Primary Owner (service / repo) | Secondary Contributors | Ownership Confidence | Evidence |
|------------|-------------------------------|------------------------|----------------------|----------|
| CAP-01 Identity & Access Management | `identity-service` (`Pacco.Services.Identity`) | `api-gateway` (validates the JWT it does not issue); `customers-service` (consumes `signed_up`) | high | `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs`; `Core/Entities/User.cs`, `RefreshToken.cs`, `Role.cs`; `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` (`validIssuer: pacco`) |
| CAP-02 Edge Routing & Access Enforcement | `api-gateway` (`Pacco.APIGateway`) | `Pacco` (selects the config via `NTRADA_CONFIG`) | high | `src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml`, `ntrada.docker.yml`, `ntrada-async.docker.yml`; `Pacco/compose/services.yml` |
| CAP-03 Customer Profile & Lifecycle Management | `customers-service` (`Pacco.Services.Customers`) | `orders-service` and `parcels-service` hold **read replicas** of `customers` populated once from `customer_created` | high (owner) / medium (replica semantics) | `Customers.Core/Entities/Customer.cs`, `State.cs`, `Core/Services/VipPolicy.cs`; `Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs`; `Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs` |
| CAP-04 Resource Availability & Reservation | `availability-service` (`Pacco.Services.Availability`) | `ordermaker-service` (selects a reservation date and issues `reserve_resource`); `orders-service` (reacts to `resource_reserved` / `resource_reservation_canceled`) | high | `Availability.Core/Entities/Resource.cs`; `Application/Commands/Handlers/ReserveResourceHandler.cs`; `OrderMaker/Services/ResourceReservationsService.cs`; `Orders.Application/Events/External/Handlers/ResourceReservedHandler.cs` |
| CAP-05 Vehicle Fleet Catalogue | `vehicles-service` (`Pacco.Services.Vehicles`) | none — read synchronously by `orders-service` and `ordermaker-service`; `availability-service` consumes only `vehicle_deleted` | high | `Vehicles.Core/Entities/Vehicle.cs`, `Variants.cs`; `Orders.Infrastructure/Services/Clients/VehiclesServiceClient.cs`; `OrderMaker/Services/Clients/VehiclesServiceClient.cs` |
| CAP-06 Parcel Catalogue & Volume Calculation | `parcels-service` (`Pacco.Services.Parcels`) | `orders-service` (holds its own `Parcel` copy inside the order aggregate) | high | `Parcels.Core/Entities/Parcel.cs`; `Parcels.Infrastructure/Mongo/Queries/Handlers/GetParcelsVolumeHandler.cs`; `Orders.Core/Entities/Parcel.cs` |
| CAP-07 Order Lifecycle Management | `orders-service` (`Pacco.Services.Orders`) | `ordermaker-service` (drives the aggregate by command); `deliveries-service` (its events advance order state); `availability-service` (`resource_reserved` triggers approval) | high | `Orders.Core/Entities/Order.cs`, `OrderStatus.cs`; `Orders.Application/Commands/Handlers/*.cs`; `Orders.Application/Events/External/Handlers/*.cs` |
| CAP-08 Order Pricing & Discounting | `pricing-service` (`Pacco.Services.Pricing`) | `customers-service` (supplies the completed-order count and VIP flag the rule reads) | high | `Pricing.Api/Core/Services/CustomerDiscountsService.cs`; `Pricing.Api/Queries/Handlers/GetOrderPricingHandler.cs`; `Pricing.Api/Services/Clients/CustomersServiceClient.cs` |
| CAP-09 Delivery Execution & Tracking | `deliveries-service` (`Pacco.Services.Deliveries`) | `orders-service` (sole consumer of all three delivery events) | high (owner) / medium (initiation path, Q1 — `service-summaries.md` G7) | `Deliveries.Core/Entities/Delivery.cs`; `Deliveries.Application/Commands/Handlers/StartDeliveryHandler.cs`; `Orders.Application/Events/External/Handlers/Delivery{Started,Completed,Failed}Handler.cs` |
| CAP-10 Automated Order Orchestration | `ordermaker-service` (`Pacco.Services.OrderMaker`) | `orders-service`, `parcels-service`, `vehicles-service`, `availability-service` — all four are commanded by the saga and none of them knows it exists | medium — reachability unproven (B1 — `service-summaries.md` G2) | `OrderMaker/Sagas/AIOrderMakingSaga.cs`, `AIMakingOrderData.cs`; `Pacco/compose/services.yml` (port `5015`); absent from `Pacco/services.yml`, `prod-services.yml`, all four `ntrada*.yml` |
| CAP-11 Operation Status Projection & Real-Time Notification | `operations-service` (`Pacco.Services.Operations`) | all eight message-publishing services (their message names populate `messages.json`); `api-gateway` (generates the correlation id) | high | `Operations.Api/Services/OperationsService.cs`; `Handlers/Generic*Handler.cs`; `Infrastructure/Subscriptions.cs`; `messages.json`; `Hubs/PaccoHub.cs`; `Operations.proto` |
| CAP-12 Asynchronous Messaging & Event Distribution | **none — no single owner.** Distributed across nine exchange-owning services; the message *catalogue* sits in `operations-service` | `Pacco` (owns the broker image and network); every publishing and subscribing service | high (topology) / **low (catalogue ownership)** | each service's `appsettings.json` → `rabbitMq`; `Operations.Api/messages.json`; `Pacco/compose/rabbitmq/Dockerfile`; `repo-inventory.md` §3.2 |
| CAP-13 Service Discovery & Load Balancing | `Pacco` (defines the Consul and Fabio containers) | all ten services participate via `consul`/`fabio` config and Convey packages | high | `Pacco/compose/consul-fabio-vault.yml`; `consul`/`fabio` sections in all ten service `appsettings.json`; `Convey.Discovery.Consul`, `Convey.LoadBalancing.Fabio` in every service `*.csproj` |
| CAP-14 Platform Observability | `Pacco` (defines Jaeger, Seq, Prometheus, Grafana) | all ten services plus `api-gateway` emit signals; **`ordermaker-service` emits no traces** | high (estate) / medium (coverage) | `Pacco/compose/grafana-seq-jaeger-prometheus.yml`, `compose/prometheus/prometheus.yml`; `jaeger`/`logger`/`metrics` sections per service; no `jaeger` section or package in `Pacco.Services.OrderMaker` |
| CAP-15 Secrets & Service-Identity Management | `Pacco` (defines Vault and documents its PKI setup) — enforcement point is `customers-service` | `availability-service` (the only certificate-presenting caller); nine of the ten services consume Vault secrets — `ordermaker-service` has no `vault` section and no `Convey.Secrets.Vault` reference | high (config) / medium (enforcement, Q7) | `Pacco/docker-images.txt`; `Pacco/compose/consul-fabio-vault.yml`; `Customers.Api/appsettings.json` (`security.certificate.acl`); `Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` |
| CAP-16 Environment & Deployment Definition | `Pacco` | each service repo owns its own `Dockerfile`, `.travis.yml` and `scripts/` | high | `Pacco/compose/*.yml`; `Pacco/services.yml`, `prod-services.yml`; `Pacco/Pacco.sln`; per-repo `Dockerfile` and `.travis.yml` |

**Notes on shared or ambiguous ownership**

1. **CAP-12 has no owner.** Nine services each own one exchange, but the platform-wide catalogue
   that defines what CAP-11 can observe — `messages.json` — lives inside `operations-service` and is
   a hand-maintained copy of names owned by eight *other* repositories, with no generation and no
   validation step. A message added anywhere is invisible to CAP-11 until someone edits that file.
   Ownership confidence for the catalogue is **low**; for the exchange topology it is **high**.
2. **CAP-03's data is owned once and replicated twice.** `orders-service` and `parcels-service`
   each persist their own `customers` collection, written from `customer_created` and never
   updated: `customer_state_changed` and `customer_became_vip` are published but handled by no
   `Events/External/Handlers/` class in any repository. Both replicas therefore diverge permanently
   from the owner after creation.
3. **CAP-07 is written by a service that does not own it.** `ordermaker-service` issues
   `create_order`, `add_parcel_to_order`, `assign_vehicle_to_order`, `approve_order` and
   `cancel_order` onto the `orders` exchange, and `reserve_resource` onto `availability`. Neither
   `orders-service` nor `availability-service` subscribes to anything from the `ordermaker`
   exchange, so the orchestration relationship is one-directional and invisible from the owning
   side.
4. **CAP-08's rule reads a fact CAP-03 owns, on every call.** `pricing-service` holds no data and
   calls `customers-service` synchronously per request. Grouping it under a "customer and
   commercial" domain is an inference, not evidence — `service-summaries.md` §2.4 flags the same
   uncertainty (`[assumption]`; this baseline records the underlying grouping premise as A5 below).
5. **CAP-13, CAP-14, CAP-15 are owned by definition in one repo and by participation in ten.** The
   `Pacco` repository defines the containers and the network; each service independently decides
   whether to participate, and participation is uneven (no Jaeger in `ordermaker-service`;
   `Convey.WebApi.Security` only in Availability and Customers).
6. **CAP-10's ownership boundary is real but its reachability is not established.** The saga owns
   the orchestration behaviour unambiguously in code; whether anything invokes it is `[unknown]`.

---

## 3 — Requirement / feature traceability

This is **not** a specification. Every row traces a capability to something directly visible in the
repositories — an HTTP endpoint registration, a CQRS command/query handler, a message subscription,
a saga step, or a configuration key. **No feature-flag system exists on the platform**: a
workspace-wide search for `LaunchDarkly`, `Unleash`, `Flagsmith`, `Split`, `featureFlag`,
`feature_flag` and `featureToggle` across `*.cs`, `*.json` and `*.yml` returned zero matches
(`repo-inventory.md` §6), so no flag-driven features can be traced.

Line references are to `src/**/Program.cs` where Convey's `UseDispatcherEndpoints` registers the
HTTP surface; repository prefixes are omitted where the capability's owning repository is
unambiguous from §2.

| Capability | Observable Feature / Requirement | Evidence Location | Confidence |
|------------|----------------------------------|-------------------|------------|
| CAP-01 | Sign-up with unique lower-cased email; role defaults to `user` | `Pacco.Services.Identity/src/Pacco.Services.Identity.Api/Program.cs:52` → `Application/Services/Identity/IdentityService.cs:85-107` | high |
| CAP-01 | Sign-in returning a JWT plus a freshly created refresh token | `…Identity.Api/Program.cs:47` → `IdentityService.cs:49-83` | high |
| CAP-01 | Current-user lookup and admin-gated user lookup | `…Identity.Api/Program.cs:35-36` (`users/{userId}`, `me`) | high |
| CAP-01 | Access-token revocation (Redis blacklist) and refresh-token use / revoke | `…Identity.Api/Program.cs:57,62,67`; `Application/Services/Identity/RefreshTokenService.cs:38-79` | high |
| CAP-01 | `sign_up` accepted as a **command off the broker**, not only over HTTP | `…Identity.Api` RabbitMQ `SubscribeCommand<SignUp>()`; `repo-inventory.md` §2.2 row `Pacco.Services.Identity` | high |
| CAP-02 | 40 declarative routes across 9 service modules plus `/`, `/docs` | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` | high |
| CAP-02 | Five admin-gated routes (`role: admin` claim requirement), identical in sync and async configs | `ntrada.yml` and `ntrada-async.yml` — 5 × `role: admin` in each | high |
| CAP-02 | Caller identity injected into downstream paths and message payloads via `@user_id` | `ntrada.yml:143` (`customers-service/customers/@user_id`), `:298` (`orders-service/orders?customerId=@user_id`); `ntrada-async.yml:132` (`bind: customerId:@user_id`) | high |
| CAP-02 | Async mode converts 20 write routes to RabbitMQ publishes | `ntrada-async.yml` — 20 × `use: rabbitmq` with `exchange` + `routing_key` (e.g. `:120-121` `availability`/`add_resource`, `:128-129` `availability`/`reserve_resource`) vs. 40 × `use: downstream` in `ntrada.yml` | high |
| CAP-03 | Customer created from an identity sign-up | `Customers.Application/Events/External/Handlers/SignedUpHandler.cs` | high |
| CAP-03 | One-time registration completion requiring non-blank full name and address | `…Customers.Api/Program.cs:38` → `Customers.Core/Entities/Customer.cs:44-65` | high |
| CAP-03 | Admin-only customer state change across the five-value state machine | `…Customers.Api/Program.cs:40`; `ntrada.yml:174-181` (`role: admin`); `Customers.Core/Entities/State.cs` | high |
| CAP-03 | VIP promotion at 20 completed orders | `Customers.Core/Services/VipPolicy.cs:8-21`; `Application/Events/External/Handlers/OrderCompletedHandler.cs:27-42` | high |
| CAP-03 | Customer state exposed as a dedicated read endpoint for other services | `…Customers.Api/Program.cs:37` (`customers/{customerId}/state`) | high |
| CAP-04 | Resource creation requiring at least one non-blank tag | `…Availability.Api/Program.cs:42` → `Availability.Core/Entities/Resource.cs:36-46` | high |
| CAP-04 | Date-keyed reservation with priority-based expropriation | `…Availability.Api/Program.cs:44` → `Resource.cs:56-79`; `Core/ValueObjects/Reservation.cs` | high |
| CAP-04 | Reservation release and resource deletion cascading cancellations | `…Availability.Api/Program.cs:45-46` → `Resource.cs:81-98` | high |
| CAP-04 | Cross-service customer-state gate before a reservation is accepted | `Availability.Application/Commands/Handlers/ReserveResourceHandler.cs:41-50` | high |
| CAP-04 | Cleanup of resources when a vehicle is deleted | `Availability.Application/Events/External/Handlers/VehicleDeletedHandler.cs` | medium — handler observed; exact cleanup semantics not read line-by-line for this baseline |
| CAP-05 | Vehicle CRUD with capacity and price validation | `…Vehicles.Api/Program.cs:35-40` → `Vehicles.Core/Entities/Vehicle.cs:17-55` | high |
| CAP-05 | Paged vehicle search | `…Vehicles.Api/Program.cs:36` (`SearchVehicles` → `PagedResult<VehicleDto>`) | high |
| CAP-05 | Admin-gated vehicle mutation at the edge | `ntrada.yml` vehicles module (`POST`/`PUT`/`DELETE /vehicles`); `repo-inventory.md` §2.3 row `Pacco.Services.Vehicles` | medium — gateway gating recorded in the inventory; the `role: admin` blocks were counted, not individually re-attributed per route |
| CAP-06 | Parcel creation requiring non-blank name and description | `…Parcels.Api/Program.cs:44` → `Parcels.Core/Entities/Parcel.cs:18-31` | high |
| CAP-06 | Volume aggregation over a supplied set of parcel ids, returning 0 for an empty set | `…Parcels.Api/Program.cs:40` → `Parcels.Infrastructure/Mongo/Queries/Handlers/GetParcelsVolumeHandler.cs:25-43` | high |
| CAP-06 | Parcel list scoped to the calling identity | `Parcels.Infrastructure/Mongo/Queries/Handlers/GetParcelsHandler.cs` (`IAppContext`); `repo-inventory.md` §2.3 | high |
| CAP-06 | Order-membership link maintained from four `orders` events | `Parcels.Application/Events/External/Handlers/` (`OrderCanceled`, `OrderDeleted`, `ParcelAddedToOrder`, `ParcelDeletedFromOrder`) | high |
| CAP-07 | Order creation, deletion, parcel attach/detach, vehicle assignment | `…Orders.Api/Program.cs:35-42` | high |
| CAP-07 | Order list scoped to the caller's identity | `Orders.Infrastructure/Mongo/Queries/Handlers/GetOrdersHandler.cs` (`IAppContext`); `ntrada.yml:298` | high |
| CAP-07 | Per-command ownership check: caller must be the order's customer or an admin | `Orders.Application/Commands/Handlers/AddParcelToOrderHandler.cs:51-58`, `AssignVehicleToOrderHandler.cs:38-42`, `ApproveOrderHandler.cs:34-38` | high |
| CAP-07 | Vehicle assignment prices the order via CAP-08 and sets the delivery date | `AssignVehicleToOrderHandler.cs:44-65` | high |
| CAP-07 | Order approval driven by a reservation event, not only by a command | `Orders.Application/Events/External/Handlers/ResourceReservedHandler.cs:24-36` | high |
| CAP-07 | Order state advanced by delivery events | `Orders.Application/Events/External/Handlers/DeliveryStartedHandler.cs:24-36`, `DeliveryCompletedHandler.cs`, `DeliveryFailedHandler.cs` | high |
| CAP-08 | Single pricing query taking `customerId` and `orderPrice` | `Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Program.cs` (`GetOrderPricing`); `repo-inventory.md` §2.2 (`GET /pricing?customerId=&orderPrice=`) | high |
| CAP-08 | Four-tier completed-order discount ladder plus VIP uplift | `Pricing.Api/Core/Services/CustomerDiscountsService.cs:7-30` | high |
| CAP-08 | Non-positive discounted price falls back to the base price | `Pricing.Api/Queries/Handlers/GetOrderPricingHandler.cs:45` | high |
| CAP-09 | Delivery start, complete, fail, and registration append | `…Deliveries.Api/Program.cs:34-39` | high |
| CAP-09 | Re-start blocked unless the previous delivery for that order failed | `Deliveries.Application/Commands/Handlers/StartDeliveryHandler.cs:28-34` | high |
| CAP-09 | Registrations accepted only while in progress | `Deliveries.Core/Entities/Delivery.cs:46-59` | high |
| CAP-10 | Saga entry point `POST /orders` on the orchestrator, distinct from the Orders API | `OrderMaker/Handlers/AIOrderMakingHandler.cs`; `OrderMaker/Program.cs`; **no route in any `ntrada*.yml`** | medium — endpoint observed, caller path `[unknown]` (B1 — `service-summaries.md` G2) |
| CAP-10 | Five-step choreography keyed by `OrderId` | `OrderMaker/Sagas/AIOrderMakingSaga.cs:45-132` | high |
| CAP-10 | Vehicle and reservation-date selection delegated to sync clients mid-saga | `AIOrderMakingSaga.cs:92-101`; `Services/Clients/VehiclesServiceClient.cs`; `Services/ResourceReservationsService.cs` | high |
| CAP-10 | Single compensation path: cancel the order if parcel attachment fails | `AIOrderMakingSaga.cs:140-146` | high |
| CAP-11 | Operation lookup by correlation id over HTTP | `Operations.Api/Program.cs` (`GET /operations/{operationId}`); `ntrada.yml:283` | high |
| CAP-11 | Real-time push over SignalR hub `/pacco` and server-streaming gRPC | `Operations.Api/Hubs/PaccoHub.cs`; `Operations.proto` (`GetOperation`, `SubscribeOperations`); `wwwroot/ui/js/app.js` | high |
| CAP-11 | Subscription set driven by a data file, types emitted at runtime | `Operations.Api/messages.json` (8 service blocks, each with one `exchange`; 24 commands, 29 events, 27 rejected events summed across all blocks — of which `orders-service` alone contributes 7 / 9 / 10); `Infrastructure/Subscriptions.cs` (`System.Reflection.Emit`) | high |
| CAP-11 | Operation state expiry window | `Operations.Api/appsettings.json:149-151` (`requests.expirySeconds: 300`) → `Services/OperationsService.cs:50-55` (`SlidingExpiration`) | high |
| CAP-12 | Exchange-per-service topic topology with `snakeCase` naming | each service `appsettings.json` → `rabbitMq` (`exchange.name`, `conventions`); `repo-inventory.md` §3.2 | high |
| CAP-12 | Transactional inbox/outbox in the seven aggregate-owning services | `outbox` section present in Availability, Customers, Deliveries, Identity, Orders, Parcels, Vehicles `appsettings.json`; absent in Operations and OrderMaker; no `rabbitMq` at all in Pricing | high |
| CAP-12 | Rejected-event convention as the failure channel (27 declared across the 8 service blocks) | `Operations.Api/messages.json`; each service's `Exceptions/` → `*_rejected` mapping | high |
| CAP-13 | Consul registration and Fabio-resolved outbound calls in all ten services | `consul` and `fabio` sections in every `hianshul100_Pacco.Services.*/src/**/appsettings.json`; `httpClient.services` maps | high |
| CAP-14 | Traces, logs and metrics per service | `jaeger`, `logger`/`seq`, `metrics` sections per service `appsettings.json`; `Pacco/compose/prometheus/prometheus.yml` | high |
| CAP-14 | Request/trace id generation and header exposure at the edge | `ntrada.yml` (`generateRequestId`, `generateTraceId`; `Request-ID`, `Trace-ID`, `Resource-ID`, `Total-Count`) | high |
| CAP-15 | Vault enabled in nine of ten services, including `pricing-service`; **not** in `ordermaker-service` | `vault.enabled: true` in nine service `appsettings.json` files; `Convey.Secrets.Vault` in those same nine `*.csproj` files; neither present in `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/` | high |
| CAP-15 | Certificate ACL granting `availability-service` the `customers:read` permission | `Customers.Api/appsettings.json:164-178` (`security.certificate.acl`, `allowedDomains: ['pacco.io']`) | high (config) / medium (enforcement, Q7) |
| CAP-15 | Client certificate attached on the one call the ACL covers | `Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` | high |
| CAP-16 | Eleven compose deployables and ten PM2 apps | `Pacco/compose/services.yml`, `compose/services-local.yml`; `Pacco/services.yml`, `prod-services.yml` | high |
| CAP-16 | Gateway operating mode selected by environment variable | `Pacco/compose/services.yml` (`NTRADA_CONFIG=ntrada-async.docker.yml`) | high |
| CAP-16 | Per-repository CI to Docker Hub | each service repo's `.travis.yml` → `scripts/build.sh`, `scripts/dockerize.sh` | high |

---

## 4 — Code evidence

Key artifacts per capability. Paths are workspace-relative from the repository root shown in the
capability heading. Class libraries, DTOs, document mappers and Convey composition roots are
omitted unless they carry capability behaviour.

- **Capability: CAP-01 — Identity & Access Management** (`Pacco.Services.Identity`)
  - `src/Pacco.Services.Identity.Core/Entities/User.cs` — aggregate; constructor-enforced email,
    password and role validity.
  - `src/Pacco.Services.Identity.Core/Entities/RefreshToken.cs`, `Role.cs` — refresh-token
    aggregate with a `Revoke` transition; the valid-role set.
  - `src/Pacco.Services.Identity.Application/Services/Identity/IdentityService.cs` — email regex,
    duplicate-email rejection, password hashing on sign-up, credential check and JWT+refresh issue
    on sign-in, `signed_up` / `signed_in` publication.
  - `src/Pacco.Services.Identity.Application/Services/Identity/RefreshTokenService.cs` — create,
    revoke, and single-use exchange of a refresh token for a new JWT.
  - `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs`, `PasswordService.cs`,
    `Rng.cs` — token minting, ASP.NET `IPasswordHasher` wrapper, token entropy source.
  - `src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs` — the platform's **only**
    schema action: a unique index created on `users` at startup.
  - `src/Pacco.Services.Identity.Api/Program.cs:34-70` — the seven HTTP endpoints.

- **Capability: CAP-02 — Edge Routing & Access Enforcement** (`Pacco.APIGateway`)
  - `src/Pacco.APIGateway/ntrada.yml` — sync routing table: 40 downstream routes, 5 admin claim
    gates, `@user_id` bindings, CORS `allowedOrigins: ['*']` with `allowCredentials: true`.
  - `src/Pacco.APIGateway/ntrada-async.yml` — same auth surface, 20 routes converted to
    `use: rabbitmq` with explicit `exchange` + `routing_key` + `bind` blocks.
  - `src/Pacco.APIGateway/ntrada.docker.yml`, `ntrada-async.docker.yml` — container variants;
    `loadBalancer.url: fabio:9999`.
  - `src/Pacco.APIGateway/Infrastructure/CorrelationContextBuilder.cs`, `SpanContextBuilder.cs`,
    `HttpRequestHook.cs` — correlation id, trace context, and request hook that make CAP-11's
    per-operation projection possible.
  - `src/Pacco.APIGateway/Program.cs` — host; config file selected by `NTRADA_CONFIG`.

- **Capability: CAP-03 — Customer Profile & Lifecycle Management** (`Pacco.Services.Customers`)
  - `src/Pacco.Services.Customers.Core/Entities/Customer.cs` — aggregate; `CompleteRegistration`
    guard, four state setters, `SetVip` idempotence, `AddCompletedOrder`.
  - `src/Pacco.Services.Customers.Core/Entities/State.cs` — the five-value state enum.
  - `src/Pacco.Services.Customers.Core/Services/VipPolicy.cs` — the 20-completed-orders threshold.
  - `src/Pacco.Services.Customers.Application/Events/External/Handlers/SignedUpHandler.cs` —
    customer creation from CAP-01's `signed_up`.
  - `src/Pacco.Services.Customers.Application/Events/External/Handlers/OrderCompletedHandler.cs` —
    completed-order accumulation and VIP evaluation.
  - `src/Pacco.Services.Customers.Api/appsettings.json` — `mongo.database: customers-service`,
    exchange `customers`, `security.certificate.acl`.

- **Capability: CAP-04 — Resource Availability & Reservation** (`Pacco.Services.Availability`)
  - `src/Pacco.Services.Availability.Core/Entities/Resource.cs` — aggregate root; tag validation,
    `AddReservation` with priority expropriation, `ReleaseReservation`, `Delete` cascade.
  - `src/Pacco.Services.Availability.Core/ValueObjects/Reservation.cs` — equality and hash code on
    `DateTime.Date` only, which is what makes a reservation date-unique per resource.
  - `src/Pacco.Services.Availability.Application/Commands/Handlers/ReserveResourceHandler.cs` —
    caller-identity check, resource existence, and the synchronous customer-state gate.
  - `src/Pacco.Services.Availability.Application/Events/External/Handlers/VehicleDeletedHandler.cs`,
    `CustomerCreatedHandler.cs` — the two inbound external subscriptions.
  - `src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` —
    the certificate-bearing call to CAP-03.
  - `src/Pacco.Services.Availability.Api/appsettings.json` — `outbox.enabled: true`,
    `security.certificate.header: Certificate`.

- **Capability: CAP-05 — Vehicle Fleet Catalogue** (`Pacco.Services.Vehicles`)
  - `src/Pacco.Services.Vehicles.Core/Entities/Vehicle.cs` — positive-capacity and positive-price
    invariants, description guard, `Standard` variant applied on construction.
  - `src/Pacco.Services.Vehicles.Core/Entities/Variants.cs` — the flags enum.
  - `src/Pacco.Services.Vehicles.Api/Program.cs:34-40` — the five HTTP endpoints including paged
    search.

- **Capability: CAP-06 — Parcel Catalogue & Volume Calculation** (`Pacco.Services.Parcels`)
  - `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs` — name/description guards, `OrderId` link,
    `AddToOrder` / `DeleteFromOrder`.
  - `src/Pacco.Services.Parcels.Core/Entities/Size.cs`, `Variant.cs` — the classification the volume
    calculation reads.
  - `src/Pacco.Services.Parcels.Infrastructure/Mongo/Queries/Handlers/GetParcelsVolumeHandler.cs` —
    empty-set short circuit, then delegation to `IParcelsService.CalculateVolume`.
  - `src/Pacco.Services.Parcels.Infrastructure/Mongo/Queries/Handlers/GetParcelsHandler.cs` —
    identity-scoped listing.
  - `tests/Pacco.Services.Parcels.PactProviderTests/` — the provider half of the platform's only
    contract test.

- **Capability: CAP-07 — Order Lifecycle Management** (`Pacco.Services.Orders`)
  - `src/Pacco.Services.Orders.Core/Entities/Order.cs` — the aggregate; every state transition and
    its guard, `CanBeDeleted`, `CanAssignVehicle`, `HasParcels`, `SetTotalPrice` guard.
  - `src/Pacco.Services.Orders.Core/Entities/OrderStatus.cs` — the five-state enum.
  - `src/Pacco.Services.Orders.Application/Commands/Handlers/AssignVehicleToOrderHandler.cs` — the
    densest cross-capability handler: identity check, parcel precondition, state precondition,
    vehicle lookup, pricing call, delivery date.
  - `src/Pacco.Services.Orders.Application/Commands/Handlers/AddParcelToOrderHandler.cs`,
    `ApproveOrderHandler.cs`, `CancelOrderHandler.cs`, `CreateOrderHandler.cs`,
    `DeleteOrderHandler.cs`, `DeleteParcelFromOrderHandler.cs`.
  - `src/Pacco.Services.Orders.Application/Events/External/Handlers/ResourceReservedHandler.cs` —
    approval triggered by CAP-04.
  - `src/Pacco.Services.Orders.Application/Events/External/Handlers/DeliveryStartedHandler.cs`,
    `DeliveryCompletedHandler.cs`, `DeliveryFailedHandler.cs` — state advanced by CAP-09.
  - `src/Pacco.Services.Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs`,
    `PricingServiceClient.cs`, `VehiclesServiceClient.cs` — the three synchronous dependencies.
  - `src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs` — the CAP-03
    read replica.
  - `tests/Pacco.Services.Orders.PactConsumerTests/` — the consumer half of the contract test.

- **Capability: CAP-08 — Order Pricing & Discounting** (`Pacco.Services.Pricing`)
  - `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` — the entire
    business rule, in hard-coded constants.
  - `src/Pacco.Services.Pricing.Api/Queries/Handlers/GetOrderPricingHandler.cs` — customer lookup,
    discount application, non-positive fallback.
  - `src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs` — the sole outbound
    dependency.
  - `src/Pacco.Services.Pricing.Api/Program.cs:31` — the single endpoint.
  - `src/Pacco.Services.Pricing.Api/appsettings.json` — no `mongo`, no `redis`, no `rabbitMq`;
    `vault.enabled: true`.

- **Capability: CAP-09 — Delivery Execution & Tracking** (`Pacco.Services.Deliveries`)
  - `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs` — aggregate; in-progress-only
    registration guard, terminal-state guards on `Complete` and `Fail`, `LastUpdate` projection.
  - `src/Pacco.Services.Deliveries.Core/Entities/DeliveryStatus.cs` — the three-state enum.
  - `src/Pacco.Services.Deliveries.Core/ValueObjects/DeliveryRegistration.cs` — value equality on
    description + timestamp, which makes duplicate scans idempotent.
  - `src/Pacco.Services.Deliveries.Application/Commands/Handlers/StartDeliveryHandler.cs` — the
    re-start guard keyed on the order.

- **Capability: CAP-10 — Automated Order Orchestration** (`Pacco.Services.OrderMaker`)
  - `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` — the whole choreography:
    `ResolveId` on `OrderId`, five handlers, one compensation.
  - `src/Pacco.Services.OrderMaker/Sagas/AIMakingOrderData.cs` — saga state including
    `AllPackagesAddedToOrder`, the gate that holds the saga until every parcel is confirmed.
  - `src/Pacco.Services.OrderMaker/Handlers/AIOrderMakingHandler.cs` — the `MakeOrder` entry point.
  - `src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs`,
    `AvailabilityServiceClient.cs`, `Services/ResourceReservationsService.cs` — mid-saga selection
    logic.
  - `src/Pacco.Services.OrderMaker/Extensions.cs` — `AddChronicle()` with **no persistence
    backend**; `AddRedis()`.

- **Capability: CAP-11 — Operation Status Projection & Real-Time Notification** (`Pacco.Services.Operations`)
  - `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` — the state machine: read
    from `IDistributedCache` under `requests:{id}`, refuse to overwrite a terminal state, write back
    with a sliding expiry, raise `OperationUpdated`.
  - `src/Pacco.Services.Operations.Api/Handlers/GenericEventHandler.cs`, `GenericCommandHandler.cs`,
    `GenericRejectedEventHandler.cs` — correlation-id requirement, saga-state header read, hub fan-out.
  - `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` — runtime type emission from
    `messages.json`.
  - `src/Pacco.Services.Operations.Api/messages.json` — the platform's de facto message catalogue.
  - `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs`, `Operations.proto`,
    `Program.cs:32,46,47` — the three protocol surfaces.
  - `src/Pacco.Services.Operations.Api/wwwroot/ui/index.html`, `wwwroot/ui/js/app.js` — the only
    frontend asset on the platform.
  - `src/Pacco.Services.Operations.Api/appsettings.json:149-151` — `requests.expirySeconds: 300`.

- **Capability: CAP-12 — Asynchronous Messaging & Event Distribution** (cross-repository)
  - `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` — the only
    platform-wide contract artifact.
  - `hianshul100_Pacco.Services.*/src/**/appsettings.json` → `rabbitMq` — exchange name, `topic`
    type, `snakeCase` conventions, queue template, `outbox` block.
  - `hianshul100_Pacco.Services.*/src/**/Events/External/*.cs` — the redeclared consumer-side
    contract classes (`CustomerCreated` exists four times; `ResourceReserved` three times).
  - `Pacco/compose/rabbitmq/Dockerfile` — broker image with management and prometheus plugins.

- **Capability: CAP-13 — Service Discovery & Load Balancing** (cross-repository)
  - `hianshul100_Pacco.Services.*/src/**/appsettings.json` → `consul`, `fabio`, `httpClient.services`.
  - `Pacco/compose/consul-fabio-vault.yml`, `Pacco/compose/infrastructure.yml`.
  - `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.docker.yml` — `loadBalancer.url: fabio:9999`.

- **Capability: CAP-14 — Platform Observability** (cross-repository)
  - `hianshul100_Pacco.Services.*/src/**/appsettings.json` → `jaeger`, `logger`/`seq`, `metrics`,
    plus per-service excluded-property masking lists.
  - `Pacco/compose/prometheus/prometheus.yml`, `Pacco/compose/grafana-seq-jaeger-prometheus.yml`.
  - `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json` — **no `jaeger`
    section**; the `*.csproj` references no Jaeger package.

- **Capability: CAP-15 — Secrets & Service-Identity Management** (cross-repository)
  - `Pacco/docker-images.txt` — Vault init, unseal, `userpass` policy, PKI roles for
    `availability-service` and `customers-service`, Mongo dynamic credentials. **Also contains five
    unseal keys and a root token in plaintext.**
  - `Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:164-178` — the ACL.
  - `Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs`.
  - `hianshul100_Pacco.Services.*/src/**/appsettings.json` → `vault` (enabled in nine of ten; no
    `vault` section in `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json`).

- **Capability: CAP-16 — Environment & Deployment Definition** (`Pacco`)
  - `compose/infrastructure.yml`, `compose/services.yml`, `compose/services-local.yml`,
    `compose/host-infrastructure.yml`, `compose/consul-fabio-vault.yml`,
    `compose/grafana-seq-jaeger-prometheus.yml`, `compose/mongo-rabbit-redis.yml`.
  - `services.yml`, `prod-services.yml` — the ten PM2 apps (ports 5000–5009).
  - `Pacco.sln`, `scripts/git-clone.sh`, `docker-images.txt`.
  - Per service repository: `Dockerfile`, `.travis.yml`, `scripts/{build,dockerize,start,test}.sh`.

---

## 5 — Confidence assessment

| Capability | Overall Confidence | Key Uncertainty | Evidence Basis |
|------------|-------------------|-----------------|----------------|
| CAP-01 Identity & Access Management | high | Whether the gateway's committed symmetric `issuerSigningKey` and this service's certificate-based signing configuration are the same trust root; no code path proves the pairing | Source code (aggregate + application services + auth infrastructure) and config |
| CAP-02 Edge Routing & Access Enforcement | high | Which of the four `ntrada*.yml` files is authoritative in production; the sync and async pairs are architecturally different systems, not environment variants (B4 — `service-summaries.md` G6) | Runtime config (four declarative routing tables) + compose environment variable |
| CAP-03 Customer Profile & Lifecycle Management | high | Business meaning of `Suspicious` and `Locked` is undocumented; the two downstream replicas of `customers` are never reconciled (Q4) | Source code (aggregate + VIP policy + external event handlers) |
| CAP-04 Resource Availability & Reservation | high | Whether `outbox.disableTransactions: true` against a single-node Mongo is deliberate; what "priority" means commercially | Source code (aggregate root + value object + command handler) and config |
| CAP-05 Vehicle Fleet Catalogue | high | `vehicle_added` and `vehicle_updated` have no consumer anywhere, so downstream capabilities can act on stale vehicle attributes (Q13 — `service-summaries.md` G9) | Source code (aggregate) + endpoint registrations + subscription search |
| CAP-06 Parcel Catalogue & Volume Calculation | high | The volume formula itself lives behind `IParcelsService.CalculateVolume` and was not read line-by-line for this baseline; how the Pact file crosses to `orders-service` is unknown (Q5) | Source code (aggregate + query handler) and test-project layout |
| CAP-07 Order Lifecycle Management | high | Whether the local `customers` collection is a projection or a second source of truth; the price is set once at vehicle assignment and never recomputed | Source code (aggregate + 7 command handlers + 7 external event handlers + 3 HTTP clients) |
| CAP-08 Order Pricing & Discounting | high | Discount tiers are hard-coded with no configuration key, no test asserting them in-repo, and no documentation of intent | Source code (one rule class + one query handler) |
| CAP-09 Delivery Execution & Tracking | medium | **Nothing in the workspace initiates a delivery**: the service subscribes to no external event and no service publishes a `deliveries` command; the only entry is a human calling `POST /deliveries` (Q1 — `service-summaries.md` G7) | Source code (aggregate + start handler) and an exhaustive publisher search |
| CAP-10 Automated Order Orchestration | medium | Reachability (no gateway route, absent from both PM2 manifests) **and** saga-state durability (`AddChronicle()` with no persistence backend) are both unresolved (B1 and Q14 respectively — `service-summaries.md` G2 and G3) | Source code (saga + saga data + clients) and deployment manifests |
| CAP-11 Operation Status Projection & Real-Time Notification | high | The **wire payloads** it receives are unknowable statically — `Subscriptions.cs` emits field-less types at runtime (Q6 — `service-summaries.md` G5). The state store, previously recorded as unknown, is now resolved: see the conflict note in §6 | Source code (`OperationsService.cs`, generic handlers, `Subscriptions.cs`) + config |
| CAP-12 Asynchronous Messaging & Event Distribution | high (topology) / low (contract governance) | `messages.json` is hand-maintained with no generation or validation step, and every consumer redeclares the payload class; a publisher-side field addition reaches nobody until each copy is edited | Config (`rabbitMq` per service) + the catalogue file + per-service handler folders |
| CAP-13 Service Discovery & Load Balancing | high | Whether Consul/Fabio are actually used in production or only in the compose environment; production topology is not in this workspace | Config (all ten services) + package references + compose definitions |
| CAP-14 Platform Observability | medium | Coverage is demonstrably uneven — `ordermaker-service`, the one component that spans four capabilities, is the single service with no tracing at all | Config + package references + compose definitions + an explicit absence check |
| CAP-15 Secrets & Service-Identity Management | medium | Whether the certificate ACL is enforced or advisory (Q7), and whether the committed Vault keys and JWT signing key are live credentials (B2) | Config (ACL, `vault` sections) + one certificate-attaching client + an operational runbook |
| CAP-16 Environment & Deployment Definition | high | Which manifest governs production: the compose stacks and the PM2 manifests disagree by exactly one service (`ordermaker-service`), and no manifest is labelled production | Deployment manifests (the strongest evidence class available in this workspace) |

**Evidence-strength note.** No runtime or orchestration manifest of the strongest kind
(Kubernetes, Helm, Terraform) exists anywhere in the workspace, so the top evidence tier available
here is Docker Compose plus PM2 process manifests. Capability confidence above is therefore
anchored primarily in **source code and configuration**, with deployment manifests used for
existence and reachability claims only.

---

## 6 — Gaps, unknowns, and assumptions

### 6.1 Conflicts between existing documentation and code

Source code is authoritative. Each item below states what the earlier document claims, what the
code actually shows, and which one this baseline follows. Nothing here is silently reconciled.

**CONFLICT-01 — The operations state store is not unknown; it is Redis.**

- *Document claim.* `repo-inventory.md` §2.3 and `service-summaries.md` §2.11 both record the
  `operations-service` persistence store as **"unknown — requires runtime validation"**, and
  `service-summaries.md` raises it as its own gap G4 / its own open question Q4 (that document's
  numbering, not this one's).
- *Code reality.* `Pacco.Services.Operations.Api/Services/OperationsService.cs` injects
  `IDistributedCache` and is the only component that reads or writes operation state. It stores
  each operation under the key `requests:{id}` with
  `SlidingExpiration = TimeSpan.FromSeconds(_options.ExpirySeconds)`.
  `Infrastructure/Extensions.cs:75` registers `.AddRedis()`, and `appsettings.json:145-148` sets
  `redis.instance: "operations:"`, making the effective key `operations:requests:{id}`.
  `appsettings.json:149-151` sets `requests.expirySeconds: 300`. A repository-wide search finds
  **no `AddMongoRepository` call and no operations document type** — `Extensions.cs:71`'s
  `.AddMongo()` registers Convey's Mongo plumbing but no collection is ever bound to it.
- *Resolution.* **Code wins.** This baseline states in CAP-11 that operation state is held in
  Redis with a 300-second sliding expiry and is not durable. The prior "unknown" is superseded.
- *Residual caveat.* That Convey's `AddRedis()` binds `IDistributedCache` to Redis rather than to
  an in-memory implementation is read from the package's public contract; the Convey source is not
  in this workspace. Confidence medium-high, not absolute — see assumption A2.

**CONFLICT-02 — The async gateway directive is `use: rabbitmq`, not `use: publish`.**

- *Document claim.* `repo-inventory.md` §3.1 describes the asynchronous gateway routes as using a
  `use: publish` directive.
- *Code reality.* `grep "use: publish"` over `ntrada-async.yml` returns nothing. Counting
  directives gives `ntrada.yml` = 40 × `use: downstream`; `ntrada-async.yml` = 20 × `use:
  downstream` + **20 × `use: rabbitmq`**.
- *Resolution.* **Code wins.** CAP-02 and CAP-12 use `use: rabbitmq`. The architectural point the
  earlier document made — that half the async gateway's routes bypass HTTP entirely and publish to
  the broker — is confirmed; only the directive name was wrong.

**CONFLICT-03 — `pricing-service` does use Vault.**

- *Document claim.* `repo-inventory.md` §2.2 omits `pricing-service` from the list of services
  configured for Vault.
- *Code reality.* `Pacco.Services.Pricing.Api/appsettings.json:106-107` sets `vault.enabled: true`,
  and `Pacco.Services.Pricing.Api.csproj` references `Convey.Secrets.Vault`.
- *Resolution.* **Code wins.** CAP-15 records Vault configuration in **nine of the ten** services,
  `pricing-service` included. `ordermaker-service` remains the only host with no `vault` section at
  all, and the only one whose `*.csproj` does not reference `Convey.Secrets.Vault`.

### 6.2 Future / intended state (not implemented)

- **`Pacco.Web` — Unverifiable, missing source evidence.** The repository is in scope and cloned,
  but contains no application source: no `package.json`, no `src`, no framework manifest. Both
  prior baselines describe a web frontend as part of the platform. No capability in this document
  is owned by, or attributed to, `Pacco.Web`. The only browser-facing asset that exists in the
  workspace is the SignalR demo page inside `operations-service` (CAP-11). Whether `Pacco.Web` is
  planned, archived, or maintained elsewhere cannot be determined from the available sources.
- **`use: rabbitmq` gateway mode as the platform default.** The async routing table exists and is
  complete, but nothing in the workspace shows which mode production runs. This baseline treats
  both as *available current-state configurations*, not as a migration in progress — no evidence
  supports either direction of travel.

### 6.3 Likely gaps

- **[likely gap] CAP-09 has no automated trigger.** No service publishes a command or event that
  starts a delivery, and `deliveries-service` subscribes to no external event. `POST /deliveries`
  called by hand is the only entry point found. Either an integration is missing from the
  workspace, or the capability is genuinely manual today. (Q1; extends `service-summaries.md` G7.)
- **[likely gap] CAP-05 publishes events nobody consumes.** `vehicle_added` and `vehicle_updated`
  have no subscriber in any of the thirteen clones — only `vehicle_deleted` is consumed
  (by CAP-04). `orders-service` reads vehicle data synchronously at assignment time, so a vehicle
  price or capacity change never reaches an existing order. (Q13; extends `service-summaries.md` G9.)
- **[likely gap] CAP-10 is unreachable from the edge.** `ordermaker-service` appears in no
  `ntrada*.yml` route and in neither PM2 manifest, and exposes no HTTP host. Its `MakeOrder`
  entry point can be reached only by publishing directly to the broker. (B1; extends
  `service-summaries.md` G2.)
- **[likely gap] CAP-10 saga state is not durable.** `Extensions.cs` calls `AddChronicle()` with no
  persistence backend registered; Chronicle's default is in-memory. A restart mid-saga would strand
  in-flight orders in a partially built state with no compensation triggered. (Q14; extends
  `service-summaries.md` G3.)
- **[likely gap] CAP-12 has no contract governance.** `messages.json` is hand-maintained, with no
  generator, schema, or validation step in any build script. Every consumer redeclares the payload
  class locally — `CustomerCreated` exists in four repositories, `ResourceReserved` in three. A
  publisher-side field addition silently reaches no consumer.
- **[likely gap] No schema or data migration tooling exists anywhere.** The single schema action on
  the platform is the unique index created on `users` at `identity-service` startup. Eight other
  services own Mongo databases with no versioning, migration, or seeding path. (Q8; extends
  `service-summaries.md` G11.)
- **[likely gap] CAP-14 coverage does not extend to the saga.** `ordermaker-service` — the only
  component that spans four capabilities in a single distributed transaction — is the one service
  with no Jaeger tracing configured, so its saga emits no spans to the platform's trace store.
- **[likely gap] CAP-07's local `customers` collection is never reconciled.** `orders-service` and
  `availability-service` each keep a replica built from `customer_created`. No compensating
  subscription updates them when CAP-03 changes a customer's state or VIP flag.

### 6.4 Unknowns

- **[unknown] Which gateway configuration is authoritative.** Four `ntrada*.yml` files exist;
  `NTRADA_CONFIG` selects one; no manifest in the workspace pins a production value. (B4;
  `service-summaries.md` G6.)
- **[unknown] The wire payload of every message CAP-11 consumes.** `Subscriptions.cs` emits types
  at runtime from `messages.json` via `System.Reflection.Emit` with no properties. What
  `operations-service` actually receives on the wire cannot be determined statically. (Q6;
  `service-summaries.md` G5.)
- **[unknown] Capability maturity, platform-wide.** No maturity attribute was retrievable for any
  capability from any source available to this stage, and none is recorded in any repository. Every
  capability's maturity is unknown rather than assumed. (Q10.)
- **[unknown] Team or domain ownership, platform-wide.** No `CODEOWNERS`, `CONTRIBUTING.md`, or
  team manifest exists in any of the thirteen clones. §2 maps capabilities to **services**, which is
  evidenced; it does not map them to teams, which is not. (Q11; `service-summaries.md` G10.)
- **[unknown] Whether the certificate ACL in CAP-15 is enforced or advisory.** The ACL is declared
  in `customers-service` config and a certificate is attached by `availability-service`'s client,
  but the enforcement code lives in the Convey package, not in this workspace. (Q7.)
- **[unknown] Whether the committed Vault unseal keys, root token, and gateway JWT signing key are
  live credentials or throwaway local-development values.** They are committed in plaintext either
  way. (B2.)
- **[unknown] How the Pact contract file travels from the `orders-service` consumer test to the
  `parcels-service` provider test.** No broker URL, no shared path, and no CI step referencing one
  appears in either repository's `.travis.yml`. (Q5; `service-summaries.md` G12.)
- **[unknown] Production runtime topology.** No Kubernetes, Helm, or Terraform artifact exists in
  any clone. Scaling, replica counts, network policy, and failover behaviour for every capability
  are outside what these sources can show.
- **[unknown] Business meaning of the `Suspicious` and `Locked` customer states.** Both are
  settable and both are treated as invalid by CAP-04's reservation gate, but no code path sets
  either and no document explains when they should apply. (Q9.)

### 6.5 Assumptions

These are the same eight assumptions carried in the consolidated ABQ table at the end of this
document, in the same order and under the same ids. The ABQ table is the source of truth; this list
gives each one its in-context narrative.

- **A1 — [assumption] Each service's `rabbitMq.exchange` value names the exchange it owns.** Used
  throughout §2 to attribute publication to a capability. Consistent across the nine services that
  declare a `rabbitMq` section (`pricing-service` declares none) and matched by `messages.json`, but
  never asserted in a document.
- **A2 — [assumption] Convey's `AddRedis()` binds `IDistributedCache` to Redis.** Underpins
  CONFLICT-01 and CAP-11. Read from the package contract; the Convey source is not in the workspace.
- **A3 — [assumption] The Docker Compose stacks describe a local development environment, not
  production.** They bind single-node infrastructure with default credentials and no replication.
  No file states its target environment.
- **A4 — [assumption] Absence of a subscriber across all thirteen clones means the message has no
  consumer.** Used for the CAP-05 and CAP-09 gaps above. Valid only if no consumer lives outside
  this workspace.
- **A5 — [assumption] Capability boundaries follow deployable service boundaries where a service
  owns a distinct business responsibility.** This holds for CAP-01 and CAP-03 through CAP-11. It
  deliberately does **not** hold for CAP-12 through CAP-16, which are cross-cutting platform
  capabilities with no single owning service; each is labelled as such in §2. It is also what makes
  the `pricing-service` grouping in §2 note 4 an inference rather than evidence.
- **A6 — [assumption] A repository's presence in the clone set means it is in scope for this
  platform.** Applied to `Pacco.Web` despite it being empty, which is why it appears in §6.2 rather
  than being dropped.
- **A7 — [assumption] The role claim the gateway checks on its admin routes is the same role the
  identity service issues in its tokens.** Needed to describe CAP-02's five `role: admin` gates as
  effective. The gateway validates a token it does not mint, and no code path or test in the
  workspace proves the two role vocabularies are the same.
- **A8 — [assumption] Capability maturity and team ownership are genuinely unrecorded, not merely
  unretrieved.** No source available to this stage held either attribute, so both are reported as
  `[unknown]` platform-wide in §6.4 rather than estimated.

---

## 7 — Architectural characteristics

Observed structural properties of each capability as it exists today. This section describes what
is, not what should change.

### CAP-01 — Identity & Access Management

- **Coupling / isolation.** Highest isolation on the platform. Zero outbound calls, zero inbound
  subscriptions, its own Mongo database, and one outbound event (`signed_up`).
- **Boundary clarity.** Sharp. Full four-project split; every rule lives in `Core` or in
  `Application/Services/Identity`; nothing else on the platform reads the `users` collection.
- **Dependency surface.** Mongo, Vault, Consul, Fabio, Jaeger, RabbitMQ (publish-only). Downstream
  dependants: none directly — CAP-02 depends on the token *format*, not on the service.
- **Observable quality signals.** The only service that creates a database index at startup. Its
  token contract with CAP-02 is a shared symmetric key committed in two repositories, verified by
  no test.

### CAP-02 — Edge Routing & Access Enforcement

- **Coupling / isolation.** Structurally isolated (no shared code with any service) but
  configurationally coupled to every one of them: 40 routes hard-code downstream service names and
  URL paths.
- **Boundary clarity.** Clear as a component, ambiguous as a configuration — four routing tables
  exist and none is marked authoritative.
- **Dependency surface.** Every HTTP-exposed service, plus Fabio, Consul and RabbitMQ. Highest
  fan-out of any component in the workspace.
- **Observable quality signals.** Zero C# business logic — the entire capability is declarative
  YAML, so a routing change requires no build. CORS is configured as `allowedOrigins: ['*']`
  together with `allowCredentials: true`, and the token signing key is committed in the repository.

### CAP-03 — Customer Profile & Lifecycle Management

- **Coupling / isolation.** Owns its data and its rules, but is read synchronously by two
  capabilities (CAP-04, CAP-08) and replicated asynchronously into two more. It is the platform's
  most-depended-upon business capability.
- **Boundary clarity.** Sharp in code — the state machine and the VIP policy are both in `Core`.
  Blurred at runtime, because two other services hold their own `customers` collections that this
  capability never updates.
- **Dependency surface.** Inbound: `signed_up` (CAP-01), `order_completed` (CAP-07), two synchronous
  HTTP readers. Outbound: none. It calls nobody.
- **Observable quality signals.** The only service carrying a certificate ACL. The VIP threshold is
  isolated in a single named policy class rather than inlined in a handler.

### CAP-04 — Resource Availability & Reservation

- **Coupling / isolation.** Mixed by design: asynchronous for vehicle and customer facts,
  **synchronous and blocking** for the customer-state check inside `ReserveResourceHandler`. A
  `customers-service` outage fails reservations even though a local replica exists.
- **Boundary clarity.** The richest domain boundary in the workspace. Expropriation, tag validation
  and cascade-on-delete all live in the aggregate root; handlers only orchestrate.
- **Dependency surface.** Inbound: `vehicle_deleted`, `customer_created`, plus commands from CAP-02
  and CAP-10. Outbound: HTTP to CAP-03; events consumed by CAP-07 and CAP-11.
- **Observable quality signals.** Five test projects — the deepest coverage on the platform,
  including an NBomber load-test project. Transactional outbox enabled with transactions disabled.

### CAP-05 — Vehicle Fleet Catalogue

- **Coupling / isolation.** Nearly standalone. Calls nothing, subscribes to nothing. Two of its
  three published events have no consumer, so it is effectively read-only to the rest of the
  platform via HTTP.
- **Boundary clarity.** Sharp and narrow — a validated catalogue aggregate and five endpoints.
- **Dependency surface.** Read synchronously by CAP-07 and CAP-10. `vehicle_deleted` consumed by
  CAP-04.
- **Observable quality signals.** Invariants are constructor-enforced, so an invalid vehicle cannot
  be constructed. No test project.

### CAP-06 — Parcel Catalogue & Volume Calculation

- **Coupling / isolation.** Well isolated, with one asymmetric edge: it holds an `OrderId` on the
  parcel, so it stores a reference to CAP-07's aggregate without subscribing to its lifecycle.
- **Boundary clarity.** Clear. The catalogue and the volume query are cleanly separated; volume is
  a query handler, not an entity method.
- **Dependency surface.** Read synchronously by CAP-07 (existence check, volume). Publishes parcel
  events consumed by CAP-11 and CAP-10.
- **Observable quality signals.** The provider half of the platform's only contract test. The only
  service whose `Core` project has no `AggregateRoot.cs` — the aggregate base is defined inline.

### CAP-07 — Order Lifecycle Management

- **Coupling / isolation.** The most coupled capability on the platform: three synchronous HTTP
  clients, seven external event subscriptions, and a replicated `customers` collection. Its state
  is advanced by events from two other capabilities (CAP-04 approves, CAP-09 delivers).
- **Boundary clarity.** The aggregate boundary is clear and every transition is guarded in `Order`.
  The *capability* boundary is not: an order becomes `Approved` because of something CAP-04 did,
  and `Completed` because of something CAP-09 did.
- **Dependency surface.** Widest of any service — outbound to Parcels, Pricing and Vehicles;
  inbound from Availability, Deliveries, Customers and the gateway.
- **Observable quality signals.** Consumer half of the Pact test. Price is captured once at vehicle
  assignment and never recomputed, so the stored total can diverge from what CAP-08 would return
  later.

### CAP-08 — Order Pricing & Discounting

- **Coupling / isolation.** Stateless and side-effect-free: no database, no message broker, one
  outbound HTTP call, one endpoint. The only capability that participates in no messaging at all.
- **Boundary clarity.** Clear responsibility, weak internal boundary — it is a single-project
  service with `Core`, `Queries` and `Services` as folders, so nothing structural separates the
  pricing rule from transport concerns.
- **Dependency surface.** Outbound: CAP-03 only. Inbound: CAP-07 and the gateway.
- **Observable quality signals.** Discount tiers are magic numbers in one class with no
  configuration key and no test. Its `.csproj` references
  `Convey.Tracing.Jaeger.RabbitMQ` while the service has no `rabbitMq` configuration — an unused
  dependency.

### CAP-09 — Delivery Execution & Tracking

- **Coupling / isolation.** Publishes into CAP-07 but consumes from nobody — a one-directional edge.
  It holds `OrderId` as a plain value with no lookup against CAP-07, so it will happily track a
  delivery for an order that does not exist.
- **Boundary clarity.** Clean aggregate with guarded transitions, but the capability's *entry* is
  unclear: nothing in the workspace initiates it.
- **Dependency surface.** Minimal. Mongo, RabbitMQ (publish-only), and the standard platform
  services. No HTTP clients.
- **Observable quality signals.** Registration is a value object with structural equality, making
  duplicate scan submissions naturally idempotent. Its `.csproj` omits `Convey.WebApi.Security`,
  which every other business service references.

### CAP-10 — Automated Order Orchestration

- **Coupling / isolation.** Maximum coupling by construction: it drives CAP-05, CAP-06, CAP-04 and
  CAP-07 through both HTTP and messaging, and its saga data mirrors state owned by each of them.
- **Boundary clarity.** The saga is a single readable file, so the *choreography* is unusually
  clear. The *deployment* boundary is not — no gateway route, no PM2 entry, no HTTP host.
- **Dependency surface.** Broadest reach, narrowest exposure. Reachable only from the broker.
- **Observable quality signals.** Compensation is implemented for exactly one of five steps. The
  saga runs under an empty identity context, so it bypasses the per-caller identity gates that
  every other path enforces. It is configured with no `jaeger` section, no `vault` section, and a
  single-URL Fabio block, where the other nine services each carry all three.

### CAP-11 — Operation Status Projection & Real-Time Notification

- **Coupling / isolation.** Coupled to every capability's message contracts and to none of their
  code. It subscribes to the entire platform through one generic handler rather than nine specific
  ones.
- **Boundary clarity.** Very clear responsibility, deliberately opaque implementation — types are
  emitted at runtime, so the boundary cannot be inspected statically.
- **Dependency surface.** Redis (state + SignalR backplane), RabbitMQ (all exchanges), and three
  outbound protocols: REST, SignalR, gRPC.
- **Observable quality signals.** Terminal states are protected against overwrite, so a late
  duplicate cannot resurrect a completed operation. State is held only in a cache with a 300-second
  sliding expiry, and a message arriving without a correlation id returns early without being
  recorded or logged.

### CAP-12 — Asynchronous Messaging & Event Distribution

- **Coupling / isolation.** The platform's principal decoupling mechanism, and simultaneously its
  most widely shared dependency — a broker outage disables nine services' integration paths.
- **Boundary clarity.** Topology is clear (one topic exchange per service, uniform naming and queue
  templates). Contract ownership is unclear — the catalogue lives inside `operations-service`, a
  consumer.
- **Dependency surface.** Every service except `pricing-service`.
- **Observable quality signals.** Uniform conventions across the nine services that declare a
  `rabbitMq` section. Transactional inbox/outbox configured in seven. No schema registry, no
  contract validation, and payload classes duplicated per consumer.

### CAP-13 — Service Discovery & Load Balancing

- **Coupling / isolation.** Ambient. Every service registers itself and resolves peers through it,
  but no business code references it — it is entirely configuration and package wiring.
- **Boundary clarity.** Clear and uniform: one `consul` block, one `fabio` block, one
  `httpClient.services` map per service.
- **Dependency surface.** All ten services plus the gateway.
- **Observable quality signals.** Uniform across the platform with one exception —
  `ordermaker-service` carries a single-URL Fabio configuration where every other service carries
  two.

### CAP-14 — Platform Observability

- **Coupling / isolation.** Ambient and non-functional; no service depends on it to operate.
- **Boundary clarity.** Clear per-signal separation: Jaeger for traces, Seq for logs, Prometheus
  and Grafana for metrics.
- **Dependency surface.** All services, plus the gateway's correlation and span context builders.
- **Observable quality signals.** Log masking is configured per service via excluded-property
  lists. Trace propagation reaches across the broker through `span_context` headers. Coverage is
  complete except for `ordermaker-service`, which has no tracing at all.

### CAP-15 — Secrets & Service-Identity Management

- **Coupling / isolation.** Split across two mechanisms with no shared implementation: Vault
  configuration in nine services, and a certificate ACL in exactly one.
- **Boundary clarity.** Weak. The ACL is declared in `customers-service`'s `appsettings.json`
  alongside ordinary application settings, and enforcement lives in a third-party package.
- **Dependency surface.** Vault, plus the certificate-attaching HTTP client in
  `availability-service`.
- **Observable quality signals.** PKI issuance and Mongo dynamic credentials are documented as an
  operational runbook. In the same file, five unseal keys and a root token are committed in
  plaintext, as is the gateway's JWT signing key in two repositories.

### CAP-16 — Environment & Deployment Definition

- **Coupling / isolation.** Centralised in the `Pacco` repository, with per-service Dockerfiles and
  CI scripts duplicated across the twelve others.
- **Boundary clarity.** Clear file-level separation by infrastructure concern. Unclear at the
  environment level — no manifest declares which environment it targets.
- **Dependency surface.** Docker, Docker Compose, PM2, Travis CI, Docker Hub.
- **Observable quality signals.** Every service builds identically through the same four scripts.
  The compose stacks and the PM2 manifests disagree by one service, and no manifest is marked
  production.

---

## 8 — Behavioral constraints

Rules that govern how each capability behaves, each one evidenced in code. Structural facts already
covered in §7 are not repeated here, and framework-internal mechanics (locking, retry, caching
strategy, serialization) are excluded unless they are externally observable to a caller.

Capabilities with no evidenced behavioural rules of their own — CAP-13 (Service Discovery & Load
Balancing) and CAP-16 (Environment & Deployment Definition) — are omitted rather than filled with
inference.

| Capability | Behavioral Constraint | Type | Confidence | Evidence |
|------------|----------------------|------|------------|----------|
| CAP-01 | Identity & Access Management rejects a sign-up whose email is already registered, before any user record is written | validation | high | `IdentityService.cs` `SignUpAsync` → `EmailInUseException` |
| CAP-01 | Identity & Access Management assigns the role `user` when a sign-up request names no role | invariant | high | `IdentityService.cs` `SignUpAsync`, role defaults to `"user"` |
| CAP-01 | Identity & Access Management stores an email and a role only in lower case, so credentials differing solely by case resolve to the same account | invariant | high | `User.cs` constructor lower-cases both values |
| CAP-01 | Identity & Access Management returns the same failure for an unknown email and for a wrong password, so a caller cannot tell which was wrong | invariant | high | `IdentityService.cs` `SignInAsync` → `InvalidCredentialsException` in both branches |
| CAP-01 | Identity & Access Management refuses a refresh token that has already been revoked, and refuses one it has never issued | state-transition | high | `RefreshTokenService.cs` `UseAsync` → `RevokedRefreshTokenException` / `InvalidRefreshTokenException` |
| CAP-01 | Identity & Access Management returns the caller's existing refresh token unchanged when issuing a new access token, so the refresh token stays valid until explicitly revoked | invariant | high | `RefreshTokenService.cs` `UseAsync` returns the same token; no rotation path exists |
| CAP-02 | Edge Routing & Access Enforcement requires an administrator claim on five routes, and rejects the request at the edge before any service is called | approval | high | `ntrada.yml` and `ntrada-async.yml`, 5 × `role: admin` in each |
| CAP-02 | Edge Routing & Access Enforcement substitutes the caller's own identity into routes bound with `@user_id`, so a caller cannot request another user's data by changing the URL | invariant | high | `ntrada.yml:143` `customers/@user_id`; `:298` `orders?customerId=@user_id` |
| CAP-02 | Edge Routing & Access Enforcement stamps every inbound request with a correlation identifier, which is what allows the request to be tracked afterwards | invariant | high | `CorrelationContextBuilder.cs`, `HttpRequestHook.cs` |
| CAP-03 | Customer Profile & Lifecycle Management accepts a registration only for a customer still in the `Incomplete` state, and rejects a second attempt | state-transition | high | `Customer.cs` `CompleteRegistration` → `CannotChangeCustomerStateException` |
| CAP-03 | Customer Profile & Lifecycle Management refuses a registration with a blank name or a blank address | validation | high | `Customer.cs` → `InvalidCustomerFullNameException`, `InvalidCustomerAddressException` |
| CAP-03 | Customer Profile & Lifecycle Management grants VIP status once a customer reaches twenty completed orders, and never removes it afterwards | invariant | high | `VipPolicy.cs` threshold of 20; `SetVip` has no inverse |
| CAP-03 | Customer Profile & Lifecycle Management ignores a repeated VIP grant, so re-evaluating an already-VIP customer changes nothing and raises no event | invariant | high | `Customer.cs` `SetVip` returns early when `IsVip` |
| CAP-03 | Customer Profile & Lifecycle Management creates a customer record automatically when a new user signs up, without any separate customer-creation request | state-transition | high | `SignedUpHandler.cs` |
| CAP-03 | Customer Profile & Lifecycle Management records a completed order against a customer only when the order carries a real identifier | validation | high | `Customer.cs` `AddCompletedOrder` ignores `Guid.Empty` |
| CAP-04 | Resource Availability & Reservation allows at most one reservation per resource per calendar day, regardless of the time of day requested | invariant | high | `Reservation.cs` equality and hash code use `DateTime.Date` only |
| CAP-04 | Resource Availability & Reservation lets a higher-priority reservation take a day already booked at a lower priority, cancelling the existing one | state-transition | high | `Resource.cs` `AddReservation` removes the colliding reservation and raises `ReservationCanceled` |
| CAP-04 | Resource Availability & Reservation refuses a reservation whose priority is not higher than the reservation already holding that day | invariant | high | `Resource.cs` → `CannotExpropriateReservationException` when `collidingReservation.Priority >= reservation.Priority` |
| CAP-04 | Resource Availability & Reservation refuses a reservation for a customer whose account is not in a valid state, checked against Customer Profile & Lifecycle Management at the moment of reservation | cross-service | high | `ReserveResourceHandler.cs` → `InvalidCustomerStateException` after `GetStateAsync` |
| CAP-04 | Resource Availability & Reservation allows a caller to reserve only for themselves, unless the caller is an administrator | approval | high | `ReserveResourceHandler.cs:27-35` → `UnauthorizedResourceAccessException` |
| CAP-04 | Resource Availability & Reservation cancels every outstanding reservation on a resource when that resource is deleted, announcing each cancellation separately | state-transition | high | `Resource.cs` `Delete` raises `ReservationCanceled` per reservation, then `ResourceDeleted` |
| CAP-04 | Resource Availability & Reservation requires at least one tag on a resource and rejects blank tags | validation | high | `Resource.cs` `ValidateTags` → `MissingResourceTagsException`, `InvalidResourceTagsException` |
| CAP-05 | Vehicle Fleet Catalogue refuses a vehicle whose payload capacity or loading capacity is not greater than zero | validation | high | `Vehicle.cs` → `InvalidVehicleCapacityException` |
| CAP-05 | Vehicle Fleet Catalogue refuses a vehicle priced at zero or below | validation | high | `Vehicle.cs` → `InvalidVehiclePricePerServiceException` |
| CAP-05 | Vehicle Fleet Catalogue gives every new vehicle the `Standard` variant, whatever variants were requested | invariant | high | `Vehicle.cs` constructor always calls `AddVariants(Variants.Standard)` |
| CAP-06 | Parcel Catalogue & Volume Calculation refuses a parcel with a blank name or a blank description | validation | high | `Parcel.cs` → `InvalidParcelNameException`, `InvalidParcelDescriptionException` |
| CAP-06 | Parcel Catalogue & Volume Calculation reports a volume of zero when asked about an empty set of parcels, rather than failing | exception | high | `GetParcelsVolumeHandler.cs:25-43` short-circuits on null or empty input |
| CAP-06 | Parcel Catalogue & Volume Calculation marks a parcel as belonging to an order by recording that order on the parcel, and clears it when the parcel is removed | state-transition | high | `Parcel.cs` `AddToOrder` / `DeleteFromOrder`; `AddedToOrder => OrderId.HasValue` |
| CAP-07 | Order Lifecycle Management allows an order to be deleted only while it is still `New` | state-transition | high | `Order.cs` `CanBeDeleted => Status == OrderStatus.New` |
| CAP-07 | Order Lifecycle Management refuses to assign a vehicle to an order that has no parcels | sequencing | high | `AssignVehicleToOrderHandler.cs:44` → `OrderHasNoParcelsException` |
| CAP-07 | Order Lifecycle Management accepts a vehicle assignment only for an order that is `New` or `Canceled`, and ignores the request otherwise without reporting an error | state-transition | high | `Order.cs` `CanAssignVehicle`; `AssignVehicleToOrderHandler.cs:49` returns silently |
| CAP-07 | Order Lifecycle Management sets an order's total price only while the order is `New`, and rejects a negative price | invariant | high | `Order.cs` `SetTotalPrice` → `CannotChangeOrderPriceException`, `InvalidOrderPriceException` |
| CAP-07 | Order Lifecycle Management refuses to add the same parcel to an order twice | validation | high | `Order.cs` `AddParcel` → `ParcelAlreadyAddedToOrderException` |
| CAP-07 | Order Lifecycle Management approves an order in response to a successful resource reservation, not in response to a direct request from the customer | cross-service | high | `ResourceReservedHandler.cs` matches on resource and date, then calls `order.Approve()` |
| CAP-07 | Order Lifecycle Management moves an order to `Delivering` only from `Approved`, and completes it only from `Delivering` | state-transition | high | `Order.cs` `SetDelivering`, `Complete` → `CannotChangeOrderStateException` |
| CAP-07 | Order Lifecycle Management refuses to cancel an order that is already completed or already cancelled | state-transition | high | `Order.cs` `Cancel` guards on `Completed` and `Canceled` |
| CAP-08 | Order Pricing & Discounting gives a ten percent discount at ten or more completed orders, five percent above three, and two percent above zero | invariant | high | `CustomerDiscountsService.cs` tier thresholds |
| CAP-08 | Order Pricing & Discounting adds a further ten percent for a VIP customer, on top of whichever completed-order tier applies | invariant | high | `CustomerDiscountsService.cs` `if (customer.IsVip) discount += 0.1m` |
| CAP-08 | Order Pricing & Discounting charges the full order price when the calculated discounted price is not above zero | exception | high | `GetOrderPricingHandler.cs:45` falls back to `query.OrderPrice` |
| CAP-08 | Order Pricing & Discounting refuses to price an order for a customer it cannot find | cross-service | high | `GetOrderPricingHandler.cs:29-32` → `CustomerNotFoundException` |
| CAP-09 | Delivery Execution & Tracking refuses to start a second delivery for an order that already has one in progress or completed | sequencing | high | `StartDeliveryHandler.cs:28-34` → `DeliveryAlreadyStartedException` |
| CAP-09 | Delivery Execution & Tracking allows a delivery that previously failed to be started again | state-transition | high | `StartDeliveryHandler.cs:28` excludes `DeliveryStatus.Failed` from the guard |
| CAP-09 | Delivery Execution & Tracking accepts a tracking registration only while the delivery is in progress | state-transition | high | `Delivery.cs` `AddRegistration` → `CannotAddDeliveryRegistrationException` |
| CAP-09 | Delivery Execution & Tracking refuses to complete or fail a delivery that has already completed or already failed | state-transition | high | `Delivery.cs` `Complete` / `Fail` → `CannotChangeDeliveryStateException` |
| CAP-09 | Delivery Execution & Tracking treats two registrations with the same description and timestamp as the same registration, so a repeated scan does not duplicate the record | invariant | high | `DeliveryRegistration.cs` value equality on description and timestamp |
| CAP-10 | Automated Order Orchestration holds the order at the parcel stage until every parcel it intended to add has been confirmed added | sequencing | high | `AIOrderMakingSaga.cs` returns early until `Data.AllPackagesAddedToOrder` |
| CAP-10 | Automated Order Orchestration selects the vehicle and the reservation slot itself, without any input from the customer | invariant | high | `AIOrderMakingSaga.cs` calls `GetBestAsync()` and `GetBestAsync(VehicleId)` |
| CAP-10 | Automated Order Orchestration identifies every step of a run by the order it belongs to, so messages for different orders never interfere | invariant | high | `AIOrderMakingSaga.cs` `ResolveId` returns `OrderId` for all five messages |
| CAP-10 | Automated Order Orchestration cancels the order it created if the run fails after a parcel has been added, and takes no compensating action for a failure at any other step | exception | high | `AIOrderMakingSaga.cs` — `CompensateAsync` is implemented only for `ParcelAddedToOrder` |
| CAP-10 | Automated Order Orchestration acts with no user identity attached, so its requests are not subject to the per-caller ownership checks that apply to customer requests | cross-service | high | `AIOrderMakingSaga.cs` constructor sets `CorrelationContext.User = new CorrelationContext.UserContext()` |
| CAP-10 | Automated Order Orchestration marks each message it publishes as pending, completed, or rejected, which is how progress becomes visible to the customer | invariant | high | `AIOrderMakingSaga.cs` sets `headers[SagaHeader]` from `SagaStates` |
| CAP-11 | Operation Status Projection & Real-Time Notification never overwrites an operation that has already completed or been rejected | invariant | high | `OperationsService.cs` `TrySetAsync` returns `(false, operation)` for terminal states |
| CAP-11 | Operation Status Projection & Real-Time Notification ignores any message that arrives without a correlation identifier, recording nothing for it | exception | high | `GenericEventHandler.cs` returns early when `messageProperties?.CorrelationId` is blank |
| CAP-11 | Operation Status Projection & Real-Time Notification treats a message carrying no explicit progress marker as a completed operation | invariant | high | `GenericEventHandler.cs` — `GetSagaState() ?? OperationState.Completed` |
| CAP-11 | Operation Status Projection & Real-Time Notification notifies connected clients only when the stored status actually changed | invariant | high | `GenericEventHandler.cs` pushes to the hub only `if (updated)` |
| CAP-11 | Operation Status Projection & Real-Time Notification forgets an operation five minutes after it was last read or written | invariant | high | `OperationsService.cs` sliding expiry from `requests.expirySeconds`; `appsettings.json:149-151` = `300` |
| CAP-12 | Asynchronous Messaging & Event Distribution carries a correlation identifier and a trace context on every message, which is what lets a request be followed across services | invariant | high | `message_context` / `span_context` headers; gateway context builders |
| CAP-12 | Asynchronous Messaging & Event Distribution records an outgoing message in the sending service's own database before it reaches the broker, in seven of the ten services | invariant | medium-high | `outbox` enabled in seven `appsettings.json` files; Convey implements the mechanism |
| CAP-14 | Platform Observability omits configured sensitive fields from logged message payloads | invariant | high | per-service excluded-property lists in each `appsettings.json` logger section |
| CAP-15 | Secrets & Service-Identity Management accepts a caller certificate only from a permitted issuer and domain, and grants only the permissions named for that caller | approval | medium | `Customers.Api/appsettings.json:164-178` ACL; enforcement lives in the Convey package |

---

## Service Lookup Index

> Compact cross-reference for AI-agent and tooling consumption. For full capability descriptions and evidence, see Sections 1–4 above.

Row names are **deployable names** as they appear in the repositories — the `container_name` /
`image` entries in `Pacco/compose/services.yml` for the eleven containerised deployables, the
project name for the one non-containerised deployable, and the repository name for the one
in-scope component that ships no deployable of its own. Repository names (`Pacco.Services.Orders`)
are bound as aliases in Notes, never used as the row name.

`Pacco.Web` is **excluded**: it maps to no capability in this baseline. Its entire tracked content
is `README.md` (verified — `git ls-files` returns exactly one path), so there is no evidence to map.
It remains in scope for the platform per assumption A6.

| Service / Repo | Capabilities Owned (Primary) | Capabilities Supported (Secondary) | Confidence | Notes |
|----------------|------------------------------|-------------------------------------|------------|-------|
| `api-gateway` | CAP-02 Edge Routing & Access Enforcement | CAP-01 Identity & Access Management, CAP-11 Operation Status Projection & Real-Time Notification, CAP-12 Asynchronous Messaging & Event Distribution, CAP-14 Platform Observability | medium | Repo `Pacco.APIGateway`. Validates a JWT it cannot issue and generates the correlation id CAP-11 keys on. Sync/async is a whole-config swap (`NTRADA_CONFIG`), not a per-route flag, so 20 write routes change transport between the two modes — which config is authoritative is unresolved (B4) |
| `identity-service` | CAP-01 Identity & Access Management | CAP-12 Asynchronous Messaging & Event Distribution | high | Repo `Pacco.Services.Identity`. Sole credential issuer on the platform; no boundary ambiguity. Accepts `sign_up` as a broker command as well as over HTTP |
| `customers-service` | CAP-03 Customer Profile & Lifecycle Management | CAP-08 Order Pricing & Discounting, CAP-12 Asynchronous Messaging & Event Distribution, CAP-15 Secrets & Service-Identity Management | medium | Repo `Pacco.Services.Customers`. Weakest link is CAP-15: it is the platform's only certificate-ACL **enforcement point**, and whether the ACL is enforced or merely declared is unresolved (Q7). Its `customers` data is replicated into `orders-service` and `parcels-service` and never reconciled (Q4) |
| `availability-service` | CAP-04 Resource Availability & Reservation | CAP-03 Customer Profile & Lifecycle Management, CAP-05 Vehicle Fleet Catalogue, CAP-07 Order Lifecycle Management, CAP-12 Asynchronous Messaging & Event Distribution, CAP-15 Secrets & Service-Identity Management | medium | Repo `Pacco.Services.Availability`. Weakest link is CAP-15 (Q7): it is the only certificate-presenting caller on the platform. Its `resource_reserved` event is what approves an order, so it writes into CAP-07's lifecycle without owning it |
| `vehicles-service` | CAP-05 Vehicle Fleet Catalogue | CAP-12 Asynchronous Messaging & Event Distribution | high | Repo `Pacco.Services.Vehicles`. Reference data only — it reserves nothing; date-level availability is CAP-04's. `vehicle_added` and `vehicle_updated` have no subscriber anywhere in the thirteen clones (Q13) |
| `parcels-service` | CAP-06 Parcel Catalogue & Volume Calculation | CAP-03 Customer Profile & Lifecycle Management, CAP-07 Order Lifecycle Management, CAP-12 Asynchronous Messaging & Event Distribution | medium | Repo `Pacco.Services.Parcels`. Weakest link is the CAP-03 replica semantics: it holds a `customers` copy written once from `customer_created` and never updated (Q4). Pact **provider** for the CAP-06/CAP-07 boundary |
| `orders-service` | CAP-07 Order Lifecycle Management | CAP-03 Customer Profile & Lifecycle Management, CAP-04 Resource Availability & Reservation, CAP-05 Vehicle Fleet Catalogue, CAP-06 Parcel Catalogue & Volume Calculation, CAP-08 Order Pricing & Discounting, CAP-09 Delivery Execution & Tracking, CAP-12 Asynchronous Messaging & Event Distribution | medium | Repo `Pacco.Services.Orders`. The platform's hub and its heaviest coupling surface — 7 commands, 9 events, 10 rejected events, 3 sync callees, 7 external event subscriptions. Weakest link is the CAP-03 replica semantics (Q4). Pact **consumer** |
| `pricing-service` | CAP-08 Order Pricing & Discounting | CAP-03 Customer Profile & Lifecycle Management | high | Repo `Pacco.Services.Pricing`. Boundary ambiguity is real but does not weaken ownership: it shares none of the platform's conventions — verified to have no `rabbitMq`, no `mongo` and no `redis` section — so its grouping under a customer/commercial subsystem is an inference (A5), not evidence. Owns no data; reads CAP-03 on every call |
| `deliveries-service` | CAP-09 Delivery Execution & Tracking | CAP-07 Order Lifecycle Management, CAP-12 Asynchronous Messaging & Event Distribution | medium | Repo `Pacco.Services.Deliveries`. Ownership is high; medium reflects the initiation path — nothing in any repository publishes `start_delivery` or calls the service, so its trigger is unevidenced (Q1). Its three events are what close CAP-07's lifecycle |
| `ordermaker-service` | CAP-10 Automated Order Orchestration | CAP-04 Resource Availability & Reservation, CAP-05 Vehicle Fleet Catalogue, CAP-06 Parcel Catalogue & Volume Calculation, CAP-07 Order Lifecycle Management, CAP-12 Asynchronous Messaging & Event Distribution | medium | Repo `Pacco.Services.OrderMaker`. Commands four capabilities that do not know it exists — the orchestration edge is one-directional and invisible from the owning side. Reachability unproven: no gateway route, absent from both PM2 manifests (B1). Verified to have no `vault` and no `jaeger` section, so it sits outside CAP-15 and emits no traces into CAP-14 |
| `operations-service` | CAP-11 Operation Status Projection & Real-Time Notification | CAP-12 Asynchronous Messaging & Event Distribution, CAP-14 Platform Observability | low | Repo `Pacco.Services.Operations`. **The `low` reflects CAP-12 only, not CAP-11** — ownership of CAP-11 is `high`. This service hosts `messages.json`, the platform-wide message catalogue, which is a hand-maintained copy of names owned by eight *other* repositories with no generation or validation step; §2 rates that catalogue ownership `low`. A message added anywhere is invisible to CAP-11 until someone edits that file |
| `Pacco.Services.Operations.GrpcClient` | — | CAP-11 Operation Status Projection & Real-Time Notification | high | Second deployable of repo `Pacco.Services.Operations` (`OutputType: Exe`, `Protobuf Include="Operations.proto"`). A non-containerised console **client** of the CAP-11 gRPC surface — it owns no capability and appears in no compose stack or PM2 manifest. Whether downstream stages should treat it as a platform component or a sample is open (Q16) |
| `Pacco` | CAP-13 Service Discovery & Load Balancing, CAP-14 Platform Observability, CAP-15 Secrets & Service-Identity Management, CAP-16 Environment & Deployment Definition | CAP-02 Edge Routing & Access Enforcement, CAP-12 Asynchronous Messaging & Event Distribution | medium | Ships **no runtime deployable of its own** — the row name is the repository name because there is none to use. Owns CAP-13/14/15 *by definition* while all ten services own them *by participation*, and participation is uneven (no Jaeger and no Vault in `ordermaker-service`; `Convey.WebApi.Security` only in Availability and Customers). Medium reflects CAP-14 coverage and CAP-15 enforcement (Q7) |

**Capability with no owning component.** CAP-12 Asynchronous Messaging & Event Distribution appears
in the Secondary column of eleven rows and the Primary column of none. This is not an omission: §2
records that it has no single owner — nine services each own one exchange, and the catalogue that
defines what CAP-11 can observe lives inside a service that owns none of the names in it. Tooling
that batches over this index will find CAP-12 unassigned; that is the evidenced state (Q15).

### Ownership statements

`api-gateway` implements the Edge Routing & Access Enforcement capability.

`identity-service` implements the Identity & Access Management capability.

`customers-service` implements the Customer Profile & Lifecycle Management capability.

`availability-service` implements the Resource Availability & Reservation capability.

`vehicles-service` implements the Vehicle Fleet Catalogue capability.

`parcels-service` implements the Parcel Catalogue & Volume Calculation capability.

`orders-service` implements the Order Lifecycle Management capability.

`pricing-service` implements the Order Pricing & Discounting capability.

`deliveries-service` implements the Delivery Execution & Tracking capability.

`ordermaker-service` implements the Automated Order Orchestration capability.

`operations-service` implements the Operation Status Projection & Real-Time Notification capability.

`Pacco` implements the Service Discovery & Load Balancing capability.

`Pacco` implements the Platform Observability capability.

`Pacco` implements the Secrets & Service-Identity Management capability.

`Pacco` implements the Environment & Deployment Definition capability.

Components in scope: api-gateway, identity-service, customers-service, availability-service, vehicles-service, parcels-service, orders-service, pricing-service, deliveries-service, ordermaker-service, operations-service, Pacco.Services.Operations.GrpcClient, Pacco

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything in this baseline that is not directly proven by code or configuration is collected
> here. Each assumption states what was taken as true and what breaks if it is wrong. Each blocker
> and open question is tagged either **[ACTION NOW]** — someone must answer it before the next
> stage can rely on this document — or **[handled later by \<stage\>]**, meaning a named later
> stage will resolve it and no action is needed now.

### Assumptions

| ID | Assumption | Why it was needed | Impact if wrong | Validation Path |
|----|------------|-------------------|-----------------|-----------------|
| A1 | Each service's `rabbitMq.exchange` value names the exchange that service owns and publishes to | Used to attribute every published event to an owning capability in §2 | Ownership attribution in §2 would shift for the affected capability, and the messaging topology in CAP-12 would be misdrawn | Inspect the live RabbitMQ management API (`GET /api/exchanges`) and compare declared exchanges and their bindings against the `rabbitMq.exchange` value in each service's `appsettings.json` |
| A2 | Convey's `AddRedis()` makes `IDistributedCache` resolve to Redis rather than to an in-memory store | Underpins the finding that operation status lives in Redis (CONFLICT-01, CAP-11) | Operation status would be per-instance and lost on every restart, making CAP-11 far less reliable than described | Read the `Convey.Persistence.Redis` 0.4.x source or decompiled package to confirm the `IDistributedCache` registration; or run `operations-service`, drive one operation, and check for an `operations:requests:{id}` key in Redis |
| A3 | The Docker Compose stacks describe a local development environment, not production | Needed because no manifest declares its target environment | Production would be running single-node infrastructure with default credentials, changing the risk picture for CAP-15 and CAP-16 | Ask a platform operator which manifest deploys production, or obtain the production deployment definition (resolved together with B3) |
| A4 | A message with no subscriber anywhere in the thirteen cloned repositories genuinely has no consumer | Basis for the unconsumed-event gaps on CAP-05 and the no-trigger gap on CAP-09 | Those gaps would be false, and consumers would exist in code outside this workspace | Inspect queue bindings for the `vehicles` and `deliveries` exchanges on the running broker; a binding with no repository behind it proves a consumer outside this workspace |
| A5 | Capability boundaries follow deployable service boundaries where a service owns a distinct business responsibility | The organising principle for CAP-01 and CAP-03 through CAP-11 | Capabilities would need regrouping, though the underlying evidence per service stays valid | Review this capability list with a platform architect or domain owner; a capability catalogue, if one is ever produced, supersedes it |
| A6 | `Pacco.Web` being present in the clone set means it is in scope for this platform even though it holds no source | Kept it visible as unverifiable rather than dropping it silently | A frontend capability exists that this baseline does not cover at all | Check the repository's commit history and default branch upstream, and ask the scope owner for issue 12998 whether a web client exists elsewhere |
| A7 | The role claim the gateway checks on its admin routes is the same role the identity service issues in its tokens | Needed to describe the admin gates in CAP-02 as effective | Admin-only routes would either reject everyone or admit everyone, and CAP-02's access enforcement would not work as described | Decode a token minted by `identity-service` for an admin user and confirm the claim name and value match the `claims.role` gate in `ntrada.yml`; or call one admin route end to end |
| A8 | Capability maturity and team ownership are genuinely unrecorded, not merely unretrieved | No source available to this stage held either attribute | Both would be recoverable from a system this stage did not reach, and the two platform-wide unknowns in §6.4 would be answerable | Query an enterprise capability catalogue or portfolio system and an engineering directory, if either exists for this organisation (see Q10, Q11) |
| A9 | A synchronous HTTP call to a capability's owner, a subscription to one of its events, or a command published onto its exchange is enough to record the caller as a **secondary contributor** to that capability in the Service Lookup Index | The Secondary column needed a stated rule; code shows the edge but never labels it as contribution | The Secondary column would over-report: a service that merely reads another's data would be indistinguishable from one that genuinely shares responsibility for the capability, and tooling batching over the index would follow edges that carry no ownership | Review the Secondary column with a platform architect against §3.1 and §3.2 of `repo-inventory.md`, and split "consumes" from "contributes to" if the distinction matters downstream |

### Blockers

**On the Owner column.** No repository in this workspace records a named owner for anything — there
is no `CODEOWNERS`, `CONTRIBUTING.md`, or team manifest in any of the thirteen clones (see Q11).
The Owner column therefore names the **role that must supply the answer**, and assigning a person to
that role is itself part of resolving the blocker. Target dates are **proposed** by this stage
relative to the analysis date of 2026-09-04; none is an agreed commitment.

| ID | Blocker | Blocks | Why it blocks | Owner | Resolution Path | Target Date |
|----|---------|--------|---------------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Nobody in the available sources can say whether `ordermaker-service` runs in any real environment. It has no gateway route, appears in neither process manifest, and can only be reached by publishing directly to the message broker | CAP-10 in §§1–8; the four capabilities it commands (CAP-04, CAP-05, CAP-06, CAP-07) inherit an unverified inbound writer; any later stage that scopes or prioritises CAP-10 | CAP-10 is described as a current capability. If it is dead code, a capability that spans four other capabilities is being carried into every later stage for no reason | Platform owner / operator for the Pacco runtime (no named individual recorded — see Q11) | Confirm whether `ordermaker-service` is deployed in any environment, and inspect the `ordermaker` exchange on the running broker for a publisher; if it is deployed but unreachable, record it as dead code and mark CAP-10 accordingly | 2026-09-11 (proposed) |
| B2 | **[ACTION NOW]** Five Vault unseal keys, a Vault root token, and the gateway's token signing key are committed in plaintext in the repositories | CAP-15's status as a real control rather than a placeholder; any security review of CAP-01 and CAP-02 that depends on the token signing key being secret | This is a live security exposure independent of any architecture work, and it also determines whether CAP-15 is a working control or a placeholder | Security owner for the Pacco platform (no named individual recorded — see Q11) | Confirm whether the committed values are throwaway local-development credentials; if they are not, rotate the Vault root token and unseal keys and the JWT signing key, and purge them from history | 2026-09-08 (proposed) — treat as immediate if the values are live |
| B3 | **[handled later by runtime and deployment validation]** No production runtime manifest exists in any repository — no Kubernetes, Helm, or Terraform | Every scaling, availability, and network-policy statement about all sixteen capabilities; the evidence-strength ceiling recorded at the end of §5; assumption A3 | Scaling, replica counts, failover, and network policy for every capability cannot be established from these sources at any confidence | The runtime and deployment validation stage (owner assigned when that stage is scheduled) | Obtain read access to the deployment platform itself and re-derive the topology there, rather than from source repositories | n/a — resolves when the runtime and deployment validation stage runs; no action required from this stage |
| B4 | **[ACTION NOW]** Four gateway routing configurations exist and nothing states which one production uses. The synchronous and asynchronous variants are behaviourally different systems, not environment variants | CAP-02's described behaviour; whether CAP-11 is on the critical path for 20 write operations or merely observational; the runtime flows drawn in `architecture-views.md` | CAP-02's actual behaviour, and whether twenty operations reach services over HTTP or over the message broker, both depend on the answer | Platform owner / operator for the Pacco runtime (no named individual recorded — see Q11) | Read the effective `NTRADA_CONFIG` value in each running environment and record which `ntrada*.yml` is authoritative per environment | 2026-09-11 (proposed) |

### Open Questions

**On the Decision Owner column.** As with the Blockers table, no named individual is recorded
anywhere in the workspace (Q11). Decision Owner names the **role empowered to settle the question**;
naming a person to it is part of answering. A *Proposed Answer* is this stage's best reading of the
evidence, offered so the owner can confirm or reject rather than start from nothing — it is **not**
a finding, and nothing elsewhere in this baseline rests on it.

| ID | Question | Why it matters | Proposed Answer (if any) | Decision Owner | Where the answer would come from |
|----|----------|----------------|--------------------------|----------------|----------------------------------|
| Q1 | **[ACTION NOW]** What starts a delivery? Nothing in any repository sends `deliveries-service` a message or calls it, other than a person calling the endpoint directly | CAP-09 sits in the middle of the order lifecycle. If its trigger is manual, that is a significant fact about how the platform actually runs | Likely genuinely manual today: the service subscribes to no external event and declares no inbound command, which is a consistent shape rather than a missing handler. Not asserted — an external integration would look identical from inside this workspace | Platform owner for the Pacco runtime | A platform owner, or an integration living outside these thirteen repositories |
| Q2 | **[ACTION NOW]** Are the discount percentages in `CustomerDiscountsService` and the twenty-order VIP threshold the intended business rules? Both are hard-coded, uncommented, untested, and undocumented | These two numbers set what every customer pays. Nothing in the codebase confirms they are deliberate | None. The values are internally consistent (a monotonic ladder plus a VIP uplift) but nothing in the workspace distinguishes deliberate business rules from placeholder constants | Business owner for pricing | Whoever owns pricing for the business |
| Q3 | **[ACTION NOW]** Is losing operation status acceptable? Operation status is held only in a cache entry that expires five minutes after it was last touched, and is lost entirely if the cache restarts | A customer watching a long-running order would simply stop receiving updates. Whether that is a defect or an accepted design choice changes how CAP-11 should be read | Probably an accepted design choice for a status projection: the expiry is explicitly configured (`requests.expirySeconds: 300`) rather than defaulted, and CAP-11 writes no domain state. Whether five minutes suits the real order lifecycle is the actual question | Platform owner, or the original author of `operations-service` | A platform owner or the original author |
| Q4 | **[handled later by data and integration analysis]** Should the copies of customer data held by `orders-service` and `availability-service` be reconciled when a customer changes? They are built once from the customer-created event and never updated | Two capabilities can act on a stale customer state or a stale VIP flag | Likely an oversight rather than a design: `customer_state_changed` and `customer_became_vip` are both published and both declared in `messages.json`, yet no handler exists for either in any repository | Data and integration analysis stage, with the `orders-service` and `availability-service` owners | A stage that examines data flow and consistency requirements across services |
| Q5 | **[handled later by data and integration analysis]** How does the Pact contract file get from the `orders-service` consumer test to the `parcels-service` provider test? No broker, shared path, or build step references one | Determines whether the platform's only contract test actually protects the boundary between CAP-06 and CAP-07 | Likely nothing transports it today, so the two suites run independently against separately committed files. Both `PACT/` directories exist and both reference Pactify 1.1.0, but neither `.travis.yml` mentions a broker or a shared artifact | Data and integration analysis stage, or the engineer who set the tests up | A stage with access to the build pipeline, or the engineer who set the tests up |
| Q6 | **[handled later by runtime and deployment validation]** What do the messages `operations-service` receives actually look like on the wire? Its message types are built at runtime with no fields, so nothing static can show their content | CAP-11's real input contract is unknowable from source alone | Partial: the payloads are whatever each publishing service serialises, and CAP-11 reads only the correlation id and the saga-state header — so the body may not matter to it at all. That reading is unverified | Runtime and deployment validation stage | Observing live message traffic |
| Q7 | **[handled later by security review]** Is the certificate access list in `customers-service` enforced, or only declared? The rule is written in configuration but the code that would apply it lives in a third-party package outside this workspace | Determines whether service-to-service access control in CAP-15 is real | Probably enforced: `Convey.WebApi.Security` is referenced by exactly the two services that participate, and `availability-service`'s client attaches a certificate on the one call the ACL covers, which would be pointless otherwise. The package source is not in the workspace, so this is inference | Security review stage | A security review, or a test against a running environment |
| Q8 | **[handled later by data and integration analysis]** How is a database schema changed once a service is live? Only one service creates an index at startup, and no service has any migration or versioning tooling | Nine capabilities own data with no evidenced path for changing its shape | Likely relies on MongoDB's schemaless documents absorbing shape changes, with no explicit migration step at all. No tooling of any kind was found to contradict or confirm this | Platform owner, with the data and integration analysis stage | A platform owner, or a stage that examines data lifecycle |
| Q9 | **[ACTION NOW]** What do the customer states `Suspicious` and `Locked` mean, and what should set them? Both make a customer unable to reserve anything, but no code ever sets either | An unreachable blocking state in CAP-03 either indicates missing behaviour or a leftover | None. The states are reachable only through the admin state-change endpoint, so they may be intended as purely manual interventions rather than dead code — but no evidence supports either reading | Business owner for customer operations | Whoever owns customer operations for the business |
| Q10 | **[handled later by capability and domain modelling]** Is capability maturity recorded anywhere outside these repositories? No maturity information was retrievable for any capability, so all sixteen are marked unknown | A later stage that needs maturity to prioritise will find nothing here | None. No capability catalogue entry for this platform was retrievable by this stage; whether that means none exists or none was reachable is itself unresolved (A8) | Capability and domain modelling stage | A capability catalogue or portfolio system, if one exists |
| Q11 | **[handled later by capability and domain modelling]** Which team owns each capability? No repository contains a code-owners file, a contributing guide, or any team reference | §2 maps capabilities to services, which is evidenced, but cannot map them to people | None. This question also gates the Owner and Decision Owner columns in these tables, which currently name roles rather than people | Capability and domain modelling stage, with engineering leadership | An engineering directory or team-ownership record maintained outside the code |
| Q12 | **[handled later by security review]** Why does the order-making automation run with no user identity, bypassing the ownership checks every customer request passes through? | Requests made by this capability are not subject to the per-caller checks described in CAP-04 and CAP-07 | Likely a pragmatic choice so the saga can act across customers, since it constructs an empty `UserContext` explicitly rather than omitting one. Whether that was intended to carry a service identity instead is unknown | Security review stage, with the `ordermaker-service` author | A security review, or the engineer who wrote the saga |
| Q13 | **[ACTION NOW]** Are `vehicle_added` and `vehicle_updated` meant to have consumers? Both are published and declared in `messages.json`, but only `vehicle_deleted` has a subscriber anywhere in the thirteen clones | `orders-service` reads vehicle data synchronously at assignment time and stores the resulting price on the order, so a later vehicle price or capacity change never reaches an existing order (CAP-05, CAP-07) | Likely deliberate for pricing, since the order captures its price at assignment by design — but that makes the two events unused rather than pending. Subject to A4 | Platform owner, with the `vehicles-service` and `orders-service` owners | A platform owner, or the running broker's queue bindings for the `vehicles` exchange |
| Q14 | **[ACTION NOW]** Where does `ordermaker-service` keep its saga state, and is losing it acceptable? `Extensions.cs` calls `AddChronicle()` with no persistence backend and no `Chronicle.Persistence.*` package | A restart mid-saga would strand an in-flight order in a partially built state, and CAP-10's single compensation path would not fire (CAP-10) | Almost certainly in-memory, which is Chronicle's default when no persistence is registered. If CAP-10 turns out to be dead code (B1) the question is moot, so resolve B1 first | Platform owner, with the `ordermaker-service` author | Chronicle's package defaults, plus a restart test against a running saga |
| Q15 | **[handled later by capability and domain modelling]** Which component should downstream tooling treat as the owner of CAP-12 Asynchronous Messaging & Event Distribution? It is the only capability in the Service Lookup Index with no Primary row | Any consumer of this index that assumes every capability has exactly one owning component will either drop CAP-12 or attach it to an arbitrary service. CAP-12 is the substrate every other capability integrates over, so mis-assigning it mis-draws the whole platform | None, and the absence is the finding rather than a gap in retrieval: nine services each own one exchange, while the catalogue that binds them (`messages.json`) sits in `operations-service`, which owns none of the names in it. A candidate answer is to split CAP-12 into a per-service exchange capability and a separately owned catalogue capability, but that is a modelling decision this stage cannot make | Capability and domain modelling stage, with the platform architect | A capability-modelling decision, not further code reading — the code position is already fully established in §2 note 1 |
| Q16 | **[handled later by capability and domain modelling]** Is `Pacco.Services.Operations.GrpcClient` an in-scope platform component or a sample that should be dropped from the component list? | It is on the `Components in scope` line, so a downstream stage that batches over that line will analyse it. If it is a demo, that is wasted scope; if it is a real client, it is an unexamined consumer of the CAP-11 gRPC contract | Likely a sample: `repo-inventory.md` §2.3 records it as a console demo client, it has no `container_name` in either compose stack and no PM2 entry, and its `Program.cs` drives the gRPC surface directly. It was kept in the list because it is a genuine second deployable of its repository (`OutputType: Exe`), and dropping a real component is the worse error | Capability and domain modelling stage, or the original author of `operations-service` | The author, or a statement of intent for the repository's second project |

