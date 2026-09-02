# Repository summary — `Pacco.Services.Parcels`

**Repository:** `Pacco.Services.Parcels` (workspace clone path: `hianshul100_Pacco.Services.Parcels`)
**Deployable:** `parcels-service` (also known as: Parcels Service, `Pacco.Services.Parcels.Api`, image `devmentors/pacco.services.parcels`). **Repository: `Pacco.Services.Parcels`, path: `src/Pacco.Services.Parcels.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Parcels
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Owns the **parcel catalogue** — the individual items a customer wants shipped. A parcel has a variant, size, and calculated volume; parcels are created by a customer, attached to and detached from orders, and deleted. The service also exposes an aggregate `volume` query, which is what lets the platform reason about whether a vehicle can carry a given set of parcels.

**Evidence:** `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`, `src/Pacco.Services.Parcels.Api/Program.cs`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice plus RabbitMQ consumer, in one process, using the canonical four-project clean-architecture layering (`.Api`, `.Application`, `.Core`, `.Infrastructure`) on Convey.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` / `GetWebHostBuilder` | `src/Pacco.Services.Parcels.Api/Program.cs` — `GetWebHostBuilder` is **`public`** specifically so the Pact provider tests can boot the host in-process |
| RabbitMQ subscriptions | `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` → `UseInfrastructure` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Parcels.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh`, **`scripts/start-test.sh`** (unique to this repository — starts the host for provider verification) |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Parcels.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Parcels.Api` | Host, endpoint map, `appsettings.json` |
| `src/Pacco.Services.Parcels.Application` | `Commands/` (`AddParcel`, `DeleteParcel`) + handlers; `Queries/` (`GetParcel`, `GetParcels`, `GetParcelsVolume`) + handlers; `Events/`, `Events/External/`, `Events/Rejected/`; `DTO/` |
| `src/Pacco.Services.Parcels.Core` | `Entities/Parcel.cs`, `Entities/Customer.cs`, `Entities/AggregateRoot.cs`, `Events/`, `Exceptions/`, `Repositories/IParcelRepository`, `ICustomerRepository` |
| `src/Pacco.Services.Parcels.Infrastructure` | `Mongo/Documents/ParcelDocument.cs`, `CustomerDocument.cs`, `Mongo/Repositories/`, `Decorators/` (outbox), `Contexts/`, `Exceptions/ExceptionToResponseMapper.cs` + `ExceptionToMessageMapper.cs`, `Logging/`, `Extensions.cs` |
| `tests/Pacco.Services.Parcels.PactProviderTests` | **Pactify 1.1.0** provider-side contract verification — `PACT/ParcelsApiPactProviderTests.cs`, plus a `MongoDbFixture` |

**Contract testing:** this repository is the **provider** side of the platform's single Pact pair; `Pacco.Services.Orders` is the consumer. The contract covers `GET /parcels/{parcelId}` only. The other five cross-service HTTP calls in the platform have no contract tests. The provider tests are the only tests in this repository — there are no unit or integration tests for the parcel domain itself.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| RabbitMQ | in + out | exchange `parcels`, topic |
| MongoDB | out | database `parcels-service` |
| Redis | out | instance prefix `parcels:` |
| Consul | out | registers `parcels-service` on port `5007` |
| Fabio | out | `http://localhost:9999` |
| Vault | out | KV v2 `kv/parcels-service/settings`; PKI role `parcels-service`, CN `parcels-service.pacco.io`; dynamic Mongo credentials |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

`httpClient.services` is **empty** — no outbound HTTP calls to other services. It is called *by* `orders-service` (`GET /parcels/{id}`).

## 6. Data stores / state handling

- **Store:** MongoDB, database `parcels-service`.
- **Collections — two domain collections:**
  - `parcels` — `AddMongoRepository<ParcelDocument, Guid>("parcels")`
  - **`customers`** — `AddMongoRepository<CustomerDocument, Guid>("customers")`
  - plus `inbox` and `outbox`.
- **Access mechanism:** Convey `IMongoRepository<>` over `MongoDB.Driver`. **No ORM.**
- **Migration tool: none.** No Flyway, Liquibase, Alembic, or EF Core migrations.
- **Document shapes:**
  - `ParcelDocument` — parcel id, `CustomerId`, variant, size, name, calculated volume, timestamps.
  - `CustomerDocument` — **`public Guid Id { get; set; }` and nothing else.** An id-only replica, byte-for-byte the same design as the one in `orders-service`.
- **Cross-domain coupling:** this service maintains a **local `customers` collection replicating customer identity owned by `customers-service`**, populated by the `customer_created` event — the same pattern as `orders-service`. There is no database foreign key (MongoDB, separate logical databases, no relational constraints); the replica is an existence check, storing no customer attributes. If `customer_created` is missed, this service does not recognise the customer and rejects their parcel creation.
  Parcel documents also carry order-related identifiers by way of the order events they consume, again as logical references with no enforcement.
