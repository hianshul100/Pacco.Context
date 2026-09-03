# Repository: `Pacco.Services.Parcels`

`parcels-service` (also known as: Parcels Service, `Pacco.Services.Parcels`, Docker image
`devmentors/pacco.services.parcels`) owns the parcel catalogue — each parcel's size, variant and
owning customer — and aggregates parcel volume.

- **Repository:** `Pacco.Services.Parcels`, path: `src/Pacco.Services.Parcels.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5007`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- The four-project clean-architecture split, and `Core/Entities/{Parcel,Size,Variant,Customer}.cs`.
- All five HTTP endpoints and the RabbitMQ `parcels` exchange with its six messages.
- MongoDB database `parcels-service` with **two** collections, `parcels` and a replicated
  `customers`.
- The five external events this service subscribes to — four of them from `orders`.
- `tests/Pacco.Services.Parcels.PactProviderTests` — **Pact provider** verification
  (`Pactify` 1.1.0), the counterpart to the consumer tests in `Pacco.Services.Orders`.
- The `GetWebHostBuilder` seam in `Program.cs` that exists so the Pact provider tests can host the
  application in-process.

**Stale doc:** none identified.

**Unknown:** how the Pact contract file reaches this repository from `Pacco.Services.Orders`; no
Pact Broker configuration exists in either repository or either Travis pipeline.

---

## 1. Primary purpose

Maintain the parcels a customer owns, expose them individually and as a list, compute aggregate
parcel volume, and keep parcel state consistent with the orders they are attached to by reacting to
order events.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process, using Convey CQRS dispatcher
endpoints. It is additionally a **Pact provider** — its HTTP surface is contract-verified against
`orders-service`'s expectations.

## 3. Key entrypoints

- `src/Pacco.Services.Parcels.Api/Program.cs` — composition root and route table. It exposes a
  `GetWebHostBuilder` seam so the Pact provider test project can host the application in-process.
- `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` — DI composition root.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Parcels.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Project | Role |
|---|---|
| `Pacco.Services.Parcels.Api` | Host, route table, configuration, `certs/`, the Pact test seam |
| `Pacco.Services.Parcels.Application` | Commands (`AddParcel`, `DeleteParcel`), queries, events, DTOs, handlers |
| `Pacco.Services.Parcels.Core` | `Entities/Parcel.cs`, `Entities/Size.cs`, `Entities/Variant.cs`, `Entities/Customer.cs`, repository interfaces, domain exceptions |
| `Pacco.Services.Parcels.Infrastructure` | Mongo documents and repositories, RabbitMQ broker, decorators, contexts, logging |
| `tests/Pacco.Services.Parcels.PactProviderTests` | Pact **provider** verification (`Pactify` 1.1.0) with a `PACT/` directory |

