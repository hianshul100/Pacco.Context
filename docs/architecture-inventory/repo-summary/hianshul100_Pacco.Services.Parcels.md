# Repository summary — `hianshul100_Pacco.Services.Parcels`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Parcels` (also known as: Pacco.Services.Parcels, the Parcels service)
**Deployable:** `Pacco.Services.Parcels.Api` (also known as: `parcels-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.parcels`, its published image). Repository: `hianshul100_Pacco.Services.Parcels`, path: `src/Pacco.Services.Parcels.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

Owns the parcel: the physical item a customer wants delivered, with its variant, size, description and the order it has been attached to. It also answers a volume query used when deciding whether a vehicle can carry an order.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, Convey-based, serving HTTP dispatcher endpoints and consuming RabbitMQ messages in the same process.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Parcels.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Parcels.Api`, `dotnet run` |
| Test run | `scripts/start-test.sh` — present only in this repository |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Parcels.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Parcels.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Dispatches to |
|---|---|---|
| GET | `parcels` | `GetParcels` |
| GET | `parcels/{parcelId}` | `GetParcel` |
| GET | `parcels/volume` | `GetParcelsVolume` |
| POST | `parcels` | `AddParcel` |
| DELETE | `parcels/{parcelId}` | `DeleteParcel` |

## 4. Important modules / packages

Four source projects plus one test project, listed in `Pacco.Services.Parcels.sln`:

| Project | Path | Role |
|---|---|---|
| `Pacco.Services.Parcels.Api` | `src/Pacco.Services.Parcels.Api` | host, endpoint registration, configuration |
| `Pacco.Services.Parcels.Application` | `src/Pacco.Services.Parcels.Application` | commands, events, external events, rejected events, queries, DTOs |
| `Pacco.Services.Parcels.Core` | `src/Pacco.Services.Parcels.Core` | `Parcel`, `Customer`, `Size`, `Variant` |
| `Pacco.Services.Parcels.Infrastructure` | `src/Pacco.Services.Parcels.Infrastructure` | Mongo documents, repositories, three query handlers, RabbitMQ broker, decorators, logging, exception mappers |
| `Pacco.Services.Parcels.PactProviderTests` | `tests/Pacco.Services.Parcels.PactProviderTests` | **`Pactify 1.1.0`** — `PACT/ParcelsApiPactProviderTests.cs`, `Fixtures/MongoDbFixture.cs`, `Fixtures/MongoDbFixtureInitializer.cs`, its own `appsettings.json`; the provider side of the contract that `orders-service` consumes |

Application commands: `AddParcel`, `DeleteParcel`. Events: `ParcelAdded`, `ParcelDeleted`. Rejected events: `AddParcelRejected`, `DeleteParcelRejected`. External events consumed: `CustomerCreated`, `OrderCanceled`, `OrderDeleted`, `ParcelAddedToOrder`, `ParcelDeletedFromOrder`. Queries: `GetParcel`, `GetParcels`, `GetParcelsVolume`.

Packages are the platform's standard Convey `0.4.*` set.

## 5. External integrations

MongoDB (database `parcels-service`), Redis (instance prefix `parcels:`), RabbitMQ (exchange `parcels`), Consul (service `parcels-service`, port 5007, ping endpoint `ping`), Fabio (`http://localhost:9999`), Vault (kv v2 mount `kv`, path `parcels-service/settings`; PKI common name `parcels.pacco.io`; dynamic MongoDB credentials with auto-renewal), Jaeger (service name `parcels`, UDP `localhost:6831`, `sampler: const`), Prometheus, Seq.

`httpClient.services` is **empty** — no outbound service-to-service HTTP. It is called by `orders-service` but calls no one.

## 6. Data stores and state handling

**Store:** MongoDB, database `parcels-service`. **Query mechanism:** Convey MongoDB repositories over the MongoDB .NET driver. **No ORM. No migration tool.**

