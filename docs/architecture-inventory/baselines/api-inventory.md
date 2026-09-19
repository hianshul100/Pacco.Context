# Pacco — API, Tool & Event Contract Inventory (Discovery)

**Project Key:** Common Architecture
**Stage:** `architecture_discovery` — observed current-state contract surface.
**No ADRs, no recommendations, no target state, no KG JSON, no test inventory.**
**Date of analysis:** 2026-09-04
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`

This document is the **single authoritative contract reference** for the Pacco platform. It covers
three surfaces: HTTP and RPC endpoints (Part 1), MCP tool contracts (Part 2), and event and message
schemas carried over the broker (Part 3). It answers *what can be called, what goes in, what comes
back, who publishes it and who consumes it* — nothing else.

**Inputs used**

- The thirteen cloned source repositories (read-only), which are the **source of truth** for every
  statement below. Where a prior document and the code disagree, the code wins and the disagreement
  is stated explicitly in §10.
- `docs/architecture-inventory/baselines/capability-baseline.md` — capability identifiers `CAP-01`
  to `CAP-16`, used in the *Related capability* column of every table here. Capability definitions
  are **not** restated in this file.
- `docs/architecture-inventory/baselines/architecture-baseline.md` — messaging topology and
  runtime composition, cross-checked against source and consistent (§10 preamble).
- `docs/architecture-inventory/baselines/service-summaries.md`, `repo-inventory.md`,
  `architecture-views.md` — prior discovery documents, used for cross-checking only.
- `.attachments/01_product_backlog_20260903_170135_37cf143b.xlsx` — backlog issue **12998**
  "Pacco - Discovery - Attempt-2", which fixes the thirteen-repository scope.
- No externally held API catalogue, contract registry, or interface standard for Pacco was
  retrievable for this baseline. Every contract below is reconstructed from the cloned source.
  No platform-wide API standard (versioning policy, pagination policy, media-type policy) is
  asserted anywhere in this document because none is recorded in any repository (see Q4).

**Contract source ranking applied.** The mandated search order for high-confidence contract
sources was executed in full across all fourteen clone directories:

| Source type | Found | Evidence |
|-------------|-------|----------|
| OpenAPI / Swagger **document** (`openapi.*`, `swagger.json`, `*.yaml` spec) | **None** | No such file exists in any clone. Swagger is generated at runtime by `AddWebApiSwaggerDocs()`; no spec is committed |
| Postman / Insomnia collection | **None** | No `*.postman_collection.json` in any clone |
| `.http` / `.rest` request fixtures | **12 files** | `Pacco.APIGateway/Pacco.rest`, `Pacco.APIGateway/Pacco-sample-scenario.rest`, and one `Pacco.Services.<Name>.rest` per service repository |
| Protobuf IDL | **1 contract, 2 identical copies** | `Pacco.Services.Operations.Api/Operations.proto`, `Pacco.Services.Operations.GrpcClient/Operations.proto` |
| Declarative routing manifest | **4 files** | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` |
| Message-name catalogue | **1 file** | `Pacco.Services.Operations.Api/messages.json` — physical routing keys per exchange |
| MVC controllers (`*Controller.cs`) | **None** | No controller class exists in any repository; all HTTP routing is either Convey dispatcher endpoints or explicit `UseEndpoints` delegates |

**Reconstruction rule applied.** Every contract below was traced statically through the code —
endpoint registration → command/query type → handler → DTO → document mapper — before any field was
declared unknown. Payloads that are assembled deterministically in code (SignalR messages, saga
publications, error responses) are recorded as **observed**, not deferred to runtime capture. The
only surface where static reconstruction genuinely fails is `operations-service`'s inbound message
bodies, whose types are emitted at runtime with no fields (§9.5, Q3).

**Evidence taxonomy used throughout.** *Observed* — read directly from a source or config file,
path cited, with the field names copied verbatim from the declaring type. *Inferred* — a conclusion
drawn from two or more observed facts, labelled as such. *Assumption* — belief beyond what evidence
shows, labelled `[assumption]` and rolled into the ABQ section. *Unknown* — labelled `[unknown]`,
never filled with a guess.

**Confidence scale used in every table.**

- **High** — the contract is fully readable from committed source: route, verb, every request field
  with its declared type, every response field, and the status code path.
- **Medium** — the route and verb are observed, but at least one element (a status code, a header,
  a response body shape) comes from framework behaviour whose implementation is a NuGet package
  outside this workspace, or from a `.rest` fixture rather than the declaring type.
- **Low** — existence is observed but the contract is not statically determinable.

**Cross-reference convention.** `A#` / `B#` / `Q#` refer to the *Assumptions, Blockers & Open
Questions* tables at the end of **this** file and nowhere else. `GAP-#` refers to §11 of this file.
`CAP-##` refers to `capability-baseline.md` §1. Identifiers from `capability-baseline.md`'s own ABQ
tables are **not** referenced here; where the same underlying fact matters to contracts, it is
restated with its own `Q#` in this file.

**Alias bindings, made once here and used verbatim thereafter.** The deployable service names used
throughout this document are `api-gateway` (the Ntrada edge process, repository
`hianshul100_Pacco.APIGateway`), `identity-service`, `customers-service`, `availability-service`,
`vehicles-service`, `parcels-service`, `orders-service`, `pricing-service`, `deliveries-service`,
`ordermaker-service`, and `operations-service`. `operations-grpc-client`
(`Pacco.Services.Operations.GrpcClient`) is the second deployable of the `operations-service`
repository. `pacco-web` is the repository `hianshul100_Pacco.Web`, which contains no code (§10
note 3). These names are taken from the `app.service` setting in each service's `appsettings.json`
and from the Consul and Fabio registrations; they are the names used in every contract statement.

## Table of contents