**Key packages:** `Convey`, `Convey.CQRS.Commands/.Events/.Queries`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Swagger`; plus `Pactify` 1.1.0 in the test project.

## 5. External integrations

Consul (registration, `pingEndpoint: ping`), Fabio, RabbitMQ (exchange `parcels`), MongoDB
(database `parcels-service`), Redis (prefix `parcels:`), Vault (kv v2 path
`parcels-service/settings`, PKI role `parcels-service`, common name `parcels-service.pacco.io`,
MongoDB dynamic credentials), Jaeger, Seq, Prometheus.

**It calls no other service.** `httpClient.services` is empty — a leaf in the synchronous call
graph, and the target of `orders-service`'s parcel-validation call.

## 6. Data stores / state

- **Store:** MongoDB, database `parcels-service`.
- **Query mechanism:** Convey `IMongoRepository<ParcelDocument, Guid>` and
  `IMongoRepository<CustomerDocument, Guid>` over the MongoDB .NET driver. **Not a relational ORM.**
- **Registrations** (`src/Pacco.Services.Parcels.Infrastructure/Extensions.cs`):
  `AddMongoRepository<CustomerDocument, Guid>("customers")` and
  `AddMongoRepository<ParcelDocument, Guid>("parcels")`.
- **Collections for the primary domain:**
  - **`parcels`** — `Infrastructure/Mongo/Documents/ParcelDocument.cs`.
  - **`customers`** — `Infrastructure/Mongo/Documents/CustomerDocument.cs`, a **replica of another
    service's domain**.
- **Framework collections:** `inbox`, `outbox` (`type: sequential`, `disableTransactions: true`).
- **Migration tool:** **none.** No migration files or tooling in the repository.
- **Cross-domain coupling — flagged.** MongoDB has no foreign keys; the equivalent coupling here is
  the same pattern found in `orders-service`:
  1. **`customers` collection** — owned by `customers-service`, replicated here and populated
     **only** by the `customer_created` event. `customer_state_changed` and `customer_became_vip`
     are published by the owner but **not consumed here**, so this copy is a creation-time snapshot
     that never updates.
  2. **`ParcelDocument` carries an `OrderId`** (owned by `orders-service`), kept in step by
     subscribing to four `orders` events rather than by any referential constraint. This is the
     more disciplined of the two couplings: parcel-to-order state is genuinely event-maintained,
     whereas the customer copy is not.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `parcels`, type `topic`, durable.
- **Conventions:** `snakeCase`; queue template `parcels-service/{{exchange}}.{{message}}`; headers
  `message_context` and `span_context`.
- **Outbox:** enabled (`AddMessageOutbox(o => o.AddMongo())`) with outbox decorators on handlers.

**Commands consumed:** `add_parcel`, `delete_parcel`.

**Events published:**

| Event | Observable payload fields |
|---|---|
| `parcel_added` | `ParcelId`, `CustomerId` |
| `parcel_deleted` | `ParcelId` |

**Rejected events published:** `add_parcel_rejected`, `delete_parcel_rejected` — each with
`Reason` and `Code`.

**External events consumed** (`Application/Events/External/Handlers/`):

| Event | Source exchange | Purpose |
|---|---|---|
| `customer_created` | `customers` | Populates the local `customers` replica |
| `order_canceled` | `orders` | Detaches parcels from the cancelled order |
| `order_deleted` | `orders` | Detaches parcels from the deleted order |
| `parcel_added_to_order` | `orders` | Marks the parcel as attached to an order |
| `parcel_deleted_from_order` | `orders` | Marks the parcel as detached |

This is the **most order-coupled service on the platform** — four of its five subscriptions come
from the `orders` exchange.

**Consumers of this service's events:** `parcel_deleted` → `orders-service`. `parcel_added` has
**no domain consumer**. `operations-service` observes both. **Needs validation.**

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Parcels.Api/Program.cs`, verbatim):

| Method | Route | Dispatches |
|---|---|---|
| `GET` | `parcels/volume` | query — aggregate parcel volume |
| `GET` | `parcels/{parcelId}` | query — single parcel |
| `GET` | `parcels` | query — parcel list, scoped by `IAppContext` (caller identity) |
| `POST` | `parcels` | `AddParcel` |
| `DELETE` | `parcels/{parcelId}` | `DeleteParcel` |

Swagger UI at route prefix `docs`.

**Route ordering note:** `parcels/volume` is registered **before** `parcels/{parcelId}`, which is
what keeps the literal segment from being captured as a parcel id.

**Consumed:** none.

**Inbound synchronous callers:** `orders-service` calls `GET /parcels/{parcelId}` to validate a
parcel before adding it to an order. **This is the call the Pact contract governs.**

**Upstream:** the gateway module `parcels` fronts `GET /` (rewritten to
`parcels-service/parcels?customerId=@user_id`), `GET /volume`, and the two write routes; in async
mode the writes arrive as RabbitMQ commands instead.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Parcels.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.parcels`.
  **`scripts/test.sh` is not invoked by CI, so the Pact provider verification does not run there.**
- Port `5007` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5007:80`), and the
  gateway `localUrl`.
- Consul service name `parcels-service`; `httpClient.type: fabio` (configured but unused, since the
  service calls nothing).
- **Runtime dependency note:** `orders-service` cannot add a parcel to an order while this service
  is unreachable — it is on the order-creation critical path.

## 10. Security / auth clues

- **JWT bearer** with `certs/localhost.cer`, `validIssuer: pacco`.
- **Identity-scoped reads:** `GetParcelsHandler` filters by `IAppContext`, so the parcel list
  returns only the caller's parcels — a real in-service check, matching `orders-service`.
- **No role gate at the edge** for the `parcels` module — any authenticated caller can add a parcel
  or delete one by id. Whether deletion verifies ownership is **Unknown — needs validation**.