| Collection | Fields |
|---|---|
| `parcels` | `Id` (Guid), `CustomerId` (Guid), `Variant` (enum), `Size` (enum), `Name` (string), `Description` (string), `CreatedAt` (DateTime), `OrderId` (nullable Guid), `AddedToOrder` (bool) |
| `customers` | `Id` (Guid) only — a replicated read model of a domain owned by `customers-service` |
| `inbox` | Convey inbox, de-duplication |
| `outbox` | Convey outbox, pending publishes |

Outbox: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Cross-domain coupling.** Same replication pattern as `orders-service`: a local `customers` collection holding nothing but identifiers, populated by the `CustomerCreated` handler so the service can validate a customer without a network call. `ParcelDocument.CustomerId` points at that replicated customer; `ParcelDocument.OrderId` points at an order owned by `orders-service`. Neither is a database foreign key.

`OrderId` and `AddedToOrder` are two representations of the same fact, kept in step by the `ParcelAddedToOrder` and `ParcelDeletedFromOrder` handlers. They can disagree if a handler fails part-way, and nothing detects that.

The parcel's `Name`, `Variant` and `Size` are copied into the order document by `orders-service` when the parcel is attached; changes here do not propagate.

Redis is registered but no cache call appears in this service's own code, so what it holds is **unknown — requires runtime capture**.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Mongo-backed transactional outbox and the Jaeger RabbitMQ plugin.

