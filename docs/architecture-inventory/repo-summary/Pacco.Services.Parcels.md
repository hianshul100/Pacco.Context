---
title: "Repository Summary — Pacco.Services.Parcels"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Parcels"
status: "evidence-based inventory"
---

# Pacco.Services.Parcels

**Primary name:** `Pacco.Services.Parcels` (aliases used in this file: `parcels-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `parcels` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Parcels`, path: `hianshul100_Pacco.Services.Parcels/`

---

## 1. Primary purpose

Owns the parcels a customer wants delivered: their size, variant and volume. It is the provider side of the platform's only consumer-driven contract.

Evidence: `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs`, `Size.cs`, `Variant.cs`, `src/Pacco.Services.Parcels.Core/Services/ParcelsService.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service using Convey dispatcher endpoints, plus a RabbitMQ subscriber. Listens on `5007`. Its host builder is exposed as a public `GetWebHostBuilder` so the provider contract tests can start it in process.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` — `Main` → public `GetWebHostBuilder` | `src/Pacco.Services.Parcels.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |
| `scripts/start-test.sh` — starts the service against `appsettings.test.json` | `scripts/start-test.sh` |

## 4. Modules / packages

Four source projects plus one test project: `Pacco.Services.Parcels.Api`, `.Application`, `.Core`, `.Infrastructure`, and `tests/Pacco.Services.Parcels.PactProviderTests`.

- **Core:** `Entities/Parcel.cs`, `Customer.cs`, `Size.cs`, `Variant.cs`; `Services/IParcelsService.cs`, `ParcelsService.cs`; `Repositories/IParcelRepository.cs`, `ICustomerRepository.cs`.
- **Application:** commands `AddParcel`, `DeleteParcel` with handlers; integration events `ParcelAdded`, `ParcelDeleted`; external events `CustomerCreated`, `OrderCanceled`, `OrderDeleted`, `ParcelAddedToOrder`, `ParcelDeletedFromOrder` with five handlers; rejected events `AddParcelRejected`, `DeleteParcelRejected`; queries `GetParcel`, `GetParcels`, `GetParcelsVolume`.
- **Infrastructure:** MongoDB documents, repositories and query handlers, plus the shared outbox decorators, event mapper, message broker and exception mappers.
- **Tests:** `PACT/ParcelsApiPactProviderTests.cs` using `Pactify 1.1.0`, with `Fixtures/MongoDbFixture.cs` and `Fixtures/MongoDbFixtureInitializer.cs`.

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus. No outbound HTTP service clients are defined.

## 6. Data stores / state

- **Store:** MongoDB, database `parcels-service`. The test configuration uses a separate database, `test-parcels-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with explicit document classes and hand-written repositories.
- **Collections:** a parcels collection and a customers collection, plus `inbox` and `outbox`.
- **Migration tool:** none anywhere in the repository. The test fixture `MongoDbFixtureInitializer.cs` seeds data directly instead.
- **Cross-domain coupling:** this service keeps its **own copy of the customer**, `Core/Entities/Customer.cs` with `ICustomerRepository`, filled from the `customer_created` event. `Pacco.Services.Orders` keeps a second, independent copy. There are no foreign keys; the three copies of a customer are kept in step only by events. A parcel also carries the order identifier it was added to, updated by reacting to order events.
- **Cache:** Redis.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `parcels`, snake-case naming, queue template `parcels-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`. Transactional outbox and inbox on MongoDB (`inbox`, `outbox`).

**Commands consumed:** `add_parcel`, `delete_parcel`.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `parcel_added` | `Application/Events/ParcelAdded.cs` | parcel identifier, customer identifier |
| `parcel_deleted` | `Application/Events/ParcelDeleted.cs` | parcel identifier |

**Rejected events published:** `add_parcel_rejected`, `delete_parcel_rejected`.

**External events consumed** (five, each with a handler in `Application/Events/External/Handlers/`): `customer_created` from `customers`, and `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order` from `orders`. The order events are how a parcel learns it has been attached to or released from an order.

Wire names are confirmed against `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `parcels/volume` | `GetParcelsVolume` |
| `GET` | `parcels` | `GetParcels` |
| `GET` | `parcels/{parcelId}` | `GetParcel` |
| `POST` | `parcels` | `AddParcel`, responds `Created` |
| `DELETE` | `parcels/{parcelId}` | `DeleteParcel` |

Note the route ordering: `parcels/volume` is registered before `parcels/{parcelId}` so that the literal segment is matched first.

Called by: `Pacco.APIGateway` (module `parcels`; the list route is rewritten to `parcels-service/parcels?customerId=@user_id`) and by `Pacco.Services.Orders` through its `ParcelsServiceClient`.

This service consumes no other service's HTTP API.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.parcels`, published `5007:80` per the platform port map, `restart: unless-stopped`, network `pacco`. Consul registration on port `5007`.

Settings files: `appsettings.json`, plus `appsettings.test.json` used by `scripts/start-test.sh` and by the provider contract tests. The test settings point at database `test-parcels-service` and set `consul`, `fabio`, `vault`, `jaeger` and `metrics` all to `enabled: false`, so the tests run without any shared infrastructure.

CI: `.travis.yml` runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` on success — and the test step has the provider contract tests to run.

## 10. Security / auth clues

- JWT validation following the platform pattern, `validIssuer: pacco`.
- Vault: KV path `parcels-service/settings`, PKI role for `parcels-service`.
- The gateway rewrites the parcel list route to filter by the signed-in user, so ownership is enforced at the edge. A caller reaching this service directly on the platform network could list any customer's parcels. **Needs validation.**
- No caller access-control list is defined here, even though `Pacco.Services.Orders` calls it directly.

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: parcels`, including RabbitMQ span propagation; structured logging via `Convey.Logging` and `Convey.Logging.CQRS` with a message-to-log-template mapper; Prometheus metrics via `Convey.Metrics.AppMetrics`. All of these are switched off in the test configuration.

## 12. Files carrying major architecture decisions; feature flags

- `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs` — the provider half of the platform's only consumer-driven contract; paired with `Pacco.Services.Orders`.
- `src/Pacco.Services.Parcels.Api/Program.cs` — the decision to expose the host builder publicly so tests can run the real service in process.
- `src/Pacco.Services.Parcels.Api/appsettings.test.json` — the decision to make every shared dependency optional for testing.
- `src/Pacco.Services.Parcels.Core/Services/ParcelsService.cs` — the parcel sizing and volume rules.
- `src/Pacco.Services.Parcels.Core/Entities/Customer.cs` — the decision to replicate customer data locally.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json` and `appsettings.test.json`, which are deployment and test configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` and `tests/` contain only C# projects. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes a parcels service on .NET Core 3.1 built with Convey, following clean architecture | Confirmed: four layered projects, Convey `0.4.*`, `netcoreapp3.1` | Confirmed |
| The platform is described as event-driven with independent data per service | Confirmed, and this service holds the platform's third copy of customer data | Confirmed |
| Contract testing is mentioned as a platform practice | It exists on exactly one boundary, between the orders service as consumer and this service as provider | Stale doc — it is a single worked example, not a platform-wide practice |
| The platform README describes a uniform test approach | Test coverage across the platform is uneven: this repository has provider contract tests only, and five of the twelve service repositories have no tests at all | Stale doc |

**Docs-only claims:** none identified.
**Disk-only components:** the volume query, the separate test settings file and `scripts/start-test.sh`, and the MongoDB seeding fixtures — present on disk, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The copy of customer data held here is kept up to date only by messages from the customers service. | The only place it is written is the handler for the customer-created message. |
| A2 | Restricting a customer to their own parcels is done by the gateway, not by this service. | The filter is written into the gateway routing file, and no equivalent check appears in the service code inspected. |
| A3 | The separate test settings file exists so the contract tests can run without the shared infrastructure. | It points at its own database and turns every shared dependency off. |

### Blockers

_None._

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Can a caller inside the platform network list another customer's parcels by calling this service directly? | **[ACTION NOW]** Confirm with the requesting team, since ownership checking currently sits only at the edge. |
| Q2 | Is the single contract between the orders service and this one meant to be extended to the other service boundaries? | **[handled later by the ADR authoring stage]** Record the intended scope of contract testing. |
| Q3 | What keeps the three separate copies of a customer in step if a message is lost? | **[handled later by the ADR authoring stage]** Record how duplicated customer data is repaired. |
