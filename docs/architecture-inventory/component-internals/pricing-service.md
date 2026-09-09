# Component internals — `pricing-service`

| | |
| --- | --- |
| **Component** | `pricing-service` |
| **Source repository** | `hianshul100_Pacco.Services.Pricing` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository; the single project is `src/Pacco.Services.Pricing.Api`) |
| **Base ref** | `feature/12998/aidlc` (`HEAD` = `ed66901`, *Added security package*) |
| **Batch** | 6 of 7 |
| **Status** | New artifact — no prior `component-internals/pricing-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` §2.4 and `repo-summary/Pacco.Services.Pricing.md` remain valid and are **complemented**, not replaced: those catalogue the surface, this document models the internals. Where this model corrects a baseline it says so and names the section (§8.4). |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full — **39 tracked
> files**, of which **15 are C# source** and 6 are project/configuration files, all under one project,
> `src/Pacco.Services.Pricing.Api`, the repository's only `*.csproj`. Six of the 39 are committed
> Rider IDE state under `src/Pacco.Services.Pricing.Api/.idea/` (§3.31). There is **no test project of
> any kind**, and no `tests/` directory. `Convey 0.4.*` — which supplies the query dispatcher, the typed
> HTTP client, the error-handler middleware, the dispatcher-bound endpoint mapping, Consul/Fabio
> registration and the Vault bootstrap — is a NuGet reference with **no source in this workspace**;
> mechanisms it owns are marked `[convey]` and, where their exact semantics would change a
> conclusion, flagged `Unverifiable — Missing Source Evidence`. The upstream half of this service's
> only route is modelled in `component-internals/api-gateway.md`; the downstream half of its only
> outbound call is modelled in `component-internals/customers-service.md`.

---

## Contents

