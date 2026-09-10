# Repository Summary — `hianshul100_Pacco.Services.Parcels`

**Primary name:** `parcels-service` (aliases used in this file: `Pacco.Services.Parcels.Api` — the .NET project and assembly name; `devmentors/pacco.services.parcels` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Parcels`, path: `src/Pacco.Services.Parcels.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Owns the parcel entity — the physical items a customer wants delivered. It registers and deletes parcels, serves parcel details, and computes an aggregate parcel *volume* used by other parts of the platform. It also keeps its own view of which parcels belong to which order by reacting to order events.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`, with an in-process RabbitMQ consumer. Layered `.Api` / `.Application` / `.Core` / `.Infrastructure`.

`Program.cs` exposes a `GetWebHostBuilder` method so tests can boot the host — the same pattern `availability-service` uses. Here it exists to support PACT provider verification.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Parcels.Api/Program.cs` |
| Test-facing host builder | `GetWebHostBuilder` in the same file |
| Dependency wiring | `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Parcels.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Parcels.sln`: `src/Pacco.Services.Parcels.Api`, `.Application`, `.Core`, `.Infrastructure`. `.Core` has no NuGet references. `.Infrastructure` carries the standard Convey stack.

**One test project exists on disk but is not in this repository's own solution file:** `tests/Pacco.Services.Parcels.PactProviderTests`, containing `PACT/ParcelsApiPactProviderTests.cs`, `Fixtures/MongoDbFixture.cs`, `Fixtures/MongoDbFixtureInitializer.cs` and its own `appsettings.json`. It is referenced only from the aggregate `Pacco.sln` in `hianshul100_Pacco`.

## 5. External integrations

- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — through Convey extensions.
- No outbound HTTP calls to peer services: `httpClient.services` in `src/Pacco.Services.Parcels.Api/appsettings.json` is empty. This service is called by `orders-service`; it calls no one.

## 6. Data stores & state

- **MongoDB.** Database `parcels-service`, connection string `mongodb://localhost:27017`, `seed: false`.
- **Collections:** two —
  - `parcels` — `.AddMongoRepository<ParcelDocument, Guid>("parcels")`
  - `customers` — `.AddMongoRepository<CustomerDocument, Guid>("customers")`
- **Outbox collections:** `outbox` and `inbox` (`outbox.enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `connectionString: localhost`, key prefix `parcels:`.
- **Query mechanism:** Convey `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver. **No ORM.**
- **Migration tool:** none. `tests/Pacco.Services.Parcels.PactProviderTests/Fixtures/MongoDbFixtureInitializer.cs` seeds data for the contract test; it is a test fixture, not a migration mechanism.
- **Cross-domain coupling.** The `customers` collection here is a **local read-model replica** of data owned by `customers-service`, populated by subscribing to `CustomerCreated`. `orders-service` maintains an identical replica in its own database. Customer records therefore live in three separate MongoDB databases with no foreign key and no reconciliation process. A parcel document also carries an order identifier owned by `orders-service`, kept in step only by the `parcel_added_to_order` and `parcel_deleted_from_order` events.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `parcels`, queue template `parcels-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Mongo-backed outbox; Jaeger plugin registered on the broker.

**Consumed** (`src/Pacco.Services.Parcels.Infrastructure/Extensions.cs`, lines 89–95):

| Kind | Message | Wire name | Origin |
|---|---|---|---|
| Command | `AddParcel` | `add_parcel` | `api-gateway` async mode |
| Command | `DeleteParcel` | `delete_parcel` | `api-gateway` async mode |
| Event | `CustomerCreated` | `customer_created` | `customers-service` |
| Event | `OrderCanceled` | `order_canceled` | `orders-service` |
| Event | `OrderDeleted` | `order_deleted` | `orders-service` |
| Event | `ParcelAddedToOrder` | `parcel_added_to_order` | `orders-service` |
| Event | `ParcelDeletedFromOrder` | `parcel_deleted_from_order` | `orders-service` |

**Published**, per the `parcels-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: events `parcel_added`, `parcel_deleted`; rejection events `add_parcel_rejected`, `delete_parcel_rejected`.

`orders-service` subscribes to `parcel_deleted`, so deleting a parcel here removes it from any order holding it. `ordermaker-service` subscribes to `parcel_added_to_order`, which `orders-service` publishes — so this service and the saga both react to the same order events without knowing about each other.

**Payload key fields:** `api-gateway` binds `customerId:@user_id` into commands on this exchange, and exposes `GET parcels?customerId=@user_id`, so a parcel is customer-scoped. The complete field set of each message is **unknown — requires runtime capture**.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.Parcels.Api/Program.cs`:

| Method | Path | Dispatched to | Response |
|---|---|---|---|
| GET | `parcels` | `GetParcels` | collection of parcel DTOs |
| GET | `parcels/{parcelId}` | `GetParcel` | parcel DTO |
| GET | `parcels/volume` | `GetParcelsVolume` | `ParcelsVolumeDto` |
| POST | `parcels` | `AddParcel` | — |
| DELETE | `parcels/{parcelId}` | `DeleteParcel` | — |

Swagger UI at `docs`. Consul health endpoint `ping`.

Note the route ordering: `parcels/volume` is declared **after** `parcels/{parcelId}` in `Program.cs`. Whether `volume` is correctly resolved rather than being captured as a `parcelId` depends on ASP.NET Core route precedence, which favours literal segments over parameters. It resolves correctly, but the ordering is fragile.

**Consumed by:**

- `api-gateway` — exposes `GET parcels?customerId=@user_id` and `GET parcels/volume` publicly, and routes `add_parcel` and `delete_parcel` through RabbitMQ.
- `orders-service` — over HTTP through Fabio.

**Consumes:** nothing over HTTP.

**Contract testing.** `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs` verifies this service against the expectations declared by `orders-service` in `hianshul100_Pacco.Services.Orders/tests/Pacco.Services.Orders.PactConsumerTests`. This service is the **provider** side of the platform's only formal inter-service contract. `Fixtures/MongoDbFixture.cs` and `Fixtures/MongoDbFixtureInitializer.cs` stand up a real MongoDB for the verification run, and the test project carries its own `appsettings.json`.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5007`.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.parcels`.
- The PACT provider test needs a running MongoDB, so continuous integration must provide one for `./scripts/test.sh` to pass — assuming the test runs at all (see open questions).

## 10. Security & auth clues

- No `security` access-control list and no certificate authentication.
- JWT validated against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- Customer scoping is applied by `api-gateway` through the `customerId:@user_id` binding, not by this service. The service's own `GET parcels` route accepts any `customerId`.
- Vault: `enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, key-value path `parcels-service/settings`, PKI `roleName: parcels-service`, `commonName: parcels-service.pacco.io`, `lease.mongo` dynamic MongoDB credentials with `autoRenewal: true`.
- **Checked-in credentials:** `vault.token: "secret"`, RabbitMQ `guest`/`guest`, Seq `apiKey: secret` in `src/Pacco.Services.Parcels.Api/appsettings.json`, plus a second `appsettings.json` inside the PACT test project.
- `.UsePublicContracts<ContractAttribute>()` exposes the message contract surface.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: parcels`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker.
- **Correlation:** `Correlation-Context` header on inbound requests; the `Saga` header is forwarded.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]` and the standard `excludeProperties` redaction list. `.AddHandlersLogging()` logs each handler.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` — the composition root and the seven-message subscription list.
- `src/Pacco.Services.Parcels.Api/Program.cs` — the route table and the decision to expose `GetWebHostBuilder` for contract verification.
- `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs` — the provider half of the platform's only contract test, and the decision to verify against a real MongoDB rather than a stub.
- `src/Pacco.Services.Parcels.Api/appsettings.json` — the configuration contract.
- `Pacco.Services.Parcels.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- **`PactProviderTests` is absent from `Pacco.Services.Parcels.sln`.** It exists on disk and is referenced only from the aggregate `Pacco.sln`. Whether `./scripts/test.sh` in Travis picks it up is **Unknown**. **Needs validation** — if not, the provider side of the platform's only contract test never runs.
- What `GET parcels/volume` aggregates over — all parcels, a customer's parcels, or an order's parcels — is **Unknown** from the route alone. The gateway exposes it with no query parameters, which suggests a global figure.
- Why this service reacts to `order_canceled` and `order_deleted` is **Unknown**; presumably it detaches parcels from the cancelled order, but the effect on parcel state is not visible from configuration.
- The declaration order of `parcels/volume` after `parcels/{parcelId}` is fragile even though it currently resolves correctly. **Needs validation** if routing behaviour is ever customised.
- How `customers` replica records are corrected if a `customer_created` event is missed is **Unknown**; no reconciliation job exists.
- Full payload field sets are **unknown — requires runtime capture**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Parcels.Api/`, `src/Pacco.Services.Parcels.Application/`, `src/Pacco.Services.Parcels.Core/`, `src/Pacco.Services.Parcels.Infrastructure/`, `tests/`, `tests/Pacco.Services.Parcels.PactProviderTests/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Parcels` directory". | No such directory. The runnable project is `src/Pacco.Services.Parcels.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5007`." | `appsettings.json` sets the Consul port to `5007`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.parcels`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Parcels.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README is the shared platform template and says nothing service-specific. | Seven message subscriptions, a `customers` read-model replica, and the provider half of the platform's only contract test. | **Docs gap.** |

**On disk but not mentioned in `README.md`:** the messaging surface, the `customers` replica, the outbox, the Vault integration, and the entire `tests/` directory including the PACT verification and its MongoDB fixtures.

**Disk-only component not in this repository's own solution:** `tests/Pacco.Services.Parcels.PactProviderTests`.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The `customers` collection in the `parcels-service` database is a read-model replica, not the system of record. | `customers-service` owns the customer domain and publishes `customer_created`; this service subscribes to it and registers a `customers` repository of its own. | Data ownership across the platform would be misdescribed. | Read the `CustomerCreated` handler under `src/Pacco.Services.Parcels.Application/Events/External/`. |
| A2 | `GET parcels/volume` resolves to the volume query rather than being captured as a parcel identifier, despite being declared after `parcels/{parcelId}`. | ASP.NET Core route matching prefers literal segments over route parameters. | The endpoint would return a not-found error for every call, and the gateway route `parcels/volume` would be broken. | Call the endpoint against a running instance. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does continuous integration actually run `Pacco.Services.Parcels.PactProviderTests`, given it is missing from `Pacco.Services.Parcels.sln`? | It is the provider half of the platform's only contract test, and it also needs a running MongoDB. If it is not running, the contract is unverified from both ends. | Add the project to the repository's own solution file and ensure the build provides MongoDB. | Service owner |
| Q2 | **[handled later by the platform inventory review]** What does `GET parcels/volume` aggregate over? | It is exposed publicly at the gateway with no parameters, so it may be returning a platform-wide figure to any authenticated caller. | Read the `GetParcelsVolume` handler. | Service owner |
| Q3 | **[handled later by the platform inventory review]** What corrects a `customers` replica that missed a `customer_created` event? | Three databases hold customer records with no reconciliation process. | No mechanism exists today. | Platform architect |