**Broker settings:** exchange `parcels`, `conventionsCasing: snakeCase`, queue name template `parcels-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `parcels`:**

- Events: `parcel_added`, `parcel_deleted`
- Rejected events: `add_parcel_rejected`, `delete_parcel_rejected`
- Commands accepted on the same exchange: `add_parcel`, `delete_parcel`

Payload key fields: the parcel events carry the parcel identifier and the customer identifier; exact field lists are **unknown — requires runtime capture**, since no `[Contract]`-annotated payload was read for these classes.

**Consumed:**

| Kind | Message | Origin exchange | Declared in |
|---|---|---|---|
| Command | `AddParcel`, `DeleteParcel` | `parcels` | own commands |
| Event | `CustomerCreated` | `customers` | `Events/External/CustomerCreated.cs`, `[Message("customers")]` |
| Event | `OrderCanceled`, `OrderDeleted`, `ParcelAddedToOrder`, `ParcelDeletedFromOrder` | `orders` | `Events/External/*.cs`, `[Message("orders")]` |

**Related defect, owned elsewhere.** `parcel_deleted` is published here on exchange `parcels`, but `hianshul100_Pacco.Services.Orders` declares its copy of `ParcelDeleted` as `[Message("deliveries")]`. The mismatch means Orders very likely never receives this event. The fix belongs in the Orders repository; it is recorded here because this service is the publisher and the visible symptom — orders keeping deleted parcels — looks like a parcels problem.

## 8. APIs exposed and consumed

**Exposed:** the five HTTP routes in section 3, Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5007`; port 80 in the container.

Through the gateway, `GET /parcels` is rewritten to `parcels-service/parcels?customerId=@user_id`, so a caller sees only their own parcels. `GET /parcels/volume` is exposed unchanged.

`GET /parcels/{parcelId}` is also called service-to-service by `orders-service` through Fabio, and it is the endpoint pinned by the Pact contract. This repository holds the provider side of that contract in `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs`; the consumer side lives in `hianshul100_Pacco.Services.Orders`. This is the only contract-tested interface in the platform.

**Consumed:** no outbound HTTP calls.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Parcels.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `parcels-service` on host port 5007.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`, plus a separate `appsettings.json` inside the Pact provider test project.
- `scripts/start-test.sh` exists here and nowhere else, which suggests the provider test needs the service running against a fixture database.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.parcels`.

## 10. Security and auth clues

- JWT validated against the committed `certs/localhost.cer`, `validIssuer: pacco`.
- `Convey.Security` is composed; no certificate authentication here.
- Ownership is enforced by the gateway rewriting `customerId` from the token, not by the service. Anything that can reach `parcels-service` on port 5007 can read or delete any customer's parcels.
- `logger.excludeProperties` keeps secret-like values out of logs; `httpClient.requestMasking` is configured.
- Vault supplies settings and dynamic MongoDB credentials; RabbitMQ credentials remain the plain defaults in `appsettings.json`.
- The Pact provider test project carries its own `appsettings.json`; test configuration files are a common place for credentials to leak, and it should be checked before the repository is published more widely.

## 11. Observability, logging and tracing clues

Jaeger under service name `parcels`; Prometheus metrics via `Convey.Metrics.AppMetrics` (`metrics.database: pacco`, interval 5); structured logs to console, file and Seq with `/`, `/ping` and `/metrics` excluded. `Infrastructure/Logging/MessageToLogTemplateMapper.cs` gives each command and event its own log template.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Parcels.Infrastructure/Extensions.cs` (technology choices and the seven subscriptions), `src/Pacco.Services.Parcels.Api/Program.cs` (HTTP surface), `src/Pacco.Services.Parcels.Api/appsettings.json` (integration endpoints), `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`, `Size.cs` and `Variant.cs` (the parcel vocabulary that `orders-service` copies into its own documents), `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs` (the platform's only contract commitment).

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Parcels.Api/`, `src/Pacco.Services.Parcels.Application/`, `src/Pacco.Services.Parcels.Core/`, `src/Pacco.Services.Parcels.Infrastructure/`, `tests/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5007`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.parcels`; `Pacco.Services.Parcels.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the Pact provider test and the contract commitment to `orders-service`; `scripts/start-test.sh`; the seven message subscriptions; the replicated `customers` read model; the Vault integration; the transactional outbox.

**Conflicts to surface:**

- **Stale doc.** The README instructs running `dotnet run` in `/src/Pacco.Services.Parcels`. That directory does not exist — the host project is `src/Pacco.Services.Parcels.Api`. `scripts/start.sh` has the correct path.
- **Cross-repository conflict.** This service publishes `parcel_deleted` on exchange `parcels`; `orders-service` subscribes to it on exchange `deliveries`. The two repositories disagree about where the event lives. Recorded as Q1 below and as a blocker in the Orders summary.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`. The Pact provider test is the only test, and whether it can pass in CI depends on a MongoDB fixture and a running service that CI does not obviously provide.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The replicated `customers` collection exists only to check that a customer is known. | The document has an `Id` field and nothing else, matching the same pattern in `orders-service`. |
| A2 | `GetParcelsVolume` computes total volume from the parcels' `Size` values. | The query has no other input and `Size` is the only dimensional field on the document. |
| A3 | `scripts/start-test.sh` exists to support the Pact provider test. | It is unique to the repository that holds the provider test. |
| A4 | `messages.json` in the Operations repository is meant to be the complete catalogue of this service's messages. | Operations builds live subscriptions from it at start-up. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** `OrderId` and `AddedToOrder` duplicate the same fact with no invariant tying them together. | A partially applied handler leaves a parcel that claims to be free while still holding an order identifier, or the reverse. | Parcels service maintainer: derive one from the other. |
| B2 | **[ACTION NOW]** Parcel ownership is enforced only at the gateway. | Anything on the internal network can read or delete any customer's parcels. | Parcels service maintainer, with the security-architecture owner. |
| B3 | **[ACTION NOW]** The Pact provider test's `appsettings.json` has not been reviewed for embedded credentials. | Test configuration is a common leak point, and the platform already has secrets committed elsewhere. | Parcels service maintainer. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Which exchange is `parcel_deleted` meant to be published on? | This service says `parcels` and `orders-service` listens on `deliveries`; one of them is wrong and an order-cleanup path is broken. | Parcels service maintainer with the Orders service maintainer. |
| Q2 | **[handled later by the API-contract stage]** What fields do `parcel_added` and `parcel_deleted` carry? | Three services react to parcel changes and none of the payloads is asserted in source. | API-contract stage. |
| Q3 | **[handled later by the domain-model stage]** What are the valid values of `Variant` and `Size`, and how do they map to volume? | `orders-service` copies both into its own documents as strings, so they are effectively a public vocabulary. | Domain-model stage. |
| Q4 | **[handled later by the data-architecture stage]** What is stored in Redis under the `parcels:` prefix? | Not answerable from source; determines whether Redis is a hard dependency here. | Data-architecture stage, by inspecting a running instance. |