1. [Purpose & boundary](#1-purpose--boundary)
2. [Core concepts (exhaustive)](#2-core-concepts-exhaustive)
3. [Per concept](#3-per-concept)
4. [Primary control flows](#4-primary-control-flows)
5. [Persistence & schema evolution](#5-persistence--schema-evolution)
6. [Surface → internals map](#6-surface--internals-map)
7. [Change/extension guide](#7-changeextension-guide)
8. [Assumptions, Blockers & Open Questions](#8-assumptions-blockers--open-questions)

---

## 1. Purpose & boundary

### 1.1 What this component is responsible for

`pricing-service` answers exactly one question: **given a customer and a base order price, what
discount applies and what is the final price?** It is a **pure function fronted by HTTP**. It owns no
data, publishes no events, subscribes to nothing, and holds no state between requests. Everything it
knows about a customer it fetches, synchronously, from `customers-service` on every single call.

Three properties characterise it, and each one is a deliberate divergence from every other service
on the platform:

1. **It is the only deployable with no broker participation at all.** `appsettings.json` has no
   `rabbitMq` section, and the project references no `Convey.MessageBrokers.*` package
   (`src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj:10-22`). It cannot publish, and
   it cannot be reached asynchronously. Contrast `vehicles-service`, which does both
   (`component-internals/vehicles-service.md` §3.31).
2. **It is the only service with no persistence.** No `mongo` section, no `redis` section, no
   `Convey.Persistence.*` package. There is no repository interface, no document type and no mapping
   layer anywhere in the tree.
3. **It does not follow the platform's four-project layering.** Where every other service splits
   `Core` → `Application` → `Infrastructure` → `Api` into four assemblies
   ([[inward-dependency-service-skeleton]]), this one is a **single project with folders of the same
   names**: `Core/`, `DTO/`, `Exceptions/`, `Infrastructure/`, `Queries/`, `Services/`. The
   dependency direction is therefore enforced by nothing but convention — the compiler cannot stop
   `Core/Services/CustomerDiscountsService.cs` from referencing an HTTP client (§3.18).

| Responsibility | Where it lives |
| --- | --- |
| Accept one HTTP query carrying a customer id and a base order price | `…Api/Program.cs:31`; `…Api/Queries/GetOrderPricing.cs:9-10` |
| Fetch the customer's VIP flag and completed-order list from `customers-service` | `…Api/Services/Clients/CustomersServiceClient.cs:19-20` |
| Collapse the completed-order list to a count and build a local `Customer` | `…Api/DTO/Extensions.cs:8-9` |
| Apply the four-band discount ladder | `…Api/Core/Services/CustomerDiscountsService.cs:11-22` |
| Add the VIP bonus on top of the band | `…Api/Core/Services/CustomerDiscountsService.cs:24-27` |
| Compute the discounted price and guard it against a non-positive result | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:35,45` |
| Fail the request when the customer does not exist | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:29-32` |
| Translate exceptions into a `{code, reason}` HTTP 400 body | `…Api/Exceptions/ExceptionToResponseMapper.cs:13-20` |
| Register with Consul, address peers through Fabio, emit Jaeger spans, expose Prometheus metrics | `…Api/Infrastructure/Extensions.cs:32-35` |
| Fetch its own settings and a PKI certificate from Vault at startup | `…Api/Program.cs:33` (`UseVault`); `…Api/appsettings.json` keys `vault.kv.*`, `vault.pki.*` |

### 1.2 What this component explicitly is **not**

- **Not a bounded context.** It owns no aggregate, no invariant over persisted state, and no
  lifecycle. `Core/Entities/Customer.cs` is a request-scoped calculation input, not an entity with
  identity — it does not even keep the id it is handed (§3.3). `service-summaries.md` §2.4 already
  states this; this document confirms it from the code and adds the reason.
- **Not the owner of the discount inputs.** VIP status and the completed-order set are
  `customers-service` data, read over HTTP on every call (§3.9). This service has **no cache, no
  replica and no fallback**: if `customers-service` is unavailable, pricing is unavailable.
- **Not an authenticator or an authorizer.** There is no `AddJwt`, no `UseAuthentication`, no
  `UseAuthorization` and no `[Authorize]` anywhere in `src/`. The `jwt` block in `appsettings.json`
  and the committed `certs/localhost.cer` are **inert configuration no code reads** (§3.21). The
  only security registration is `AddSecurity()` (`…Api/Infrastructure/Extensions.cs:37`), which
  registers Convey's cryptography helpers and is **never injected anywhere** (§3.22). Access control
  happens exclusively at the gateway (§3.33).
- **Not a certificate-presenting client.** Unlike `availability-service`, which attaches a
  Vault-PKI client certificate before calling `customers-service`
  (`hianshul100_Pacco.Services.Availability/…/Clients/CustomersServiceClient.cs:16-34`), this
  service's client constructor takes **only** `IHttpClient` and `HttpClientOptions` and sets no
  headers (`…Api/Services/Clients/CustomersServiceClient.cs:13-17`). It calls a service that has
  certificate authentication switched on. See **B-1** and **Q-1**.
- **Not a currency-aware calculator.** `decimal` is used throughout, but no currency code, no
  rounding, no minor-unit handling and no locale appear anywhere. The service returns full-precision
  `decimal` products (§3.35).
- **Not idempotency-sensitive.** It performs no writes, so repeat calls are harmless — but it is also
  **not** deterministic across time, because the answer depends on `customers-service` state at the
  moment of the call.
- **Not tested.** There is no test project. `scripts/test.sh` runs `dotnet test`, which over this
  solution discovers nothing and exits successfully — a green CI step that asserts nothing (§3.30).
- **Not an aggregator.** It calls one service. It does not fan out, join, or orchestrate.

### 1.3 The single-transport boundary

Every other write-capable service on the platform has two entry points with different failure
semantics ([[dual-mode-edge-write]]). `pricing-service` has **one**, and that simplifies its failure
model to a single row:

| | HTTP path (the only path) |
| --- | --- |
| Entry | `UseDispatcherEndpoints` → `Get<GetOrderPricing, OrderPricingDto>("pricing")` (`…Api/Program.cs:29-31`) |
| Identity source | **none** — no `IAppContext`, no `IIdentityContext`, no `Correlation-Context` reader exists in this repository (§3.19) |
| Caller trust | the `customerId` query parameter is believed unconditionally; the gateway is the only thing that binds it to the token (§3.33) |
| De-duplication | not applicable — read-only |
| Failure surfaced as | HTTP **400**, always, with `{code, reason}` (§3.13) |
| Failure when unmapped | generic 400 `{code:"error", reason:"There was an error."}` |
| Caller learns outcome | synchronously, in the response body |

The consequence worth holding on to: **this service cannot report a failure asynchronously**, so
`operations-service` never sees a pricing failure and no rejected event exists for it
([[rejected-event-failure-contract]] is not instantiated here). A caller that gets a 400 is the only
party that knows.

### 1.4 Position in the platform

| Direction | Counterpart | Mechanism | Evidence |
| --- | --- | --- | --- |
| Inbound (sync) | `api-gateway` | `GET /pricing`, `auth: true`, downstream rewritten to inject `customerId=@user_id` | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:400-412`; identical block in `ntrada-async.yml:468-480`, `ntrada.docker.yml`, `ntrada-async.docker.yml` |
| Inbound (sync) | `orders-service` | `GET /pricing?customerId=&orderPrice=` via a typed client | `hianshul100_Pacco.Services.Orders/…/Services/Clients/PricingServiceClient.cs:20-21` |
| Outbound (sync) | `customers-service` | `GET /customers/{id}`, **no certificate header** | `…Api/Services/Clients/CustomersServiceClient.cs:19-20` |
| Inbound (async) | **nobody** | no exchange, no queue, no subscription | `appsettings.json` has no `rabbitMq` key |
| Outbound (async) | **nobody** | publishes nothing | no `IMessageBroker`, no `Convey.MessageBrokers.*` reference |

The gateway exposes `pricing` in **both** the synchronous and the asynchronous profile as
`use: downstream` — this is the only write-capable-looking route family that is a plain proxy in
`ntrada-async.yml` as well, because there is nothing to publish it to.

---

## 2. Core concepts (exhaustive)

Every distinct internal mechanism this service implements or deliberately omits. The **Owner** column
names the file that *defines* the concept; a concept whose owner is a configuration key or the
repository layout is one the code does not define anywhere else.

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | `GetOrderPricing` — the only query | `…Api/Queries/GetOrderPricing.cs` | §3.1 |
| 2 | `OrderPricingDto` — the response contract | `…Api/DTO/OrderPricingDto.cs` | §3.2 |
| 3 | `Customer` — the request-scoped calculation input | `…Api/Core/Entities/Customer.cs` | §3.3 |
| 4 | `CustomerDto` — the wire shape read from `customers-service` | `…Api/DTO/CustomerDto.cs` | §3.4 |
| 5 | `AsEntity` — the DTO→input mapping and the `Count()` collapse | `…Api/DTO/Extensions.cs:8-9` | §3.5 |
| 6 | The discount ladder — four bands on completed-order count | `…Api/Core/Services/CustomerDiscountsService.cs:11-22` | §3.6 |
| 7 | The VIP bonus — an additive `+0.1` | `…Api/Core/Services/CustomerDiscountsService.cs:24-27` | §3.7 |
| 8 | Final-price arithmetic and the `> 0` fallback | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:35,45` | §3.8 |
| 9 | `ICustomersServiceClient` / `CustomersServiceClient` — the only outbound dependency | `…Api/Services/Clients/*.cs` | §3.9 |
| 10 | Logical-name addressing and the Fabio hop | `appsettings.json` keys `httpClient.type`, `httpClient.services.customers` | §3.10 |
| 11 | Null-customer detection and `CustomerNotFoundException` | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:29-32` | §3.11 |
| 12 | `AppException` and the error-code derivation | `…Api/Exceptions/AppException.cs`; `ExceptionToResponseMapper.cs:22-39` | §3.12 |
| 13 | `ExceptionToResponseMapper` — everything is HTTP 400 | `…Api/Exceptions/ExceptionToResponseMapper.cs` | §3.13 |
| 14 | Dispatcher-bound endpoints and query-string binding | `…Api/Program.cs:29-31` | §3.14 |
| 15 | The unnamed root endpoint | `…Api/Program.cs:30` | §3.15 |
| 16 | The in-memory query dispatcher and handler discovery | `…Api/Infrastructure/Extensions.cs:29-30` | §3.16 |
| 17 | Statelessness — the absence of persistence, as a property | repository layout; `appsettings.json` | §3.17 |
| 18 | The single-project layout — folders where other services have assemblies | `Pacco.Services.Pricing.sln`; folder tree | §3.18 |
| 19 | Absence of caller context — no `IAppContext`, no identity | repository layout | §3.19 |
| 20 | Correlation and span propagation — what survives, what does not | `…Api/Infrastructure/Extensions.cs:35,44` | §3.20 |
| 21 | Inert JWT configuration and the committed certificate | `appsettings.json` key `jwt.*`; `…Api/certs/localhost.cer` | §3.21 |
| 22 | `AddSecurity()` — registered, never injected | `…Api/Infrastructure/Extensions.cs:37` | §3.22 |
| 23 | The dangling `Convey.Tracing.Jaeger.RabbitMQ` reference | `…Api/Pacco.Services.Pricing.Api.csproj:20` | §3.23 |
| 24 | Consul registration and Fabio addressing | `…Api/Infrastructure/Extensions.cs:32-33`; `appsettings.json` keys `consul.*`, `fabio.*` | §3.24 |
| 25 | Vault — KV settings and PKI, with no lease | `…Api/Program.cs:33`; `appsettings.json` key `vault.*` | §3.25 |
| 26 | Environment layering — four `appsettings` profiles | `…Api/appsettings*.json`; `Dockerfile:10`; `scripts/start.sh:2` | §3.26 |
| 27 | Logging — level, path exclusion, property redaction, sinks | `…Api/Program.cs:32`; `appsettings.json` key `logger.*` | §3.27 |
| 28 | Swagger docs surface | `…Api/Infrastructure/Extensions.cs:36,43`; `appsettings.json` key `swagger.*` | §3.28 |
| 29 | Metrics and tracing wiring | `…Api/Infrastructure/Extensions.cs:34-35,44,46` | §3.29 |
| 30 | Absence of a test suite, and the green-but-empty test step | `scripts/test.sh`; `.travis.yml` | §3.30 |
| 31 | Committed IDE state and the absent `LICENSE` | `src/Pacco.Services.Pricing.Api/.idea/**` | §3.31 |
| 32 | Deployment identity — port, image, PM2 app, Prometheus job | `Dockerfile`; `scripts/dockerize.sh`; `hianshul100_Pacco/services.yml`, `prod-services.yml`, `compose/services.yml` | §3.32 |
| 33 | Gateway exposure and the `customerId=@user_id` rewrite | `hianshul100_Pacco.APIGateway/…/ntrada.yml:400-412` | §3.33 |
| 34 | The unauthenticated read of a certificate-protected endpoint | `…Api/Services/Clients/CustomersServiceClient.cs:13-17` vs `hianshul100_Pacco.Services.Customers/…/Extensions.cs:79,91` | §3.34 |
| 35 | Currency, units and rounding — what the numbers mean | `…Api/DTO/OrderPricingDto.cs`; `…Queries/Handlers/GetOrderPricingHandler.cs:37-39` | §3.35 |
| 36 | Threshold coupling — 10/4/1 here, 20 in `customers-service` | `CustomerDiscountsService.cs:11-22` vs `component-internals/customers-service.md` §3.5 | §3.36 |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement**, **Extension procedure**, **Failure modes**.

### 3.1 `GetOrderPricing` — the only query

**Definition.** A two-property message implementing `IQuery<OrderPricingDto>`
(`…Api/Queries/GetOrderPricing.cs:7-11`): `Guid CustomerId` and `decimal OrderPrice`. Both are
**settable** (`{ get; set; }`), unlike every command on the platform, which uses get-only properties
plus a constructor. That difference is not stylistic: it is what makes query-string binding work
without a constructor-binding convention.

**Representation & storage.** Not stored. It exists only for the duration of a request, materialised
by Convey's endpoint binder from the query string and handed straight to the dispatcher.

**Lifecycle.** Bound → dispatched → handled → discarded. There is no validation stage between
binding and handling: the handler receives whatever the binder produced.

**Invariants & enforcement.**

| Expected invariant | Enforced? | Where it would have to live |
| --- | --- | --- |
| `CustomerId` is present | **No.** An absent or unparseable `customerId` yields `Guid.Empty`, which is sent to `customers-service` as `/customers/00000000-0000-0000-0000-000000000000` | a guard in `GetOrderPricingHandler` or a `FluentValidation`-style filter — neither exists |
| `OrderPrice` is positive | **No.** Negative and zero prices are accepted and flow into the arithmetic | see §3.8 — the `> 0` fallback silently rewrites the result instead |
| `OrderPrice` is parseable | `[framework]` — a malformed decimal is an ASP.NET Core model-binding failure, not an application failure, and does **not** pass through `ExceptionToResponseMapper`. The exact status code is `Unverifiable — Missing Source Evidence` (Convey owns the endpoint binder) |

**Extension procedure.** Adding a field means: add the property here, extend
`Pacco.Services.Pricing.rest` so the manual probe still exercises it, and — because the gateway
rewrites the query string — add it to the `downstream` template in **all four** `ntrada*.yml` files
(§3.33), or the gateway will silently drop it.

**Failure modes.** The `Guid.Empty` path is the important one: it does not fail fast. It produces a
well-formed outbound HTTP call for a customer that cannot exist, and surfaces as
`customer_not_found` (§3.11) — a misleading code for what is really a missing parameter.

### 3.2 `OrderPricingDto` — the response contract

**Definition.** Three `decimal` properties (`…Api/DTO/OrderPricingDto.cs:5-7`): `OrderPrice`,
`CustomerDiscount`, `OrderDiscountPrice`. This is the entire public output of the service.

**Representation & storage.** Serialized to JSON by the framework's default camel-casing. The
consumer that matters is `orders-service`, whose `PricingServiceClient` deserializes into its own
copy of the shape (`hianshul100_Pacco.Services.Orders/…/DTO/OrderPricingDto.cs`) — **two independent
declarations of one contract, coupled by nothing but matching property names**. Renaming a property
here compiles cleanly and silently produces `0` in `orders-service`.

**Lifecycle.** Constructed once, at `…Api/Queries/Handlers/GetOrderPricingHandler.cs:41-46`, using an
object initializer. Never mutated.

**Invariants & enforcement.**

| Invariant | Status |
| --- | --- |
| `CustomerDiscount` is a **fraction**, not a percentage or a money amount | Holds by construction (§3.6) but is **undocumented on the wire**. The log line at `GetOrderPricingHandler.cs:37-39` formats it as `"{discount} $"` — a unit error in the message text; the value is a ratio |
| `OrderDiscountPrice ≤ OrderPrice` | Holds whenever `OrderPrice > 0`, because the discount is in `[0, 0.2]`. Does **not** hold for negative `OrderPrice`, where the fallback returns `OrderPrice` unchanged (§3.8) |
| `OrderDiscountPrice > 0` | Intended by the ternary, but **not** achieved: when the computed price is `≤ 0` the fallback substitutes `OrderPrice`, which for a zero or negative input is itself `≤ 0` |

**Extension procedure.** Adding a field is additive and safe for `orders-service` (unknown properties
are ignored on its side). *Renaming* one is a silent breaking change on two consumers — this service
has no contract test, and the platform's only PACT pair is `orders` ↔ `parcels`, which is itself
disabled ([[consumer-driven-contract-test-pair]]; `index.md` §4).

**Failure modes.** Silent zeroing in `orders-service` on any rename. There is no version marker, no
`[Contract]` attribute (this service has no `UsePublicContracts` registration at all — §3.28) and no
schema published anywhere except Swagger.

### 3.3 `Customer` — the request-scoped calculation input

**Definition.** A three-property class in `Core/Entities/`
(`…Api/Core/Entities/Customer.cs:5-16`): `Guid Id`, `bool IsVip`, `int CompletedOrdersNumber`, all
with private setters, plus a constructor `Customer(Guid id, bool isVip, int completedOrdersNumber)`.

**Representation & storage.** Never persisted — this service has no store (§3.17). It exists for the
duration of one `CalculateDiscount` call.

**Lifecycle.** Created by `AsEntity` (§3.5), read once by `CustomerDiscountsService`, discarded.

**Invariants & enforcement.** **The constructor never assigns `Id`.** It sets `IsVip` and
`CompletedOrdersNumber` and drops its `id` parameter on the floor
(`…Api/Core/Entities/Customer.cs:11-15`). Every `Customer` instance therefore has
`Id == Guid.Empty`, regardless of what was passed.

This is currently **harmless**, because nothing reads `Id`: `CalculateDiscount` uses only
`CompletedOrdersNumber` and `IsVip` (`…Api/Core/Services/CustomerDiscountsService.cs:9-28`), and the
handler logs `query.CustomerId`, not `customer.Id`
(`…Api/Queries/Handlers/GetOrderPricingHandler.cs:37-39`). It is a **latent trap**: the first
feature that needs per-customer logic — a customer-specific override, an audit line, a cache key —
will read `Id`, get `Guid.Empty`, and behave identically for every customer on the platform. Because
`Guid.Empty` is a valid `Guid`, nothing throws.

There is also no validation: a negative `completedOrdersNumber` is accepted (it simply falls through
all four bands to `0.0m`).

**Extension procedure.** Fix `Id` before adding any code that reads it — one line,
`Id = id;`, and there is no test to update because there are no tests (§3.30). Adding a field means
extending `CustomerDto` (§3.4) and `AsEntity` (§3.5) at the same time, since the constructor is the
only way in.

**Failure modes.** Identity erasure, as above. Also note the class is in `Core/Entities/` but has no
behaviour — the discount logic lives in `Core/Services/`, so this is an anaemic input record wearing
an entity's folder name.

### 3.4 `CustomerDto` — the wire shape read from `customers-service`

**Definition.** `Guid Id`, `bool IsVip`, `IEnumerable<Guid> CompletedOrders`, all settable
(`…Api/DTO/CustomerDto.cs:8-10`). This is `pricing-service`'s **private, partial copy** of a contract
owned elsewhere.

**Representation & storage.** Deserialized from the `customers-service` response body. That service
returns `CustomerDetailsDto`, which extends `CustomerDto` with `Email`, `FullName`, `Address`,
`IsVip` and `CompletedOrders`
(`hianshul100_Pacco.Services.Customers/…/Application/DTO/CustomerDetailsDto.cs:5-16`) — so the
upstream payload is a **superset**, and the three fields this service names line up by property name.

**Lifecycle.** One instance per request, produced by `IHttpClient.GetAsync<CustomerDto>` `[convey]`,
consumed by `AsEntity`, discarded.

**Invariants & enforcement.** None declared. Two properties matter and neither is guarded:

| Property | Risk |
| --- | --- |
| `CompletedOrders` | **May be `null`.** Convey's typed client deserializes absent JSON properties to `null`, and `AsEntity` calls `.Count()` on it unconditionally (§3.5) |
| `IsVip` | Defaults to `false` when absent. A missing field silently removes the VIP bonus rather than failing |

**Extension procedure.** Only add fields that `customers-service` actually returns — verify against
`CustomerDetailsDto` in that repository, because nothing here will tell you if you guess wrong; an
unmatched property deserializes to its default.

**Failure modes.** Silent default-valued fields on any upstream rename. This is
[[event-carried-reference-replica]] inverted: instead of a replicated read model, the service takes a
synchronous dependency and duplicates the *shape* without duplicating the *data*, so it inherits both
the coupling and the availability cost.

### 3.5 `AsEntity` — the DTO→input mapping and the `Count()` collapse

**Definition.** A single static extension method on an `internal static class`
(`…Api/DTO/Extensions.cs:6-9`): `dto.AsEntity()` returns
`new Customer(dto.Id, dto.IsVip, dto.CompletedOrders.Count())`.

**Representation & storage.** No storage. This is where the platform's richest customer signal — the
**set of completed order ids** — is collapsed to a single `int`. Everything downstream of this line
knows only *how many*, never *which*, so no recency, value or category weighting is possible without
changing this method.

**Lifecycle.** Called exactly once per request, inline inside the `CalculateDiscount` argument at
`…Api/Queries/Handlers/GetOrderPricingHandler.cs:34`, between the HTTP fetch and the calculation.

**Invariants & enforcement.**

- **`CompletedOrders` is dereferenced without a null check.** If `customers-service` returns a
  customer whose `completedOrders` is absent or `null`, `.Count()` throws
  `ArgumentNullException`. That is **not** an `AppException`, so `ExceptionToResponseMapper` falls
  through to its generic arm and the caller receives `400 {code:"error", reason:"There was an
  error."}` (§3.13) — a 400 for what is a server-side null dereference. The one-character fix is
  `dto.CompletedOrders?.Count() ?? 0`.
- **`.Count()` on `IEnumerable<Guid>` is O(n)** but enumerates a materialised array here, so cost is
  negligible.
- The `id` argument is passed and then discarded by the constructor (§3.3).

**Extension procedure.** To weight the discount by anything other than raw count, this is the seam:
change `Customer` to carry the collection (or a derived score), change this mapping, then change
`CustomerDiscountsService`. All three live in one project, so the change is a single compile unit.

**Failure modes.** `ArgumentNullException` → generic 400, as above. Silent under-counting if
`customers-service` ever pages or truncates `completedOrders` — this service would read the page
size as the lifetime count and cannot detect the difference.

### 3.6 The discount ladder — four bands on completed-order count

**Definition.** `CustomerDiscountsService.CalculateDiscount(Customer)`
(`…Api/Core/Services/CustomerDiscountsService.cs:9-28`) assigns a base discount from
`CompletedOrdersNumber` through a chain of `if`/`else if` with hard-coded thresholds and rates:

| Completed orders | Discount | Line |
| --- | --- | --- |
| `>= 10` | `0.10` | `CustomerDiscountsService.cs:11-14` |
| `4` – `9` (`< 10 && > 3`) | `0.05` | `CustomerDiscountsService.cs:15-18` |
| `1` – `3` (`<= 3 && > 0`) | `0.02` | `CustomerDiscountsService.cs:19-22` |
| `<= 0` | `0.00` (the initialiser at `:10`) | — |

The redundant left-hand conjuncts (`< 10 &&`, `<= 3 &&`) are implied by the `else if` chain; they are
harmless but they are also the reason the bands read as independent rules rather than as a ladder,
which matters when someone edits one of them.

**Representation & storage.** The rates and thresholds are **compile-time literals**. There is no
configuration key, no Vault secret, no database row and no admin endpoint that can change them. A
pricing change is a code change, a container rebuild and a redeploy.

**Lifecycle.** Evaluated per request. Stateless — the class has no fields
(`…Api/Core/Services/CustomerDiscountsService.cs:5-7`) and is registered as a scoped/singleton service
as `AddTransient<ICustomerDiscountsService, CustomerDiscountsService>()`
(`…Api/Infrastructure/Extensions.cs:25`) — a new instance per resolution, which is free because the
class holds nothing.

**Invariants & enforcement.**

- Bands are **contiguous and exhaustive** over the integers — verified by reading the chain: every
  `int` lands in exactly one arm. Negative counts fall to the `0.0m` initialiser.
- The ladder is **monotonic** in completed orders.
- **Nothing enforces that the bands stay contiguous.** Editing `> 3` to `> 5` opens a silent gap
  where counts 4–5 fall through to `0.0m` — no exception, no log, just a smaller discount. There is
  no test to catch it (§3.30).

**Extension procedure.** To make the ladder configurable: introduce an options type bound from
`appsettings.json`, register it in `…Api/Infrastructure/Extensions.cs`, and inject it here. Note the
`[[composable-per-concern-environment-stacks]]` consequence — the key must be added to
`appsettings.json` **and** left absent from the overrides, or `appsettings.local.json` /
`appsettings.docker.json` will need explicit values (§3.26).

**Failure modes.** Silent misconfiguration by edit, as above. Also: because the thresholds are
duplicated nowhere and documented nowhere, the only statement of the business rule is this method —
`README.md` does not describe it.

### 3.7 The VIP bonus — an additive `+0.1`

**Definition.** After the ladder, `if (customer.IsVip) { discount += 0.1m; }`
(`…Api/Core/Services/CustomerDiscountsService.cs:24-27`).

**Representation & storage.** A literal, like the ladder. `IsVip` itself is **owned by
`customers-service`** and simply believed here.

**Lifecycle.** Applied once per calculation, after the band, before return.

**Invariants & enforcement.**

- The composed discount is bounded: maximum `0.10 + 0.10 = 0.20`, minimum `0.00`. **This bound is
  emergent, not asserted** — no clamp exists. Raise the top band to `0.95` and add the VIP bonus and
  the service will happily return `1.05`, producing a *negative* discounted price, which the `> 0`
  fallback (§3.8) then converts back into the full price. The failure is invisible.
- The bonus is **additive, not multiplicative** — a VIP with 10+ orders pays 80% of list, not
  `0.9 × 0.9 = 81%`. Worth stating because both readings are plausible from the field name.

**Extension procedure.** Any new bonus should be added here and a clamp
(`Math.Min(discount, MaxDiscount)`) introduced at the same time; the clamp is the missing invariant,
not a nicety.

**Failure modes.** Unbounded composition, as above. Also: `IsVip` arriving `false` because the field
was renamed upstream (§3.4) removes the bonus with no diagnostic.

### 3.8 Final-price arithmetic and the `> 0` fallback

**Definition.** In the handler (`…Api/Queries/Handlers/GetOrderPricingHandler.cs:35,41-46`):

- `orderDiscountPrice = query.OrderPrice - customerDiscount * query.OrderPrice`
- the DTO's `OrderDiscountPrice` is set to `orderDiscountPrice > 0 ? orderDiscountPrice : query.OrderPrice`

**Representation & storage.** `decimal` throughout — the correct choice for money, and consistently
applied (`OrderPrice`, `CustomerDiscount`, `OrderDiscountPrice` are all `decimal`). No rounding is
performed at any point (§3.35).

**Lifecycle.** Computed once per request, immediately before the response is constructed.

**Invariants & enforcement.** The ternary is the only guard in the arithmetic path, and it is a
**silent** one:

| Input | Computed | Returned | What the caller sees |
| --- | --- | --- | --- |
| `OrderPrice = 100`, discount `0.1` | `90` | `90` | correct |
| `OrderPrice = 0`, any discount | `0` | **`0`** (fallback substitutes `OrderPrice`, which is `0`) | a free order, no error |
| `OrderPrice = -50`, discount `0.1` | `-45` | **`-50`** — the fallback makes the result *worse* than the computation | a negative price, no error |
| discount `> 1` (only reachable by editing §3.6/§3.7) | negative | `OrderPrice` | full price, silently — the pricing bug is masked |

The guard's evident intent is "never return a non-positive price", and it **does not achieve that**:
it substitutes `OrderPrice`, which is non-positive in exactly the cases that trigger it. A guard that
rejected the input (`throw new AppException`) or clamped to zero would fail loudly; this one fails
silently and returns a number that looks legitimate.

**Extension procedure.** The correct place for input validation is the top of
`HandleAsync`, before the outbound HTTP call — validating first also avoids a pointless call to
`customers-service` for a request that cannot succeed. Adding a new `AppException` subclass there
requires a matching arm in `ExceptionToResponseMapper` (§3.13) or it degrades to the generic 400.

**Failure modes.** Silent acceptance of zero and negative prices; silent masking of an
over-100% discount. Both are invisible in logs, because the log line at
`GetOrderPricingHandler.cs:37-39` is emitted at `Information` and prints the *computed*
`orderDiscountPrice` — so a log will show `-45` while the response body shows `-50`. That
discrepancy is the only observable trace.

### 3.9 `ICustomersServiceClient` / `CustomersServiceClient` — the only outbound dependency

**Definition.** A one-method interface (`…Api/Services/Clients/ICustomersServiceClient.cs:7-10`,
`Task<CustomerDto> GetAsync(Guid id)`) and an `internal sealed` implementation
(`…Api/Services/Clients/CustomersServiceClient.cs:8-21`) that delegates to Convey's `IHttpClient`:
`_httpClient.GetAsync<CustomerDto>($"{_url}/customers/{id}")`.

**Representation & storage.** The base URL is captured **once, in the constructor**, from
`options.Services["customers"]` (`CustomersServiceClient.cs:16`). Registration is
`AddTransient<ICustomersServiceClient, CustomersServiceClient>()`
(`…Api/Infrastructure/Extensions.cs:24`), so the dictionary lookup re-runs per resolution — the
options object is bound once at startup, so the value cannot change without a restart.

**Lifecycle.** One instance per request scope, one outbound call per request, no connection state of
its own (`IHttpClient` wraps `IHttpClientFactory` `[convey]`).

**Invariants & enforcement.**

| Invariant | Enforcement | Failure style |
| --- | --- | --- |
| The `customers` key exists in `httpClient.services` | **Indexer access, not `TryGetValue`** — a missing key throws `KeyNotFoundException` inside the DI resolution for the *first request*, not at startup | **Fails loudly, but late**: the service starts healthy, registers in Consul, passes its `ping` check, and then 500s (or 400s, via the generic mapper) on the first real request |
| The response body is a customer | Not checked — see §3.11 |
| The caller is authorized to read this customer | **Not enforced at all** — no certificate, no token, no header (§3.34) |

**Extension procedure.** A second outbound dependency follows the same three-step shape: interface +
`internal sealed` client in `Services/Clients/`, an `AddTransient` line in
`…Api/Infrastructure/Extensions.cs`, and a new key under `httpClient.services` in **`appsettings.json`
and `appsettings.local.json` and `appsettings.docker.json`** — the local profile *replaces* the whole
`services` object rather than merging into it (`appsettings.local.json:9-15`), so a key added only to
the base file is absent locally and the indexer throws. That replace-not-merge behaviour is the
single most likely way to break this service by editing configuration.

**Failure modes.** Timeouts, connection refusals and non-2xx responses from `customers-service`
propagate as whatever `IHttpClient` raises `[convey]`; none of them is an `AppException`, so all of
them become the generic `400 {code:"error"}` (§3.13). **A caller cannot distinguish "customer does not
exist" from "customers-service is down" by status code** — only by the `code` field, and only for the
first case.

`httpClient.retries: 3` (`appsettings.json:25`) means the failure is retried before it surfaces
`[convey]`; whether those retries are applied to `GetAsync<T>` and whether they back off is
`Unverifiable — Missing Source Evidence`.

### 3.10 Logical-name addressing and the Fabio hop

**Definition.** The client is handed a **logical service name**, not a URL. In the base and docker
profiles `httpClient.services.customers` is `customers-service` (`appsettings.json:27`,
`appsettings.docker.json:26`) and `httpClient.type` is `fabio` (`appsettings.json:24`), so Convey's
Fabio-aware handler rewrites the request to the Fabio load balancer, which resolves the name through
Consul ([[registry-mediated-discovery-and-routing]]).

**Representation & storage.** Three keys carry the whole mechanism:

| Key | Base | `local` | `docker` | Effect |
| --- | --- | --- | --- | --- |
| `httpClient.type` | `fabio` | `""` (`appsettings.local.json:10`) | `fabio` | selects the outbound handler; empty means "plain HTTP, use the address verbatim" |
| `httpClient.services.customers` | `customers-service` | `http://localhost:5002` | `customers-service` | the address the client concatenates with `/customers/{id}` |
| `fabio.url` | `http://localhost:9999` | (disabled at `appsettings.local.json:6-8`) | `http://fabio:9999` | where the rewritten request actually goes |

**Lifecycle.** Bound at startup. The rewrite happens per request inside the Convey HTTP stack.

**Invariants & enforcement.** The two keys must agree: `type: fabio` with a literal `http://…`
address, or `type: ""` with a bare logical name, both produce requests that go nowhere useful. There
is **no validation of that pairing** — it is a convention held only by the three files agreeing with
each other. `appsettings.local.json` is the demonstration that the convention exists: it changes
*both* keys together.

**Extension procedure.** When adding an environment profile, change `httpClient.type` and every entry
in `httpClient.services` in the same edit, and remember the object is replaced, not merged (§3.9).

**Failure modes.** Silent misrouting: with `type: fabio` and a literal URL, Fabio receives a request
for a service name that looks like a URL and fails to route it — surfacing as the generic 400.

### 3.11 Null-customer detection and `CustomerNotFoundException`

**Definition.** `if (customer is null) throw new CustomerNotFoundException(query.CustomerId);`
(`…Api/Queries/Handlers/GetOrderPricingHandler.cs:29-32`). The exception carries
`Code = "customer_not_found"` and the message `$"Customer not found: {id}."`
(`…Api/Exceptions/CustomerNotFoundException.cs:7-11`).

**Representation & storage.** Not stored. The `Code` is an instance property with an initialiser, so
it is *not* derived from the type name — the derivation fallback in `GetCode` (§3.12) never runs for
this type.

**Lifecycle.** Thrown inside the handler, caught by Convey's error-handler middleware
(`…Api/Infrastructure/Extensions.cs:28,42`), mapped by `ExceptionToResponseMapper`, serialized as
HTTP 400.

**Invariants & enforcement.** The load-bearing assumption is that **`IHttpClient.GetAsync<T>` returns
`null` for a 404** rather than throwing. Convey's HTTP client is not in this workspace, so this is
`Unverifiable — Missing Source Evidence` — but the platform relies on it uniformly: the same
null-check-then-throw shape appears in `orders-service`
(`hianshul100_Pacco.Services.Orders/…/Commands/Handlers/CreateOrderHandler.cs`) and in
`availability-service`. If Convey ever changed that behaviour, every service on the platform would
switch from `customer_not_found` to the generic `error` code simultaneously
([[narrow-synchronous-point-read]]).

A second, weaker assumption: `null` means *absent*, not *malformed*. A 200 response with an empty
body would also deserialize to `null` and be reported as "customer not found".

**Extension procedure.** New failure conditions should each get their own `AppException` subclass with
an explicit `Code` — that is the only way to give a caller a distinguishable error, since the status
code is always 400 (§3.13).

**Failure modes.** Conflation, as above: three distinct causes (no such customer, empty body,
upstream returning `null` JSON) collapse into one code.

### 3.12 `AppException` and the error-code derivation

**Definition.** `public abstract class AppException : Exception` with a `virtual string Code`
(`…Api/Exceptions/AppException.cs:5-12`). `CustomerNotFoundException` is its only subclass in the
repository.

**Representation & storage.** `ExceptionToResponseMapper.GetCode` maintains a
`static readonly ConcurrentDictionary<Type, string> Codes`
(`…Api/Exceptions/ExceptionToResponseMapper.cs:11`) as a **process-lifetime memoisation** of
type→code. On a miss it takes the exception's own `Code` when non-blank, otherwise derives one by
`exception.GetType().Name.Underscore().Replace("_exception", string.Empty)`
(`…Api/Exceptions/ExceptionToResponseMapper.cs:30-34`) — `Humanizer`'s `Underscore()`, reached through
Convey's root namespace.

**Lifecycle.** Populated lazily, never evicted, never bounded. Because the key set is the finite set
of exception types in the assembly, unbounded growth is not a practical risk.

**Invariants & enforcement.**

- **`Code` is authoritative when set, derived otherwise.** The derivation is a fallback, not a
  convention check: nothing verifies that an explicit `Code` matches what the derivation would
  produce. Here it happens to: `CustomerNotFoundException` → `customer_not_found` either way.
- The cache is keyed on `Type`, and the first observed instance's `Code` wins for all later instances.
  Since `Code` is a fixed initialiser on the only subclass, this is currently safe. **An
  `AppException` subclass whose `Code` varied per instance would be silently pinned to whichever value
  was thrown first** — a real trap for anyone adding a parameterised code.

**Extension procedure.** Subclass `AppException`, set `Code` with an initialiser (not a
constructor-computed value, per the caching caveat), and — if the caller needs a status other than
400 — extend the `switch` in `Map` (§3.13).

**Failure modes.** Per-instance codes silently frozen; codes that drift from the type name if someone
renames a class without updating `Code`.

### 3.13 `ExceptionToResponseMapper` — everything is HTTP 400

**Definition.** A two-arm switch expression
(`…Api/Exceptions/ExceptionToResponseMapper.cs:13-20`): `AppException` →
`{code, reason}` at `HttpStatusCode.BadRequest`; **everything else** →
`{code:"error", reason:"There was an error."}` at `HttpStatusCode.BadRequest`.

**Representation & storage.** Registered as the error handler by
`AddErrorHandler<ExceptionToResponseMapper>()` (`…Api/Infrastructure/Extensions.cs:28`) and activated
by `UseErrorHandler()` (`…Api/Infrastructure/Extensions.cs:42`) `[convey]`.

**Lifecycle.** One `Map` call per unhandled exception escaping the pipeline.

**Invariants & enforcement.** The consequential property is that **there is no 404, no 401, no 403 and
no 500 anywhere in this service.**

| Situation | Status | Body `code` |
| --- | --- | --- |
| Customer does not exist | **400** | `customer_not_found` |
| `customers-service` unreachable / timed out | **400** | `error` |
| `CompletedOrders` was `null` → `ArgumentNullException` (§3.5) | **400** | `error` |
| `httpClient.services` missing the `customers` key (§3.9) | **400** | `error` |
| Any future bug | **400** | `error` |

A caller — including `orders-service`, which is a machine — therefore cannot tell "your input was
wrong" from "we are broken". **This is a deliberate platform-wide convention**, identical in shape to
`vehicles-service` (`component-internals/vehicles-service.md` §3.26) and every other service, so
changing it here alone would make this service inconsistent with the rest; it belongs in
[[framework-supplied-platform-conventions]], not in a local fix.

The generic arm also **discards the exception detail from the response** — the message is a constant.
The detail survives only in the log, which is where a 400-with-`error` has to be diagnosed from.

**Extension procedure.** To return a different status for a new failure class, add an arm above the
discard: `SomeException ex => new ExceptionResponse(…, HttpStatusCode.NotFound)`. Order matters — the
`AppException` arm matches subclasses, so a new subclass-specific arm must come *before* it.

**Failure modes.** Diagnosis by status code is impossible; monitoring that alerts on 5xx will never
fire for this service, no matter what breaks.

### 3.14 Dispatcher-bound endpoints and query-string binding

**Definition.** `UseDispatcherEndpoints` maps routes directly to CQRS messages with no controller,
no action method and no attribute routing (`…Api/Program.cs:29-31`)
([[dispatcher-bound-cqrs-endpoints]]). The only real route is
`Get<GetOrderPricing, OrderPricingDto>("pricing")` (`…Api/Program.cs:31`).

**Representation & storage.** The binding is positional-by-type: Convey resolves
`IQueryHandler<GetOrderPricing, OrderPricingDto>` from the container, binds a `GetOrderPricing` from
the request, dispatches, and serializes the returned `OrderPricingDto` `[convey]`.

**Lifecycle.** Registered once at startup, inside `Configure`.

**Invariants & enforcement.**

- **The query type must have a public parameterless constructor and settable properties.** Both hold
  (§3.1). Nothing enforces it; a query with get-only properties would bind to defaults silently —
  this is exactly the failure `vehicles-service` exhibits on its AMQP delete path
  (`component-internals/vehicles-service.md` §3.13).
- **The handler must be discoverable.** `AddQueryHandlers()` scans the assembly
  (`…Api/Infrastructure/Extensions.cs:29`) `[convey]`; a handler in a different assembly would not be
  found, and the failure surfaces as a DI resolution error at request time, not at startup.
- There is **no route-level authorization**: `UseDispatcherEndpoints` accepts an optional auth
  argument in other services, and it is not used here.

**Extension procedure.** A new read is three files: the query, the handler, and one line in
`Program.cs`. A new *write* would be a much larger change — this service has no command dispatcher,
no `AddCommandHandlers()`, and no broker (§3.17, §1.1).

**Failure modes.** Silent default binding if the message shape convention is broken, as above.

### 3.15 The unnamed root endpoint

**Definition.** `Get("", ctx => ctx.Response.WriteAsync(ctx.RequestServices.GetService<AppOptions>().Name))`
(`…Api/Program.cs:30`) — `GET /` returns the plain-text string `Pacco Pricing Service`
(`appsettings.json:3`).

**Representation & storage.** `AppOptions` is bound from the `app` section by `AddConvey()`
`[convey]`. Note `GetService<T>()`, not `GetRequiredService<T>()`: if the section were absent this
line would `NullReferenceException` rather than fail with a clear message.

**Lifecycle.** Per request. No caching, no dependencies beyond options.

**Invariants & enforcement.** `/` is excluded from both logging and tracing
(`logger.excludePaths` at `appsettings.json:36`; `jaeger.excludePaths` at `appsettings.json:77`), so
probe traffic does not pollute either. This is the convention every Pacco service follows.

Worth noting what this endpoint is **not**: it is not the Consul health check. Consul is configured
with `pingEndpoint: "ping"` (`appsettings.json:14`), a *different* path served by
`UseConvey()` `[convey]` (`…Api/Infrastructure/Extensions.cs:45`). `/` is a human-readable identity
banner; `/ping` is the liveness probe. Neither checks whether `customers-service` is reachable, so
**Consul will keep this instance in rotation while every real request fails** (§4.3).

**Extension procedure.** A meaningful readiness check would need a new endpoint that exercises the
outbound client, plus a change to `consul.pingEndpoint` in all three profiles.

**Failure modes.** False-healthy, as above.

### 3.16 The in-memory query dispatcher and handler discovery

**Definition.** `AddQueryHandlers().AddInMemoryQueryDispatcher()`
(`…Api/Infrastructure/Extensions.cs:29-30`) registers every `IQueryHandler<,>` in the assembly and an
`IQueryDispatcher` that resolves and invokes them in-process `[convey]`.

**Representation & storage.** No queue, no thread pool of its own, no persistence. "Dispatch" here is
a container resolution plus a direct `await` — the whole CQRS apparatus is a naming discipline, not a
transport.

**Lifecycle.** Registered at startup; one resolution per request.

**Invariants & enforcement.** One handler per query type, enforced only by the container: registering
two handlers for the same query would resolve the last-registered one, silently. There is exactly one
here (`…Api/Queries/Handlers/GetOrderPricingHandler.cs:11`), declared `internal sealed`, which is the
platform convention for handlers.

Note the asymmetry with every other service: there is **no** `AddCommandHandlers()`, no
`AddInMemoryCommandDispatcher()`, and no `AddEventHandlers()`. The `Convey.CQRS.Commands` package is
not even referenced (`…Api/Pacco.Services.Pricing.Api.csproj:10-22`). This service is queries-only at
the package level, not just by convention.

**Extension procedure.** Adding commands means adding `Convey.CQRS.Commands`, the registrations, and
— because a command implies a state change and this service has no state — almost certainly a
persistence stack too. That is the point at which this stops being `pricing-service` as designed.

**Failure modes.** Handler-not-found surfaces as a DI exception at request time → generic 400.

### 3.17 Statelessness — the absence of persistence, as a property

**Definition.** This service stores nothing. There is no `mongo` section, no `redis` section, no
`outbox` section in any of the four `appsettings*.json` files; no `Convey.Persistence.*` package
reference; no repository interface, document type, or `AsDocument` mapping anywhere in `src/`.

**Representation & storage.** The only in-process state that outlives a request is the
`ConcurrentDictionary` code cache in `ExceptionToResponseMapper` (§3.12) and whatever the Convey/ASP.NET
stack holds. Nothing is durable.

**Lifecycle.** The process can be killed and restarted at any point with no recovery step, no
migration and no data loss. `docker-compose down` and `docker-compose up` are equivalent to a restart
for this service alone.

**Invariants & enforcement.** Three consequences follow, and they are the reason this concept is
listed at all:

1. **Horizontal scaling is free.** Any number of instances are interchangeable; Fabio can round-robin
   without affinity.
2. **There is no cache, so `customers-service` load is exactly proportional to pricing load.** Every
   `GET /pricing` is one `GET /customers/{id}`. The platform has a shared Redis with
   [[prefix-partitioned-shared-cache]] — `vehicles-service` uses it with prefix `vehicles:`
   (`component-internals/vehicles-service.md` §3.36) — and this service does not participate.
3. **The service's answer is not reproducible.** Two identical requests seconds apart can return
   different discounts if the customer completed an order in between, and nothing records which answer
   was given. `orders-service` stores the resulting price on the order, so the *decision* is durable
   there, not here.

**Extension procedure.** If pricing ever needs to be auditable ("why did this customer get 15%?"),
the change is not a new field — it is introducing persistence, which this repository has no scaffolding
for at all: no Mongo package, no `Infrastructure/Mongo` folder, no `mongo` configuration, and no
`vault.lease` section to obtain database credentials (§3.25). Compare `vehicles-service`, which has
all four.

**Failure modes.** None internal. The externalised risk is availability: **`pricing-service` is
exactly as available as `customers-service`, with no degraded mode**, and no circuit breaker in
between.

### 3.18 The single-project layout — folders where other services have assemblies

**Definition.** The solution declares exactly one project,
`src\Pacco.Services.Pricing.Api\Pacco.Services.Pricing.Api.csproj`
(`Pacco.Services.Pricing.sln:8`), with root namespace `Pacco.Services.Pricing.Api`
(`…Api/Pacco.Services.Pricing.Api.csproj:6`). Inside it, six folders carry the names the platform
elsewhere gives to *assemblies*: `Core/`, `DTO/`, `Exceptions/`, `Infrastructure/`, `Queries/`,
`Services/`.

**Representation & storage.** Namespaces mirror folders (`Pacco.Services.Pricing.Api.Core.Entities`,
`…Api.Infrastructure`, …), so the *appearance* of layering is complete. What is missing is the
enforcement: with one assembly there is no project reference graph, so **the compiler will not stop
`Core` from depending on `Infrastructure`**.

**Lifecycle.** Fixed at repository creation; unchanged through `HEAD` (`ed66901`).

**Invariants & enforcement.**

| Layering rule | Elsewhere on the platform | Here |
| --- | --- | --- |
| `Core` references nothing | enforced by an empty `<ItemGroup>` in `…Core.csproj` (`component-internals/vehicles-service.md` §3.2) | **convention only** — `Core/Services/CustomerDiscountsService.cs` currently references only `Core/Entities`, and nothing prevents that changing |
| `Application` does not reference `Infrastructure` | enforced by project references | no `Application` folder exists; `Queries/` plays that role |
| `Api` composes the rest | enforced | `Program.cs` and `Infrastructure/Extensions.cs` sit in the same assembly as everything else |

The convention *is* currently respected — verified by reading every `using` directive in the 15 C#
files: no file under `Core/` imports `Infrastructure`, `DTO` or `Services`. The value of recording
this is that the respect is **voluntary and unverified**.

Note also the split that *is* unusual: `DTO/Extensions.cs` (the DTO→entity mapping) lives outside
`Core/` and imports `Core.Entities` — the direction is correct, but it means the entity's only
construction site is in a different conceptual layer from the entity.

**Extension procedure.** Two options, and the choice should be deliberate:

1. **Keep the single project.** Cheapest; correct for a service this size. If chosen, the layering
   rule should be written down (it currently is not, anywhere in the repository) — this document is
   now that record.
2. **Split into four projects** to match [[inward-dependency-service-skeleton]]. That means four
   `.csproj` files, four namespace roots, moving `ExceptionToResponseMapper` into `Infrastructure`,
   and updating `Dockerfile:4` (`dotnet publish src/Pacco.Services.Pricing.Api`) and
   `scripts/start.sh:3`. Worth it only if the service grows a persistence or messaging stack.

**Failure modes.** Silent architectural drift. There is no ArchUnit-style test, and no test project
at all (§3.30), so a `Core → Infrastructure` reference would be caught only in review.

### 3.19 Absence of caller context — no `IAppContext`, no identity

**Definition.** Every other Pacco service defines `IAppContext`, `IIdentityContext` and a
`CorrelationContext`, populated from the `Correlation-Context` header, and registers an
`IAppContextFactory` (see `component-internals/vehicles-service.md` §3.38 for the fully wired case).
**This repository contains none of them.** There is no `Contexts/` folder, no `IAppContext.cs`, and
no reference to `Correlation-Context` in any of the 15 C# files.

**Representation & storage.** Nothing to store. The handler's only inputs are its constructor
dependencies and the bound query (`…Api/Queries/Handlers/GetOrderPricingHandler.cs:17-23,25`).

**Lifecycle.** N/A.

**Invariants & enforcement.** The consequence is precise and worth stating plainly: **inside this
service, nothing knows who is asking.** The `customerId` in the query string is the *subject* of the
pricing calculation, and the service treats it as authoritative with no way to check that it matches
the caller. That check exists, but it lives at the gateway (§3.33), which rewrites the downstream
query string to `customerId=@user_id` — the value from the token — so an end user cannot price
another user's order *through the gateway*.

`orders-service`, however, calls this service **directly**, service-to-service, with whatever
`customerId` it chooses (`hianshul100_Pacco.Services.Orders/…/Clients/PricingServiceClient.cs:20-21`).
That is legitimate, but it means the gateway's binding is a *perimeter* control, not an invariant of
the service — anything inside the cluster can ask for any customer's discount.

**Extension procedure.** If a future change needs identity (per-user rate limiting, audit, or a
"customers may only price their own orders" rule enforced in-service), the whole context stack must be
introduced: the `IAppContext`/`IIdentityContext` pair, an `AppContextFactory`, and a registration —
copy the shape from `vehicles-service`, where it is already wired but, notably, also unused
(`component-internals/vehicles-service.md` §3.40). Both services build or omit caller context in a way
that is inert today; only the reason differs.

**Failure modes.** No in-service authorization is possible today. This is not a defect to fix
unilaterally — it is [[edge-enforced-authentication-with-identity-binding]] working as designed — but
it is a property to know before assuming `customerId` is trustworthy.

### 3.20 Correlation and span propagation — what survives, what does not

**Definition.** Two distinct propagation mechanisms exist on the platform, and this service
participates in exactly one.

| Mechanism | Present? | Evidence |
| --- | --- | --- |
| **Jaeger trace context** over HTTP | **Yes** — `AddJaeger()` / `UseJaeger()` (`…Api/Infrastructure/Extensions.cs:35,44`), plus Convey's outbound HTTP instrumentation `[convey]` | inbound spans are created for non-excluded paths; the outbound call to `customers-service` continues the trace |
| **`Correlation-Context` message header** | **No** — nothing reads or writes it here (§3.19) | the header is a broker-side concern; this service has no broker |
| **`span_context` message header** | **No** | same |
| Gateway `Request-ID` / `Trace-ID` | generated upstream (`ntrada.yml:16-17`, exposed via CORS at `ntrada.yml:38-40`) and forwarded (`forwardRequestHeaders: true`, `ntrada.yml:14`) | this service does not read them; they reach it only as opaque headers |

**Representation & storage.** `jaeger.serviceName` is **`pricing`** (`appsettings.json:72`), not
`pricing-service` — so the name in Jaeger differs from the name in Consul (`pricing-service`,
`appsettings.json:10`), from the Prometheus job (`pricing-service`,
`hianshul100_Pacco/compose/prometheus/prometheus.yml:46`), and from the compose container
(`pricing-service`, `hianshul100_Pacco/compose/services.yml:98`). Correlating a Jaeger trace with a
Prometheus series requires knowing that mapping; it is written down nowhere in the repository.

**Lifecycle.** Per request; sampler is `const` (`appsettings.json:76`), i.e. **every** request is
sampled when Jaeger is enabled — full-fidelity tracing, and full tracing cost.

**Invariants & enforcement.** `jaeger.excludePaths` (`appsettings.json:77`) keeps `/`, `/ping` and
`/metrics` out of the trace stream; without it, Consul's 3-second ping (`appsettings.json:15`) would
dominate every trace view.

**Extension procedure.** Aligning `jaeger.serviceName` with the Consul name is a one-key change in
`appsettings.json` and `appsettings.docker.json` — but it renames the service in Jaeger's UI and
breaks any saved query, so it is a coordinated change, not a local one.

**Failure modes.** Trace/metric correlation by name fails silently for anyone who assumes the names
match ([[correlation-and-span-propagation]]).

### 3.21 Inert JWT configuration and the committed certificate

**Definition.** `appsettings.json:79-87` carries a full `jwt` block —
`certificate.location: certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`,
`validateIssuer: true`, `validateLifetime: true` — and `…Api/certs/localhost.cer` is committed
(cited by path only, per the folder convention in `index.md` §3.2). The `.csproj` copies the whole
`certs\**` tree into the publish output (`…Api/Pacco.Services.Pricing.Api.csproj:26`), and
`appsettings.local.json:28-32` blanks `certificate.location`.

**The block is read by nothing.** There is no `Convey.Auth` package reference
(`…Api/Pacco.Services.Pricing.Api.csproj:10-22`), no `AddJwt()`, no `UseAuthentication()` and no
`UseAuthorization()` in `…Api/Infrastructure/Extensions.cs` or `…Api/Program.cs`.

**Representation & storage.** Configuration keys with no binder, and a certificate file shipped into
every container image for no runtime purpose.

**Lifecycle.** Loaded into `IConfiguration` at startup, never bound to an options type, never used.

**Invariants & enforcement.** None — that is the point. The risk here is **misleading evidence**: an
engineer reading `appsettings.json` will reasonably conclude the service validates JWTs. It does not.
Any request that reaches port 5008 directly is served, unauthenticated. The only thing between a
caller and this service is the gateway's `auth: true` (`ntrada.yml:407`) and network topology.

Note the difference from the certificate in §3.34: this one is a **JWT signing certificate**
(`localhost.cer`, an issuer public key), not a client certificate for mutual TLS. Vault's PKI section
(§3.25) would produce the latter, and nothing consumes that either.

**Extension procedure.** Either wire it or delete it. Wiring: add `Convey.Auth`, call `AddJwt()` in
`AddInfrastructure`, `UseAuthentication()`/`UseAuthorization()` in `UseInfrastructure`, and pass the
auth flag through `UseDispatcherEndpoints`. Deleting: remove the `jwt` blocks from all three profiles,
remove `certs/**` from the `.csproj` and the repository. **Do not leave it as is without a note** —
this document is that note.

**Failure modes.** Misdiagnosis (assuming auth exists); image bloat and a committed certificate that
implies a rotation obligation nobody owns.

### 3.22 `AddSecurity()` — registered, never injected

**Definition.** `AddSecurity()` is the last call in the builder chain
(`…Api/Infrastructure/Extensions.cs:37`), backed by the `Convey.Security` package
(`…Api/Pacco.Services.Pricing.Api.csproj:18`). It registers Convey's hashing/encryption/signing
helpers (`IHasher`, `IEncryptor`, `ISigner`, `IRng` `[convey]`).

**Representation & storage.** DI registrations only. **No type in this repository injects any of
them** — verified by reading all 15 C# files: the only constructor dependencies anywhere are
`IHttpClient`, `HttpClientOptions`, `ICustomersServiceClient`, `ICustomerDiscountsService` and
`ILogger<T>`.

**Lifecycle.** Registered at startup, resolved never.

**Invariants & enforcement.** None. This is dead wiring, and `git log -1` names it directly: the
`HEAD` commit is **"Added security package"** (`ed66901`) — the most recent change to this repository
added a dependency that nothing uses. That is useful context for anyone wondering whether the omission
is an oversight or a work-in-progress: it is the latter, abandoned at the registration step.

There is also **no `security` section in any `appsettings*.json`**, whereas `customers-service` has
one carrying its certificate ACL (`hianshul100_Pacco.Services.Customers/…/appsettings.json`). So even
if certificate authentication were switched on here, there would be no policy to enforce.

**Extension procedure.** If a future change needs signed URLs or hashed values, the helpers are
already available — inject them, no registration change needed. If not, `AddSecurity()` and the
package reference can be dropped together.

**Failure modes.** None functional. The cost is a misleading signal in both the code and the commit
history.

### 3.23 The dangling `Convey.Tracing.Jaeger.RabbitMQ` reference

**Definition.** `…Api/Pacco.Services.Pricing.Api.csproj:20` references
`Convey.Tracing.Jaeger.RabbitMQ` `0.4.*`. That package exists to add a **RabbitMQ tracing plugin** —
in `vehicles-service` it is consumed as
`AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`
(`component-internals/vehicles-service.md` §3.31). This service has no RabbitMQ, no
`Convey.MessageBrokers.RabbitMQ` reference, and no call site: `…Api/Infrastructure/Extensions.cs`
calls only `AddJaeger()` (line 35), from the base `Convey.Tracing.Jaeger` package (line 19).

**Representation & storage.** A package reference that contributes assemblies to the publish output
and nothing else.

**Lifecycle.** Restored at build, shipped in the image, never loaded in a meaningful way.

**Invariants & enforcement.** None. Its presence is the clearest evidence available that this project
was **scaffolded from a broker-enabled service template** and then had the broker removed
incompletely — consistent with the `jwt` block (§3.21) and `AddSecurity()` (§3.22), which tell the
same story from different angles. Recording this matters because a reader auditing `.csproj` files for
"which services use RabbitMQ?" will get a false positive here.

**Extension procedure.** Safe to delete: nothing references it. If messaging is ever added, it comes
back alongside `Convey.MessageBrokers.RabbitMQ`, not on its own.

**Failure modes.** False positives in dependency audits; a slightly larger image. No runtime effect.

### 3.24 Consul registration and Fabio addressing

**Definition.** `AddConsul()` and `AddFabio()` (`…Api/Infrastructure/Extensions.cs:32-33`) make this
instance discoverable and make its outbound calls routable
([[registry-mediated-discovery-and-routing]]).

**Representation & storage.** Configuration, per profile:

| Key | Base | `docker` | `local` |
| --- | --- | --- | --- |
| `consul.enabled` | `true` | `true` | **`false`** (`appsettings.local.json:3`) |
| `consul.url` | `http://localhost:8500` | `http://consul:8500` | — |
| `consul.service` | `pricing-service` | `pricing-service` | — |
| `consul.address` | **`docker.for.win.localhost`** (`appsettings.json:11`) | `pricing-service` | — |
| `consul.port` | `"5008"` | `"80"` | — |
| `consul.pingEndpoint` / `pingInterval` / `removeAfterInterval` | `ping` / `3` / `3` | same | `pingEndpoint: ""` |
| `fabio.enabled` / `fabio.url` | `true` / `http://localhost:9999` | `true` / `http://fabio:9999` | `false` |

**Lifecycle.** Registration happens at startup and deregistration on graceful shutdown `[convey]`;
with `removeAfterInterval: 3`, a hard-killed instance is evicted after roughly three failed pings.

**Invariants & enforcement.** Two things are worth flagging:

1. **`consul.address` in the base profile is `docker.for.win.localhost`** — a Docker Desktop for
   *Windows* hostname. The base profile is therefore not portable: running with
   `ASPNETCORE_ENVIRONMENT` unset (or `development`, which is `{}` — §3.26) on Linux or macOS
   registers an address other services cannot resolve. The service still starts and still passes its
   own ping; only *callers* fail. This same literal appears across the platform's services, so it is a
   platform-wide default, not a local mistake.
2. **The ping check proves liveness, not readiness** (§3.15). Consul cannot tell that
   `customers-service` is unreachable, so a broken instance stays in rotation.

**Extension procedure.** Adding an environment means adding an `appsettings.<env>.json` with the
matching `consul.address`, `consul.port` and `fabio.url` — and, because
`appsettings.development.json` is `{}`, remembering that `development` inherits the Windows-specific
base (§3.26).

**Failure modes.** Unreachable registration (case 1) and false-healthy registration (case 2). Both are
silent from this service's own point of view.

### 3.25 Vault — KV settings and PKI, with no lease

**Definition.** `UseVault()` (`…Api/Program.cs:33`) hooks Vault into the *host builder*, so it runs
before configuration is finalised `[convey]`. The `vault` section (`appsettings.json:106-124`) turns
on two of Vault's three roles.

| Sub-section | Configured | Consumed by anything in this repository? |
| --- | --- | --- |
| `vault.kv` (`engineVersion: 2`, `mountPoint: kv`, `path: pricing-service/settings`) | yes, `enabled: true` | **Yes, implicitly** — KV entries are merged into `IConfiguration`, so they can override *any* key above |
| `vault.pki` (`roleName: pricing-service`, `commonName: pricing-service.pacco.io`) | yes, `enabled: true` | **No** — nothing reads the issued certificate (§3.34) |
| `vault.lease` (dynamic database credentials) | **absent** | n/a — there is no database (§3.17) |

Authentication is `authType: token` with `token: "secret"` (`appsettings.json:109-110`) — the
unchanged Convey default, quoted here per `index.md` §3.2 precisely because *being* the default is the
evidence. `username`/`password` are also present as defaults for the unused `userpass` path.

**Representation & storage.** KV values live in Vault and are injected at startup; they are not in the
repository. This is the mechanism by which a real deployment supplies real secrets without committing
them — [[vault-issued-dynamic-credentials-and-service-pki]].

**Lifecycle.** Read once, at startup. **There is no lease renewal and no re-read**, because there is
no `vault.lease` section: a rotated value takes effect only on restart.

**Invariants & enforcement.**

- **KV overrides are invisible in the repository.** Any key in this document's tables can be
  different at runtime if `kv/pricing-service/settings` sets it. That includes
  `httpClient.services.customers`. Nothing in the code logs the effective value.
- **Vault is disabled in both non-production profiles**: `appsettings.local.json:37-44` and
  `appsettings.docker.json:75-84` set `enabled: false` for the section and both sub-sections. So the
  *only* profile that exercises Vault is the base one — the least-tested path.
- PKI is enabled and unused: a certificate is requested at startup and discarded. That is a
  **wasted Vault round-trip on every boot** and, more importantly, a strong signal that certificate
  authentication was intended here (§3.34).

**Extension procedure.** To make this service present its Vault certificate to `customers-service`,
the change is: read the issued certificate from Convey's Vault integration and attach it as a header
in `CustomersServiceClient`, mirroring
`hianshul100_Pacco.Services.Availability/…/Clients/CustomersServiceClient.cs:16-34`, **and** add
`pricing-service` to the `customers-service` ACL (§3.34, **B-1**). Both halves, or the call starts
failing.

**Failure modes.** A Vault outage at startup is a startup failure, not a request failure — `[convey]`
behaviour on an unreachable Vault is `Unverifiable — Missing Source Evidence`. Silent configuration
divergence between environments, as above.

### 3.26 Environment layering — four `appsettings` profiles

**Definition.** ASP.NET Core loads `appsettings.json` then `appsettings.{ASPNETCORE_ENVIRONMENT}.json`
on top ([[composable-per-concern-environment-stacks]]). Four profiles exist:

| File | Size | Selected by | Net effect |
| --- | --- | --- | --- |
| `appsettings.json` | 125 lines | always (base) | everything on: Consul, Fabio, Jaeger, metrics, Seq, file logging, Vault, Swagger |
| `appsettings.local.json` | 46 lines | `scripts/start.sh:2` (`export ASPNETCORE_ENVIRONMENT=local`) and both `launchSettings.json` profiles (`…Api/Properties/launchSettings.json:14,22`) | everything platform-side **off**; `customers` addressed as `http://localhost:5002`; log level `verbose` |
| `appsettings.docker.json` | 85 lines | `Dockerfile:10` (`ENV ASPNETCORE_ENVIRONMENT docker`) | container hostnames (`consul`, `fabio`, `seq`, `jaeger`, `influx`); **Vault off**; file logging off |
| `appsettings.development.json` | `{}` | `ASPNETCORE_ENVIRONMENT=development` | **nothing** — falls through entirely to the base profile |

**Representation & storage.** Plain JSON in the repository, plus the Vault KV overlay (§3.25) which
is applied *after* both files.

**Lifecycle.** Read at startup only. No `reloadOnChange` behaviour is configured explicitly; the
default host builder enables it, but nothing in this service re-reads options after binding, so a
changed file has no effect until restart.

**Invariants & enforcement.** Three properties matter, and none is enforced by anything:

1. **JSON objects are merged key-by-key, but a redefined object replaces only the keys it names.**
   The practical trap is `httpClient.services` (§3.9): `appsettings.local.json:12-14` names only
   `customers`, so a *second* service added to the base file **would** still be inherited locally —
   but the `type: ""` on line 10 means the inherited logical name would be used verbatim as a URL.
   Adding a service therefore requires touching the local profile too.
2. **`appsettings.development.json` being empty is a live hazard.** Running under the
   conventional .NET development environment produces the *production-shaped* base configuration,
   including `consul.address: docker.for.win.localhost` (§3.24) and `vault.enabled: true` pointing at
   `http://localhost:8200`. The intended dev profile is `local`, not `development` — a distinction
   carried only by `scripts/start.sh` and `launchSettings.json`.
3. **The docker profile disables Vault**, so the container image as built runs with every value from
   the repository and none from Vault.

**Extension procedure.** Any new configuration key must be considered against all four profiles.
The safe default is: add it to `appsettings.json` with a working value, and override in `local` and
`docker` only when the value must differ.

**Failure modes.** Wrong-environment startup (case 2) produces a service that *runs* and *registers*
but is unreachable — the worst failure shape available, because health checks pass.

### 3.27 Logging — level, path exclusion, property redaction, sinks

**Definition.** `UseLogging()` on the host builder (`…Api/Program.cs:32`) installs Serilog configured
from the `logger` section (`appsettings.json:34-69`) `[convey]`.

**Representation & storage.**

| Key | Base | `local` | `docker` | Effect |
| --- | --- | --- | --- | --- |
| `logger.level` | `information` | `verbose` | (inherits `information`) | minimum level |
| `logger.excludePaths` | `["/", "/ping", "/metrics"]` | inherits | inherits | suppresses probe noise (§3.15) |
| `logger.excludeProperties` | 12 names (§below) | inherits | inherits | redaction list |
| `logger.console.enabled` | `true` | inherits | `true` | stdout — the sink that matters under Docker/PM2 |
| `logger.file.enabled` / `path` / `interval` | `true` / `logs/logs.txt` / `day` | **`false`** | **`false`** | daily rolling file, on only in the base profile |
| `logger.elk.enabled` | `false` | inherits | `false` (url `http://elk:9200`) | never on in this repository |
| `logger.seq.enabled` / `url` / `apiKey` | `true` / `http://localhost:5341` / `"secret"` | **`false`** | `true` / `http://seq:5341` / `"secret"` | structured log sink |
| `logger.tags` | `{}` | inherits | inherits | no static enrichment — nothing tags logs with the service name |

`logger.seq.apiKey: "secret"` (`appsettings.json:66`, `appsettings.docker.json:45`) is the unchanged
Convey default, quoted verbatim per `index.md` §3.2 because the fact that it is a default *is* the
finding.

**Lifecycle.** Configured once at startup; sinks are flushed on shutdown by Serilog.

**Invariants & enforcement.** The redaction list —
`api_key`, `access_key`, `ApiKey`, `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`,
`Password`, `Email`, `Login`, `Secret`, `Token` (`appsettings.json:37-50`) — is
[[structured-logging-with-property-redaction]], and it has a **specific hole in this service**:

> The only log statement in the entire service is an **interpolated string**
> (`…Api/Queries/Handlers/GetOrderPricingHandler.cs:37-39`), not a Serilog message template. The
> interpolation happens in C# *before* Serilog sees it, so there are no structured properties to
> exclude — `excludeProperties` cannot redact anything in this message, and the customer id, base
> price, discount and final price are baked into a flat string. Nothing sensitive by the list's own
> definition is present, so there is no leak today; but the mechanism is inert here, and a future log
> line that interpolated an email or token would bypass redaction entirely.

The same line also loses queryability: `customerId` is not a searchable field in Seq, only substring
text.

There is a second, complementary masking mechanism —
`httpClient.requestMasking.enabled: true` with `maskTemplate: "*****"` (`appsettings.json:29-32`) —
which masks outbound HTTP request logging `[convey]`. Note it is present in the base profile only:
`appsettings.local.json:9-15` and `appsettings.docker.json:22-28` both redefine `httpClient` **without**
`requestMasking`. Because JSON configuration merges by key, the base value survives — the key is not
*removed*, merely not restated. `Unverifiable — Missing Source Evidence` as to what Convey masks.

**Extension procedure.** New log statements should use Serilog message templates
(`_logger.LogInformation("… {CustomerId} …", query.CustomerId)`) rather than interpolation, so that
redaction and structured search work. Converting the existing line is a safe, isolated improvement.

**Failure modes.** Redaction bypass via interpolation, as above; no `logger.tags`, so distinguishing
this service's logs in a shared Seq instance relies on the sink's own metadata.

### 3.28 Swagger docs surface

**Definition.** `AddWebApiSwaggerDocs()` (`…Api/Infrastructure/Extensions.cs:36`) and
`UseSwaggerDocs()` (`…Api/Infrastructure/Extensions.cs:43`), configured by `swagger`
(`appsettings.json:97-105`): `enabled: true`, `routePrefix: docs`, `name`/`version` `v1`,
`title: API`, `reDocEnabled: false`, `includeSecurity: true`.

**Representation & storage.** Generated at runtime from the dispatcher endpoint registrations
`[convey]`, not from controllers — `AddWebApiSwaggerDocs` is the CQRS-aware variant, which is why it
can document `GET /pricing` at all despite there being no action method.

**Lifecycle.** Served on every request to `/docs`.

**Invariants & enforcement.** Two things to know:

- **It is enabled in every profile**, including `docker` (`appsettings.docker.json:66-74`), and
  neither `local` nor `docker` turns it off. The API documentation is therefore reachable from
  anywhere that can reach port 5008 — which is also anywhere that can call the service
  unauthenticated (§3.21).
- `includeSecurity: true` adds an authorization affordance to the generated document, describing a
  security scheme this service does not implement (§3.21) — a second place where the repository
  advertises auth it does not have.
- `/docs` is **not** in `logger.excludePaths` or `jaeger.excludePaths`, so documentation hits are
  logged and traced like real traffic.

Note the contrast with services that publish message contracts: this service does **not** call
`UsePublicContracts<T>()` (compare `component-internals/vehicles-service.md` §3.34), because it has no
messages to publish. Swagger is its only machine-readable surface description.

**Extension procedure.** A new route is documented automatically; no annotation step exists. To
suppress documentation in production, set `swagger.enabled: false` in the deployed profile.

**Failure modes.** Information disclosure of the API shape to anyone who can reach the port; no
functional risk.

### 3.29 Metrics and tracing wiring

**Definition.** `AddMetrics()` / `UseMetrics()` (`…Api/Infrastructure/Extensions.cs:34,46`) and
`AddJaeger()` / `UseJaeger()` (`…Api/Infrastructure/Extensions.cs:35,44`), plus `UseConvey()`
(`…Api/Infrastructure/Extensions.cs:45`) which serves `/ping` and the framework endpoints `[convey]`.

**Representation & storage.** `metrics` (`appsettings.json:88-96`): `enabled: true`,
`prometheusEnabled: true`, `influxEnabled: false`, `database: pacco`, `env: local` (base) /
`docker` (`appsettings.docker.json:63`), `interval: 5`. The Prometheus scrape target is declared
outside this repository, as job `pricing-service` against host `pricing-service`
(`hianshul100_Pacco/compose/prometheus/prometheus.yml:46-48`).

**Lifecycle.** `UseMetrics()` is the **last** call in `UseInfrastructure`
(`…Api/Infrastructure/Extensions.cs:42-46`); middleware order there is
`UseErrorHandler → UseSwaggerDocs → UseJaeger → UseConvey → UseMetrics`. That ordering means the error
handler wraps everything (so all exceptions become 400s, §3.13) and metrics see the request last.

**Invariants & enforcement.** The metrics exposed are Convey/AppMetrics defaults — request counts,
durations, status codes. **There is no application-level metric**: no counter for discounts applied,
no histogram of discount values, no counter for `customer_not_found`. Since every failure is an HTTP
400 (§3.13), a Prometheus alert cannot distinguish "callers are sending bad ids" from
"`customers-service` is down" — both show as 400s at the same rate.

`metrics.env` is `local` in the base profile, which is the same portability smell as
`consul.address` (§3.24): a deployment that forgets to set an environment tags its series `local`.

**Extension procedure.** Adding a domain metric means injecting AppMetrics' `IMetrics` into
`GetOrderPricingHandler` and recording alongside the existing log line. That is also the natural place
to make §3.8's silent fallbacks observable — a counter incremented when the `> 0` branch is taken
would turn an invisible failure into an alertable one.

**Failure modes.** Undifferentiated 400 rate, as above.

### 3.30 Absence of a test suite, and the green-but-empty test step

**Definition.** There is no test project. `scripts/test.sh` is two lines —
`#!/bin/bash` and `dotnet test` (`scripts/test.sh:1-2`) — run by CI at `.travis.yml:14`, between
`build.sh` and `dockerize.sh`.

**Representation & storage.** `dotnet test` executed at the repository root discovers projects from
`Pacco.Services.Pricing.sln`, which declares one non-test project
(`Pacco.Services.Pricing.sln:8`). The command finds nothing to run and exits `0`.

**Lifecycle.** Every CI build on `master` and `develop` (`.travis.yml:6-9`) runs this step, and it has
always passed.

**Invariants & enforcement.** **Nothing about this service's behaviour is verified anywhere.** In
particular, none of the following is covered:

| Behaviour | Would a test have caught it? |
| --- | --- |
| `Customer.Id` never assigned (§3.3) | a two-line constructor test, yes |
| Discount band boundaries at 0/1/3/4/9/10 (§3.6) | a table test, yes — and it is the single highest-value test to add |
| VIP bonus additive and the `0.2` ceiling (§3.7) | yes |
| `> 0` fallback returning `OrderPrice` for non-positive inputs (§3.8) | yes |
| `AsEntity` NRE on null `CompletedOrders` (§3.5) | yes |
| The `customers` configuration key being present (§3.9) | only an integration test |

Compare the platform norm: `orders-service` and `parcels-service` each carry unit, integration and
(disabled) contract projects ([[layered-service-test-suite]]). This repository has none of the three.

**Extension procedure.** The cheapest meaningful addition is a single xUnit project testing
`CustomerDiscountsService` — it has no dependencies at all (`…Api/Core/Services/CustomerDiscountsService.cs:5-7`),
so the test needs no mocks, no fixtures and no infrastructure. Adding it requires: a
`tests/Pacco.Services.Pricing.Tests.Unit` project, a `<ProjectReference>` to the API project, and a
solution entry — after which `scripts/test.sh` starts asserting something without any change to
`.travis.yml`.

**Failure modes.** A green CI signal that carries no information. This is worse than no test step,
because the badge in `README.md:15-16` implies verification that does not exist.

### 3.31 Committed IDE state and the absent `LICENSE`

**Definition.** Six of the repository's 39 tracked files are JetBrains Rider project state under
`src/Pacco.Services.Pricing.Api/.idea/.idea.Pacco.Services.Pricing.dir/.idea/` —
`contentModel.xml`, `indexLayout.xml`, `modules.xml`, `vcs.xml`, `workspace.xml`, and
`riderModule.iml`.

**Representation & storage.** Version-controlled XML describing one developer's local IDE
configuration. `workspace.xml` in particular records editor and run-configuration state and is
rewritten by the IDE on ordinary use.

**Lifecycle.** Committed once; regenerated locally by anyone who opens the project in Rider, producing
spurious diffs.

**Invariants & enforcement.** `.gitignore` does not exclude them (they are tracked, so it could not
retroactively). `.dockerignore` also does not list `.idea` (`.dockerignore:1-29`), so the `COPY . .`
at `Dockerfile:3` pulls them into the build context — harmless for the final image, since only
`/app/out` is copied forward (`Dockerfile:8`), but it enlarges the context.

There is also **no `LICENSE` file** in this repository, while the upstream project it is derived from
is described as open source (`README.md:6`). Recorded as an observation, not a legal conclusion.

**Extension procedure.** `git rm -r --cached src/Pacco.Services.Pricing.Api/.idea` plus a `.gitignore`
entry, in a commit of its own. Note this repository is a **read-only clone in this workspace** and was
not modified.

**Failure modes.** Merge conflicts on `workspace.xml`; no runtime effect.

### 3.32 Deployment identity — port, image, PM2 app, Prometheus job

**Definition.** The service is known by five different names across five files, and they do not all
agree:

| Context | Identifier | Evidence |
| --- | --- | --- |
| Convey app name | `Pacco Pricing Service` | `appsettings.json:3` |
| Convey/Consul service | `pricing-service` | `appsettings.json:4,10,21` |
| Docker image | `devmentors/pacco.services.pricing` | `hianshul100_Pacco/compose/services.yml:97`; built as `$DOCKER_USERNAME/pacco.services.pricing` (`scripts/dockerize.sh:16`) |
| Compose container / network alias | `pricing-service`, `5008:80` | `hianshul100_Pacco/compose/services.yml:96-103` |
| PM2 app (dev) | `pricing` | `hianshul100_Pacco/services.yml:34-37` |
| PM2 app (prod) | `pricing`, `ASPNETCORE_URLS: http://*:5008` | `hianshul100_Pacco/prod-services.yml:50-55` |
| Prometheus job | `pricing-service` | `hianshul100_Pacco/compose/prometheus/prometheus.yml:46-48` |
| Jaeger service | **`pricing`** | `appsettings.json:72` (§3.20) |
| Gateway service key | `pricing-service`, `localUrl: localhost:5008` | `ntrada.yml:410-412` |

**Representation & storage.** Port `5008` is the constant that ties them together, and it appears in
six places: `appsettings.json:12`, `…Api/Properties/launchSettings.json:6,20`,
`hianshul100_Pacco/compose/services.yml:101`, `hianshul100_Pacco/prod-services.yml:55`,
`ntrada.yml:411`, and `Pacco.Services.Pricing.rest:1`. Inside the container the app listens on **80**
(`Dockerfile:9`, `ENV ASPNETCORE_URLS http://*:80`) and 5008 is only the host-side mapping.

**Lifecycle.** Image tags are branch-derived: `master` → `latest` + build number, `develop` → `dev` +
`dev-<build>`, anything else → **empty tags** (`scripts/dockerize.sh:2-14`). A build on a third branch
would run `docker build -t $REPOSITORY: …` with an empty tag — but `.travis.yml:6-9` restricts CI to
`master` and `develop`, so the case does not arise in CI. It would arise for a manual run of the
script.

**Invariants & enforcement.** Nothing enforces that the seven names and the port agree; they agree by
inspection today. **`hianshul100_Pacco/services.yml:34-37` (the dev PM2 manifest) sets no
`ASPNETCORE_URLS` and runs `dotnet run` from the project directory**, so the port and environment come
from `launchSettings.json` — meaning dev PM2 runs the `local` profile on 5008, while prod PM2 runs a
published DLL with the *base* profile (no `ASPNETCORE_ENVIRONMENT` is set at
`prod-services.yml:50-55`) on 5008. **The production manifest therefore runs the profile that points
Consul at `docker.for.win.localhost`** (§3.24) — recorded as **B-2**.

**Extension procedure.** Changing the port means editing all six locations plus the compose mapping;
there is no single source of truth. [[independent-per-repository-release]] means this repository can
be released alone, but a port change cannot be — it requires a coordinated change in
`hianshul100_Pacco` and `hianshul100_Pacco.APIGateway`.

**Failure modes.** Name drift between observability systems (§3.20); the prod-profile mismatch above.

### 3.33 Gateway exposure and the `customerId=@user_id` rewrite

**Definition.** The gateway declares a single-route `pricing` module in all four Ntrada
configurations, identically (`ntrada.yml:400-412`, `ntrada.docker.yml:400-412`,
`ntrada-async.yml:468-480`, `ntrada-async.docker.yml:468-480`):

| Aspect | Value | Line (`ntrada.yml`) |
| --- | --- | --- |
| Module path | `pricing` | `:401` |
| Upstream | `GET /pricing` (module path + `/`) | `:403-404` |
| Mode | `use: downstream` — a proxy, **in the async profile too** | `:405` |
| Downstream | `pricing-service/pricing?customerId=@user_id` | `:406` |
| Auth | `auth: true` (with global `auth.enabled: true, global: false`, `ntrada.yml:1-3`) | `:407` |
| Service resolution | `localUrl: localhost:5008`, `url: pricing-service`, selected by `useLocalUrl: true` (`ntrada.yml:18`) and `loadBalancer.enabled: false` (`ntrada.yml:19-20`) | `:410-412` |

**Representation & storage.** `@user_id` is an Ntrada token substituted from the authenticated
principal `[ntrada]`. `passQueryString: true` (`ntrada.yml:13`) forwards the caller's own query
string, and the `?customerId=@user_id` in the downstream template supplies the customer id from the
token — so `orderPrice` arrives from the caller and `customerId` from the token.

**Lifecycle.** Per request, at the edge.

**Invariants & enforcement.** This is the **only** place on the platform where a user is prevented
from pricing another customer's order ([[edge-enforced-authentication-with-identity-binding]]). It is
also the one behaviour of the pricing surface that is **not implemented in this repository at all** —
`pricing-service` will happily answer for any id (§3.19).

Two subtleties:

- **The async profile does not change pricing.** Every other write-capable module gains
  `use: rabbitmq` routes in `ntrada-async.yml`; pricing stays `downstream` because it is a read with a
  synchronous answer ([[dual-mode-edge-write]] does not apply). It is a useful negative example of
  where that pattern stops.
- **Precedence of the two `customerId` values is `[ntrada]`-owned.** If a caller supplies
  `?customerId=<someone-else>` and `passQueryString` merges it with the template's `@user_id`, which
  wins is not determinable from this workspace — **`Unverifiable — Missing Source Evidence`**, recorded
  as **Q-2**. It is the single most security-relevant unknown in this model, because the whole
  isolation guarantee rests on it.

**Extension procedure.** Adding a query parameter to `GET /pricing` requires editing the `downstream`
template in **all four** `ntrada*.yml` files, or the parameter reaches the service only on some
profiles.

**Failure modes.** If `@user_id` resolves to empty (a token without the expected claim), the downstream
receives `customerId=` → `Guid.Empty` → `customer_not_found` (§3.1). The user sees "customer not
found" for what is really a token problem.

### 3.34 The unauthenticated read of a certificate-protected endpoint

**Definition.** `customers-service` enables certificate authentication —
`AddCertificateAuthentication()` and `UseCertificateAuthentication()`
(`hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Infrastructure/Extensions.cs:79,91`)
— configured by a `security.certificate` section
(`hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-182`)
whose ACL contains **exactly one entry**: `availability-service`, with `validIssuer: localhost` and
permission `customers:read`. That mechanism is modelled from the other side in
`component-internals/customers-service.md` §3.25.

`pricing-service` calls `GET /customers/{id}` on that service
(`…Api/Services/Clients/CustomersServiceClient.cs:19-20`) with a client whose constructor takes only
`IHttpClient` and `HttpClientOptions` and sets **no headers**
(`…Api/Services/Clients/CustomersServiceClient.cs:13-17`). Compare `availability-service`, whose
client injects `ICertificatesService`, `VaultOptions` and `SecurityOptions`, fetches the Vault-issued
certificate for `vault.pki.roleName`, and attaches it under the configured header name
(`hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs:16-34`).

`pricing-service` **has** the ingredients — `vault.pki` is enabled with `roleName: pricing-service`
(`appsettings.json:119-123`) and `Convey.Security` is referenced (§3.22) — and uses none of them. It
also has **no `security` section** in any profile, so `SecurityOptions` would bind to defaults.

**Representation & storage.** An asymmetry across two repositories; nothing in either one records it.

**Lifecycle.** Every pricing request performs this call.

**Invariants & enforcement.** What actually happens to an uncertificated request is
**`Unverifiable — Missing Source Evidence`**: `Convey.WebApi.Security`'s middleware is not in this
workspace, and `customers-service`'s route registrations
(`hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/Program.cs:33-41`) pass no
per-route permission argument, so which routes the `customers:read` permission gates cannot be read
from source. Two readings are possible and they have opposite consequences:

1. **The middleware rejects requests without a valid certificate.** Then `pricing-service` has never
   worked against a Vault-enabled `customers-service`: every call fails, `IHttpClient` returns `null`
   or throws, and the caller sees `customer_not_found` or the generic `error` (§3.13). This is
   consistent with `vault.enabled: false` in `appsettings.docker.json:76` — the deployed compose
   environment has Vault off, so the ACL is presumably not enforced there either, and the problem
   would only appear in a fully-provisioned environment.
2. **The middleware only *identifies* certificated callers and enforces permissions where a route
   asks for one.** Then the call succeeds and the ACL is simply not applied to this route.

Either way the asymmetry is real and undocumented, and it is the highest-value open item in this
model — **B-1** and **Q-1**.

**Extension procedure.** See §3.25's extension procedure: attach the certificate *and* add
`pricing-service` to the ACL in `customers-service`'s `security.certificate.acl`, in a coordinated
two-repository change. Doing only the first fails closed; doing only the second changes nothing.

**Failure modes.** Under reading 1, a total and silent outage of pricing in any environment where
`customers-service` has Vault and certificate auth fully enabled — surfaced to users as
"customer not found".

### 3.35 Currency, units and rounding — what the numbers mean

**Definition.** All three response fields are bare `decimal`
(`…Api/DTO/OrderPricingDto.cs:5-7`). There is no currency code, no `CultureInfo`, no `Math.Round`, and
no minor-unit convention anywhere in the service.

**Representation & storage.** `decimal` is the right .NET type for money — exact base-10, 28–29
significant digits, no binary rounding drift. That part is correct and consistent.

**Lifecycle.** The single arithmetic operation is
`OrderPrice - discount * OrderPrice` (`…Api/Queries/Handlers/GetOrderPricingHandler.cs:35`).

**Invariants & enforcement.**

- **Results are not rounded.** With the fixed discount rates (`0.02`, `0.05`, `0.1`, `0.2`) and a
  price with two decimal places, the product has at most four decimal places — e.g.
  `99.99 × 0.95 = 94.9905`. The service returns `94.9905`, and **whoever displays or charges it owns
  the rounding**. `orders-service` stores the returned value
  (`hianshul100_Pacco.Services.Orders/…/Clients/PricingServiceClient.cs:20-21`) without rounding it.
- **`CustomerDiscount` is a ratio in `[0, 0.2]`**, not a percentage and not an amount — yet the only
  log line labels it `"$"` (`…Api/Queries/Handlers/GetOrderPricingHandler.cs:38`). A reader of Seq
  will see `discount: 0.15 $` and reasonably conclude it is money. **The field name is right and the
  log label is wrong**; the response contract carries no unit information either way.
- **There is no currency.** The platform has no multi-currency concept anywhere, so this is
  consistent — but it means adding one later touches this contract, `orders-service`'s copy of it, and
  the discount arithmetic.

**Extension procedure.** If rounding is ever needed, it belongs *after* the discount computation and
*before* the DTO, as `Math.Round(orderDiscountPrice, 2, MidpointRounding.ToEven)`, and it must be
introduced together with a decision about `CustomerDiscount`'s precision. Changing it silently changes
prices.

**Failure modes.** Unbounded decimal places reaching a payment surface; unit confusion from the log
label.

### 3.36 Threshold coupling — 10/4/1 here, 20 in `customers-service`

**Definition.** This service's discount ladder keys off `CompletedOrdersNumber` with thresholds
`10`, `3` and `0` (`…Api/Core/Services/CustomerDiscountsService.cs:11,15,19`). It *also* consumes
`IsVip` (`…Api/DTO/CustomerDto.cs:9`), which is computed inside `customers-service` against **its own,
different** completed-order threshold (`component-internals/customers-service.md` §3.5).

**Representation & storage.** Two unrelated constants in two repositories, both derived from the same
underlying quantity — the number of orders a customer has completed.

**Lifecycle.** Independent. Either can be changed without the other noticing.

**Invariants & enforcement.** None. The composite behaviour is worth writing out, because it is not
obvious from either side alone:

| Completed orders | Band (here) | VIP (there) | Total discount |
| --- | --- | --- | --- |
| 0 | `0.00` | no | **0%** |
| 1–3 | `0.02` | no | **2%** |
| 4–9 | `0.05` | no | **5%** |
| 10–19 | `0.10` | no | **10%** |
| 20+ | `0.10` | **yes** | **20%** |

So the ladder has a **fourth, invisible step at the VIP threshold** — a step this repository does not
contain and cannot see. Lowering the VIP threshold in `customers-service` doubles the top discount
with no change to, and no review of, `pricing-service`. The exact VIP threshold is
`customers-service`'s to state; this model deliberately cites it rather than restating a number that
could drift.

**Extension procedure.** Any change to either constant is a **pricing change** and should be reviewed
as one across both repositories. If the coupling is ever to be made explicit, the cleanest option is
for `customers-service` to expose the completed-order count it used and for this service to derive VIP
locally — or for the whole ladder to move behind one owner.

**Failure modes.** Uncoordinated pricing changes, invisible in either repository's diff.

---

## 4. Primary control flows

The service has exactly **three** end-to-end flows: one request path, one startup path, and one
failure path that is really a family. All three are traced here in full.

### 4.1 `GET /pricing` — the happy path, end to end

| # | Step | Where | Notes |
| --- | --- | --- | --- |
| 1 | User calls `GET /pricing?orderPrice=100` on the gateway with a bearer token | `ntrada.yml:401-407` | `auth: true`; global auth is `enabled: true, global: false` (`ntrada.yml:1-3`), so it applies per route |
| 2 | Ntrada validates the token, generates `Request-ID` and `Trace-ID` | `ntrada.yml:16-17` | forwarded downstream (`forwardRequestHeaders: true`, `:14`) |
| 3 | Ntrada rewrites the downstream URL to `pricing-service/pricing?customerId=@user_id`, resolving `@user_id` from the principal | `ntrada.yml:406` | `useLocalUrl: true` (`:18`) picks `localhost:5008` (`:411`); `loadBalancer.enabled: false` (`:19-20`) means no Fabio hop at the edge |
| 4 | ASP.NET Core routes `GET /pricing` to the dispatcher endpoint | `…Api/Program.cs:31` | no controller, no filter pipeline of the usual kind |
| 5 | Convey binds `GetOrderPricing` from the query string | `[convey]` | settable properties make this work (§3.1); an absent `customerId` binds `Guid.Empty` without error |
| 6 | The dispatcher resolves and invokes `GetOrderPricingHandler` | `…Api/Infrastructure/Extensions.cs:29-30` | in-process, synchronous `await` |
| 7 | The handler calls `ICustomersServiceClient.GetAsync(query.CustomerId)` | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:27` | **the only I/O in the request** |
| 8 | The client issues `GET {customers-service}/customers/{id}` | `…Api/Services/Clients/CustomersServiceClient.cs:19-20` | `httpClient.type: fabio` routes it through Fabio (§3.10); **no client certificate is attached** (§3.34); `httpClient.retries: 3` applies `[convey]` |
| 9 | `customers-service` returns `CustomerDetailsDto`; Convey deserializes the three fields this service declares | `…Api/DTO/CustomerDto.cs:8-10` | superset → subset (§3.4) |
| 10 | `customer.AsEntity()` maps to `Customer`, collapsing `CompletedOrders` to a count | `…Api/DTO/Extensions.cs:8-9` | **`Id` is dropped** (§3.3); **`.Count()` on a null list would throw here** (§3.5) |
| 11 | `CalculateDiscount` applies the band, then the VIP bonus | `…Api/Core/Services/CustomerDiscountsService.cs:9-29` | result in `[0, 0.2]` |
| 12 | `orderDiscountPrice = OrderPrice - discount * OrderPrice` | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:35` | no rounding (§3.35) |
| 13 | One `Information` log line, interpolated | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:37-39` | not a Serilog template → not redactable, not queryable (§3.27); labels the discount ratio `"$"` |
| 14 | `OrderPricingDto` is built; `OrderDiscountPrice` passes through the `> 0` ternary | `…Api/Queries/Handlers/GetOrderPricingHandler.cs:41-46` | the silent fallback (§3.8) |
| 15 | Convey serializes the DTO; response 200 | `[convey]` | |
| 16 | Ntrada returns the body to the caller unchanged — the `pricing` route declares no `onSuccess` transformation | `ntrada.yml:403-407` | contrast the `vehicles` list route, which unwraps `response.data.items` |

**The whole request is one outbound HTTP call and about twenty lines of arithmetic.** There is no
database round-trip, no message publish, no cache lookup and no lock.

### 4.2 `orders-service` calling directly — the second caller

`orders-service` calls this service without going through the gateway:

| # | Step | Where |
| --- | --- | --- |
| 1 | Order creation needs a price | `hianshul100_Pacco.Services.Orders/…/Commands/Handlers/…` |
| 2 | `PricingServiceClient` issues `GET {url}/pricing?customerId={customerId}&orderPrice={orderPrice}` | `hianshul100_Pacco.Services.Orders/…/Services/Clients/PricingServiceClient.cs:20-21` |
| 3 | Address comes from **its own** `httpClient.services["pricing"]` | same file | 
| 4 | Steps 4–15 of §4.1 run identically | — |

Two differences matter. **First, no `@user_id` substitution happens** — `orders-service` supplies the
customer id itself, so the gateway's isolation guarantee (§3.33) does not apply on this path.
**Second, a failure here becomes an order failure**: this service's 400 propagates into
`orders-service`'s own error handling, where — because the code is `error` for every infrastructure
fault (§3.13) — it cannot be retried intelligently.

### 4.3 Startup — what must be true before the first request

| # | Step | Where | Fails how |
| --- | --- | --- | --- |
| 1 | Host builder created; `appsettings.json` + `appsettings.{env}.json` loaded | `…Api/Program.cs:21` | a malformed JSON file fails startup loudly |
| 2 | `UseVault()` fetches `kv/pricing-service/settings` and merges it over configuration; requests a PKI certificate for role `pricing-service` | `…Api/Program.cs:33`; `appsettings.json:106-124` | Vault unreachable → `[convey]`, `Unverifiable — Missing Source Evidence`. **Off in `local` and `docker`** (§3.25) |
| 3 | `AddConvey()` binds `AppOptions`, `HttpClientOptions`, … | `…Api/Program.cs:23` | a missing `httpClient` section would leave `Services` empty — surfacing later, at step 7 of §4.1 |
| 4 | `AddInfrastructure()` registers the two application services and the Convey chain | `…Api/Infrastructure/Extensions.cs:22-38` | DI misconfiguration surfaces at first resolution, not here |
| 5 | `UseInfrastructure()` builds the pipeline: error handler → Swagger → Jaeger → Convey → metrics | `…Api/Infrastructure/Extensions.cs:42-46` | — |
| 6 | Dispatcher endpoints mapped | `…Api/Program.cs:29-31` | — |
| 7 | `UseLogging()` configures Serilog sinks | `…Api/Program.cs:32` | an unreachable Seq does not stop startup `[convey]` |
| 8 | Consul registration; `/ping` begins answering every 3s | `…Api/Infrastructure/Extensions.cs:32`; `appsettings.json:13-16` | **registers `docker.for.win.localhost` under the base profile** (§3.24) |
| 9 | First request resolves `CustomersServiceClient`, which reads `options.Services["customers"]` | `…Api/Services/Clients/CustomersServiceClient.cs:16` | **`KeyNotFoundException` on the first request, not at startup** (§3.9) — the instance is already in Consul rotation by then |

The load-bearing observation: **steps 8 and 9 are in the wrong order for safety.** The service
advertises itself as healthy before it has ever proven it can reach its only dependency.

### 4.4 The failure path — every way this service can fail, and what the caller sees

| Cause | Where detected | Exception | HTTP | Body `code` | Loud or silent? |
| --- | --- | --- | --- | --- | --- |
| Customer does not exist | `GetOrderPricingHandler.cs:29-32` | `CustomerNotFoundException` | 400 | `customer_not_found` | **loud** |
| `customerId` absent/unparseable → `Guid.Empty` | not detected | `CustomerNotFoundException` (downstream 404) | 400 | `customer_not_found` | **misleading** — reports a data problem for an input problem |
| Token lacks the id claim → `@user_id` empty | not detected | as above | 400 | `customer_not_found` | **misleading** (§3.33) |
| `customers-service` down / timeout / 5xx | Convey HTTP stack | `[convey]`-specific | 400 | `error` | **loud but uninformative** |
| Certificate rejected by `customers-service` (§3.34, reading 1) | Convey HTTP stack | as above | 400 | `error` | **loud but uninformative** |
| `CompletedOrders` null | `DTO/Extensions.cs:9` | `ArgumentNullException` | 400 | `error` | **loud but misattributed** — a server bug reported as a client error |
| `httpClient.services` missing `customers` | `CustomersServiceClient.cs:16` | `KeyNotFoundException` | 400 | `error` | **loud, late** (§4.3 step 9) |
| `OrderPrice` zero or negative | not detected | none | 200 | — | **silent** (§3.8) |
| Discount > 1 after a code edit | not detected | none | 200 | — | **silent** (§3.8) |
| `Customer.Id` always `Guid.Empty` | not detected | none | 200 | — | **silent** (§3.3) |
| Malformed `orderPrice` (not a decimal) | model binder | `[framework]` | `Unverifiable — Missing Source Evidence` | — | — |

**Four of the eleven fail silently or misleadingly.** All four are internal-arithmetic or
input-validation gaps, and all four would be closed by a validation block at the top of `HandleAsync`
plus assigning `Customer.Id`. That is the single most valuable change available in this repository —
see §7.1.

---

## 5. Persistence & schema evolution

### 5.1 There is no persistence

This section is short because the answer is "none", and it is worth stating precisely what "none"
means here:

| Store | Present | Evidence |
| --- | --- | --- |
| MongoDB | **no** | no `mongo` section in any of the four `appsettings*.json`; no `Convey.Persistence.MongoDB` reference (`…Api/Pacco.Services.Pricing.Api.csproj:10-22`) |
| Redis | **no** | no `redis` section; no `Convey.Persistence.Redis` reference — this service is absent from [[prefix-partitioned-shared-cache]] |
| Outbox / inbox collections | **no** | no `outbox` section; no `Convey.MessageBrokers.Outbox*` reference — [[transactional-outbox-handler-decorator]] is not instantiated |
| Relational | **no** | no provider reference of any kind |
| Local files | **only logs** | `logger.file.path: logs/logs.txt` (`appsettings.json:60`), enabled in the base profile only and **not** in `docker` (`appsettings.docker.json:37-41`) |

Consequences already stated in §3.17: free horizontal scaling, no cache, no audit trail, and exact
proportionality between pricing load and `customers-service` load.

### 5.2 The schemas that *do* evolve

Three contracts have versioning consequences even without a database:

| Contract | Owner | Consumers | What a change costs |
| --- | --- | --- | --- |
| `OrderPricingDto` (response) | this service (`…Api/DTO/OrderPricingDto.cs`) | `orders-service`'s own copy; any gateway caller | **additive** = safe; **rename** = silent zeroing in `orders-service` (§3.2) |
| `GetOrderPricing` (request) | this service (`…Api/Queries/GetOrderPricing.cs`) | the gateway's `downstream` template ×4; `orders-service`'s client | a new required parameter must be added to all four `ntrada*.yml` and to `PricingServiceClient` before it can be relied on |
| `CustomerDto` (upstream read) | **`customers-service`** | this service | a rename there silently defaults the field here (§3.4) |

None of the three is validated by a test, a schema registry, or a contract check. The platform's only
contract-testing mechanism — [[consumer-driven-contract-test-pair]] — is instantiated once, between
`orders` and `parcels`, and is disabled on both sides (`index.md` §4).

### 5.3 Configuration as schema

Because there is no database, **configuration is the closest thing this service has to a schema**, and
it evolves under the same hazards:

| Change | Silent failure risk | Mitigation that exists |
| --- | --- | --- |
| Add a key under `httpClient.services` | high — the `local` profile redefines the object and the `type: ""` there changes its meaning (§3.9, §3.26) | none |
| Change `consul.port` / `ASPNETCORE_URLS` | high — six locations must agree (§3.32) | none |
| Change `jaeger.serviceName` | medium — breaks saved traces (§3.20) | none |
| Add a Vault KV override | **invisible** — nothing logs effective configuration (§3.25) | none |
| Add a key only to `appsettings.json` | low for `docker`, **high** for `development` (which is `{}`) (§3.26) | none |

### 5.4 What a "migration" would look like

If this service ever acquires state — the most plausible trigger is auditability (§3.17) — the change
is not incremental. It requires: a `Convey.Persistence.MongoDB` reference, a `mongo` section in three
profiles, a `vault.lease.mongo` section to obtain credentials (which this service, uniquely among
stateful Pacco services, does not have), an `Infrastructure/Mongo` folder with document types and
`AsDocument`/`AsEntity` mappings, and a repository registration
([[database-per-service-with-document-mapping]]). `vehicles-service` is the closest working reference
for every one of those steps.

---

## 6. Surface → internals map

Every externally reachable entry point, mapped to the code that serves it.

### 6.1 HTTP routes

| Method | Path | Bound message | Handler | Response | Auth |
| --- | --- | --- | --- | --- | --- |
| `GET` | `/` | — (inline delegate) | `…Api/Program.cs:30` | `text/plain` app name from `appsettings.json:3` | none |
| `GET` | `/pricing` | `GetOrderPricing` (`…Api/Queries/GetOrderPricing.cs`) | `GetOrderPricingHandler` (`…Api/Queries/Handlers/GetOrderPricingHandler.cs`) | `OrderPricingDto` | none in-service; `auth: true` at the gateway (`ntrada.yml:407`) |
| `GET` | `/ping` | — | `UseConvey()` `[convey]` (`…Api/Infrastructure/Extensions.cs:45`) | framework | none |
| `GET` | `/metrics` | — | `UseMetrics()` `[convey]` (`…Api/Infrastructure/Extensions.cs:46`) | Prometheus exposition | none |
| `GET` | `/docs` | — | `UseSwaggerDocs()` (`…Api/Infrastructure/Extensions.cs:43`) | OpenAPI UI | none (§3.28) |

`/`, `/ping` and `/metrics` are excluded from logging and tracing (`appsettings.json:36,77`);
`/docs` is **not**.

### 6.2 Messages

**None.** No exchange, no queue, no subscription, no publish. This is the only Pacco service with an
empty row here — `service-summaries.md` §3 records zero `rabbitMq` occurrences in its
`appsettings.json`, and this model confirms the absence extends to the package references
(`…Api/Pacco.Services.Pricing.Api.csproj:10-22`) and to the code.

### 6.3 Outbound calls

| Target | Call | Client | Address source | Credential |
| --- | --- | --- | --- | --- |
| `customers-service` | `GET /customers/{id}` | `CustomersServiceClient` (`…Api/Services/Clients/CustomersServiceClient.cs:19-20`) | `httpClient.services.customers` (`appsettings.json:27`) | **none** (§3.34) |
| Consul | registration + `/ping` | `AddConsul()` `[convey]` | `consul.url` | none |
| Fabio | outbound routing | `AddFabio()` `[convey]` | `fabio.url` | none |
| Jaeger | UDP spans | `AddJaeger()` `[convey]` | `jaeger.udpHost:6831` | none |
| Seq | structured logs | `UseLogging()` `[convey]` | `logger.seq.url` | `logger.seq.apiKey` (§3.27) |
| Vault | KV read + PKI issue at startup | `UseVault()` `[convey]` | `vault.url` | `vault.token` (§3.25) |

### 6.4 Inbound callers

| Caller | Path | Identity binding |
| --- | --- | --- |
| End user via `api-gateway` | `GET /pricing` | `customerId` forced to `@user_id` (`ntrada.yml:406`) |
| `orders-service` | `GET /pricing?customerId=&orderPrice=` | **none** — caller-chosen id (§4.2) |
| Prometheus | `GET /metrics` | none |
| Consul | `GET /ping` | none |

---

## 7. Change/extension guide

Ordered by how likely a maintainer is to be asked for each.

### 7.1 Add input validation (the highest-value change)

**Why first:** four of the eleven failure modes in §4.4 are silent or misattributed, and all four are
closed here.

1. In `…Api/Queries/Handlers/GetOrderPricingHandler.cs`, at the top of `HandleAsync` — **before** the
   outbound call, so an invalid request does not cost a round-trip:
   - reject `query.CustomerId == Guid.Empty` with a new `AppException` subclass (e.g. code
     `invalid_customer_id`);
   - reject `query.OrderPrice <= 0` with another (e.g. `invalid_order_price`).
2. Add each subclass under `…Api/Exceptions/`, following `CustomerNotFoundException.cs:5-12`: derive
   from `AppException`, set `Code` with an **initialiser** (not a computed value — §3.12).
3. No change to `ExceptionToResponseMapper` is needed; the `AppException` arm already matches
   subclasses (`…Api/Exceptions/ExceptionToResponseMapper.cs:16-17`).
4. Once `OrderPrice > 0` is guaranteed, the `> 0` ternary at
   `…Api/Queries/Handlers/GetOrderPricingHandler.cs:45` becomes dead. **Remove it** rather than
   leaving it — it is the mechanism that hides an over-100% discount (§3.8).
5. Update `Pacco.Services.Pricing.rest`, whose current sample uses
   `customerId = 00000000-…` and `orderPrice = 0` (`Pacco.Services.Pricing.rest:2-3`) — **both of
   which the new guards reject**. This is the one place the change visibly breaks something.
6. Update §3.1, §3.8 and §4.4 of this document (§7.9).

### 7.2 Fix `Customer.Id`

One line: `Id = id;` in `…Api/Core/Entities/Customer.cs:11-15`. Nothing reads `Id` today, so the
change is behaviour-preserving *now* and prevents a whole class of future bug (§3.3). Do it before
any feature that needs per-customer logic, not after.

### 7.3 Change a discount rate or threshold

1. Edit the literals in `…Api/Core/Services/CustomerDiscountsService.cs:11-27`.
2. **Check band contiguity by hand** — the `else if` chain has no gap today, and nothing will tell you
   if your edit introduces one (§3.6).
3. **Consider the VIP interaction**: the effective top rate is `band + 0.1`, and the VIP threshold
   lives in `customers-service` (§3.36). A change to the top band changes the VIP total too.
4. **Consider adding a ceiling** — `Math.Min(discount, 0.2m)` — as part of any change that raises a
   rate; the current bound is emergent, not enforced (§3.7).
5. This is a **pricing change**. It ships as a container rebuild and a redeploy; there is no runtime
   toggle (§3.6).

### 7.4 Make the discount ladder configurable

1. Add an options class (e.g. `DiscountOptions`) with the bands as a list.
2. Bind it in `…Api/Infrastructure/Extensions.cs:22-38` via `builder.Services.Configure<…>` or
   Convey's `GetOptions<T>` `[convey]`.
3. Inject it into `CustomerDiscountsService` — which currently has no constructor at all
   (`…Api/Core/Services/CustomerDiscountsService.cs:5-7`).
4. Add the section to `appsettings.json` **only**; leave `local` and `docker` alone so they inherit
   (§3.26). Remember `appsettings.development.json` is `{}` and inherits everything.
5. Note the new capability this creates: Vault KV can now override pricing at runtime-of-startup
   (§3.25). Decide deliberately whether that is wanted.

### 7.5 Add a second outbound dependency

1. Interface + `internal sealed` client in `…Api/Services/Clients/`, mirroring
   `CustomersServiceClient.cs:8-21`.
2. `AddTransient<IFooServiceClient, FooServiceClient>()` in `…Api/Infrastructure/Extensions.cs:24-25`.
3. Add the key to `httpClient.services` in **`appsettings.json`, `appsettings.local.json` and
   `appsettings.docker.json`** — the local profile uses literal URLs with `type: ""`, so the value
   differs by profile (§3.10).
4. Prefer `TryGetValue` over the indexer in the new client, so a missing key fails at startup with a
   clear message rather than on the first request (§4.3 step 9).
5. Decide about the certificate: if the target enables certificate authentication, follow
   `availability-service`'s client shape (§3.25 extension procedure), not this repository's.

### 7.6 Attach the Vault PKI certificate to the `customers-service` call

A two-repository change; **both halves or neither** (§3.34).

1. **Here:** inject `ICertificatesService`, `VaultOptions` and `SecurityOptions` into
   `CustomersServiceClient`; guard on `vaultOptions.Enabled && vaultOptions.Pki?.Enabled == true`;
   `SetHeaders` with the raw certificate data under `securityOptions.Certificate.GetHeaderName()` —
   copying `hianshul100_Pacco.Services.Availability/…/Clients/CustomersServiceClient.cs:16-34`
   verbatim. Add a `security.certificate` section (this repository has none) so `SecurityOptions`
   binds meaningfully. `Convey.WebApi.Security` must be added to the `.csproj`.
2. **In `hianshul100_Pacco.Services.Customers`:** add a `pricing-service` entry to
   `security.certificate.acl` with `validIssuer` and the `customers:read` permission, alongside the
   existing `availability-service` entry (`…Customers.Api/appsettings.json:173-180`).
3. Verify in an environment with `vault.enabled: true` — **`appsettings.docker.json:76` disables
   Vault**, so the compose environment will not exercise this path (§3.25).

### 7.7 Add a route

1. Query class in `…Api/Queries/` with settable properties (§3.14).
2. Handler in `…Api/Queries/Handlers/`, `internal sealed`, implementing `IQueryHandler<,>` — it is
   discovered automatically by `AddQueryHandlers()` (`…Api/Infrastructure/Extensions.cs:29`).
3. One line in `…Api/Program.cs:29-31`.
4. Add it to `Pacco.Services.Pricing.rest`.
5. **Gateway exposure is a separate repository**: add a route to the `pricing` module in all four
   `ntrada*.yml` (`ntrada.yml:400-412` and its three siblings), or the route is reachable only
   in-cluster.

### 7.8 Add the first test

See §3.30. `CustomerDiscountsService` is dependency-free, so the first test project needs no mocks.
The table in §3.36 is directly executable as a parameterised test and is the recommended starting
point; add boundary cases at `0`, `1`, `3`, `4`, `9`, `10` and the VIP variants of each.

### 7.9 The maintenance contract

**Any change to this component's internals must update this document in the same change.** Concretely:

| If you change… | Update at least… |
| --- | --- |
| `CustomerDiscountsService` rates or thresholds | §3.6, §3.7, §3.36 (the composite table), §7.3 |
| `GetOrderPricing` or `OrderPricingDto` shape | §3.1, §3.2, §5.2, §6.1, and the gateway note in §3.33 |
| `GetOrderPricingHandler` control flow | §3.8, §3.11, §4.1, §4.4 |
| `Customer` or `AsEntity` | §3.3, §3.4, §3.5, §4.1 step 10 |
| `CustomersServiceClient` (address, headers, method) | §3.9, §3.10, §3.34, §6.3, §7.6 |
| `ExceptionToResponseMapper` or any `AppException` subclass | §3.12, §3.13, §4.4 |
| `…Api/Infrastructure/Extensions.cs` registrations or middleware order | §3.14, §3.16, §3.22, §3.24, §3.29 |
| Any `appsettings*.json` key | the corresponding §3 table, §5.3, and §3.26 if a profile is added |
| `Program.cs` endpoints or host builder | §3.14, §3.15, §6.1, §4.3 |
| Project structure, `.csproj`, `Dockerfile`, `scripts/`, `.travis.yml` | §3.18, §3.23, §3.30, §3.32 |
| Adding persistence or messaging | §1.1, §1.2, §3.17, §5.1, §5.4, §6.2 — and re-check §1.2's list of "explicitly nots" |
| The `pricing` module in any `ntrada*.yml` | §3.33, §4.1, §6.4 |
| `customers-service`'s `CustomerDetailsDto`, VIP threshold, or certificate ACL | §3.4, §3.34, §3.36 — **and the corresponding sections of `customers-service.md`** |

Adding a concept to §3 also means adding its row to the §2 table and, if the count changes, updating
this component's row in `component-internals/index.md` §1.

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

| # | Assumption | Basis | How to falsify |
| --- | --- | --- | --- |
| **A-1** | `IHttpClient.GetAsync<T>` returns `null` for a 404 rather than throwing | the null-check-then-throw shape at `…Api/Queries/Handlers/GetOrderPricingHandler.cs:29-32` only makes sense under this behaviour, and the same shape is used platform-wide | read `Convey.HTTP` 0.4.\* source, or call the service for a non-existent customer and observe whether `code` is `customer_not_found` or `error` |
| **A-2** | `httpClient.retries: 3` (`appsettings.json:25`) applies to `GetAsync<T>` | the key exists and Convey documents a retry policy | as above; observe request counts at `customers-service` for one failed pricing call |
| **A-3** | JSON configuration profiles merge key-by-key, so keys present only in `appsettings.json` survive under `local`/`docker` | standard ASP.NET Core `IConfiguration` layering `[framework]`; `appsettings.development.json` being `{}` and the service still working depends on it | set a base-only key and read it back under `ASPNETCORE_ENVIRONMENT=docker` |
| **A-4** | `AddQueryHandlers()` discovers handlers in the calling assembly only | single-project layout means there is no way to observe the difference here | add a handler in a second assembly and see whether it resolves |
| **A-5** | Vault KV values are merged over file configuration (KV wins) | `UseVault()` runs on the host builder at `…Api/Program.cs:33`, after the JSON providers | set a KV key that collides with `appsettings.json` and read the effective value |
| **A-6** | The `certs/localhost.cer` committed at `…Api/certs/` is a JWT issuer certificate, not a client certificate | it is referenced only by `jwt.certificate.location` (`appsettings.json:81`), a JWT key | inspect the certificate's extended key usage |
| **A-7** | `orders-service` and the gateway are the only callers of `GET /pricing` | grep across the workspace found `PricingServiceClient` in `orders-service` and the `pricing` module in the four `ntrada*.yml`, and no other reference | search a deployed environment's access logs |

### 8.2 Blockers

| # | Blocker | Impact | What would unblock it |
| --- | --- | --- | --- |
| **B-1** | `pricing-service` calls `customers-service` **without a client certificate**, while that service enables certificate authentication with an ACL naming only `availability-service` (§3.34) | If the middleware fails closed, pricing is **completely broken** in any environment with Vault and certificate auth enabled — and it surfaces to users as `customer_not_found`, not as an auth error | `Convey.WebApi.Security` 0.4.\* source, or an integration test against a Vault-enabled `customers-service`. This is the single highest-priority item in this model |
| **B-2** | `hianshul100_Pacco/prod-services.yml:50-55` sets no `ASPNETCORE_ENVIRONMENT`, so the **production** PM2 process runs the base profile — the one that registers `consul.address: docker.for.win.localhost` (§3.24) and enables Vault against `http://localhost:8200` | production instances register an address peers cannot resolve; the failure is invisible from the service's own health check | confirm whether a `prod` profile is injected elsewhere in the deployment pipeline (not present in this workspace), or add `appsettings.prod.json` and set the environment variable |
| **B-3** | No test project exists (§3.30); `scripts/test.sh` passes vacuously and CI reports green (`.travis.yml:14`) | every behaviour in §3.6–§3.8 is unverified, and the README badge implies otherwise | add the unit project described in §7.8 |
| **B-4** | `Customer.Id` is never assigned (§3.3) | latent: any future per-customer logic silently treats all customers as `Guid.Empty` | one-line fix (§7.2); no external dependency |
| **B-5** | `AsEntity` dereferences `CompletedOrders` without a null guard (§3.5) | a `customers-service` response without that field produces `ArgumentNullException` → generic 400, misattributed as a client error | `dto.CompletedOrders?.Count() ?? 0`; requires deciding whether "no data" should mean "no discount" or "reject" |

### 8.3 Open questions

| # | Question | Why it matters | Where the answer lives |
| --- | --- | --- | --- |
| **Q-1** | Does `Convey.WebApi.Security`'s certificate middleware reject uncertificated requests outright, or only gate routes that declare a permission? | determines whether **B-1** is an active outage or a latent inconsistency | `Convey.WebApi.Security` 0.4.\* source — not in this workspace |
| **Q-2** | With `passQueryString: true` (`ntrada.yml:13`) and a `downstream` template that also sets `customerId=@user_id` (`ntrada.yml:406`), which value wins if the caller supplies their own `customerId`? | the platform's only guarantee that a user cannot price another customer's order rests entirely on this (§3.33) | `Ntrada` source — not in this workspace `[ntrada]` |
| **Q-3** | Is the `> 0` fallback at `…Api/Queries/Handlers/GetOrderPricingHandler.cs:45` intentional, and if so, what business rule does it encode? | it currently makes negative prices *worse* and masks over-100% discounts (§3.8); §7.1 recommends deleting it | product owner; no evidence in the repository, `README.md`, or commit history |
| **Q-4** | Should the discount ladder and the VIP threshold have a single owner? | they are unrelated constants in two repositories that jointly determine one number (§3.36) | product owner + `customers-service` maintainers |
| **Q-5** | Is `pricing-service` intended to remain stateless, or is pricing auditability a requirement? | determines whether §5.4's migration is speculative or planned | product backlog; `service-summaries.md` does not say |
| **Q-6** | Why does `jaeger.serviceName` differ from the Consul/Prometheus name (§3.20)? | cross-tool correlation depends on knowing the mapping | platform convention — the same divergence appears in sibling services, so this is a platform question, not a local one |
| **Q-7** | Was `Convey.Tracing.Jaeger.RabbitMQ` (§3.23) left over from a template, or does it anticipate planned messaging? | decides whether to delete it or to expect a broker | commit history shows no messaging work; `HEAD` (`ed66901`, "Added security package") suggests scaffolding, not intent |
| **Q-8** | Should `OrderPricingDto` values be rounded before leaving the service (§3.35)? | `orders-service` stores them unrounded; a payment surface will eventually round them somewhere | product owner |

### 8.4 Cross-references, related patterns and baseline reconciliation

**Related patterns.** [[narrow-synchronous-point-read]] (the `customers-service` call is the canonical
instance — this service is *entirely* one), [[dispatcher-bound-cqrs-endpoints]],
[[registry-mediated-discovery-and-routing]], [[composable-per-concern-environment-stacks]],
[[vault-issued-dynamic-credentials-and-service-pki]] (partially instantiated — PKI issued, never
used), [[structured-logging-with-property-redaction]] (configured but bypassed by interpolation, §3.27),
[[correlation-and-span-propagation]], [[edge-enforced-authentication-with-identity-binding]],
[[framework-supplied-platform-conventions]], [[independent-per-repository-release]],
[[declarative-configuration-driven-api-gateway]] (from the gateway side).

**Patterns this service is a deliberate counter-example to.**
[[inward-dependency-service-skeleton]] (folders, not assemblies — §3.18),
[[database-per-service-with-document-mapping]] (no database — §3.17),
[[service-owned-topic-exchange-messaging]] and [[rejected-event-failure-contract]] (no broker — §6.2),
[[transactional-outbox-handler-decorator]] (nothing to make transactional),
[[transport-agnostic-caller-context]] (no context at all — §3.19),
[[dual-mode-edge-write]] (one transport — §1.3),
[[prefix-partitioned-shared-cache]] (no Redis — §3.17),
[[layered-service-test-suite]] and [[consumer-driven-contract-test-pair]] (no tests — §3.30),
[[aggregate-buffered-domain-events]] (no aggregate — §1.2).

**Other component models.**
`component-internals/customers-service.md` — §3.5 (the VIP threshold this service depends on), §3.25
(the certificate ACL behind **B-1**), §3.18 (the identical error-mapping convention).
`component-internals/api-gateway.md` — the upstream half of §3.33.
`component-internals/orders-service.md` — the second caller (§4.2) and the second declaration of
`OrderPricingDto` (§3.2).
`component-internals/vehicles-service.md` — the batch-6 sibling, and the nearest available reference
for every stack this service lacks (§5.4).

**Baseline reconciliation.** `baselines/service-summaries.md` §2.4 and §3 are consistent with
everything found here — in particular the zero-`rabbitMq` observation. This model **adds** rather than
corrects: the discount ladder (§3.6), the `Customer.Id` defect (§3.3), the `> 0` fallback (§3.8), the
certificate asymmetry (§3.34) and the cross-repository threshold coupling (§3.36) are not derivable
from the baselines. No baseline statement about `pricing-service` was found to be wrong.

**Explicitly unverifiable in this workspace.** Convey 0.4.\* internals (HTTP client null/retry
semantics — A-1, A-2; query binding; Vault failure behaviour; certificate middleware — Q-1); Ntrada
query-string precedence (Q-2); ASP.NET Core model-binder status codes for malformed decimals (§3.1).

---

*Component-internals model for `pricing-service`, batch 6 of 7. Registered in
`component-internals/index.md` §1. Any change to this component's internals must update this document
in the same change (§7.9).*