- **No certificate ACL**, even though `orders-service` calls this service synchronously.
  `customers-service` has an ACL for its one synchronous caller; this service does not, and
  `orders-service`'s client attaches no certificate. The inbound call from `orders-service` is
  therefore not certificate-authenticated. **Needs validation** of whether that is intentional.
- **Vault:** kv v2 settings, PKI role `parcels-service`, MongoDB dynamic credentials with lease
  auto-renewal.
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties.
- **`GET /parcels/volume` exposes an aggregate** with no role gate and no identity scoping visible
  in its route registration — it appears to report platform-wide volume to any authenticated
  caller. **Needs validation.**

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: parcels`, UDP `6831`, `const` sampler rate 1, with the
  `Convey.Tracing.Jaeger.RabbitMQ` plugin propagating `span_context` across AMQP.
- **Logging:** console, file and Seq sinks enabled; ELK sink present but `enabled: false`.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Parcels.sln` | Four-project clean-architecture split plus a contract-test project |
| `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` | Capability chain, two Mongo repository registrations (including the customer replica), outbox decorators |
| `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`, `Entities/Size.cs`, `Entities/Variant.cs` | That parcel size and variant are modelled as domain concepts rather than free-form fields — this is what makes `GET /parcels/volume` computable |
| `src/Pacco.Services.Parcels.Application/Events/External/Handlers/` | That parcel-to-order state is maintained by subscribing to four `orders` events rather than by querying `orders-service` |
| `tests/Pacco.Services.Parcels.PactProviderTests/` | That this service's HTTP surface is governed by a consumer-driven contract with `orders-service` (`Pactify` 1.1.0) |
| `src/Pacco.Services.Parcels.Api/Program.cs` | The `GetWebHostBuilder` seam that exists specifically to support provider verification |
| `src/Pacco.Services.Parcels.Api/appsettings.json` | Exchange, outbox with `disableTransactions: true`, Vault PKI |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. How the Pact contract file travels from `Pacco.Services.Orders`, and whether either side's tests
   run anywhere.
2. Whether the local `customers` replica is meant to stay a creation-time snapshot.
3. Why `parcel_added` has no domain consumer.
4. Whether `GET /parcels/volume` is meant to be platform-wide or caller-scoped.
5. Whether parcel deletion verifies caller ownership, given the list query does.
6. Why the inbound synchronous call from `orders-service` carries no client certificate, when the
   equivalent Availability → Customers call does.