- **Outbox:** enabled, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: inbox`, `outboxCollection: outbox`, `disableTransactions: true`.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `parcels`; `conventionsCasing: snakeCase`; queue template `parcels-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — commands:**

| Message | Wire name | Key payload fields |
|---|---|---|
| `AddParcel` | `add_parcel` | `ParcelId`, `CustomerId`, `Variant`, `Size`, `Name` |
| `DeleteParcel` | `delete_parcel` | `ParcelId` |

**Consumed — external events (five):**

| Message | Wire name | Origin | Effect |
|---|---|---|---|
| `CustomerCreated` | `customer_created` | `customers-service` | records the customer in the local `customers` replica |
| `ParcelAddedToOrder` | `parcel_added_to_order` | `orders-service` | marks the parcel as attached to an order |
| `ParcelDeletedFromOrder` | `parcel_deleted_from_order` | `orders-service` | marks the parcel as detached |
| `OrderCanceled` | `order_canceled` | `orders-service` | releases the parcels held by that order |
| `OrderDeleted` | `order_deleted` | `orders-service` | releases the parcels held by that order |

**Published — events:**

| Event | Wire name | Key payload fields |
|---|---|---|
| `ParcelAdded` | `parcel_added` | `ParcelId` |
| `ParcelDeleted` | `parcel_deleted` | `ParcelId` |

**Published — rejection events:** `add_parcel_rejected`, `delete_parcel_rejected`, each with `Reason` and `Code`, produced by `Infrastructure/Exceptions/ExceptionToMessageMapper.cs`.

**Bidirectional relationship with `orders-service`:** this service consumes four order events and publishes `parcel_deleted`, which `orders-service` consumes to remove the parcel from any order holding it. The two services keep a mutual view of parcel-to-order membership through events in both directions, with no shared store and no reconciliation mechanism — a design that works as long as no message is lost, and has no way to detect it if one is.

**Reliability:** outbox/inbox decorators wrap every command and event handler. The `Saga` header is forwarded.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5007`, container port `80`):

| Method | Path | Maps to | Gateway exposure |
|---|---|---|---|
| GET | `parcels` | `GetParcels` | `/parcels` → `parcels-service/parcels?customerId=@user_id` — scoped to the caller |
| GET | `parcels/volume` | `GetParcelsVolume` | `/parcels/volume` |
| GET | `parcels/{parcelId}` | `GetParcel` | `/parcels/{parcelId}` — **also the Pact-covered contract with `orders-service`** |
| POST | `parcels` | `AddParcel` | `/parcels` — gateway generates `parcelId`, binds `customerId: @user_id` |
| DELETE | `parcels/{parcelId}` | `DeleteParcel` | `/parcels/{parcelId}` |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**Route ordering note:** `GET parcels/volume` is registered **before** `GET parcels/{parcelId}` in `Program.cs`, which is what prevents the literal segment `volume` from being captured as a parcel id.

**Consumed:** none over HTTP.

**Called by:** `orders-service` → `GET /parcels/{id}` (unauthenticated; no caller credential is presented).

**Ownership scoping:** `GET /parcels` is scoped to `@user_id` and `POST /parcels` binds `customerId` from the token. `GET /parcels/{parcelId}` and `DELETE /parcels/{parcelId}` are addressed by id with **no ownership binding at the gateway**; whether the handlers check `CustomerId` is **Needs validation**. `GET /parcels/volume` takes no customer scope at all — whether it returns a platform-wide aggregate to any authenticated caller is **Needs validation**.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Parcels.Api.dll`.
- Composed as `parcels-service` on `5007:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5007`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- `scripts/start-test.sh` exists only here, to start the host for Pact provider verification. Whether CI invokes it, and where the consumer's pact file comes from, is **Unknown** — no pact broker URL appears in either repository.

## 10. Security/auth clues