1. [Surface census](#1--surface-census)

**Part 1 — HTTP / RPC API inventory**

2. [Edge surface — `api-gateway` routes, synchronous configuration](#2--edge-surface--api-gateway-routes-synchronous-configuration)
3. [Edge surface — `api-gateway` routes, asynchronous configuration](#3--edge-surface--api-gateway-routes-asynchronous-configuration)
4. [Service HTTP endpoints](#4--service-http-endpoints)
5. [Non-HTTP RPC and streaming surfaces](#5--non-http-rpc-and-streaming-surfaces)
6. [Inter-service HTTP calls](#6--inter-service-http-calls)
7. [Cross-cutting HTTP contract facts](#7--cross-cutting-http-contract-facts)

**Part 2 — MCP tool contract inventory**

8. [MCP tool contract inventory](#8--mcp-tool-contract-inventory)

**Part 3 — Event and message schema inventory**

9. [Event and message schema inventory](#9--event-and-message-schema-inventory)

**Findings**

10. [Conflicts between sources](#10--conflicts-between-sources)
11. [Gaps](#11--gaps)

- [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions) — mandatory final section

## 1 — Surface census

*Observed.* Counts below are of distinct contract entries catalogued in this document, not of code
constructs.

| Surface | Count | Where inventoried |
|---------|-------|-------------------|
| `api-gateway` upstream routes, synchronous configuration (`ntrada.yml`) | 41 (40 proxied + 1 root) | §2 |
| `api-gateway` upstream routes, asynchronous configuration (`ntrada-async.yml`) | 41 (20 broker-published + 20 proxied + 1 root) | §3 |
| Service HTTP endpoints across the ten services | 35 business endpoints + 2 root endpoints | §4 |
| gRPC methods | 2 (1 unary, 1 server-streaming) | §5.1 |
| SignalR hub methods and server-to-client messages | 1 hub, 1 client-callable method, 5 server-emitted messages | §5.2 |
| Inter-service HTTP calls | 7 | §6 |
| MCP tools | **0** | §8 |
| Broker exchanges | 9 declared (8 carry traffic) | §9.1 |
| Distinct messages catalogued (commands + events + rejected events) | 80 in `messages.json`, plus 3 code-declared types it omits | §9.3 |
| Domain-service consumer bindings (subscriber × message) | 44, plus `operations-service`'s 80 catalogue-driven bindings | §9.4 |

**Two gateway configurations exist and they are not environment variants of one system — they are
behaviourally different systems.** In `ntrada.yml` all 40 routes proxy over HTTP. In
`ntrada-async.yml` the 20 write routes stop proxying and instead publish a message to the broker,
returning `202 Accepted`; the 20 read routes still proxy. Both are inventoried separately (§2, §3)
rather than merged, because merging them would state a contract that neither file expresses. Which
one runs in production is not recorded anywhere in the workspace (B1).

*Observed.* Configuration selection, in the order the value is resolved by
`Pacco.APIGateway/src/Pacco.APIGateway/Program.cs` (`args?.FirstOrDefault() ?? ntradaConfig ?? "ntrada.yml"`):

| Source | Value | File |
|--------|-------|------|
| Compose stack, default | `NTRADA_CONFIG=ntrada-async.docker.yml` | `hianshul100_Pacco/compose/services.yml:9` |
| Compose stack, local variant | `NTRADA_CONFIG=ntrada.docker` | `hianshul100_Pacco/compose/services-local.yml:9` |
| Image default | `ENV NTRADA_CONFIG ntrada.docker` | `hianshul100_Pacco.APIGateway/Dockerfile:11` |
| Start script | `export NTRADA_CONFIG=ntrada-async` | `hianshul100_Pacco.APIGateway/scripts/start-async.sh:3` |

## 2 — Edge surface — `api-gateway` routes, synchronous configuration

**Evidence base for the whole section:**
`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, section `modules.<module>.routes`.
`ntrada.docker.yml` is the same file with `localUrl` replaced by container hostnames; route shapes
are identical, so it is not tabulated separately.

**How to read the upstream path.** Ntrada composes the full upstream path as
`/<module.path>/<route.upstream>`. Every path in the first column below is the composed value —
what a client actually calls. The gateway listens on `http://localhost:5000` (*observed*,
`Pacco.APIGateway/Pacco.rest:1`, `@url = http://localhost:5000`).

**Auth column semantics.** `Bearer` — `auth: true` on the route, or inherited from
`auth.enabled: true` with no route-level override; the token is a JWT validated against
`validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`
(*observed*, `ntrada.yml` `extensions.jwt`). `Bearer + role admin` — the route additionally carries
`claims: role: admin`, checked against claim type
`http://schemas.microsoft.com/ws/2008/06/identity/claims/role` (*observed*, `ntrada.yml`
`auth.claims.role`). `none` — the route declares `auth: false`. No route in either configuration
uses a session cookie or an API key; **no cookie-based or API-key authentication exists anywhere in
the platform** (*observed*, absence across all fourteen clones).

**Status-code caveat that applies to every `200, empty body` cell in §2 and §4.** For a command
endpoint registered without an explicit `afterDispatch`, the response status is produced by
`Convey.WebApi.CQRS 0.4.*`, a NuGet package that is not vendored into this workspace. The route,
verb, and request contract of those rows are *observed* at High confidence; the status code itself
is **framework-determined and Medium confidence** and is the single reason those rows are not
end-to-end High. Rows with an explicit `afterDispatch` — `Created(...)` and `NoContent()` — are
fully observed. See §7.3 and Q1.

**Binding notation.** `bind: customerId: @user_id` injects the authenticated subject id into the
downstream request body, so that field is **not** supplied by the caller even though the downstream
service requires it. Every such injection is called out in the Request column.

### 2.1 — `availability` module → `availability-service`

Downstream `localUrl: localhost:5001`, service name `availability-service`. Related capability
**CAP-04**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* / *inferred* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /availability/resources` | `availability-service/resources` | List resources filtered by tag | Query `tags` (JSON array literal, e.g. `tags=["vehicle","armor"]`), `matchAllTags` (bool). *Observed*, `Pacco.rest` | `200` + `ResourceDto[]` — *inferred* from `GetResources` return type; status code is Convey framework behaviour | Bearer | High |
| `GET /availability/resources/{resourceId}` | `availability-service/resources/{resourceId}` | Read one resource with its reservations | Path `resourceId` (Guid) | `200` + `ResourceDto`; `404` when the handler returns null | Bearer | High |
| `POST /availability/resources` | `availability-service/resources` | Register a reservable resource | Body `{ resourceId: Guid?, tags: string[] }` | `201` + `Location: resources/{resourceId}`, `Resource-ID` header | Bearer | High |
| `POST /availability/resources/{resourceId}/reservations/{dateTime}` | same path | Reserve a resource for a date | Path `resourceId`, `dateTime`; body `{ priority: int }`; **`customerId` injected** from `@user_id`; `resourceId` and `dateTime` re-bound from the path | `200`, empty body | Bearer | High |
| `DELETE /availability/resources/{resourceId}/reservations/{dateTime}` | same path | Release a reservation | Path `resourceId`, `dateTime`, both re-bound into the body | `200`, empty body | Bearer | High |
| `DELETE /availability/resources/{resourceId}` | same path | Remove a resource | Path `resourceId`, re-bound into the body | `200`, empty body | Bearer | High |

`ResourceDto` = `{ id: Guid, tags: string[], reservations: [{ dateTime: DateTime, priority: int }] }`
(*observed*, `Pacco.Services.Availability.Application/DTO/ResourceDto.cs`,
`ReservationDto.cs`).

### 2.2 — `customers` module → `customers-service`

Downstream `localUrl: localhost:5002`. Related capability **CAP-03**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* / *inferred* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /customers` | `customers-service/customers` | List all customers | none | `200` + `CustomerDto[]` | Bearer + role `admin` | High |
| `GET /customers/me` | `customers-service/customers/@user_id` | Read the calling customer | none — the path segment **is** the injected `@user_id` | `200` + `CustomerDetailsDto` | Bearer | High |
| `GET /customers/{customerId}` | `customers-service/customers/{customerId}` | Read any customer | Path `customerId` (Guid) | `200` + `CustomerDetailsDto`; `404` when null | Bearer + role `admin` | High |
| `GET /customers/{customerId}/state` | `.../customers/{customerId}/state` | Read a customer's state flag | Path `customerId` | `200` + `CustomerStateDto` = `{ id: Guid, state: string }` | Bearer + role `admin` | High |
| `POST /customers` | `customers-service/customers` | Complete registration for the signed-up user | Body `{ fullName: string, address: string }`; **`customerId` injected** from `@user_id`. Route declares `payload: create_customer` and `schema: create_customer.schema` — **neither file exists in the repository** (§10 note 2, GAP-2) | `201` + `Location: customers/{customerId}` | Bearer | Medium — request validation contract unresolvable |
| `PUT /customers/{customerId}/state/{state}` | `.../customers/{customerId}/state/{state}` | Change a customer's state | Path `customerId`, `state` (string), both re-bound into the body | `204` (`afterDispatch: NoContent()`) | Bearer + role `admin` | High |

`CustomerDto` = `{ id: Guid, state: string, createdAt: DateTime }`; `CustomerDetailsDto` extends it
with `{ email: string, fullName: string, address: string, isVip: bool, completedOrders: Guid[] }`
(*observed*, `Pacco.Services.Customers.Application/DTO/`).

### 2.3 — `deliveries` module → `deliveries-service`

Downstream `localUrl: localhost:5003`. Related capability **CAP-09**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* / *inferred* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /deliveries/{deliveryId}` | `deliveries-service/deliveries/{deliveryId}` | Read a delivery and its registration trail | Path `deliveryId` (Guid) | `200` + `DeliveryDto`; `404` when null | Bearer | High |
| `POST /deliveries` | `deliveries-service/deliveries` | Start a delivery for an order | Body `{ orderId: Guid, description: string, dateTime: DateTime }`; route sets `resourceId: { property: deliveryId, generate: true }`, so the gateway **generates** the `deliveryId` and returns it | `201` + `Resource-ID` header carrying the generated `deliveryId` (*observed*, `Pacco-sample-scenario.rest` reads `{{start_delivery.response.headers.Resource-ID}}`) | Bearer | High |
| `POST /deliveries/{deliveryId}/fail` | `.../fail` | Mark a delivery failed | Path `deliveryId` re-bound; body `{ reason: string }` | `200`, empty body | Bearer | High |
| `POST /deliveries/{deliveryId}/complete` | `.../complete` | Mark a delivery complete | Path `deliveryId` re-bound | `200`, empty body | Bearer | High |
| `POST /deliveries/{deliveryId}/registrations` | `.../registrations` | Append a tracking entry | Path `deliveryId` re-bound; body `{ description: string, dateTime: DateTime }` | `200`, empty body | Bearer | High |

`DeliveryDto` = `{ id: Guid, orderId: Guid, status: DeliveryStatus, notes: string, lastUpdate: DateTime?, registrations: [{ description: string, dateTime: DateTime }] }`.
`lastUpdate` is **computed**, not stored: it is the maximum `DateTime` across `registrations`
(*observed*, `Pacco.Services.Deliveries.Application/DTO/DeliveryDto.cs`).

### 2.4 — `identity` module → `identity-service`

Downstream `localUrl: localhost:5004`. Related capability **CAP-01**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `POST /identity/sign-up` | `identity-service/sign-up` | Create an account | Body `{ email: string, password: string, role: string, permissions: string[] }`, `userId` optional; route sets `resourceId: { property: userId, generate: true }` | `201` + `Location: identity/me` (*observed*, `Identity.Api/Program.cs` `Response.Created("identity/me")`) | none (`auth: false`) | High |
| `POST /identity/sign-in` | `identity-service/sign-in` | Exchange credentials for tokens | Body `{ email: string, password: string }`; route pins `responseHeaders: content-type: application/json` | `200` + `AuthDto` = `{ accessToken: string, refreshToken: string, role: string, expires: long }` (*observed*, `Identity.Application/DTO/AuthDto.cs`; confirmed on the wire by `Pacco-sample-scenario.rest` reading `sign_in.response.body.$.accessToken`) | none (`auth: false`) | High |
| `GET /identity/me` | `identity-service/me` | Read the calling user | none; the service re-parses the bearer token itself | `200` + `UserDto`; `401` when the token yields `Guid.Empty` (*observed*, `Identity.Api/Program.cs`) | Bearer | High |
| `GET /identity/users/{userId}` | `identity-service/users/{userId}` | Read any user | Path `userId` (Guid) | `200` + `UserDto`; `404` when null | Bearer + role `admin` | High |

`UserDto` = `{ id: Guid, email: string, role: string, createdAt: DateTime, permissions: string[] }`
(*observed*, `Pacco.Services.Identity.Application/DTO/UserDto.cs`).

**Three token-lifecycle endpoints exist on `identity-service` but have no gateway route in either
configuration** — `POST /access-tokens/revoke`, `POST /refresh-tokens/use`,
`POST /refresh-tokens/revoke`. They are inventoried in §4.4 and the exposure gap is recorded in
§10 note 6 and GAP-3.

### 2.5 — `operations` module → `operations-service`

Downstream `localUrl: localhost:5005`. Related capability **CAP-11**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /operations/{operationId}` | `operations-service/operations/{operationId}` | Poll the status of a previously accepted write | Path `operationId` (string — the correlation id, **not** a Guid-typed parameter) | `200` + `OperationDto` = `{ id: string, userId: string, name: string, state: OperationState, code: string, reason: string }`; `404` when the cache entry is absent or expired | **none — `auth: false`** | High |

`OperationState` is the enum `{ Pending, Completed, Rejected }` (*observed*,
`Pacco.Services.Operations.Core/Types/OperationState.cs`). The record lives in Redis under key
`requests:{id}` with a sliding expiry of `requests.expirySeconds: 300` (*observed*,
`Operations.Api/Services/OperationsService.cs`, `Operations.Api/appsettings.json`). This route being
unauthenticated while the record carries a `userId` is recorded as a conflict in §10 note 7.

### 2.6 — `orders` module → `orders-service`

Downstream `localUrl: localhost:5006`. Related capability **CAP-07**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* / *inferred* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /orders` | `orders-service/orders?customerId=@user_id` | List the caller's orders | none — `customerId` is **appended by the gateway** from `@user_id` | `200` + `OrderDto[]` | Bearer | High |
| `GET /orders/{orderId}` | `orders-service/orders/{orderId}` | Read one order | Path `orderId` (Guid) | `200` + `OrderDto`; `404` when null | Bearer | High |
| `POST /orders` | `orders-service/orders` | Create an order | Body `{}` through the gateway — `customerId` **injected** from `@user_id`; `resourceId: { property: orderId, generate: true }` | `201` + `Resource-ID` header carrying the generated `orderId` (*observed*, `Pacco-sample-scenario.rest`) | Bearer | High |
| `DELETE /orders/{orderId}` | `orders-service/orders/{orderId}` | Delete an order | Path `orderId` | `200`, empty body | Bearer | High |
| `POST /orders/{orderId}/parcels/{parcelId}` | same path | Attach a parcel to an order | Path `orderId`, `parcelId`, both re-bound; **`customerId` injected** from `@user_id` | `200`, empty body | Bearer | High |
| `DELETE /orders/{orderId}/parcels/{parcelId}` | same path | Detach a parcel | Path `orderId`, `parcelId` | `200`, empty body | Bearer | High |
| `POST /orders/{orderId}/vehicles/{vehicleId}` | same path | Assign a vehicle and price the order | Path `orderId`, `vehicleId`, both re-bound; body `{ deliveryDate: DateTime }`; **`customerId` injected** | `200`, empty body | Bearer | High |

`OrderDto` = `{ id: Guid, customerId: Guid, vehicleId: Guid?, status: string (lower-cased), createdAt: DateTime, deliveryDate: DateTime?, totalPrice: decimal, parcels: [{ id: Guid, name: string, variant: string, size: string }] }`
(*observed*, `Pacco.Services.Orders.Application/DTO/OrderDto.cs`).

### 2.7 — `parcels` module → `parcels-service`

Downstream `localUrl: localhost:5007`. Related capability **CAP-06**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* / *inferred* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /parcels` | `parcels-service/parcels?customerId=@user_id` | List the caller's parcels | Query `includeAddedToOrders` (bool); `customerId` **appended by the gateway** | `200` + `ParcelDto[]` | Bearer | High |
| `GET /parcels/{parcelId}` | `parcels-service/parcels/{parcelId}` | Read one parcel | Path `parcelId` (Guid) | `200` + `ParcelDto`; `404` when null | Bearer | High |
| `GET /parcels/volume` | `parcels-service/parcels/volume` | Total the volume of a parcel set | Query `parcelIds` as a JSON array literal — `parcelIds=["<guid>"]` (*observed*, `Pacco.Services.Parcels.rest`) | `200` + `{ volume: double }` | Bearer | High |
| `POST /parcels` | `parcels-service/parcels` | Register a parcel | Body `{ variant: string, size: string, name: string, description: string }`; **`customerId` injected**; `resourceId: { property: parcelId, generate: true }` | `201` + `Resource-ID` header carrying the generated `parcelId` | Bearer | High |
| `DELETE /parcels/{parcelId}` | `parcels-service/parcels/{parcelId}` | Delete a parcel | Path `parcelId` | `200`, empty body | Bearer | High |

`ParcelDto` = `{ id: Guid, customerId: Guid, variant: string (lower-cased), size: string (lower-cased), name: string, description: string, createdAt: DateTime, orderId: Guid? }`.

**Route ordering caution.** `GET /parcels/volume` is declared *after* `GET /parcels/{parcelId}` in
`ntrada.yml`. Whether `volume` is captured by the `{parcelId}` template depends on Ntrada's matcher,
whose implementation is a NuGet package outside this workspace — see Q1.

### 2.8 — `pricing` module → `pricing-service`

Downstream `localUrl: localhost:5008`. Related capability **CAP-08**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /pricing` | `pricing-service/pricing?customerId=@user_id` | Quote a discounted price for an order value | Query `orderPrice` (decimal); `customerId` **appended by the gateway** (*observed*, `Pacco.rest` sends only `?orderPrice=100`) | `200` + `OrderPricingDto` = `{ orderPrice: decimal, customerDiscount: decimal, orderDiscountPrice: decimal }` | Bearer | High |

### 2.9 — `vehicles` module → `vehicles-service`

Downstream `localUrl: localhost:5009`. Related capability **CAP-05**.

| Method & upstream path | Downstream | Purpose | Request — *observed* | Response — *observed* / *inferred* | Auth | Conf. |
|---|---|---|---|---|---|---|
| `GET /vehicles` | `vehicles-service/vehicles` | Search the fleet | Query `payloadCapacity` (double), `loadingCapacity` (double), `variants` (flags enum as int, e.g. `variants=1`), plus Convey paging parameters | `200` + **`response.data.items`** — the route rewrites the envelope with `onSuccess: { data: response.data.items }`, so the caller receives a bare `VehicleDto[]` and **loses the paging metadata** the service returns | Bearer | High |
| `GET /vehicles/{vehicleId}` | `vehicles-service/vehicles/{vehicleId}` | Read one vehicle | Path `vehicleId` (Guid) | `200` + `VehicleDto`; `404` when null | Bearer | High |
| `POST /vehicles` | `vehicles-service/vehicles` | Add a vehicle | Body `{ brand, model, description: string, payloadCapacity, loadingCapacity: double, pricePerService: decimal, variants: Variants }`; `resourceId: { property: vehicleId, generate: true }` | `201` + `Resource-ID` header carrying the generated `vehicleId` | Bearer | High |
| `PUT /vehicles/{vehicleId}` | `vehicles-service/vehicles/{vehicleId}` | Update a vehicle | Path `vehicleId`; body `{ description: string, pricePerService: decimal, variants: Variants }` | `200`, empty body | Bearer | High |
| `DELETE /vehicles/{vehicleId}` | `vehicles-service/vehicles/{vehicleId}` | Remove a vehicle | Path `vehicleId` | `200`, empty body | Bearer | High |

`VehicleDto` = `{ id: Guid, brand: string, model: string, description: string, payloadCapacity: double, loadingCapacity: double, pricePerService: decimal, variants: string[] }`.
**No gateway route exposes `POST`, `PUT` or `DELETE` on `vehicles` behind a role claim** — any
authenticated caller can mutate the fleet catalogue (§10 note 8).

### 2.10 — Gateway-local route

| Method & upstream path | Purpose | Request | Response | Auth | Cap | Conf. |
|---|---|---|---|---|---|---|
| `GET /` | Liveness / banner | none | `200` + the literal string `Welcome to Pacco API!` (*observed*, `ntrada.yml` `modules.home`) | inherited (`auth.global: false`, no route-level `auth: true`) → none | CAP-02 | High |

## 3 — Edge surface — `api-gateway` routes, asynchronous configuration

**Evidence base for the whole section:**
`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada-async.yml`, section `modules.<module>.routes`.

In this configuration **20 routes change from `use: downstream` to `use: rabbitmq`**. Instead of
proxying an HTTP call, the gateway publishes a message to a topic exchange with a fixed routing key
and returns immediately. The remaining 20 routes and the `home` route are byte-for-byte identical to
§2 and are not repeated here — **every route not listed below behaves exactly as documented in §2**.

**Response contract for every broker-published route.** The caller does not receive a domain
response. It receives an acknowledgement carrying the correlation id, and must then poll
`GET /operations/{operationId}` (§2.5) to learn the outcome. The acknowledgement status code and
body shape are produced by `Ntrada.Extensions.RabbitMq 0.4.*`, whose implementation is a NuGet
package outside this workspace; the correlation-id-plus-polling pattern is confirmed by
`operations-service` reading `IMessageProperties.CorrelationId` on every inbound message
(*observed*, `Operations.Api/Handlers/`). The exact acknowledgement payload is therefore **not
statically determinable** and is recorded as GAP-1 / Q2, not guessed.

**Broker connection contract** (*observed*, `ntrada-async.yml` `extensions.rabbitmq`):
`connectionName: api-gateway`, `port: 5672`, `exchange.declareExchange: true`, `durable: true`,
`autoDelete: false`, `type: topic`, `messageContext.enabled: true` with header `message_context`,
`spanContextHeader: span_context`.

**Reading the table.** *Physical exchange* and *Physical routing key* are copied character-for-
character from the route's `config:` block. Neither is a logical name; the logical command type each
key maps to is given in the last column and is resolved through
`Operations.Api/messages.json` plus the snake-case naming convention
(`rabbitMq.conventionsCasing: snakeCase`).

| Method & upstream path | Physical exchange | Physical routing key | Request — *observed* | Logical command type | Cap | Conf. |
|---|---|---|---|---|---|---|
| `POST /availability/resources` | `availability` | `add_resource` | Body `{ resourceId: Guid?, tags: string[] }` | `AddResource` | CAP-04 | High |
| `POST /availability/resources/{resourceId}/reservations/{dateTime}` | `availability` | `reserve_resource` | Body `{ priority: int }`; bind `resourceId`, `dateTime` from path, `customerId` from `@user_id` | `ReserveResource` | CAP-04 | High |
| `DELETE /availability/resources/{resourceId}/reservations/{dateTime}` | `availability` | `release_resource` | bind `resourceId`, `dateTime` | `ReleaseResourceReservation` — **the routing key `release_resource` does not match the command type name**; see §10 note 9 | CAP-04 | High |
| `DELETE /availability/resources/{resourceId}` | `availability` | `delete_resource` | bind `resourceId` | `DeleteResource` | CAP-04 | High |
| `POST /customers` | `customers` | `complete_customer_registration` | Body `{ fullName, address: string }`; bind `customerId` from `@user_id`; declares `payload: complete_customer_registration` and `schema: complete_customer_registration.schema` — **neither file exists** (§10 note 2) | `CompleteCustomerRegistration` | CAP-03 | Medium |
| `PUT /customers/{customerId}/state/{state}` | `customers` | `change_customer_state` | bind `customerId`, `state`; `claims: role: admin` | `ChangeCustomerState` | CAP-03 | High |
| `POST /deliveries` | `deliveries` | `start_delivery` | Body `{ orderId: Guid, description: string, dateTime: DateTime }`; `resourceId: { property: deliveryId, generate: true }` | `StartDelivery` | CAP-09 | High |
| `POST /deliveries/{deliveryId}/fail` | `deliveries` | `fail_delivery` | Body `{ reason: string }`; bind `deliveryId` | `FailDelivery` | CAP-09 | High |
| `POST /deliveries/{deliveryId}/complete` | `deliveries` | `complete_delivery` | bind `deliveryId` | `CompleteDelivery` | CAP-09 | High |
| `POST /deliveries/{deliveryId}/registrations` | `deliveries` | `add_delivery_registration` | Body `{ description: string, dateTime: DateTime }`; bind `deliveryId` | `AddDeliveryRegistration` | CAP-09 | High |
| `POST /orders` | `orders` | `create_order` | Body `{}`; `resourceId: { property: orderId, generate: true }`; bind `customerId` from `@user_id` | `CreateOrder` | CAP-07 | High |
| `DELETE /orders/{orderId}` | `orders` | `delete_order` | bind `orderId`, `customerId` from `@user_id` | `DeleteOrder` | CAP-07 | High |
| `POST /orders/{orderId}/parcels/{parcelId}` | `orders` | `add_parcel_to_order` | bind `orderId`, `parcelId`, `customerId` | `AddParcelToOrder` | CAP-07 | High |
| `DELETE /orders/{orderId}/parcels/{parcelId}` | `orders` | `delete_parcel_from_order` | bind `orderId`, `parcelId`, `customerId` | `DeleteParcelFromOrder` | CAP-07 | High |
| `POST /orders/{orderId}/vehicles/{vehicleId}` | `orders` | `assign_vehicle_to_order` | Body `{ deliveryDate: DateTime }`; bind `orderId`, `vehicleId`, `customerId` | `AssignVehicleToOrder` | CAP-07 | High |
| `POST /parcels` | `parcels` | `add_parcel` | Body `{ variant, size, name, description: string }`; `resourceId: { property: parcelId, generate: true }`; bind `customerId` from `@user_id` | `AddParcel` | CAP-06 | High |
| `DELETE /parcels/{parcelId}` | `parcels` | `delete_parcel` | bind `parcelId`, `customerId` from `@user_id` | `DeleteParcel` | CAP-06 | High |
| `POST /vehicles` | `vehicles` | `add_vehicle` | Body `{ brand, model, description, payloadCapacity, loadingCapacity, pricePerService, variants }`; `resourceId: { property: vehicleId, generate: true }` | `AddVehicle` | CAP-05 | High |
| `PUT /vehicles/{vehicleId}` | `vehicles` | `update_vehicle` | Body `{ description, pricePerService, variants }`; bind `vehicleId` | `UpdateVehicle` | CAP-05 | High |
| `DELETE /vehicles/{vehicleId}` | `vehicles` | `delete_vehicle` | bind `vehicleId` | `DeleteVehicle` | CAP-05 | High |

**Routes that stay `use: downstream` in the asynchronous configuration:** every `GET`, plus all four
`identity` routes, the single `operations` route, and the single `pricing` route. `identity-service`
therefore keeps a synchronous sign-in path in both configurations — necessary, since a caller with
no token cannot poll an operation.

**`ordermaker-service` has no gateway route in either configuration.** Its `POST /orders` endpoint
(§4.9) is reachable only by calling the service directly or by publishing to the `ordermaker`
exchange, and `messages.json` declares no inbound command for it (B2).

## 4 — Service HTTP endpoints

These are the endpoints each service exposes on its own port. They are what the gateway proxies to
in §2, and they are **also directly reachable** — every service repository ships a `.rest` fixture
that calls its own port with no gateway in front (*observed*, e.g.
`hianshul100_Pacco.Services.Orders/Pacco.Services.Orders.rest`). Nothing in any service enforces
that a request arrived through the gateway, so the direct surface is the real trust boundary (Q5).

**Registration mechanism.** Nine of the ten services register endpoints through Convey's
`UseDispatcherEndpoints`, which binds an HTTP route straight to a command or query type — there is
**no controller class anywhere in the platform**. `identity-service` is the exception: it uses raw
`UseEndpoints` with explicit delegates, which is why its status codes are directly observable rather
than framework-determined.

**Request-shape reading rule used for every row.** For a `Get<TQuery, TResult>` registration, the
request contract is the public settable properties of `TQuery` (bound from route values and query
string) and the response contract is `TResult`. For a `Post<TCommand>` / `Put<TCommand>` /
`Delete<TCommand>` registration, the request contract is the public properties of `TCommand`, filled
from the route template first and the JSON body second. Field names below are copied verbatim from
the declaring C# type; on the wire they are camel-cased by the default JSON serialiser.

**Ports** (*observed*, each service's `Dockerfile` / `appsettings.json` / `.rest` fixture):
`availability-service` 5001, `customers-service` 5002, `deliveries-service` 5003,
`identity-service` 5004, `operations-service` 5005, `orders-service` 5006, `parcels-service` 5007,
`pricing-service` 5008, `vehicles-service` 5009, `ordermaker-service` 5015. `api-gateway` 5000.
gRPC 50050.

### 4.1 — `availability-service` (CAP-04)

Evidence: `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Error contract | Conf. |
|---|---|---|---|---|---|---|
| `GET /` | Service banner | none | `200` + `AppOptions.Name` = `Pacco Availability Service` | none | n/a | High |
| `GET /resources` | Tag-filtered resource list | `GetResources { Tags: string[], MatchAllTags: bool }` | `IEnumerable<ResourceDto>` | none at the service | *observed* §7.4 | High |
| `GET /resources/{resourceId}` | Read one resource | `GetResource { ResourceId: Guid }` | `ResourceDto`, `404` when null | none at the service | *observed* §7.4 | High |
| `POST /resources` | Register a resource | `AddResource { ResourceId: Guid, Tags: string[] }` — an empty `ResourceId` is replaced with `Guid.NewGuid()` and a null `Tags` with an empty collection, both in the constructor | `201` + `Location: resources/{ResourceId}` | none at the service | *observed* §7.4 | High |
| `POST /resources/{resourceId}/reservations/{dateTime}` | Reserve | `ReserveResource { ResourceId: Guid, DateTime: DateTime, Priority: int, CustomerId: Guid }` | `200`, empty body | none at the service | *observed* §7.4 | High |
| `DELETE /resources/{resourceId}/reservations/{dateTime}` | Release a reservation | `ReleaseResourceReservation { ResourceId: Guid, DateTime: DateTime }` | `200`, empty body | none at the service | *observed* §7.4 | High |
| `DELETE /resources/{resourceId}` | Delete a resource | `DeleteResource { ResourceId: Guid }` | `200`, empty body | none at the service | *observed* §7.4 | High |

### 4.2 — `customers-service` (CAP-03)

Evidence: `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /customers` | List customers | `GetCustomers {}` | `IEnumerable<CustomerDto>` | none at the service | High |
| `GET /customers/{customerId}` | Read a customer | `GetCustomer { CustomerId: Guid }` | `CustomerDetailsDto`, `404` when null | none at the service | High |
| `GET /customers/{customerId}/state` | Read a customer's state | `GetCustomerState { CustomerId: Guid }` | `CustomerStateDto { Id: Guid, State: string }` | certificate header, see below | High |
| `POST /customers` | Complete registration | `CompleteCustomerRegistration { CustomerId: Guid, FullName: string, Address: string }` | `201` + `Location: customers/{CustomerId}` | none at the service | High |
| `PUT /customers/{customerId}/state/{state}` | Change state | `ChangeCustomerState { CustomerId: Guid, State: string }` | `204` (`afterDispatch: NoContent()`) | none at the service | High |

**Certificate authentication — the exact declared contract.** *Observed*,
`Customers.Api/appsettings.json` `security.certificate`:

```
enabled: true          header: "Certificate"        skipRevocationCheck: false
allowedDomains: ["pacco.io"]   allowSubdomains: true   allowedHosts: ["localhost"]
acl:
  availability-service: { validIssuer: "localhost", permissions: ["customers:read"] }
```

`customers-service` calls `.AddCertificateAuthentication()` and `.UseCertificateAuthentication()`
(*observed*, `Customers.Infrastructure/Extensions.cs:79,91`). The ACL is declared **service-wide
against the caller identity `availability-service` with permission `customers:read`** — it is *not*
scoped to a single route in configuration, so it applies to whatever the package maps
`customers:read` onto. The matching caller side is *observed* in
`Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs:26–34`: it fetches the
Vault PKI certificate for role `customers-service` and sets the header named by
`securityOptions.Certificate.GetHeaderName()` (`Certificate` in `Availability.Api/appsettings.json`)
to `certificate.GetRawCertDataString()`. Whether the ACL is *enforced*, and which routes
`customers:read` covers, depends on `Convey.WebApi.Security` package code outside this workspace —
the declaration is *observed*, the enforcement is **[unknown]** (Q6).

### 4.3 — `deliveries-service` (CAP-09)

Evidence: `hianshul100_Pacco.Services.Deliveries/src/Pacco.Services.Deliveries.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /deliveries/{deliveryId}` | Read a delivery | `GetDelivery { DeliveryId: Guid }` | `DeliveryDto`, `404` when null | none at the service | High |
| `POST /deliveries` | Start a delivery | `StartDelivery { DeliveryId: Guid, OrderId: Guid, Description: string, DateTime: DateTime }` — empty `DeliveryId` replaced with `Guid.NewGuid()` | `201` + `Location: deliveries/{DeliveryId}` | none at the service | High |
| `POST /deliveries/{deliveryId}/fail` | Fail a delivery | `FailDelivery { DeliveryId: Guid, Reason: string }` | `200`, empty body | none at the service | High |
| `POST /deliveries/{deliveryId}/complete` | Complete a delivery | `CompleteDelivery { DeliveryId: Guid }` | `200`, empty body | none at the service | High |
| `POST /deliveries/{deliveryId}/registrations` | Append a tracking entry | `AddDeliveryRegistration { DeliveryId: Guid, Description: string, DateTime: DateTime }` | `200`, empty body | none at the service | High |

### 4.4 — `identity-service` (CAP-01)

Evidence: `hianshul100_Pacco.Services.Identity/src/Pacco.Services.Identity.Api/Program.cs` — explicit
`UseEndpoints` delegates, so every status code below is *observed* in this repository rather than
framework-determined.

| Method & path | Purpose | Request — *observed* | Response — *observed* | Auth | Gateway-exposed | Conf. |
|---|---|---|---|---|---|---|
| `POST /sign-up` | Create an account | `SignUp { UserId: Guid, Email: string, Password: string, Role: string, Permissions: string[] }` — empty `UserId` replaced with `Guid.NewGuid()` | `201` + `Location: identity/me` | none | yes | High |
| `POST /sign-in` | Issue tokens | `SignIn { Email: string, Password: string }` | `200` + `AuthDto { AccessToken, RefreshToken, Role: string, Expires: long }` written by `WriteJsonAsync` | none | yes | High |
| `GET /me` | Read the caller | none; the handler calls `AuthenticateUsingJwtAsync()` and reads the subject from the token | `200` + `UserDto`; `401` when the resolved id is `Guid.Empty` | Bearer | yes | High |
| `GET /users/{userId}` | Read any user | `GetUser { UserId: Guid }` | `200` + `UserDto`; `404` when the service returns null | Bearer (gateway adds role `admin`) | yes | High |
| `POST /access-tokens/revoke` | Blacklist an access token | `RevokeAccessToken { AccessToken: string }` | `204` (`StatusCode = 204`, no body) | Bearer | **no route in either gateway config** | High |
| `POST /refresh-tokens/use` | Exchange a refresh token | `UseRefreshToken { RefreshToken: string }` | `200` + `AuthDto` written by `WriteJsonAsync` | none stated on the route | **no route in either gateway config** | High |
| `POST /refresh-tokens/revoke` | Revoke a refresh token | `RevokeRefreshToken { RefreshToken: string }` | `204` | Bearer | **no route in either gateway config** | High |

**Token contract** (*observed*, `Identity.Api/appsettings.json` `jwt`): `issuer: pacco`,
`expiryMinutes: 60`, `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`. A signing key is committed
in that file; it is referenced here by path only and is not reproduced (B3). The `expires` field in
`AuthDto` is therefore an absolute expiry 60 minutes after issue.

**The three unexposed endpoints are the entire refresh-token lifecycle.** With a 60-minute access
token and no gateway route to `refresh-tokens/use`, a client behind the gateway has no observable
way to refresh — it must sign in again. See §10 note 6 and GAP-3.

### 4.5 — `operations-service` (CAP-11)

Evidence: `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response — *observed* | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /operations/{operationId}` | Read a pending/complete/rejected operation record | `GetOperation { OperationId: string }` | `200` + `OperationDto { Id, UserId, Name: string, State: OperationState, Code, Reason: string }`; `NotFound()` when the service returns null | none — and no gateway route adds any (§2.5) | High |

`Program.cs` additionally calls `endpoints.MapHub<PaccoHub>("/pacco")` and
`endpoints.MapGrpcService<GrpcServiceHost>()`; both are inventoried in §5.

### 4.6 — `orders-service` (CAP-07)

Evidence: `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /orders` | List orders, optionally by customer | `GetOrders { CustomerId: Guid? }` | `IEnumerable<OrderDto>` | none at the service | High |
| `GET /orders/{orderId}` | Read an order | `GetOrder { OrderId: Guid }` | `OrderDto`, `404` when null | none at the service | High |
| `POST /orders` | Create an order | `CreateOrder { OrderId: Guid, CustomerId: Guid }` — empty `OrderId` replaced with `Guid.NewGuid()` | `201` + `Location: orders/{OrderId}` | none at the service | High |
| `DELETE /orders/{orderId}` | Delete an order | `DeleteOrder { OrderId: Guid }` | `200`, empty body | none at the service | High |
| `POST /orders/{orderId}/parcels/{parcelId}` | Attach a parcel | `AddParcelToOrder { OrderId: Guid, ParcelId: Guid }` | `200`, empty body | none at the service | High |
| `DELETE /orders/{orderId}/parcels/{parcelId}` | Detach a parcel | `DeleteParcelFromOrder { OrderId: Guid, ParcelId: Guid }` | `200`, empty body | none at the service | High |
| `POST /orders/{orderId}/vehicles/{vehicleId}` | Assign a vehicle and price the order | `AssignVehicleToOrder { OrderId: Guid, VehicleId: Guid, DeliveryDate: DateTime }` | `200`, empty body | none at the service | High |

`ApproveOrder { OrderId: Guid }` and `CancelOrder { OrderId: Guid, Reason: string }` are declared
commands with handlers and bus subscriptions but **have no HTTP route** — they are reachable only
over the broker (§9.4). This is *observed*, not an omission in this inventory.

### 4.7 — `parcels-service` (CAP-06)

Evidence: `hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /parcels/volume` | Sum the volume of a parcel set | `GetParcelsVolume { ParcelIds: Guid[] }` | `ParcelsVolumeDto { Volume: double }` | none at the service | High |
| `GET /parcels/{parcelId}` | Read a parcel | `GetParcel { ParcelId: Guid }` | `ParcelDto`, `404` when null | none at the service | High |
| `GET /parcels` | List parcels | `GetParcels { CustomerId: Guid?, IncludeAddedToOrders: bool }` | `IEnumerable<ParcelDto>` | none at the service | High |
| `POST /parcels` | Register a parcel | `AddParcel { ParcelId: Guid, CustomerId: Guid, Variant: string, Size: string, Name: string, Description: string }` — empty `ParcelId` replaced with `Guid.NewGuid()` | `201` + `Location: parcels/{ParcelId}` | none at the service | High |
| `DELETE /parcels/{parcelId}` | Delete a parcel | `DeleteParcel { ParcelId: Guid }` | `200`, empty body | none at the service | High |

`GET /parcels/volume` is registered **before** `GET /parcels/{parcelId}` in `Program.cs`, which is
the opposite order from `ntrada.yml` (§2.7, Q1).

### 4.8 — `pricing-service` (CAP-08)

Evidence: `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /pricing` | Quote a discounted order price | `GetOrderPricing { CustomerId: Guid, OrderPrice: decimal }` | `OrderPricingDto { OrderPrice, CustomerDiscount, OrderDiscountPrice: decimal }` | none at the service | High |

`pricing-service` is the only service with **no `rabbitMq` section and no `mongo` section** in
`appsettings.json` — it is purely a synchronous calculator with one outbound HTTP dependency (§6).
It publishes and consumes **nothing** and owns no exchange.

### 4.9 — `ordermaker-service` (CAP-10)

Evidence: `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /` | Service banner | none | `200` + the literal string `Welcome to Pacco uber AI order maker Service!` | none | High |
| `POST /orders` | Start the automated order saga | `MakeOrder { OrderId: Guid, CustomerId: Guid, ParcelId: Guid }` — empty `OrderId` replaced with `Guid.NewGuid()` | `200`, empty body — the actual work is asynchronous (§9.6) | none | High |

**Path collision across services.** `POST /orders` on port 5015 (`ordermaker-service`, starts a
saga) and `POST /orders` on port 5006 (`orders-service`, creates one order) are different contracts
sharing a path. No gateway route disambiguates them because `ordermaker-service` has no gateway
module at all.

`ordermaker-service` also sets `httpClient.type: ""` (empty) in `appsettings.json`, unlike every
other service which sets `fabio` — so its two outbound calls (§6) bypass the load balancer and
address services directly. *Observed*.

### 4.10 — `vehicles-service` (CAP-05)

Evidence: `hianshul100_Pacco.Services.Vehicles/src/Pacco.Services.Vehicles.Api/Program.cs`.

| Method & path | Purpose | Request — *observed* | Response | Auth | Conf. |
|---|---|---|---|---|---|
| `GET /vehicles/{vehicleId}` | Read a vehicle | `GetVehicle { VehicleId: Guid }` | `VehicleDto`, `404` when null | none at the service | High |
| `GET /vehicles` | Search the fleet, paged | `SearchVehicles : PagedQueryBase { PayloadCapacity: double, LoadingCapacity: double, Variants: Variants }` plus the inherited paging parameters | `PagedResult<VehicleDto>` — a paging envelope with an `items` collection | none at the service | High |
| `POST /vehicles` | Add a vehicle | `AddVehicle { VehicleId: Guid, Brand, Model, Description: string, PayloadCapacity, LoadingCapacity: double, PricePerService: decimal, Variants: Variants }` — empty `VehicleId` replaced with `Guid.NewGuid()` | `201` + `Location: vehicles/{VehicleId}` | none at the service | High |
| `PUT /vehicles/{vehicleId}` | Update a vehicle | `UpdateVehicle { VehicleId: Guid, Description: string, PricePerService: decimal, Variants: Variants }` | `200`, empty body | none at the service | High |
| `DELETE /vehicles/{vehicleId}` | Delete a vehicle | `DeleteVehicle { VehicleId: Guid }` | `200`, empty body | none at the service | High |

`Variants` is a flags enum serialised as an integer on the query string (*observed*,
`Pacco.Services.Vehicles.rest` sends `variants=1`) and as a `string[]` on `VehicleDto`
(*observed*, `Vehicles.Application/DTO/VehicleDto.cs`) — the request and response representations of
the same concept differ.

`GET /vehicles` is the only endpoint in the platform with a paging envelope, and the gateway
discards it (§2.9).

## 5 — Non-HTTP RPC and streaming surfaces

### 5.1 — gRPC — `operations-service`

Evidence: `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Operations.proto`
(an identical copy exists at `Pacco.Services.Operations.GrpcClient/Operations.proto`). Registered by
`endpoints.MapGrpcService<GrpcServiceHost>()` in `Operations.Api/Program.cs`. Related capability
**CAP-11**.

Package `Services.Operations`, service `GrpcOperationsService`. Default client address
`https://localhost:50050` (*observed*, `Operations.GrpcClient/Program.cs`).

| Method | Kind | Request — *observed* | Response — *observed* | Auth | Conf. |
|---|---|---|---|---|---|
| `GetOperation` | unary | `GetOperationRequest { string id = 1 }` | `GetOperationResponse { string id = 1; string userId = 2; string name = 3; string state = 4; string code = 5; string reason = 6 }` | **none** — no interceptor, credential, or metadata check is registered | High |
| `SubscribeOperations` | server-streaming | `Empty {}` | `stream GetOperationResponse` | **none** | High |

`GetOperationResponse` carries the same six fields as the HTTP `OperationDto`, but `state` is a
`string` here and an `OperationState` enum over HTTP — the wire representations of the same field
differ between the two transports. *Observed*.

**`SubscribeOperations` takes `Empty`, so a caller subscribes to every operation on the platform
regardless of which user it belongs to**, on an unauthenticated port. *Observed* from the IDL and
the absence of any registered interceptor (§10 note 7).

The only consumer of this contract inside the workspace is `operations-grpc-client`, a console
executable in the same repository (`OutputType: Exe`). No service calls it.

### 5.2 — SignalR — `operations-service`

Evidence: `Operations.Api/Program.cs` (`endpoints.MapHub<PaccoHub>("/pacco")`),
`Operations.Api/Hubs/PaccoHub.cs`, `Operations.Api/Services/HubService.cs`,
`Operations.Api/Services/HubWrapper.cs`. Backplane `signalR: { backplane: redis }` (*observed*,
`Operations.Api/appsettings.json`). Related capability **CAP-11**.

Hub path `/pacco` on port 5005. **No gateway route proxies `/pacco` in either configuration** — a
browser client must reach `operations-service` directly (GAP-4).

**Client-callable hub method**

| Method | Request — *observed* | Effect — *observed* | Response to caller — *observed* | Conf. |
|---|---|---|---|---|
| `InitializeAsync` | `token: string` — a raw JWT | Parses the token payload, reads `Subject`, and adds the connection to the group named by that user id | Emits `connected` to the calling connection on success, `disconnected` otherwise | High |

**Server-emitted messages.** All three are built inline in `HubService.cs` as anonymous objects and
are therefore fully determinable — *observed*, not runtime-capture material. Each is delivered by
`HubWrapper.PublishToUserAsync(userId, message, data)`, which resolves to
`_hubContext.Clients.Group(userId.ToUserGroup()).SendAsync(message, data)`, so **the delivery scope
is the single user group** and the user id is the routing key.

| Message name (verbatim) | Payload — *observed* | Emitted when | Conf. |
|---|---|---|---|
| `operation_pending` | `{ id, name }` | A command is observed on the broker with saga state `Pending` | High |
| `operation_completed` | `{ id, name }` | An event is observed, or saga state `Completed` | High |
| `operation_rejected` | `{ id, name, code, reason }` | A rejected event is observed, or saga state `Rejected` | High |
| `connected` | none | `InitializeAsync` resolved a user id from the token | High |
| `disconnected` | none | `InitializeAsync` failed to resolve a user id | High |

`id` is the correlation id taken from `IMessageProperties.CorrelationId`; `name` is the message name
`operations-service` observed on the wire (a snake-case routing key from §9.3). *Observed*,
`Operations.Api/Handlers/`.

**No client of this hub exists in the workspace.** `pacco-web` contains only a `README.md` whose
entire content is `# Pacco.Web` (§10 note 3), so the consumer side of every message above is
**Unverifiable — Missing Source Evidence** (GAP-5).

## 6 — Inter-service HTTP calls

These are service-to-service calls made in code, not through the gateway. Each is a typed client in
the calling service's `Infrastructure/Services/Clients/` folder. The `{url}` placeholder is resolved
from the caller's `httpClient.services` map in its own `appsettings.json`; with
`httpClient.type: fabio` the request is addressed through the Fabio load balancer, and with an empty
type it goes direct (§4.9). All seven are *observed*.

| # | Caller | Callee | Call | Response type | Auth on the call | Cap link | Conf. |
|---|---|---|---|---|---|---|---|
| 1 | `availability-service` | `customers-service` | `GET {url}/customers/{id}/state` | `CustomerStateDto { State: string, IsValid: bool }` | Vault PKI certificate in the `Certificate` header (§4.2) | CAP-04 → CAP-03 | High |
| 2 | `pricing-service` | `customers-service` | `GET {url}/customers/{id}` | `CustomerDto { Id: Guid, IsVip: bool, CompletedOrders: Guid[] }` | **none** | CAP-08 → CAP-03 | High |
| 3 | `orders-service` | `parcels-service` | `GET {url}/parcels/{id}` | `ParcelDto { Id: Guid, Name, Variant, Size: string }` | **none** | CAP-07 → CAP-06 | High |
| 4 | `orders-service` | `pricing-service` | `GET {url}/pricing?customerId={customerId}&orderPrice={orderPrice}` | `OrderPricingDto { OrderPrice, CustomerDiscount, OrderDiscountPrice: decimal }` | **none** | CAP-07 → CAP-08 | High |
| 5 | `orders-service` | `vehicles-service` | `GET {url}/vehicles/{id}` | `VehicleDto { Id: Guid, PricePerService: decimal }` | **none** | CAP-07 → CAP-05 | High |
| 6 | `ordermaker-service` | `vehicles-service` | `GET {url}/vehicles` | `PagedResult<VehicleDto>`; the client takes `Items.FirstOrDefault()` | **none** | CAP-10 → CAP-05 | High |
| 7 | `ordermaker-service` | `availability-service` | `GET {url}/resources/{resourceId}` | `ResourceDto { Id: Guid, Reservations: [{ DateTime, Priority }] }` | **none** | CAP-10 → CAP-04 | High |

**Caller-local DTOs are narrower than the producer's DTO, deliberately.** In rows 2, 3, 5, 6 and 7
the calling service declares its own DTO class carrying only the fields it uses — for example
`orders-service`'s `VehicleDto` has two fields where `vehicles-service` returns eight. This is a
*consumer-driven projection*, not a contract mismatch: JSON deserialisation ignores the surplus. It
does mean **the callee can add fields freely but cannot rename or remove the projected ones**
without silently zeroing them in the caller.

**Row 6 reads the whole fleet to pick one vehicle.** `ordermaker-service` calls `GET /vehicles` with
no filter and takes the first item of the page. *Observed*.

## 7 — Cross-cutting HTTP contract facts

### 7.1 — Headers

| Header | Direction | Set by | Meaning | Evidence | Conf. |
|---|---|---|---|---|---|
| `Authorization: Bearer <jwt>` | request | client | The only credential accepted on any HTTP route | `ntrada.yml` `extensions.jwt`; every `.rest` fixture | High |
| `Certificate` | request | `availability-service` | Raw Vault PKI certificate data for service-to-service authentication to `customers-service` | `Availability.../CustomersServiceClient.cs:32–34`; `Customers.Api/appsettings.json` `security.certificate.header` | High |
| `Resource-ID` | response | `api-gateway` | The identifier the gateway generated for a created resource, on routes declaring `resourceId: { generate: true }` | `ntrada.yml` `extensions.cors.exposedHeaders`; consumed in `Pacco-sample-scenario.rest` | High |
| `Request-ID` | response | `api-gateway` | Per-request identifier (`generateRequestId: true`) | `ntrada.yml` | High |
| `Trace-ID` | response | `api-gateway` | Distributed-trace identifier (`generateTraceId: true`) | `ntrada.yml` | High |
| `Total-Count` | response | declared exposed by CORS, **never set by any code in the workspace** | Intended paging total | `ntrada.yml` `exposedHeaders`; no producer found (GAP-6) | Medium |
| `Correlation-Context` | request | inter-service HTTP | JSON correlation context deserialised into `CorrelationContext` | `Orders.Infrastructure/Extensions.cs` `GetCorrelationContext` | High |

**CORS contract** (*observed*, `ntrada.yml` `extensions.cors`): `allowedOrigins: *`,
`allowedMethods: post, put, delete`, `exposedHeaders: Request-ID, Resource-ID, Trace-ID,
Total-Count`. A wildcard origin combined with bearer-token auth is recorded in §10 note 8.

### 7.2 — Authentication, consolidated

*Observed.* The **only** authentication mechanism on any HTTP route in the platform is a JWT bearer
token, and it is enforced **only at `api-gateway`**. No service validates a token on its own port;
no session cookie exists anywhere; no API key exists anywhere.

| Scope | Enforcement point | Notes |
|---|---|---|
| 35 of 40 gateway routes | `api-gateway` JWT extension | `auth.enabled: true`, `auth.global: false`, per-route `auth:` flag |
| 5 gateway routes require role `admin` | `api-gateway` claim check | `GET /customers`, `GET /customers/{customerId}`, `GET /customers/{customerId}/state`, `PUT /customers/{customerId}/state/{state}`, `GET /identity/users/{userId}` |
| 3 gateway routes are anonymous | `auth: false` | `POST /identity/sign-up`, `POST /identity/sign-in`, `GET /operations/{operationId}` |
| `identity-service` `/me` | the service itself | re-parses the bearer token; the only service-side token check in the platform |
| `customers-service` | certificate ACL | declared, enforcement [unknown] (§4.2, Q6) |
| gRPC port 50050, SignalR `/pacco` | **nothing** | no interceptor, no auth middleware (§5) |
| All 10 service ports | **nothing** | anyone with network reach bypasses every check above (Q5) |

**Ownership checks are a domain concern, not a transport one.** Where a caller must own the resource
it acts on, the check is inside the domain handler using the `customerId` the gateway injected — for
example `UnauthorizedResourceAccessException` in `availability-service`. This means a direct call to
a service port supplying an arbitrary `customerId` satisfies the check. *Inferred* from the gateway
binding rules (§2) plus the exception types in each `Core/Exceptions/` folder.

### 7.3 — Status codes

| Code | When | Source | Conf. |
|---|---|---|---|
| `200` | Successful query; command with no `afterDispatch` | Convey framework default | Medium |
| `201` | Command with `afterDispatch: ctx.Response.Created(...)`; the `Location` value is the literal string in the call site | *observed* per row in §4 | High |
| `202` | Broker-published gateway routes in the asynchronous configuration | `Ntrada.Extensions.RabbitMq` — **not verifiable in this workspace**, recorded as GAP-1 rather than asserted | Low |
| `204` | `afterDispatch: NoContent()` on `PUT /customers/{customerId}/state/{state}`; explicit `StatusCode = 204` on the two `identity-service` revoke endpoints | *observed* | High |
| `400` | **Every** handled error — see §7.4 | *observed* | High |
| `401` | `identity-service` `GET /me` when the token resolves to `Guid.Empty`; gateway JWT rejection | *observed* for the service; framework for the gateway | High / Medium |
| `404` | A `Get<TQuery, TResult>` whose handler returns null | Convey framework behaviour, confirmed by explicit `NotFound()` in `operations-service` and `identity-service` | Medium |

### 7.4 — Error contract

*Observed.* Every service registers `AddErrorHandler<ExceptionToResponseMapper>()` and
`UseErrorHandler()`. The mapper is structurally identical in every service:

```csharp
DomainException ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.BadRequest),
AppException    ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.BadRequest),
_               => new ExceptionResponse(new {code = "error", reason = "There was an error."}, HttpStatusCode.BadRequest)
```

**The platform-wide HTTP error body is therefore `{ "code": string, "reason": string }` with status
`400` — always `400`, for every error class including not-found and unauthorised-access domain
errors.** `GetCode` derives the code from the exception type when the exception does not carry an
explicit `Code`:
`exception.GetType().Name.Underscore().Replace("_exception", "")` — so
`ResourceNotFoundException` yields code `resource_not_found`. *Observed* in every
`<Service>.Infrastructure/Exceptions/ExceptionToResponseMapper.cs`.

**Two services have only the catch-all arm.** `ordermaker-service` and `operations-service` map
*every* exception to `{ "code": "error", "reason": "There was an error." }` with status `400`. A
caller of those two services cannot distinguish error causes at all. *Observed*.

**The `code` and `reason` fields are the same two fields carried by every rejected event** (§9.3) and
surfaced by `operation_rejected` over SignalR (§5.2) — one error vocabulary across all three
transports. *Inferred* from the three observed shapes.

### 7.5 — Operational endpoints

*Inferred* — these are registered by Convey extension methods whose route constants live in NuGet
packages outside this workspace. They are listed because configuration in this workspace names them
explicitly, but their exact paths are not observable here.

| Endpoint | Registered by | Evidence in this workspace | Conf. |
|---|---|---|---|
| `GET /docs` — Swagger UI, per service and on the gateway | `AddWebApiSwaggerDocs()` + `UseSwaggerDocs()`; gateway `extensions.swagger` | `swagger: { enabled: true, name: v1, title: API, version: v1, routePrefix: docs, includeSecurity: true }` in every service `appsettings.json`; `title: Pacco API, version: v1, routePrefix: docs` in `ntrada.yml` | Medium — path derived from `routePrefix`, generator is package code |
| Public contracts listing | `UsePublicContracts<ContractAttribute>()` in every service | The call site is *observed*; the route it registers is a package default and **is not stated anywhere in this workspace** | Low — existence High, path [unknown] (GAP-7) |
| `GET /metrics` — Prometheus scrape | `AddMetrics()` + `UseMetrics()` with `metrics.prometheusEnabled: true` | `logger.excludePaths` names `/metrics` in every service | Medium |
| `GET /ping` | Convey health endpoint | `logger.excludePaths` names `/ping` in every service; no other reference exists | Medium |
| `GET /` | Explicit `Get("")` registration | *observed* in `availability-service` and `ordermaker-service` only; `logger.excludePaths` names `/` in every service | High for the two, Medium elsewhere |

**No API versioning exists.** No route in any configuration or any service carries a version segment,
an `Accept` version parameter, or a version header. The `swagger.version: v1` string is documentation
metadata only and appears in no route. *Observed* (absence). See Q4.

### 7.6 — Contract statements (Part 1)

`api-gateway` exposes the HTTP route `GET /availability/resources`.
`api-gateway` exposes the HTTP route `POST /identity/sign-in`.
`api-gateway` routes `/availability/*` to `availability-service`.
`api-gateway` routes `/customers/*` to `customers-service`.
`api-gateway` routes `/deliveries/*` to `deliveries-service`.
`api-gateway` routes `/identity/*` to `identity-service`.
`api-gateway` routes `/operations/*` to `operations-service`.
`api-gateway` routes `/orders/*` to `orders-service`.
`api-gateway` routes `/parcels/*` to `parcels-service`.
`api-gateway` routes `/pricing/*` to `pricing-service`.
`api-gateway` routes `/vehicles/*` to `vehicles-service`.
`availability-service` exposes 6 HTTP endpoints plus a root banner endpoint.
`customers-service` exposes 5 HTTP endpoints.
`deliveries-service` exposes 5 HTTP endpoints.
`identity-service` exposes 7 HTTP endpoints.
`operations-service` exposes 1 HTTP endpoint.
`orders-service` exposes 7 HTTP endpoints.
`parcels-service` exposes 5 HTTP endpoints.
`pricing-service` exposes 1 HTTP endpoint.
`vehicles-service` exposes 5 HTTP endpoints.
`ordermaker-service` exposes 1 HTTP endpoint plus a root banner endpoint.
`operations-service` exposes the gRPC service `GrpcOperationsService` on port 50050.
`operations-service` exposes the gRPC method `GetOperation`.
`operations-service` exposes the gRPC method `SubscribeOperations`.
`operations-service` exposes the SignalR hub `/pacco`.
`operations-service` emits the SignalR message `operation_pending`.
`operations-service` emits the SignalR message `operation_completed`.
`operations-service` emits the SignalR message `operation_rejected`.
`operations-grpc-client` consumes the gRPC service `GrpcOperationsService`.
`availability-service` calls `customers-service` over HTTP.
`pricing-service` calls `customers-service` over HTTP.
`orders-service` calls `parcels-service` over HTTP.
`orders-service` calls `pricing-service` over HTTP.
`orders-service` calls `vehicles-service` over HTTP.
`ordermaker-service` calls `vehicles-service` over HTTP.
`ordermaker-service` calls `availability-service` over HTTP.
`api-gateway` authenticates callers with JWT bearer tokens issued by `identity-service`.

## 8 — MCP tool contract inventory

**The Pacco platform exposes no MCP tools and hosts no MCP server. This section is empty by
observation, not by omission.**

*Observed.* A case-insensitive search across all fourteen clone directories for every MCP marker —
`modelcontextprotocol`, `mcp_server`, `fastmcp`, the `@mcp.` decorator prefix, `mcp.tool`,
`list_tools`, and the `tools/call` method name — returns **exactly one** hit, and it is this
document. No other file in any repository contains any of those strings.

Corroborating absences, each *observed*:

| Expected artefact if MCP existed | Present? | Evidence |
|---|---|---|
| An MCP server manifest or transport registration | No | No `mcp.json`, `server.json`, or MCP stdio/SSE transport wiring inside any repository |
| A tool-registration decorator or builder call | No | No `@tool`, `@mcp.tool`, `AddMcpServer`, `WithTools`, or equivalent in any of the ~1,100 C# files |
| An MCP client dependency | No | No `.csproj` in any repository references an MCP package; the Convey `0.4.*` family and `Ntrada 0.4.*` are the only framework dependencies |
| A tool schema definition | No | No JSON Schema file of any kind exists in any repository — the same absence that leaves the two Ntrada `schema:` references dangling (§10 note 2) |

**Why this is expected rather than surprising.** Pacco targets .NET Core 3.1 and its dependency set
is fixed at Convey `0.4.*` (*observed*, every `.csproj`). Its integration surfaces are HTTP, gRPC,
SignalR and RabbitMQ, all inventoried in Parts 1 and 3. There is no agent-facing or tool-calling
surface in the platform at all.

**One MCP configuration file exists in the workspace, and it is not part of the platform.**
`.mcp.json` sits at the workspace root, outside every repository directory, and configures the
discovery tooling used to produce this document — not any Pacco deployable. It is named here only so
that a later reader does not mistake it for a platform artefact. Its contents include a credential
and are not reproduced (B3).

### 8.1 — Contract statements (Part 2)

No Pacco service exposes any MCP tool.
No Pacco service consumes any MCP tool.
`api-gateway` exposes no MCP tool.
`operations-service` exposes no MCP tool.
The Pacco platform hosts no MCP server.

## 9 — Event and message schema inventory

### 9.1 — Physical topology

**Broker system: RabbitMQ.** *Observed* — `Convey.MessageBrokers.RabbitMQ` is referenced by nine of
the ten services and by `api-gateway` (`Ntrada.Extensions.RabbitMq`). **No Kafka, SNS, SQS, Kinesis,
Azure Service Bus, or NATS dependency exists in any repository** — a search of every `.csproj` and
every configuration file finds none.

**Physical exchange names, copied character-for-character** from `rabbitMq.exchange.name` in each
service's `appsettings.json`. Every exchange is `type: topic`, `declare: true`, `durable: true`,
`autoDelete: false`.

| Physical exchange name | Owning service | Carries traffic? |
|---|---|---|
| `availability` | `availability-service` | yes |
| `customers` | `customers-service` | yes |
| `deliveries` | `deliveries-service` | yes |
| `identity` | `identity-service` | yes |
| `operations` | `operations-service` | **no — declared, never published to** (§10 note 5) |
| `ordermaker` | `ordermaker-service` | yes |
| `orders` | `orders-service` | yes |
| `parcels` | `parcels-service` | yes |
| `vehicles` | `vehicles-service` | yes |

`pricing-service` has **no `rabbitMq` section at all** and therefore owns no exchange and touches the
broker in no way. *Observed*.

**Physical vs. logical naming — the rule that governs every table in §9.3.** Pacco messages have two
distinct names and this inventory records both:

- The **logical name** is the C# type name, e.g. `CustomerCreated`. It is what the code publishes and
  subscribes to and what appears in every `SubscribeEvent<>` call.
- The **physical routing key** is the snake-case string that actually appears on the wire, e.g.
  `customer_created`. It is produced by `rabbitMq.conventionsCasing: snakeCase` (*observed*, every
  service `appsettings.json`) and is independently catalogued in
  `Operations.Api/messages.json`, which is the one committed file that states physical routing keys
  verbatim.

Where the two disagree, both are recorded and the disagreement is a finding, not a normalisation
(§10 note 9).

**Physical queue names.** `rabbitMq.queue.template` is `<service>/{{exchange}}.{{message}}` in every
service (*observed*). Substituting gives one durable, non-exclusive, non-auto-delete queue per
*(subscribing service, source exchange, message)* triple — for example `orders-service` subscribing
to `customer_created` on the `customers` exchange yields the physical queue
`orders-service/customers.customer_created`. Every queue name in §9.4 is derived by this rule and is
therefore *inferred* from two observed facts (the template and the subscription), not read from a
manifest.

**Ordering and partitioning.** *Observed:* RabbitMQ topic exchanges have **no partition key and no
cross-queue ordering guarantee**; nothing in any repository configures consistent-hash routing,
single-active-consumer, or a priority queue. Per-queue FIFO holds only while a queue has one
consumer, and no repository pins consumer counts. The *correlating* identifiers that do exist are:

| Identifier | Scope | Set by | Read by | Evidence |
|---|---|---|---|---|
| `IMessageProperties.CorrelationId` | one end-to-end request | `api-gateway` / originating service | `operations-service`, on every message; skipped when blank | `Operations.Api/Handlers/` |
| `message_context` header | correlation payload | `rabbitMq.context.header` | Convey message-context plumbing | every `appsettings.json` |
| `span_context` header | distributed trace | Jaeger RabbitMQ plugin | `GetSpanContext` in each `Infrastructure/Extensions.cs`, decoded `byte[]` → UTF-8 | `Orders.Infrastructure/Extensions.cs` |
| `Saga` header, values `Pending` / `Completed` / `Rejected` | one saga instance | `ordermaker-service` (`AIOrderMakingSaga`) | `operations-service` (`GetSagaState`); forwarded by `orders-service` (`GetHeadersToForward`) | `AIOrderMakingSaga.cs`; `Operations.Api/.../Extensions.GetSagaState` |
| `OrderId` | one saga instance | `AIOrderMakingSaga.ResolveId` returns `OrderId.ToString()` for **every** message it handles | Chronicle saga store | `OrderMaker.Application/Sagas/AIOrderMakingSaga.cs` |

The **`Order key` column in §9.3 is therefore `none` for every message** except those carried by the
saga, where the saga correlation key is `OrderId`. That is a property of the saga, not of the
exchange.

**Reliability contract.** Six services wrap their handlers in a Mongo-backed message outbox —
`TryDecorate(ICommandHandler<>, OutboxCommandHandlerDecorator<>)`,
`TryDecorate(IEventHandler<>, OutboxEventHandlerDecorator<>)`, `AddMessageOutbox(o => o.AddMongo())`
with `outbox: { enabled: true, type: sequential, expiry: 3600, intervalMilliseconds: 2000,
inboxCollection: inbox, outboxCollection: outbox, disableTransactions: true }` (*observed*,
`Orders.Api/appsettings.json`). Delivery is therefore **at-least-once with inbox de-duplication**,
and `disableTransactions: true` means the outbox write and the domain write are **not** atomic.
*Inferred* from the observed configuration plus the decorator registrations.

### 9.2 — How to read the message tables

- **Physical routing key** — copied character-for-character from `Operations.Api/messages.json`,
  which is the only committed catalogue of wire names. A key present in the code but absent from
  `messages.json` is flagged in the row.
- **Logical type** — the C# type name, copied verbatim from the declaring file.
- **Payload fields** — the constructor parameters and public properties of the declaring type,
  copied verbatim. These are *observed*; nothing here is guessed.
- **`[Contract]`** — the type carries Convey's `[Contract]` attribute, which publishes it through
  `UsePublicContracts<ContractAttribute>()`. Its absence is noted because it means the type is not
  advertised on the public-contracts endpoint even though it is published on the broker.
- **Consumers** — every service with a matching `SubscribeCommand<>` / `SubscribeEvent<>`
  registration, plus `operations-service` where the routing key appears in `messages.json`.
  `operations-service` is listed as a **status-only** consumer because it does not deserialise
  payload fields (§9.5).
- **Evidence** — the per-exchange evidence line above each table applies to every row; row-specific
  paths are given inline.

**Catalogue census.** `Operations.Api/messages.json` declares **80 physical routing keys** across
eight exchanges — **24 commands, 29 events, 27 rejected events**. A mechanical diff of those keys
against every message type declared in the source finds five divergences, all recorded in §10
note 9: three types exist in code but are absent from the catalogue
(`add_delivery_registration_rejected`, `complete_order_rejected`,
`release_resource_reservation_rejected`), and two catalogue keys correspond to no type at all
(`release_resource`, `reserve_resource_rejected`).

### 9.3 — Message catalogue by physical exchange

Throughout: **PUB** = publisher, **CON** = consumers. `operations-service` appears as a consumer of
every catalogued routing key and is written as `operations-service (status-only)`; the reason is
§9.5. `api-gateway (async only)` means the message is published by the gateway **only** when
`NTRADA_CONFIG` selects `ntrada-async*.yml` (§3).

#### 9.3.1 — Physical exchange `availability`

Evidence: `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Application/Commands/`,
`.../Events/`, `.../Events/Rejected/`; subscriptions
`Availability.Infrastructure/Extensions.cs:107–112`; catalogue `messages.json:2–23`; publishers
`ntrada-async.yml` `modules.availability`, `OrderMaker/Sagas/AIOrderMakingSaga.cs`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `add_resource` | `AddResource` | command | `ResourceId: Guid`, `Tags: string[]` | `api-gateway` (async only) | `availability-service`; `operations-service` (status-only) | none | High |
| `delete_resource` | `DeleteResource` | command | `ResourceId: Guid` | `api-gateway` (async only) | `availability-service`; `operations-service` (status-only) | none | High |
| `reserve_resource` | `ReserveResource` | command | `ResourceId: Guid`, `DateTime: DateTime`, `Priority: int`, `CustomerId: Guid` | `api-gateway` (async only); `ordermaker-service` | `availability-service`; `operations-service` (status-only) | `OrderId` when published by the saga | High |
| `release_resource` | **no type exists** | command | — | `api-gateway` (async only) | **none** — `availability-service` binds `release_resource_reservation` (§10 note 9) | none | High |
| `release_resource_reservation` | `ReleaseResourceReservation` | command | `ResourceId: Guid`, `DateTime: DateTime` | **none observed** | `availability-service`; **not in `messages.json`**, so `operations-service` does not observe it | none | High |
| `resource_added` | `ResourceAdded` `[Contract]` | event | `ResourceId: Guid` | `availability-service` | `operations-service` (status-only) only | none | High |
| `resource_deleted` | `ResourceDeleted` `[Contract]` | event | `ResourceId: Guid` | `availability-service` | `operations-service` (status-only) only | none | High |
| `resource_reserved` | `ResourceReserved` `[Contract]` | event | `ResourceId: Guid`, `DateTime: DateTime` | `availability-service` | `orders-service`, `ordermaker-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `resource_reservation_released` | `ResourceReservationReleased` `[Contract]` | event | `ResourceId: Guid`, `DateTime: DateTime` | `availability-service` | `operations-service` (status-only) only | none | High |
| `resource_reservation_canceled` | `ResourceReservationCanceled` `[Contract]` | event | `ResourceId: Guid`, `DateTime: DateTime` | `availability-service` | `orders-service`; `operations-service` (status-only) | none | High |
| `add_resource_rejected` | `AddResourceRejected` `[Contract]` | rejected | `ResourceId: Guid`, `Reason: string`, `Code: string` | `availability-service` | `operations-service` (status-only) | none | High |
| `delete_resource_rejected` | `DeleteResourceRejected` `[Contract]` | rejected | `ResourceId: Guid`, `Reason: string`, `Code: string` | `availability-service` | `operations-service` (status-only) | none | High |
| `release_resource_rejected` | `ReleaseResourceRejected` `[Contract]` | rejected | `ResourceId: Guid`, `DateTime: DateTime`, `Reason: string`, `Code: string` | `availability-service` — emitted when a **`ReleaseResourceReservation`** command fails | `operations-service` (status-only) | none | High |
| `release_resource_reservation_rejected` | `ReleaseResourceReservationRejected` `[Contract]` | rejected | `ResourceId: Guid`, `DateTime: DateTime`, `Reason: string`, `Code: string` | `availability-service` — emitted when a **`ReserveResource`** command fails | **none** — absent from `messages.json`, so even `operations-service` never sees it | none | High |
| `reserve_resource_rejected` | **no type exists** | rejected | — | **none** | `operations-service` (status-only), bound to a key nothing publishes | none | High |

**The rejected-event mapping for this exchange is inverted relative to its names.** *Observed*,
`Availability.Infrastructure/Exceptions/ExceptionToMessageMapper.cs`: a failing `ReserveResource`
maps to `ReleaseResourceReservationRejected` (lines 23, 29, 35, 43, 51) while a failing
`ReleaseResourceReservation` maps to `ReleaseResourceRejected` (line 45). Combined with the two
catalogue divergences above, **a failed reservation is published under a routing key that
`messages.json` does not list, so `operations-service` never marks the operation `Rejected` and the
caller polling `GET /operations/{operationId}` sees `Pending` until the 300-second expiry.**
*Inferred* from three observed facts (the mapper, the catalogue, and the polling contract).

**The mapper returns `null` for every unlisted `(exception, message)` pair** — for example a
`ResourceNotFoundException` raised while handling `AddResource`. No rejected event is published in
those cases at all, with the same polling consequence. *Observed*, `ExceptionToMessageMapper.cs:55`.

#### 9.3.2 — Physical exchange `customers`

Evidence: `Pacco.Services.Customers.Application/{Commands,Events,Events/Rejected}/`; subscriptions
`Customers.Infrastructure/Extensions.cs:93–96`; catalogue `messages.json:24–39`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `complete_customer_registration` | `CompleteCustomerRegistration` | command | `CustomerId: Guid`, `FullName: string`, `Address: string` | `api-gateway` (async only) | `customers-service`; `operations-service` (status-only) | none | High |
| `change_customer_state` | `ChangeCustomerState` | command | `CustomerId: Guid`, `State: string` | `api-gateway` (async only) | `customers-service`; `operations-service` (status-only) | none | High |
| `customer_created` | `CustomerCreated` `[Contract]` | event | `CustomerId: Guid` | `customers-service` | `availability-service`, `orders-service`, `parcels-service`; `operations-service` (status-only) | none | High |
| `customer_became_vip` | `CustomerBecameVip` `[Contract]` | event | `CustomerId: Guid` | `customers-service` | `operations-service` (status-only) only — **no domain consumer anywhere** | none | High |
| `customer_state_changed` | `CustomerStateChanged` `[Contract]` | event | `CustomerId: Guid`, `CurrentState: string`, `PreviousState: string` | `customers-service` | `operations-service` (status-only) only — **no domain consumer anywhere** | none | High |
| `complete_customer_registration_rejected` | `CompleteCustomerRegistrationRejected` `[Contract]` | rejected | `CustomerId: Guid`, `Reason: string`, `Code: string` | `customers-service` | `operations-service` (status-only) | none | High |
| `change_customer_state_rejected` | `ChangeCustomerStateRejected` `[Contract]` | rejected | `CustomerId: Guid`, `State: string`, `Reason: string`, `Code: string` | `customers-service` | `operations-service` (status-only) | none | High |

`customer_created` is the widest fan-out on the platform: three services build a local customer copy
from it. Neither `customer_state_changed` nor `customer_became_vip` has a consumer, so **those three
local copies are written once and never updated** (GAP-8).

#### 9.3.3 — Physical exchange `deliveries`

Evidence: `Pacco.Services.Deliveries.Application/{Commands,Events,Events/Rejected}/`; subscriptions
`Deliveries.Infrastructure/Extensions.cs:87–90`; catalogue `messages.json:40–59`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `start_delivery` | `StartDelivery` | command | `DeliveryId: Guid`, `OrderId: Guid`, `Description: string`, `DateTime: DateTime` | `api-gateway` (async only) | `deliveries-service`; `operations-service` (status-only) | none | High |
| `complete_delivery` | `CompleteDelivery` | command | `DeliveryId: Guid` | `api-gateway` (async only) | `deliveries-service`; `operations-service` (status-only) | none | High |
| `fail_delivery` | `FailDelivery` | command | `DeliveryId: Guid`, `Reason: string` | `api-gateway` (async only) | `deliveries-service`; `operations-service` (status-only) | none | High |
| `add_delivery_registration` | `AddDeliveryRegistration` | command | `DeliveryId: Guid`, `Description: string`, `DateTime: DateTime` | `api-gateway` (async only) | `deliveries-service`; `operations-service` (status-only) | none | High |
| `delivery_started` | `DeliveryStarted` `[Contract]` | event | `DeliveryId: Guid`, `OrderId: Guid` | `deliveries-service` | `orders-service` — **which declares only `OrderId`** (§10 note 4); `operations-service` (status-only) | none | High |
| `delivery_completed` | `DeliveryCompleted` `[Contract]` | event | `DeliveryId: Guid`, `OrderId: Guid` | `deliveries-service` | `orders-service`; `operations-service` (status-only) | none | High |
| `delivery_failed` | `DeliveryFailed` `[Contract]` | event | `DeliveryId: Guid`, `OrderId: Guid`, `Reason: string` | `deliveries-service` | `orders-service`; `operations-service` (status-only) | none | High |
| `registration_added_to_delivery` | `RegistrationAddedToDelivery` `[Contract]` | event | `DeliveryId: Guid`, `OrderId: Guid`, `Message: string` | `deliveries-service` | `operations-service` (status-only) only | none | High |
| `start_delivery_rejected` | `StartDeliveryRejected` `[Contract]` | rejected | `DeliveryId: Guid`, `OrderId: Guid`, `Reason: string`, `Code: string` | `deliveries-service` | `operations-service` (status-only) | none | High |
| `complete_delivery_rejected` | `CompleteDeliveryRejected` `[Contract]` | rejected | `DeliveryId: Guid`, `Reason: string`, `Code: string` | `deliveries-service` | `operations-service` (status-only) | none | High |
| `fail_delivery_rejected` | `FailDeliveryRejected` `[Contract]` | rejected | `DeliveryId: Guid`, `Reason: string`, `Code: string` | `deliveries-service` | `operations-service` (status-only) | none | High |
| `add_delivery_registration_rejected` | `AddDeliveryRegistrationRejected` `[Contract]` | rejected | `DeliveryId: Guid`, `Reason: string`, `Code: string` | `deliveries-service` | **none** — absent from `messages.json` (§10 note 9) | none | High |

#### 9.3.4 — Physical exchange `identity`

Evidence: `Pacco.Services.Identity.Application/{Commands,Events,Events/Rejected}/`; subscription
`Identity.Infrastructure/Extensions.cs:103`; catalogue `messages.json:60–74`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `sign_up` | `SignUp` | command | `UserId: Guid`, `Email: string`, `Password: string`, `Role: string`, `Permissions: string[]` | **none observed** — the gateway keeps `POST /identity/sign-up` synchronous in *both* configurations | `identity-service`; `operations-service` (status-only) | none | High |
| `sign_in` | `SignIn` | command | `Email: string`, `Password: string` | **none observed** | **none** — `identity-service` subscribes only `SignUp`; catalogued but unbound | none | High |
| `signed_up` | `SignedUp` `[Contract]` | event | `UserId: Guid`, `Email: string`, `Role: string` | `identity-service` | `customers-service`; `operations-service` (status-only) | none | High |
| `signed_in` | `SignedIn` `[Contract]` | event | `UserId: Guid`, `Role: string` | `identity-service` | `operations-service` (status-only) only | none | High |
| `sign_up_rejected` | `SignUpRejected` `[Contract]` | rejected | `Email: string`, `Reason: string`, `Code: string` | `identity-service` | `operations-service` (status-only) | none | High |
| `sign_in_rejected` | `SignInRejected` `[Contract]` | rejected | `Email: string`, `Reason: string`, `Code: string` | `identity-service` | `operations-service` (status-only) | none | High |

**A credential-bearing command type is published on the broker contract.** `sign_up` carries a
plaintext `Password` field. No publisher for it exists in the workspace today, but the subscription
is live, so anything with publish rights to the `identity` exchange can create an account. The three
token-lifecycle commands (`RevokeAccessToken`, `UseRefreshToken`, `RevokeRefreshToken`) are
deliberately **not** on the broker — they are HTTP-only (§4.4). *Observed*.

#### 9.3.5 — Physical exchange `orders`

Evidence: `Pacco.Services.Orders.Application/{Commands,Events,Events/Rejected}/`; subscriptions
`Orders.Infrastructure/Extensions.cs:95–108`; catalogue `messages.json:84–118`; saga publications
`OrderMaker/Sagas/AIOrderMakingSaga.cs:63,74,104,141`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `create_order` | `CreateOrder` | command | `OrderId: Guid`, `CustomerId: Guid` | `api-gateway` (async only); `ordermaker-service` | `orders-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `add_parcel_to_order` | `AddParcelToOrder` | command | `OrderId: Guid`, `ParcelId: Guid` — the `ordermaker-service` copy adds `CustomerId: Guid` | `api-gateway` (async only); `ordermaker-service` | `orders-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `delete_parcel_from_order` | `DeleteParcelFromOrder` | command | `OrderId: Guid`, `ParcelId: Guid` | `api-gateway` (async only) | `orders-service`; `operations-service` (status-only) | none | High |
| `assign_vehicle_to_order` | `AssignVehicleToOrder` | command | `OrderId: Guid`, `VehicleId: Guid`, `DeliveryDate: DateTime` | `api-gateway` (async only); `ordermaker-service` | `orders-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `delete_order` | `DeleteOrder` | command | `OrderId: Guid` | `api-gateway` (async only) | `orders-service`; `operations-service` (status-only) | none | High |
| `cancel_order` | `CancelOrder` | command | `OrderId: Guid`, `Reason: string` | `ordermaker-service` **only**, as saga compensation, with the literal reason `Because I'm saga` | `orders-service`; `operations-service` (status-only) | `OrderId` | High |
| `approve_order` | `ApproveOrder` | command | `OrderId: Guid` | **none anywhere in the workspace** (§10 note 10) | `orders-service`; `operations-service` (status-only) | none | High |
| `order_created` | `OrderCreated` `[Contract]` | event | `OrderId: Guid` | `orders-service` | `ordermaker-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `parcel_added_to_order` | `ParcelAddedToOrder` `[Contract]` | event | `OrderId: Guid`, `ParcelId: Guid` | `orders-service` | `parcels-service`, `ordermaker-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `parcel_deleted_from_order` | `ParcelDeletedFromOrder` `[Contract]` | event | `OrderId: Guid`, `ParcelId: Guid` | `orders-service` | `parcels-service`; `operations-service` (status-only) | none | High |
| `vehicle_assigned_to_order` | `VehicleAssignedToOrder` — **no `[Contract]`** | event | `OrderId: Guid`, `VehicleId: Guid` | `orders-service` | `ordermaker-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `order_approved` | `OrderApproved` `[Contract]` | event | `OrderId: Guid` | `orders-service`, from `ApproveOrderHandler` only | `ordermaker-service`; `operations-service` (status-only) | `OrderId` in the saga leg | High |
| `order_canceled` | `OrderCanceled` `[Contract]` | event | `OrderId: Guid`, `Reason: string` | `orders-service` | `parcels-service`; `operations-service` (status-only) | none | High |
| `order_deleted` | `OrderDeleted` `[Contract]` | event | `OrderId: Guid` | `orders-service` | `parcels-service`; `operations-service` (status-only) | none | High |
| `order_completed` | `OrderCompleted` `[Contract]` | event | `OrderId: Guid`, `CustomerId: Guid` | `orders-service` | `customers-service`; `operations-service` (status-only) | none | High |
| `order_delivering` | `OrderDelivering` `[Contract]` | event | `OrderId: Guid` | `orders-service` | `operations-service` (status-only) only | none | High |
| `create_order_rejected` | `CreateOrderRejected` `[Contract]` | rejected | `CustomerId: Guid`, `Reason: string`, `Code: string` — **carries no `OrderId`** | `orders-service` | `operations-service` (status-only) | none | High |
| `add_parcel_to_order_rejected` | `AddParcelToOrderRejected` `[Contract]` | rejected | `OrderId: Guid`, `ParcelId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `delete_parcel_from_order_rejected` | `DeleteParcelFromOrderRejected` `[Contract]` — **implements no interface** | rejected | `OrderId: Guid`, `ParcelId: Guid`, `Reason: string`, `Code: string` | **cannot be published through the rejected-event path** (§10 note 11) | `operations-service` (status-only), bound to a key nothing can publish | none | High |
| `assign_vehicle_to_order_rejected` | `AssignVehicleToOrderRejected` — **no `[Contract]`** | rejected | `OrderId: Guid`, `VehicleId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `approve_order_rejected` | `ApproveOrderRejected` `[Contract]` | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `cancel_order_rejected` | `CancelOrderRejected` `[Contract]` | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `delete_order_rejected` | `DeleteOrderRejected` `[Contract]` | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `delivering_order_rejected` | `DeliveringOrderRejected` `[Contract]` | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `order_for_delivery_not_found` | `OrderForDeliveryNotFound` — **no `[Contract]`** | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `order_for_reserved_vehicle_not_found` | `OrderForReservedVehicleNotFound` — **no `[Contract]`** | rejected | `VehicleId: Guid`, `Date: DateTime`, `Reason: string`, `Code: string` | `orders-service` | `operations-service` (status-only) | none | High |
| `complete_order_rejected` | `CompleteOrderRejected` `[Contract]` | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `orders-service` | **none** — absent from `messages.json` (§10 note 9) | none | High |

**`orders` is the only exchange with two publishers.** `api-gateway` and `ordermaker-service` both
publish commands onto it, and they are the platform's only cross-boundary publishers.

#### 9.3.6 — Physical exchange `parcels`

Evidence: `Pacco.Services.Parcels.Application/{Commands,Events,Events/Rejected}/`; subscriptions
`Parcels.Infrastructure/Extensions.cs:89–95`; catalogue `messages.json:119–133`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `add_parcel` | `AddParcel` | command | `ParcelId: Guid`, `CustomerId: Guid`, `Variant: string`, `Size: string`, `Name: string`, `Description: string` | `api-gateway` (async only) | `parcels-service`; `operations-service` (status-only) | none | High |
| `delete_parcel` | `DeleteParcel` | command | `ParcelId: Guid` | `api-gateway` (async only) | `parcels-service`; `operations-service` (status-only) | none | High |
| `parcel_added` | `ParcelAdded` `[Contract]` | event | `ParcelId: Guid` | `parcels-service` | `operations-service` (status-only) only | none | High |
| `parcel_deleted` | `ParcelDeleted` `[Contract]` | event | `ParcelId: Guid` | `parcels-service` | `operations-service` (status-only) only — **`orders-service` intends to consume this but binds it to the wrong exchange** (§10 note 1) | none | High |
| `add_parcel_rejected` | `AddParcelRejected` `[Contract]` | rejected | `Reason: string`, `Code: string` — **carries no parcel identifier at all** | `parcels-service` | `operations-service` (status-only) | none | High |
| `delete_parcel_rejected` | `DeleteParcelRejected` `[Contract]` | rejected | `ParcelId: Guid`, `Reason: string`, `Code: string` | `parcels-service` | `operations-service` (status-only) | none | High |

`AddParcelRejected` carrying no identifier means a failed `add_parcel` can be correlated back to the
caller **only** through `IMessageProperties.CorrelationId`. That is exactly the field
`operations-service` reads, so the polling path still works, but no other consumer could correlate
it. *Observed* + *inferred*.

#### 9.3.7 — Physical exchange `vehicles`

Evidence: `Pacco.Services.Vehicles.Application/{Commands,Events,Events/Rejected}/`; subscriptions
`Vehicles.Infrastructure/Extensions.cs:86–88`; catalogue `messages.json:134–151`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `add_vehicle` | `AddVehicle` | command | `VehicleId: Guid`, `Brand: string`, `Model: string`, `Description: string`, `PayloadCapacity: double`, `LoadingCapacity: double`, `PricePerService: decimal`, `Variants: Variants` | `api-gateway` (async only) | `vehicles-service`; `operations-service` (status-only) | none | High |
| `update_vehicle` | `UpdateVehicle` | command | `VehicleId: Guid`, `Description: string`, `PricePerService: decimal`, `Variants: Variants` | `api-gateway` (async only) | `vehicles-service`; `operations-service` (status-only) | none | High |
| `delete_vehicle` | `DeleteVehicle` | command | `VehicleId: Guid` | `api-gateway` (async only) | `vehicles-service`; `operations-service` (status-only) | none | High |
| `vehicle_added` | `VehicleAdded` `[Contract]` | event | `VehicleId: Guid` | `vehicles-service` | `operations-service` (status-only) only | none | High |
| `vehicle_updated` | `VehicleUpdated` `[Contract]` | event | `VehicleId: Guid` | `vehicles-service` | `operations-service` (status-only) only | none | High |
| `vehicle_deleted` | `VehicleDeleted` `[Contract]` | event | `VehicleId: Guid` | `vehicles-service` | `availability-service`; `operations-service` (status-only) | none | High |
| `add_vehicle_rejected` | `AddVehicleRejected` `[Contract]` | rejected | `VehicleId: Guid`, `Reason: string`, `Code: string` | `vehicles-service` | `operations-service` (status-only) | none | High |
| `update_vehicle_rejected` | `UpdateVehicleRejected` `[Contract]` | rejected | `VehicleId: Guid`, `Reason: string`, `Code: string` | `vehicles-service` | `operations-service` (status-only) | none | High |
| `delete_vehicle_rejected` | `DeleteVehicleRejected` `[Contract]` | rejected | `VehicleId: Guid`, `Reason: string`, `Code: string` | `vehicles-service` | `operations-service` (status-only) | none | High |

Every vehicle event carries only `VehicleId`, so a consumer that needs the changed values must call
`GET /vehicles/{vehicleId}` (§6 row 5). *Observed*.

#### 9.3.8 — Physical exchange `ordermaker`

Evidence: `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Events/MakeOrderCompleted.cs`,
`.../Events/Rejected/MakeOrderRejected.cs`; catalogue `messages.json:75–83`.

| Physical routing key | Logical type | Kind | Payload fields — *observed* | PUB | CON | Order key | Conf. |
|---|---|---|---|---|---|---|---|
| `make_order_completed` | `MakeOrderCompleted` — **no `[Contract]`** | event | `OrderId: Guid` | `ordermaker-service`, from the saga's `OrderApproved` step | `operations-service` (status-only) only | `OrderId` | High |
| `make_order_rejected` | `MakeOrderRejected` — **no `[Contract]`** | rejected | `OrderId: Guid`, `Reason: string`, `Code: string` | `ordermaker-service` | `operations-service` (status-only) only | `OrderId` | High |

`MakeOrder` — the saga's own start command — is **not** on this exchange. `messages.json` declares no
`commands` array for `ordermaker-service` at all, and `ordermaker-service` registers no
`SubscribeCommand<MakeOrder>()`. **The only way to start the saga is `POST /orders` on port 5015**
(§4.9). *Observed*.

#### 9.3.9 — Physical exchange `operations`

Declared by `operations-service` (`rabbitMq.exchange.name: operations`, `declare: true`) and
**never published to by any code in any repository**. `messages.json` contains no
`operations-service` entry — the catalogue excludes its own owner. Nothing binds a queue to it.
*Observed*, absence across all fourteen clones. See §10 note 5.

**No messages. This exchange carries no contract.**

### 9.4 — Consumer binding matrix

Every `SubscribeCommand<>` / `SubscribeEvent<>` registration in the platform, with the **physical
queue name** each one produces. Queue names are *inferred* by substituting into the observed
template `<service>/{{exchange}}.{{message}}`; the templates themselves are *observed* and are
identical in shape across all nine broker-connected services (`availability-service/…`,
`customers-service/…`, `deliveries-service/…`, `identity-service/…`, `operations-service/…`,
`ordermaker-service/…`, `orders-service/…`, `parcels-service/…`, `vehicles-service/…`). Every queue
is `declare: true`, `durable: true`, `exclusive: false`, `autoDelete: false`.

| Subscriber | Source exchange | Message | Physical queue name |
|---|---|---|---|
| `availability-service` | `availability` | `add_resource` | `availability-service/availability.add_resource` |
| `availability-service` | `availability` | `delete_resource` | `availability-service/availability.delete_resource` |
| `availability-service` | `availability` | `release_resource_reservation` | `availability-service/availability.release_resource_reservation` |
| `availability-service` | `availability` | `reserve_resource` | `availability-service/availability.reserve_resource` |
| `availability-service` | `customers` | `customer_created` | `availability-service/customers.customer_created` |
| `availability-service` | `vehicles` | `vehicle_deleted` | `availability-service/vehicles.vehicle_deleted` |
| `customers-service` | `customers` | `complete_customer_registration` | `customers-service/customers.complete_customer_registration` |
| `customers-service` | `customers` | `change_customer_state` | `customers-service/customers.change_customer_state` |
| `customers-service` | `identity` | `signed_up` | `customers-service/identity.signed_up` |
| `customers-service` | `orders` | `order_completed` | `customers-service/orders.order_completed` |
| `deliveries-service` | `deliveries` | `start_delivery` | `deliveries-service/deliveries.start_delivery` |
| `deliveries-service` | `deliveries` | `complete_delivery` | `deliveries-service/deliveries.complete_delivery` |
| `deliveries-service` | `deliveries` | `fail_delivery` | `deliveries-service/deliveries.fail_delivery` |
| `deliveries-service` | `deliveries` | `add_delivery_registration` | `deliveries-service/deliveries.add_delivery_registration` |
| `identity-service` | `identity` | `sign_up` | `identity-service/identity.sign_up` |
| `ordermaker-service` | `orders` | `order_created` | `ordermaker-service/orders.order_created` |
| `ordermaker-service` | `orders` | `parcel_added_to_order` | `ordermaker-service/orders.parcel_added_to_order` |
| `ordermaker-service` | `orders` | `vehicle_assigned_to_order` | `ordermaker-service/orders.vehicle_assigned_to_order` |
| `ordermaker-service` | `orders` | `order_approved` | `ordermaker-service/orders.order_approved` |
| `ordermaker-service` | `availability` | `resource_reserved` | `ordermaker-service/availability.resource_reserved` |
| `orders-service` | `orders` | `create_order` | `orders-service/orders.create_order` |
| `orders-service` | `orders` | `approve_order` | `orders-service/orders.approve_order` |
| `orders-service` | `orders` | `cancel_order` | `orders-service/orders.cancel_order` |
| `orders-service` | `orders` | `delete_order` | `orders-service/orders.delete_order` |
| `orders-service` | `orders` | `add_parcel_to_order` | `orders-service/orders.add_parcel_to_order` |
| `orders-service` | `orders` | `delete_parcel_from_order` | `orders-service/orders.delete_parcel_from_order` |
| `orders-service` | `orders` | `assign_vehicle_to_order` | `orders-service/orders.assign_vehicle_to_order` |
| `orders-service` | `customers` | `customer_created` | `orders-service/customers.customer_created` |
| `orders-service` | `deliveries` | `delivery_started` | `orders-service/deliveries.delivery_started` |
| `orders-service` | `deliveries` | `delivery_completed` | `orders-service/deliveries.delivery_completed` |
| `orders-service` | `deliveries` | `delivery_failed` | `orders-service/deliveries.delivery_failed` |
| `orders-service` | **`deliveries`** | `parcel_deleted` | `orders-service/deliveries.parcel_deleted` — **dead binding; the event is published on `parcels`** (§10 note 1) |
| `orders-service` | `availability` | `resource_reserved` | `orders-service/availability.resource_reserved` |
| `orders-service` | `availability` | `resource_reservation_canceled` | `orders-service/availability.resource_reservation_canceled` |
| `parcels-service` | `parcels` | `add_parcel` | `parcels-service/parcels.add_parcel` |
| `parcels-service` | `parcels` | `delete_parcel` | `parcels-service/parcels.delete_parcel` |
| `parcels-service` | `orders` | `order_canceled` | `parcels-service/orders.order_canceled` |
| `parcels-service` | `orders` | `order_deleted` | `parcels-service/orders.order_deleted` |
| `parcels-service` | `orders` | `parcel_added_to_order` | `parcels-service/orders.parcel_added_to_order` |
| `parcels-service` | `orders` | `parcel_deleted_from_order` | `parcels-service/orders.parcel_deleted_from_order` |
| `parcels-service` | `customers` | `customer_created` | `parcels-service/customers.customer_created` |
| `vehicles-service` | `vehicles` | `add_vehicle` | `vehicles-service/vehicles.add_vehicle` |
| `vehicles-service` | `vehicles` | `update_vehicle` | `vehicles-service/vehicles.update_vehicle` |
| `vehicles-service` | `vehicles` | `delete_vehicle` | `vehicles-service/vehicles.delete_vehicle` |

44 bindings. `operations-service` adds one binding per catalogued routing key — **80 further queues**
of the form `operations-service/<exchange>.<routing_key>` — generated at startup rather than
declared (§9.5).

### 9.5 — `operations-service` consumes contract-blind, by design

This is the one place where static contract reconstruction genuinely stops, and the reason is
architectural rather than a gap in this analysis.

*Observed*, `Operations.Api/Infrastructure/Subscriptions.cs`: the service reads `messages.json`,
then uses `AssemblyBuilder.DefineDynamicAssembly` and
`ModuleBuilder.DefineType(message, TypeAttributes.Public, base)` to **emit a field-less subclass of
`Command`, `Event`, or `RejectedEvent` for every routing key in the catalogue**, attaching a
`CustomAttributeBuilder` for `MessageAttribute(exchange, null, null, true)`, and subscribes to each
reflectively.

The three base types are *observed* to be almost empty: `Types/Event.cs` and `Types/Command.cs`
declare **no members at all**, and `Types/RejectedEvent.cs` declares only `Reason: string` and
`Code: string`.

**Consequence, stated precisely.** `operations-service` receives all 80 messages but deserialises
**no payload field** from commands or events, and only `Reason` and `Code` from rejected events.
What it actually reads is transport metadata: `IMessageProperties.CorrelationId` (skipped when
blank) and the `Saga` header, mapped by `Extensions.GetSagaState` to `Pending` / `Completed` /
`Rejected`. Where no `Saga` header is present the default is state by message kind — command →
`Pending`, event → `Completed`, rejected event → `Rejected`. *Observed*,
`Operations.Api/Handlers/`.

**This is why the *Consumers* column writes `operations-service (status-only)` everywhere.** It is
also why nothing further about its inbound payloads can be established from source: the payload
never becomes a typed value in this service. That is not a discovery gap to be filled by more
reading — it is the observed design (Q3).

`operations-service` writes each record to Redis at `requests:{id}` with
`SlidingExpiration = RequestsOptions.ExpirySeconds` (300 s), and **terminal states are immutable** —
`TrySetAsync` returns `(false, operation)` once the state is `Completed` or `Rejected`. *Observed*,
`Operations.Api/Services/OperationsService.cs`.

### 9.6 — Saga message flow — `ordermaker-service`

*Observed*, `Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`.
`Saga<AIMakingOrderData>` implementing `ISagaStartAction<MakeOrder>` and `ISagaAction<T>` for
`OrderCreated`, `ParcelAddedToOrder`, `VehicleAssignedToOrder`, `OrderApproved`. `ResolveId` returns
`OrderId.ToString()` for **every** message, so the saga correlation key is `OrderId` throughout.
Every publication carries `messageContext: _accessor.CorrelationContext` and the header
`Saga` set to `Pending`, `Completed`, or `Rejected`.

| Step | Trigger | Publishes | Target exchange | `Saga` header | Line |
|---|---|---|---|---|---|
| 1 | `MakeOrder` (HTTP `POST /orders` on 5015) | `CreateOrder(OrderId, CustomerId)` | `orders` | `Pending` | `:63` |
| 2 | `OrderCreated` | `AddParcelToOrder(OrderId, parcelId, CustomerId)` | `orders` | `Pending` | `:74` |
| 3 | `ParcelAddedToOrder` | first `GET /vehicles` then `GET /resources/{resourceId}` over HTTP (§6 rows 6–7), then `AssignVehicleToOrder(OrderId, VehicleId, ReservationDate)` | `orders` | `Pending` | `:103–104` |
| 4 | `VehicleAssignedToOrder` | `ReserveResource(VehicleId, CustomerId, Date, Priority)` | `availability` | `Pending` | `:113` |
| 5 | `OrderApproved` | `MakeOrderCompleted(OrderId)`, then `CompleteAsync()` | `ordermaker` | `Completed` | `:124` |
| Compensation | `ParcelAddedToOrder` fails | `CancelOrder(OrderId, "Because I'm saga")` | `orders` | `Rejected` | `:141` |

**Step 5 has no trigger.** `OrderApproved` is published only by `orders-service`'s
`ApproveOrderHandler`, which runs only on the `approve_order` command, which **nothing in the
workspace publishes** — no gateway route in either configuration and no saga step (§10 note 10).
The saga therefore reaches step 4 and stops. *Observed* from the publisher census plus the handler
chain.

**Step 3 mixes transports inside one saga step**, making two synchronous HTTP calls between two
broker publications. A failure of either HTTP call is not a broker message and does not carry the
`Saga` header. *Observed*.

**`ordermaker-service` sets broker retries explicitly** — `rabbitMq.retries: 3`,
`retryInterval: 2` (*observed*, `Pacco.Services.OrderMaker/appsettings.json`), the only service that
does.

### 9.7 — Contract statements (Part 3)

`availability-service` owns the topic exchange `availability`.
`customers-service` owns the topic exchange `customers`.
`deliveries-service` owns the topic exchange `deliveries`.
`identity-service` owns the topic exchange `identity`.
`operations-service` owns the topic exchange `operations`.
`ordermaker-service` owns the topic exchange `ordermaker`.
`orders-service` owns the topic exchange `orders`.
`parcels-service` owns the topic exchange `parcels`.
`vehicles-service` owns the topic exchange `vehicles`.
`pricing-service` owns no exchange and publishes no messages.
`availability-service` publishes `resource_added` events to topic exchange `availability`.
`availability-service` publishes `resource_deleted` events to topic exchange `availability`.
`availability-service` publishes `resource_reserved` events to topic exchange `availability`.
`availability-service` publishes `resource_reservation_released` events to topic exchange `availability`.
`availability-service` publishes `resource_reservation_canceled` events to topic exchange `availability`.
`availability-service` publishes `add_resource_rejected` events to topic exchange `availability`.
`availability-service` publishes `delete_resource_rejected` events to topic exchange `availability`.
`availability-service` publishes `release_resource_rejected` events to topic exchange `availability`.
`availability-service` publishes `release_resource_reservation_rejected` events to topic exchange `availability`.
`customers-service` publishes `customer_created` events to topic exchange `customers`.
`customers-service` publishes `customer_became_vip` events to topic exchange `customers`.
`customers-service` publishes `customer_state_changed` events to topic exchange `customers`.
`customers-service` publishes `complete_customer_registration_rejected` events to topic exchange `customers`.
`customers-service` publishes `change_customer_state_rejected` events to topic exchange `customers`.
`deliveries-service` publishes `delivery_started` events to topic exchange `deliveries`.
`deliveries-service` publishes `delivery_completed` events to topic exchange `deliveries`.
`deliveries-service` publishes `delivery_failed` events to topic exchange `deliveries`.
`deliveries-service` publishes `registration_added_to_delivery` events to topic exchange `deliveries`.
`identity-service` publishes `signed_up` events to topic exchange `identity`.
`identity-service` publishes `signed_in` events to topic exchange `identity`.
`orders-service` publishes `order_created` events to topic exchange `orders`.
`orders-service` publishes `order_approved` events to topic exchange `orders`.
`orders-service` publishes `order_canceled` events to topic exchange `orders`.
`orders-service` publishes `order_completed` events to topic exchange `orders`.
`orders-service` publishes `order_deleted` events to topic exchange `orders`.
`orders-service` publishes `order_delivering` events to topic exchange `orders`.
`orders-service` publishes `parcel_added_to_order` events to topic exchange `orders`.
`orders-service` publishes `parcel_deleted_from_order` events to topic exchange `orders`.
`orders-service` publishes `vehicle_assigned_to_order` events to topic exchange `orders`.
`parcels-service` publishes `parcel_added` events to topic exchange `parcels`.
`parcels-service` publishes `parcel_deleted` events to topic exchange `parcels`.
`vehicles-service` publishes `vehicle_added` events to topic exchange `vehicles`.
`vehicles-service` publishes `vehicle_updated` events to topic exchange `vehicles`.
`vehicles-service` publishes `vehicle_deleted` events to topic exchange `vehicles`.
`ordermaker-service` publishes `make_order_completed` events to topic exchange `ordermaker`.
`ordermaker-service` publishes `make_order_rejected` events to topic exchange `ordermaker`.
`api-gateway` publishes `add_resource` commands to topic exchange `availability`.
`api-gateway` publishes `reserve_resource` commands to topic exchange `availability`.
`api-gateway` publishes `release_resource` commands to topic exchange `availability`.
`api-gateway` publishes `delete_resource` commands to topic exchange `availability`.
`api-gateway` publishes `complete_customer_registration` commands to topic exchange `customers`.
`api-gateway` publishes `change_customer_state` commands to topic exchange `customers`.
`api-gateway` publishes `start_delivery` commands to topic exchange `deliveries`.
`api-gateway` publishes `complete_delivery` commands to topic exchange `deliveries`.
`api-gateway` publishes `fail_delivery` commands to topic exchange `deliveries`.
`api-gateway` publishes `add_delivery_registration` commands to topic exchange `deliveries`.
`api-gateway` publishes `create_order` commands to topic exchange `orders`.
`api-gateway` publishes `delete_order` commands to topic exchange `orders`.
`api-gateway` publishes `add_parcel_to_order` commands to topic exchange `orders`.
`api-gateway` publishes `delete_parcel_from_order` commands to topic exchange `orders`.
`api-gateway` publishes `assign_vehicle_to_order` commands to topic exchange `orders`.
`api-gateway` publishes `add_parcel` commands to topic exchange `parcels`.
`api-gateway` publishes `delete_parcel` commands to topic exchange `parcels`.
`api-gateway` publishes `add_vehicle` commands to topic exchange `vehicles`.
`api-gateway` publishes `update_vehicle` commands to topic exchange `vehicles`.
`api-gateway` publishes `delete_vehicle` commands to topic exchange `vehicles`.
`ordermaker-service` publishes `create_order` commands to topic exchange `orders`.
`ordermaker-service` publishes `add_parcel_to_order` commands to topic exchange `orders`.
`ordermaker-service` publishes `assign_vehicle_to_order` commands to topic exchange `orders`.
`ordermaker-service` publishes `cancel_order` commands to topic exchange `orders`.
`ordermaker-service` publishes `reserve_resource` commands to topic exchange `availability`.
`availability-service` consumes `add_resource` commands from topic exchange `availability`.
`availability-service` consumes `delete_resource` commands from topic exchange `availability`.
`availability-service` consumes `release_resource_reservation` commands from topic exchange `availability`.
`availability-service` consumes `reserve_resource` commands from topic exchange `availability`.
`availability-service` consumes `customer_created` events from topic exchange `customers`.
`availability-service` consumes `vehicle_deleted` events from topic exchange `vehicles`.
`customers-service` consumes `complete_customer_registration` commands from topic exchange `customers`.
`customers-service` consumes `change_customer_state` commands from topic exchange `customers`.
`customers-service` consumes `signed_up` events from topic exchange `identity`.
`customers-service` consumes `order_completed` events from topic exchange `orders`.
`deliveries-service` consumes `start_delivery` commands from topic exchange `deliveries`.
`deliveries-service` consumes `complete_delivery` commands from topic exchange `deliveries`.
`deliveries-service` consumes `fail_delivery` commands from topic exchange `deliveries`.
`deliveries-service` consumes `add_delivery_registration` commands from topic exchange `deliveries`.
`identity-service` consumes `sign_up` commands from topic exchange `identity`.
`orders-service` consumes `create_order` commands from topic exchange `orders`.
`orders-service` consumes `approve_order` commands from topic exchange `orders`.
`orders-service` consumes `cancel_order` commands from topic exchange `orders`.
`orders-service` consumes `delete_order` commands from topic exchange `orders`.
`orders-service` consumes `add_parcel_to_order` commands from topic exchange `orders`.
`orders-service` consumes `delete_parcel_from_order` commands from topic exchange `orders`.
`orders-service` consumes `assign_vehicle_to_order` commands from topic exchange `orders`.
`orders-service` consumes `customer_created` events from topic exchange `customers`.
`orders-service` consumes `delivery_started` events from topic exchange `deliveries`.
`orders-service` consumes `delivery_completed` events from topic exchange `deliveries`.
`orders-service` consumes `delivery_failed` events from topic exchange `deliveries`.
`orders-service` consumes `resource_reserved` events from topic exchange `availability`.
`orders-service` consumes `resource_reservation_canceled` events from topic exchange `availability`.
`parcels-service` consumes `add_parcel` commands from topic exchange `parcels`.
`parcels-service` consumes `delete_parcel` commands from topic exchange `parcels`.
`parcels-service` consumes `order_canceled` events from topic exchange `orders`.
`parcels-service` consumes `order_deleted` events from topic exchange `orders`.
`parcels-service` consumes `parcel_added_to_order` events from topic exchange `orders`.
`parcels-service` consumes `parcel_deleted_from_order` events from topic exchange `orders`.
`parcels-service` consumes `customer_created` events from topic exchange `customers`.
`vehicles-service` consumes `add_vehicle` commands from topic exchange `vehicles`.
`vehicles-service` consumes `update_vehicle` commands from topic exchange `vehicles`.
`vehicles-service` consumes `delete_vehicle` commands from topic exchange `vehicles`.
`ordermaker-service` consumes `order_created` events from topic exchange `orders`.
`ordermaker-service` consumes `parcel_added_to_order` events from topic exchange `orders`.
`ordermaker-service` consumes `vehicle_assigned_to_order` events from topic exchange `orders`.
`ordermaker-service` consumes `order_approved` events from topic exchange `orders`.
`ordermaker-service` consumes `resource_reserved` events from topic exchange `availability`.
`operations-service` consumes every catalogued routing key from all eight traffic-carrying exchanges.
`orders-service` binds a queue for `parcel_deleted` to topic exchange `deliveries`, which never carries that event.
`ordermaker-service` runs the saga `AIOrderMakingSaga` correlated by `OrderId`.

## 10 — Conflicts between sources

**Documentation cross-check.** Every contract-bearing claim in the three prior discovery documents in
this repository was checked against the source, and **no conflict was found**. Specifically:
`architecture-baseline.md:102` ("eight topic exchanges, one owned per service"),
`:391` ("roughly 80 distinct messages across eight topic exchanges that carry traffic"),
`:396–402` (the `operations` exchange declared but never published to), `:411–413` (the queue
template producing one queue per subscriber/exchange/message triple), `:416` (snake-case routing
keys), `:425–429` (`ordermaker-service` as the only cross-exchange publisher), `:454–466` (twenty
write routes converted to messages in the asynchronous configuration, twenty reads proxied),
`:497–502` (SignalR `/pacco` with a Redis backplane; gRPC `GetOperation` unary and
`SubscribeOperations` streaming) and `:544–547` (the saga step table) all match what the code shows.
`capability-baseline.md`'s capability-to-service ownership matches the endpoint and exchange
ownership recorded here. The conflicts below are therefore **code-versus-code and
code-versus-configuration**, not documentation errors.

**Note 1 — `orders-service` binds `parcel_deleted` to the wrong exchange.**
*Doc/config claim:* `orders-service` consumes parcel deletions.
*Code shows:*
`Pacco.Services.Orders.Application/Events/External/ParcelDeleted.cs` declares
`[Message("deliveries")] public class ParcelDeleted : IEvent { Guid ParcelId }`, and
`Orders.Infrastructure/Extensions.cs:106` calls `SubscribeEvent<ParcelDeleted>()`.
`parcels-service` publishes `ParcelDeleted` on the **`parcels`** exchange
(`Parcels.Application/Events/ParcelDeleted.cs`, `messages.json:119–128`). `deliveries-service`
publishes no `parcel_deleted`, and its four events are all delivery events.
*Truth:* the code wins — the binding is on `deliveries`. The queue
`orders-service/deliveries.parcel_deleted` is declared and consumes nothing; **`orders-service` never
learns that a parcel it holds has been deleted.** Every neighbouring external event in the same
folder (`DeliveryStarted`, `DeliveryCompleted`, `DeliveryFailed`) legitimately carries
`[Message("deliveries")]`, which is consistent with a copy-paste of the attribute.
Confidence: **High**. → GAP-9, Q7.

**Note 2 — The gateway's two `payload:`/`schema:` references point at files that do not exist.**
*Config claims:* `ntrada.yml` `modules.customers` declares `payload: create_customer` and
`schema: create_customer.schema`; `ntrada-async.yml` declares
`payload: complete_customer_registration` and `schema: complete_customer_registration.schema`.
*Repository shows:* a full listing of `hianshul100_Pacco.APIGateway` contains **no `Payloads/`
directory, no `*.schema` file, and no `*.json` file other than `appsettings.json`**. No JSON Schema
file exists in any of the fourteen clones.
*Truth:* the request-validation contract for `POST /customers` is **[unknown]** in both
configurations. Whether Ntrada fails startup, logs a warning, or silently skips validation depends
on package code outside this workspace.
Confidence: **High** for the absence, **[unknown]** for the effect. → GAP-2, Q8.

**Note 3 — `pacco-web` contains no code.**
*Repository shows:* `hianshul100_Pacco.Web` contains exactly one file, `README.md`, whose entire
content is `# Pacco.Web`.
*Truth:* every browser-facing contract in this document — the SignalR hub `/pacco`, the CORS policy,
the `Resource-ID` header, the operation-polling pattern — has **no observable consumer**. Those
consumer sides are **Unverifiable — Missing Source Evidence**, not absent.
Confidence: **High**. → GAP-5, Q9.

**Note 4 — `orders-service`'s `DeliveryStarted` declares fewer fields than the publisher sends.**
*Code shows:* `deliveries-service` publishes `DeliveryStarted { DeliveryId: Guid, OrderId: Guid }`;
`Orders.Application/Events/External/DeliveryStarted.cs` declares `{ OrderId: Guid }` only.
*Truth:* JSON deserialisation tolerates this, so it is a working consumer-driven projection rather
than a break — the same pattern as the HTTP clients in §6. It is recorded because the two
declarations of one contract now drift independently.
Confidence: **High**.

**Note 5 — The `operations` exchange is declared and never used.**
*Config shows:* `Operations.Api/appsettings.json` sets `rabbitMq.exchange.name: operations`,
`declare: true`. *Code shows:* no publication to it anywhere; `messages.json` has no
`operations-service` entry; nothing binds a queue to it.
*Truth:* the exchange exists at runtime and carries nothing.
Confidence: **High**.

**Note 6 — The entire refresh-token lifecycle is unreachable through the gateway.**
*Code shows:* `identity-service` exposes `POST /access-tokens/revoke`, `POST /refresh-tokens/use`
and `POST /refresh-tokens/revoke` (`Identity.Api/Program.cs`), and
`Pacco.Services.Identity.rest` exercises all three directly on port 5004.
*Config shows:* neither `ntrada.yml` nor `ntrada-async.yml` declares any route for them.
*Truth:* with `jwt.expiryMinutes: 60`, a client that can only reach the gateway must sign in again
every hour; the `refreshToken` in `AuthDto` is issued but cannot be redeemed at the edge.
Confidence: **High**. → GAP-3, Q10.

**Note 7 — Two `operations-service` read surfaces are unauthenticated while the data is per-user.**
*Code and config show:* `ntrada.yml` and `ntrada-async.yml` both set `auth: false` on
`GET /operations/{operationId}`; `OperationDto` carries `UserId`; the gRPC service registers no
interceptor and `SubscribeOperations` takes `Empty`, streaming every operation regardless of owner;
the SignalR hub is reached directly because no gateway route proxies `/pacco`.
*Truth:* anyone who knows or guesses an operation id can read its `userId`, `name`, `code` and
`reason` over HTTP, and anyone who can reach port 50050 can stream all of them.
Confidence: **High**. → Q11.

**Note 8 — Edge policy gaps that the route tables make visible.**
*Config shows:* `cors.allowedOrigins: *` combined with bearer-token auth; exactly five routes carry
`claims: role: admin`, none of them on `vehicles`, so any authenticated customer can `POST`, `PUT`
or `DELETE` fleet vehicles; `cors.allowedMethods` lists `post, put, delete` but not `get`.
*Truth:* stated as observed configuration. Whether each is intended is a decision, not a reading.
Confidence: **High** for the configuration. → Q12.

**Note 9 — `messages.json` and the code disagree on five routing keys.**
A mechanical diff of every snake-cased type name against the catalogue gives exactly five
divergences:

| Divergence | Code | `messages.json` | Effect |
|---|---|---|---|
| `release_resource` | no type exists | declared as an `availability` command; **published by `api-gateway` in the asynchronous configuration** | `DELETE /availability/resources/{resourceId}/reservations/{dateTime}` publishes a message no service consumes |
| `release_resource_reservation` | `ReleaseResourceReservation`, subscribed by `availability-service` | absent | the live command is invisible to `operations-service` |
| `release_resource_reservation_rejected` | `ReleaseResourceReservationRejected`, published on **`ReserveResource` failure** | absent | a failed reservation never reaches `operations-service`, so the caller polls `Pending` until expiry |
| `reserve_resource_rejected` | no type exists | declared | `operations-service` binds a queue nothing publishes to |
| `add_delivery_registration_rejected`, `complete_order_rejected` | both exist and are published | absent | those two failures never surface through `GET /operations/{operationId}` |

*Truth:* the code wins in every row. `messages.json` is a hand-maintained catalogue that has drifted
from the types it describes.
Confidence: **High**. → GAP-10, B1.

**Note 10 — The `ordermaker-service` saga has no path to completion.**
*Code shows:* the saga's final step handles `OrderApproved`
(`AIOrderMakingSaga.cs:124`). `OrderApproved` is emitted only by `orders-service`'s `EventMapper`
from `ApproveOrderHandler` (`Orders.Infrastructure/Services/EventMapper.cs:27`), which runs only on
the `approve_order` command. A publisher census across all fourteen clones finds **no publisher of
`approve_order`**: no gateway route in either configuration, and the saga publishes `CreateOrder`,
`AddParcelToOrder`, `AssignVehicleToOrder`, `ReserveResource`, `MakeOrderCompleted` and
`CancelOrder` but never `ApproveOrder`.
*Truth:* the saga advances to step 4 and stops; `make_order_completed` cannot be published from
inside the platform, and `CompleteAsync()` is never reached.
Confidence: **High**. → GAP-11, B2.

**Note 11 — `DeleteParcelFromOrderRejected` implements no interface.**
*Code shows:* nine of the ten `orders-service` rejected events declare
`: IRejectedEvent`; `Events/Rejected/DeleteParcelFromOrderRejected.cs` declares
`[Contract] public class DeleteParcelFromOrderRejected` with no base type or interface, while
carrying the same four fields as its siblings.
*Truth:* it cannot flow through the `IRejectedEvent` publication path, so the catalogued routing key
`delete_parcel_from_order_rejected` has a bound queue and no possible publisher. A failed
`DELETE /orders/{orderId}/parcels/{parcelId}` never surfaces as `Rejected`.
Confidence: **High**. → GAP-10.

**Note 12 — Committed development credentials are present and are not reproduced here.**
`identity-service` and `operations-service` `appsettings.json`, both `ntrada*.yml` files, and every
service's `vault` and `rabbitMq` sections contain plaintext values including a JWT signing key, a
Vault token, and broker credentials. They are named by **file path only** in this document and their
values are deliberately not quoted anywhere in it.
Confidence: **High**. → B3.

## 11 — Gaps

Contracts, or parts of contracts, that could not be established from the available evidence. Each
row states what is missing, what it stops this inventory from asserting, and where it is carried
forward in §12. **Every row here appears in §12; nothing is left only in this table.**

| ID | Gap | What it prevents | Why it could not be closed here | Carried forward |
|----|-----|------------------|---------------------------------|-----------------|
| GAP-1 | The acknowledgement status code and body returned by the 20 broker-published gateway routes | Stating the client-visible response contract of every write operation in the asynchronous configuration | The response is produced inside `Ntrada.Extensions.RabbitMq 0.4.*`; no source, spec, or fixture in the workspace records it | Q2 |
| GAP-2 | The request-validation contract for `POST /customers` | Stating which fields are required or rejected on the one route that declares a schema | Both `payload:` and `schema:` files referenced by the gateway are absent from the repository (§10 note 2) | Q8 |
| GAP-3 | Any edge route for the three `identity-service` token-lifecycle endpoints | Describing how a gateway-only client refreshes a 60-minute token | The routes simply do not exist in either configuration (§10 note 6) | Q10 |
| GAP-4 | Any gateway route proxying the SignalR hub `/pacco` | Describing how a browser reaches real-time notifications through the edge | No `signalr`, `pacco`, or WebSocket route exists in either `ntrada*.yml` | Q9 |
| GAP-5 | Any consumer of the SignalR messages, the CORS policy, the `Resource-ID` header or the operation-polling pattern | Confirming that the client-facing half of those contracts is used as designed | `pacco-web` contains only a one-line `README.md` (§10 note 3) — **Unverifiable — Missing Source Evidence** | Q9 |
| GAP-6 | Any producer of the `Total-Count` response header | Stating whether paging metadata reaches clients at all | The header is declared in `cors.exposedHeaders` and set by nothing in any repository; the one paged endpoint has its envelope stripped by the gateway (§2.9) | Q14 |
| GAP-7 | The route path registered by `UsePublicContracts<ContractAttribute>()` | Naming the endpoint on which 70-plus `[Contract]` types are published | The call site is observed in every service; the route constant lives in `Convey.WebApi.CQRS 0.4.*` and is stated nowhere in the workspace | Q15 |
| GAP-8 | Any consumer of `customer_state_changed` or `customer_became_vip` | Explaining how the three services holding local customer copies learn of changes | No subscription to either event exists anywhere; the absence is the finding (§9.3.2) | Q13 |
| GAP-9 | A working consumer for `parcel_deleted` in `orders-service` | Asserting that `orders-service` reacts to parcel deletion at all | The subscription exists but is bound to the `deliveries` exchange, which never carries the event (§10 note 1) | Q7 |
| GAP-10 | Reliable rejection reporting for five operations | Asserting that `GET /operations/{operationId}` reports every failure | Five routing-key divergences plus one rejected event that implements no interface make those failures unpublishable or uncatalogued (§10 notes 9 and 11) | B1 |
| GAP-11 | Any publisher of `approve_order` | Asserting that the automated order saga can complete | A publisher census across all fourteen clones finds none (§10 note 10) | B2 |
| GAP-12 | Any committed API specification, contract registry entry, versioning policy, or pagination policy | Stating a platform-wide API standard of any kind | No OpenAPI document, Postman collection, JSON Schema, or written interface standard exists in any repository; Swagger is generated at runtime and never captured | Q4 |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything in this baseline that is not directly proven by code or configuration is collected
> here. Each assumption states what was taken as true and what breaks if it is wrong. Each blocker
> and open question is tagged either **[ACTION NOW]** — someone must answer it before the next
> stage can rely on this document — or **[handled later by \<stage\>]**, meaning a named later
> stage will resolve it and no action is needed now.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|----|------------|-----------|-----------------|-----------------|
| A1 | The snake-case names in `messages.json` are the actual strings on the wire, and a message type not listed there still travels under the snake-case form of its class name | Every service sets `rabbitMq.conventionsCasing: snakeCase`, and 75 of the 80 catalogued names match the snake-case form of a real class exactly. Nothing else in the workspace states a wire name | Every physical routing key and every physical queue name in §9 would be wrong, and so would the five divergences in §10 note 9 | Read the exchange bindings on a running broker and compare the key strings |
| A2 | Physical queue names are the observed template `<service>/{{exchange}}.{{message}}` with the two placeholders filled by the source exchange and the routing key | The template is observed identically in all nine broker-connected services; the placeholders have no other plausible filling | The 44 queue names in §9.4 would be wrong, though the bindings they represent would still be real | List queues on a running broker |
| A3 | Convey returns `200` for a command endpoint with no `afterDispatch`, and `404` for a query whose handler returns null | Two services (`identity-service`, `operations-service`) write those codes explicitly, which is consistent with the framework doing the same for the other eight | Roughly 15 response rows in §2 and §4 would carry the wrong status code; routes, verbs and payloads would be unaffected | Call any service endpoint directly and read the status line, or read `Convey.WebApi.CQRS 0.4.*` source |
| A4 | Field names are camel-cased on the wire even though this document lists them in the C# `PascalCase` form of the declaring type | No service overrides the serialiser naming policy, and the `.rest` fixtures send camel-cased JSON (`customerId`, `orderPrice`, `parcelIds`) | Every request and response body in Parts 1 and 3 would need re-casing; the field sets themselves would be unchanged | Compare a captured request body against the fixtures |
| A5 | The 12 `.rest` fixtures reflect request shapes that actually worked against these services | They are checked in beside the code they exercise, use consistent variable chaining across files, and agree with the command and query types in every case checked | A handful of query-string encodings — notably the JSON-array-literal form `parcelIds=["<guid>"]` — would be wrong | Replay the fixtures against a running environment |
| A6 | `messages.json` governs only what `operations-service` subscribes to, and is not read by any publisher | It lives inside `operations-service` and is loaded only by `Operations.Api/Infrastructure/Subscriptions.cs`; no other project references it | The five divergences in §10 note 9 would affect publishing too, widening their blast radius considerably | Search the deployed images for other readers of the file, or confirm with the platform owner |

### Blockers

**On the Owner column.** No repository in this workspace records a named owner for anything — there
is no `CODEOWNERS`, `CONTRIBUTING.md`, or team manifest in any of the fourteen clones. The Owner
column therefore names the **role that must supply the answer**, and assigning a person to that role
is itself part of resolving the blocker. Target dates are **proposed** by this stage relative to the
analysis date of 2026-09-04; none is an agreed commitment.

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|----|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Five message names in `Operations.Api/messages.json` disagree with the code, and one rejected event cannot be published at all. Because of this, at least five kinds of failure never reach the operation record a caller polls — a failed reservation, a failed delivery-registration, a failed order completion, a failed parcel removal from an order, and anything the availability error mapper does not cover. The caller sees `Pending` until the record expires after five minutes and then sees nothing | GAP-10; the claim in §2.5 and §3 that `GET /operations/{operationId}` is a complete status channel; any later stage that treats the asynchronous write path as observable | Platform owner for the Pacco runtime, with the `availability-service`, `deliveries-service` and `orders-service` owners | Decide whether `messages.json` or the code is meant to be authoritative, reconcile the five names, make `DeleteParcelFromOrderRejected` implement `IRejectedEvent`, and confirm against the bindings on a running broker | 2026-09-11 (proposed) |
| B2 | **[ACTION NOW]** The automated order saga in `ordermaker-service` has no path to completion. Its last step waits for `order_approved`, which is only ever published in response to an `approve_order` command, and nothing anywhere publishes that command | GAP-11; the `ordermaker` exchange contract in §9.3.8; the saga flow in §9.6; any later stage that treats CAP-10 as a working capability | Platform owner for the Pacco runtime, with the `ordermaker-service` author | Confirm whether an approval trigger exists outside these repositories. If not, decide whether the saga is unfinished work or dead code, and record which — the answer changes whether four other exchanges have a live inbound writer | 2026-09-11 (proposed) |
| B3 | **[ACTION NOW]** A token signing key, a Vault token, and broker credentials are committed in plaintext in `identity-service`, `operations-service`, both gateway configurations, and every service's `vault` section | Any security assessment of the bearer-token contract in §7.2; the trustworthiness of every `Auth` column in Parts 1 and 3 | Security owner for the Pacco platform | Confirm whether the committed values are throwaway local-development credentials. If they are not, rotate them and purge them from history. This document names the files but reproduces no value | 2026-09-08 (proposed) — treat as immediate if the values are live |
| B4 | **[ACTION NOW]** Nothing states which gateway configuration production runs. The compose stack defaults to the asynchronous one, the image defaults to the synchronous one, and they are behaviourally different systems | Whether §2 or §3 is the real edge contract; whether 20 write operations return a domain response or an acknowledgement; whether the operation-polling path in §2.5 is on the critical path at all | Platform owner for the Pacco runtime | Read the effective `NTRADA_CONFIG` value in each running environment and record which `ntrada*.yml` is authoritative per environment | 2026-09-11 (proposed) |

### Open Questions

**On the Decision Owner column.** As with the Blockers table, no named individual is recorded
anywhere in the workspace. Decision Owner names the **role empowered to settle the question**;
naming a person to it is part of answering. A *Proposed Answer* is this stage's best reading of the
evidence, offered so the owner can confirm or reject rather than start from nothing — it is **not**
a finding, and nothing elsewhere in this inventory rests on it.

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|----|----------|----------------|--------------------------|----------------|
| Q1 | **[handled later by runtime and deployment validation]** What status codes does the framework actually return for command endpoints, and does `GET /parcels/volume` reach its handler or get captured by the `{parcelId}` route ahead of it in the gateway configuration? | Roughly 15 response rows in §2 and §4 depend on the first answer; the second decides whether a working endpoint is reachable through the edge | Likely `200` for commands, and likely that Ntrada matches literal segments before templates — but both live in package code that is not in this workspace, so neither is asserted. The service itself registers `volume` first, which would be pointless if order did not matter | Runtime and deployment validation stage |
| Q2 | **[handled later by runtime and deployment validation]** What exactly does a caller receive from the 20 broker-published gateway routes — which status code, and what in the body? | This is the client-visible contract of every write operation whenever the asynchronous configuration is live. It is currently the largest single unknown in Part 1 | Almost certainly a `202` plus a correlation identifier, since `operations-service` reads `CorrelationId` on every message and the polling endpoint exists for exactly this purpose. The specific shape is not asserted | Runtime and deployment validation stage |
| Q3 | **[handled later by runtime and deployment validation]** What do the message bodies `operations-service` receives actually look like? | Its inbound contract is the one surface in this document that static reading genuinely cannot reach | Partial and probably sufficient: the bodies are whatever each publisher serialises, and `operations-service` reads only the correlation id and the `Saga` header, so the body may not matter to it at all. That reading is unverified | Runtime and deployment validation stage |
| Q4 | **[ACTION NOW]** Is there an API standard for this platform — versioning, pagination, media types, error format — recorded anywhere outside these repositories? | GAP-12. Without one, this inventory can describe what the endpoints do but cannot say whether any of it is intended. It also means nothing detects a breaking change | None, and the absence looks real rather than a retrieval failure: no route carries a version, the one paged response has its envelope discarded at the edge, and the single consistent convention — `{code, reason}` with status `400` — is a framework side effect rather than a written rule | Platform architect for Pacco |
| Q5 | **[ACTION NOW]** Are the ten service ports reachable from anywhere other than the gateway? | Every authentication and ownership check in the platform sits at the gateway. If the service ports are reachable, §7.2's whole table is advisory rather than enforced | Probably not in a real deployment, since the compose stack puts services on an internal network — but nothing in the workspace states a network policy, and every repository ships a `.rest` fixture that calls its own port directly | Platform owner for the Pacco runtime |
| Q6 | **[handled later by security review]** Is the certificate access list in `customers-service` enforced, and which routes does the `customers:read` permission cover? | Decides whether the one service-to-service authentication control in the platform is real, and whether it protects `GET /customers/{customerId}/state` specifically | Probably enforced: the ACL names `availability-service` explicitly, and `availability-service` attaches a Vault certificate on exactly the one call it makes, which would be pointless otherwise. The route mapping is genuinely unknown — the ACL grants a permission, not a path | Security review stage |
| Q7 | **[ACTION NOW]** Is `orders-service` supposed to react when a parcel is deleted? Today it subscribes to `parcel_deleted` on the `deliveries` exchange, where that event never appears | GAP-9. An order can keep referencing a parcel that no longer exists, and no failure is raised anywhere | Almost certainly a copy-paste of the `[Message("deliveries")]` attribute from the three delivery events in the same folder. That is the mechanism; whether the subscription is wanted at all is the decision | Platform owner, with the `orders-service` owner |
| Q8 | **[ACTION NOW]** What should validate the `POST /customers` request body? The gateway names a payload file and a schema file, and neither exists | GAP-2. This is the only route in the platform that declares request validation, so its absence means nothing validates any request body at the edge | None. Whether Ntrada fails startup, warns, or silently skips validation when the files are missing cannot be determined from this workspace, and the intended schema contents are unrecoverable | Platform owner, with the `api-gateway` owner |
| Q9 | **[ACTION NOW]** Is there a client for this platform outside these repositories? | GAP-4 and GAP-5. The SignalR hub, the CORS policy, the `Resource-ID` header and the whole operation-polling pattern exist for a client that is not in the workspace | Likely yes, or likely intended: `pacco-web` is an empty placeholder repository rather than a deleted one, and four distinct server-side features only make sense with a browser client. Its absence here is a scope fact, not evidence that no client exists | Platform owner for the Pacco runtime |
| Q10 | **[ACTION NOW]** How is a client meant to refresh an expiring token? The three token-lifecycle endpoints exist on `identity-service` and have no gateway route | GAP-3. With a 60-minute token and no refresh route at the edge, either clients re-authenticate hourly or they reach `identity-service` directly, which contradicts Q5 | Likely an oversight: `sign-in` returns a `refreshToken` that has no redeemable endpoint at the edge, which is a self-inconsistent contract rather than a deliberate design | Platform owner, with the `identity-service` owner |
| Q11 | **[handled later by security review]** Should the operation-status surfaces be unauthenticated? `GET /operations/{operationId}` is `auth: false`, the gRPC port has no interceptor, and `SubscribeOperations` streams every operation to any caller | Operation records carry `userId`, `code` and `reason`. Anyone who can guess an id reads one; anyone who can reach port 50050 reads all of them | Partly deliberate, partly not: the HTTP route arguably needs to be anonymous so a caller can poll an operation created before sign-in completes, but nothing explains the unauthenticated streaming method | Security review stage |
| Q12 | **[ACTION NOW]** Are the two edge policy gaps intended — a wildcard CORS origin alongside bearer tokens, and fleet mutation routes with no `admin` claim while four customer read routes have one? | Any authenticated customer can add, update or delete vehicles. The five routes that do carry the claim show the mechanism was available and simply not applied here | Likely an oversight on `vehicles`: the claim is applied consistently across `customers` and once on `identity`, and no comment or config distinguishes the fleet routes. The CORS wildcard is more likely a development convenience | Security owner for the Pacco platform |
| Q13 | **[handled later by data and integration analysis]** Should the local customer copies in `availability-service`, `orders-service` and `parcels-service` be updated when a customer changes state or becomes a VIP? | GAP-8. Three services act on a customer snapshot taken at creation and never refreshed, while both change events are published and catalogued | Likely an oversight rather than a design: publishing two events that no service subscribes to is a strange thing to do deliberately. The alternative reading — that the snapshot is intentionally immutable — is not supported by anything in the code | Data and integration analysis stage, with the three service owners |
| Q14 | **[ACTION NOW]** What is meant to set the `Total-Count` response header the gateway exposes through CORS? | GAP-6. The header is advertised to browsers and set by nothing. The one paged endpoint in the platform has its paging envelope stripped by the gateway, so a client has no way to page at all | Likely a leftover from a paging design that was started and not finished: the header, the envelope-stripping rewrite on `GET /vehicles`, and the paged query type are three pieces of one feature that do not connect | Platform owner, with the `api-gateway` owner |
| Q15 | **[handled later by runtime and deployment validation]** On what path does each service publish its `[Contract]` types, and is that path exposed anywhere? | GAP-7. Over 70 message types are marked `[Contract]` and published by every service; if the endpoint is reachable it is a live, machine-readable contract source that would supersede parts of this document | Likely a fixed framework route on each service port, unreachable through the gateway since no `ntrada*.yml` route proxies it. The path itself is stated nowhere in the workspace and is not guessed here | Runtime and deployment validation stage |