7. Whether `outbox.disableTransactions: true` is deliberate.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Parcels.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Parcels.Application/`,
`src/Pacco.Services.Parcels.Core/`, `src/Pacco.Services.Parcels.Infrastructure/`, `tests/`, and the
repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor). The only browser-facing surface is the Convey Swagger UI at
`/docs`, generated by `Convey.WebApi.Swagger`.

---

## Evidence

| Fact | File |
|---|---|
| Route table, route ordering, Pact host builder seam | `src/Pacco.Services.Parcels.Api/Program.cs` |
| DI composition, both Mongo repository registrations, outbox decorators | `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` |
| Domain model for size and variant | `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`, `Entities/Size.cs`, `Entities/Variant.cs`, `Entities/Customer.cs` |
| Persistence documents, including the customer replica | `src/Pacco.Services.Parcels.Infrastructure/Mongo/Documents/ParcelDocument.cs`, `CustomerDocument.cs` |
| Commands | `src/Pacco.Services.Parcels.Application/Commands/AddParcel.cs`, `DeleteParcel.cs` |
| Published events and payloads | `src/Pacco.Services.Parcels.Application/Events/ParcelAdded.cs`, `ParcelDeleted.cs` |
| Rejected events | `src/Pacco.Services.Parcels.Application/Events/Rejected/*.cs` |
| Consumed external events | `src/Pacco.Services.Parcels.Application/Events/External/Handlers/*.cs` |
| Identity-scoped parcel list | `src/Pacco.Services.Parcels.Application/Queries/Handlers/GetParcelsHandler.cs` |
| Volume aggregation | `src/Pacco.Services.Parcels.Application/Queries/Handlers/` (volume query handler) |
| Pact provider verification | `tests/Pacco.Services.Parcels.PactProviderTests/`, `tests/Pacco.Services.Parcels.PactProviderTests/PACT/` |
| Exchange, outbox, Vault, JWT, logging, metrics, tracing | `src/Pacco.Services.Parcels.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Package set | `src/Pacco.Services.Parcels.Infrastructure/Pacco.Services.Parcels.Infrastructure.csproj`, `src/Pacco.Services.Parcels.Api/Pacco.Services.Parcels.Api.csproj`, `tests/Pacco.Services.Parcels.PactProviderTests/*.csproj` |
| Project list | `Pacco.Services.Parcels.sln` |
| Container build and CI, and that tests are not run in CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Pact consumer counterpart and the call it governs | `../hianshul100_Pacco.Services.Orders/tests/Pacco.Services.Orders.PactConsumerTests/`, `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Services/Clients/ParcelsServiceClient.cs` |
| Customer replica counterpart | `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Gateway routes and `@user_id` rewriting | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The local `customers` collection is a read-only projection, written only by the `customer_created` handler | The handler is the only writer visible in the repository, and no command touches customer data | If anything else writes it, this service becomes another source of truth for customer data with no reconciliation against `customers-service` | Read every write path against `CustomerDocument` in this repository |
| A2 | The parcel-to-order relationship is kept correct purely by the four `orders` subscriptions | Those handlers are the only code that changes a parcel's order association, and no synchronous call to `orders-service` exists | A missed or failed event would leave a parcel attached to an order that no longer holds it, with nothing to detect or repair it | Test each of the four handlers against a running platform, including a broker outage |
| A3 | The `PACT/` directory in this repository holds the contract produced by `Pacco.Services.Orders` | Both repositories use `Pactify` 1.1.0 and the consumer/provider pairing matches the `orders-service` → `parcels-service` call | If the file is stale or hand-edited, provider verification passes against expectations the consumer no longer has | Compare the two `PACT/` directories and establish how the file is produced |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** How does the Pact contract get here from `Pacco.Services.Orders`, and does the verification actually run? | No Pact Broker is configured in either repository, and neither Travis pipeline invokes `scripts/test.sh`. The platform's only contract-testing arrangement may be verifying a hand-copied file that nobody executes — which would make the Orders → Parcels call unprotected in practice while appearing governed | Introduce a Pact Broker or a shared contract artefact, and wire the tests into CI on both sides | Whoever owns the Orders/Parcels contract tests |
| Q2 | **[ACTION NOW]** Should this service consume `customer_state_changed` and `customer_became_vip`? | Its `customers` copy is written once at creation and never updated, so every later change leaves it stale with nothing detecting the drift | Either subscribe to the state events, or state that only the creation snapshot matters here | Domain owners for Parcels and Customers |
| Q3 | **[ACTION NOW]** Is `GET /parcels/volume` meant to return platform-wide volume to any authenticated caller? | The parcel list is scoped to the caller, but the volume route has no such scoping visible and no role gate. If it aggregates across all customers, it leaks a business metric to every signed-in user | Confirm whether the aggregate is meant to be admin-only or caller-scoped | Domain owner for Parcels |
| Q4 | **[ACTION NOW]** Should the inbound call from `orders-service` be certificate-authenticated? | `customers-service` requires a Vault-issued client certificate from its one synchronous caller and holds a matching ACL. This service has the same situation and no such protection, so anything on the network can call it as if it were `orders-service` | Either add the ACL and certificate as Availability/Customers do, or record why this call is treated differently | Whoever owns Pacco authentication |
| Q5 | **[ACTION NOW]** Does parcel deletion check that the caller owns the parcel? | The list query is identity-scoped, but `DELETE /parcels/{parcelId}` takes an id and has no role gate at the gateway. If the handler does not check ownership, any authenticated user who knows a parcel id could delete someone else's parcel | Read the delete handler for an ownership check and confirm against a running instance | Whoever owns Pacco authentication |
| Q6 | **[handled later by HLD]** Should `parcel_added` have a consumer? | It is published on every parcel creation and nobody listens, while `parcel_deleted` is consumed by `orders-service`. The asymmetry suggests either a missing consumer or an event emitted only for visibility | Either add the consumer or record the event as observability-only | Domain owner for Parcels |
| Q7 | **[handled later by HLD]** Is `outbox.disableTransactions: true` the intended setting? | Without transactions the parcel write and the outbox write are not atomic, so a crash between them can delete a parcel that `orders-service` never hears about — leaving an order holding a parcel that no longer exists | Likely a single-node MongoDB constraint in development; confirm the production topology | Platform architect |