- **JWT bearer** validation via `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- `.AddSecurity()` is registered, but there is **no `security.certificate` block** — this service does not verify client certificates, so the inbound call from `orders-service` is unauthenticated at the application layer.
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- Log redaction via `logger.excludeProperties`.
- **Authorisation is enforced only at the gateway**, and only for the collection-level routes (§8). Direct access to port `5007` bypasses it entirely.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: parcels-service`, UDP `localhost:6831`, `sampler: const`) with the RabbitMQ Jaeger plugin.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`. Handler logging via `.AddHandlersLogging()`.
- **Correlation:** `Correlation-Context` header; `Saga` header forwarded.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs` | The parcel model — variant, size, and the **volume calculation**, which is the rule that determines what fits in a vehicle |
| `src/Pacco.Services.Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs` | The decision to replicate customer identity locally as an id-only document, mirroring `orders-service` |
| `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` | Composition, and the subscription set — five external events, four of them from `orders-service` |
| `src/Pacco.Services.Parcels.Api/Program.cs` | `public GetWebHostBuilder` to support in-process provider verification; route ordering that protects `parcels/volume` |
| `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs` | The decision to verify one of six cross-service calls by contract |
| `src/Pacco.Services.Parcels.Api/appsettings.json` | Outbox with `disableTransactions: true` |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`). The parcel variant and size vocabulary and the volume calculation are compiled in, not configurable.

## 13. Open questions / ambiguities

- **`GET /parcels/volume` has no visible customer scoping.** Whether an authenticated caller receives a platform-wide total is **Needs validation**.
- **Ownership checks** on `GET`/`DELETE /parcels/{parcelId}` are **Needs validation** — the gateway does not bind them to `@user_id`.
- **The volume calculation** — how variant and size map to a number — was read at the entity level but not verified against any test. It is the rule that determines vehicle capacity fit, and it has **no unit tests**. **Needs validation.**
- **No unit or integration tests for the parcel domain.** The only tests are the Pact provider verification of a single endpoint.
- **Pact publication is unproven.** A provider verification exists here and a consumer contract in `Pacco.Services.Orders`, but no broker or publish step was found, so whether the two are actually exchanged rather than kept in step by hand is **Unknown**.
- **The mutual parcel-to-order view** maintained by events in both directions has no reconciliation or drift detection. **Needs validation.**
- **`disableTransactions: true`** on the outbox, as everywhere else.

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (four C# projects only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Parcels service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5007`. — **Confirmed** (`appsettings.json` `consul.port: 5007`, `Pacco/compose/services.yml` `5007:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Parcels` directory"**; the actual host project is **`/src/Pacco.Services.Parcels.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **The Pact provider tests** and the fact that this repository is one half of the platform's only contract-testing pair — along with `scripts/start-test.sh`, which exists solely to support them and appears in no other repository.
- **The volume calculation** and the `GET /parcels/volume` endpoint, the rule the platform relies on to decide what fits in a vehicle.
- **The local `customers` replica collection** and its dependence on the `customer_created` event.
- The five external event subscriptions, four of them from `orders-service`, and the bidirectional parcel-to-order relationship they maintain.
- The transactional outbox/inbox and the handler decorators; `scripts/`.

**Unknown (neither pass yielded proof):**
- Whether the Pact contract is published and verified automatically or maintained by hand.
- Whether the volume query is scoped to a customer.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The local `customers` collection is an existence check only, populated solely by the `customer_created` event — the same pattern as in `orders-service`. | `CustomerDocument` holds one field, `Id`, and the service subscribes to `customer_created`. | Customer data could be duplicated across services with no reconciliation defined. | Confirm the only insert path in the customer repository is the event handler. |
| A2 | Parcel-to-order membership is kept consistent purely by events flowing in both directions between this service and `orders-service`, with no reconciliation. | Four order events are consumed here and `parcel_deleted` is consumed there; no shared store, no periodic sync, and no drift check exist. | A single lost message would leave a parcel permanently attached to a cancelled order, or free when it should be held, with nothing to detect it. | Deliberately drop a message in a test environment and observe whether either side recovers. |
| A3 | The volume calculation on `Parcel` is the platform's authoritative rule for whether a set of parcels fits a vehicle. | It is the only volume computation in the workspace, and `GET /parcels/volume` is its only consumer-facing surface. | Capacity decisions elsewhere could be based on a different or absent rule, allowing over-allocated vehicles. | Ask the product owner to confirm the rule, and add unit tests covering each variant and size. |

### Blockers

_None identified for this repository._

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does `GET /parcels/volume` return a platform-wide total to any authenticated caller? It carries no customer binding at the gateway. | If so, every customer can see aggregate platform activity, which is a business-information leak rather than a personal-data one. | Unknown — the query was not traced to confirm whether it filters by customer. | Security owner |
| Q2 | **[ACTION NOW]** Can a caller read or delete a parcel belonging to someone else? `GET` and `DELETE /parcels/{parcelId}` are not bound to the token's user id at the gateway. | Parcel records carry customer-identifying context, and deletion is destructive. | Unknown — the handlers were not confirmed to check `CustomerId`. | Security owner / service owner |
| Q3 | **[handled later by architecture_evolution_generation]** Is the Pact contract between Orders and Parcels published and verified automatically, or kept in step by hand? | An unpublished contract gives the appearance of contract testing without the protection, and only one of six cross-service calls is covered either way. | Unknown — no broker URL or publish step appears in either repository. | Service owner |
| Q4 | **[ACTION NOW]** Should the volume calculation have unit tests? It currently has none, and it is the rule that decides vehicle capacity fit. | A silent error there would over- or under-allocate vehicles across the whole platform, and nothing would catch it. | Yes — it is small, pure, and central. | Service owner |
