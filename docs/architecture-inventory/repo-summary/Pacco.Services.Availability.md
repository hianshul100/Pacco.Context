---
title: "Repository Summary — Pacco.Services.Availability"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Availability"
status: "evidence-based inventory"
---

# Pacco.Services.Availability

**Primary name:** `Pacco.Services.Availability` (aliases used in this file: `availability-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `availability` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Availability`, path: `hianshul100_Pacco.Services.Availability/`

---

## 1. Primary purpose

Owns the platform's scarce-resource model: resources (such as vehicles) and the time-slot reservations placed against them. It is the service that makes the platform's central idea — "limited resources availability" — concrete.

Evidence: `src/Pacco.Services.Availability.Core/Entities/Resource.cs`, `src/Pacco.Services.Availability.Core/ValueObjects/Reservation.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service using the Convey dispatcher endpoint style (`UseDispatcherEndpoints`), plus a RabbitMQ subscriber. Listens on `5001`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` — `Main` → `CreateWebHostBuilder` (declared public so tests can reuse it) | `src/Pacco.Services.Availability.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

`Program.cs` also calls `.UseLogging()` and `.UseVault()`.

## 4. Modules / packages

Four source projects and five test projects.

- `Pacco.Services.Availability.Core` — `Entities/AggregateId.cs`, `Entities/AggregateRoot.cs`, `Entities/Resource.cs`, `ValueObjects/Reservation.cs`, `Repositories/IResourcesRepository.cs`, domain events `ReservationAdded`, `ReservationCanceled`, `ReservationReleased`, `ResourceCreated`, `ResourceDeleted`, and exceptions including `CannotExpropriateReservationException`, `InvalidResourceTagsException`, `MissingResourceTagsException`.
- `Pacco.Services.Availability.Application` — commands, integration events, rejected events, queries, DTOs (`ResourceDto`, `ReservationDto`, `CustomerStateDto`), and `Services/Clients/ICustomersServiceClient.cs`.
- `Pacco.Services.Availability.Infrastructure` — `Mongo/Documents/ResourceDocument.cs`, `Mongo/Documents/ReservationDocument.cs`, `Mongo/Repositories/ResourcesMongoRepository.cs`, `Mongo/Queries/Handlers/GetResourceHandler.cs`, `GetResourcesHandler.cs`, `Decorators/OutboxCommandHandlerDecorator.cs`, `Decorators/OutboxEventHandlerDecorator.cs`, `Jaeger/JaegerCommandHandlerDecorator.cs`, `Logging/MessageToLogTemplateMapper.cs`, `Metrics/CustomMetricsMiddleware.cs`, `Metrics/MetricsJob.cs`, `Services/EventMapper.cs`, `Services/EventProcessor.cs`, `Services/MessageBroker.cs`, `Services/Clients/CustomersServiceClient.cs`, `Exceptions/ExceptionToMessageMapper.cs`, `Exceptions/ExceptionToResponseMapper.cs`.
- `Pacco.Services.Availability.Tests.Unit`, `.Tests.Integration`, `.Tests.EndToEnd`, `.Tests.Performance`, `.Tests.Shared`.

Convey packages in use: `Convey`, `Convey.Auth`, `Convey.CQRS.Commands`, `Convey.CQRS.Events`, `Convey.CQRS.Queries`, `Convey.Discovery.Consul`, `Convey.HTTP`, `Convey.LoadBalancing.Fabio`, `Convey.Logging`, `Convey.Logging.CQRS`, `Convey.MessageBrokers.CQRS`, `Convey.MessageBrokers.Outbox`, `Convey.MessageBrokers.Outbox.Mongo`, `Convey.MessageBrokers.RabbitMQ`, `Convey.Metrics.AppMetrics`, `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis`, `Convey.Secrets.Vault`, `Convey.Security`, `Convey.Tracing.Jaeger`, `Convey.Tracing.Jaeger.RabbitMQ`, `Convey.WebApi`, `Convey.WebApi.CQRS`, `Convey.WebApi.Security`, `Convey.WebApi.Swagger` — all `0.4.*`.

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus, and `Pacco.Services.Customers` over HTTP (`httpClient.services.customers: customers-service`, `httpClient.type: fabio`).

## 6. Data stores / state

- **Store:** MongoDB, database `availability-service`.
- **Access mechanism:** no ORM. The official MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with hand-written repository classes (`ResourcesMongoRepository`) and explicit document types (`ResourceDocument`, `ReservationDocument`).
- **Collections (the MongoDB equivalent of tables):** a resources collection derived from `ResourceDocument`, plus `inbox` and `outbox` created by the outbox configuration.
- **Migration tool:** none. There is no Entity Framework migration folder, no Flyway, no Liquibase and no equivalent anywhere in the repository. Schema is implicit in the document classes.
- **Cross-domain coupling:** no foreign keys exist, because MongoDB has none. Coupling is by identifier only — `ResourceDocument` carries the resource identifier that other services reference, and customer state is fetched over HTTP rather than joined.
- **Cache:** Redis, instance prefix `availability:`.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `availability` (durable), queue template `availability-service/{{exchange}}.{{message}}`, `conventionsCasing: "snakeCase"`, message context header `message_context`, span context header `span_context`.

**Transactional outbox and inbox:** enabled, MongoDB-backed, with `inboxCollection: "inbox"`, `outboxCollection: "outbox"`, `type: "sequential"`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Commands consumed** (`add_resource`, `delete_resource`, `release_resource`, `reserve_resource`): implemented by `AddResource`, `DeleteResource`, `ReleaseResourceReservation`, `ReserveResource` in `Application/Commands/`.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `resource_added` | `Application/Events/ResourceAdded.cs` | resource identifier |
| `resource_deleted` | `Application/Events/ResourceDeleted.cs` | resource identifier |
| `resource_reserved` | `Application/Events/ResourceReserved.cs` | `ResourceId` (Guid), `DateTime` (DateTime) — read directly from the class |
| `resource_reservation_released` | `Application/Events/ResourceReservationReleased.cs` | resource identifier, reservation date |
| `resource_reservation_canceled` | `Application/Events/ResourceReservationCanceled.cs` | resource identifier, reservation date |

**Rejected events published:** `add_resource_rejected`, `delete_resource_rejected`, `release_resource_rejected`, `reserve_resource_rejected` — classes `AddResourceRejected`, `DeleteResourceRejected`, `ReleaseResourceRejected`, `ReleaseResourceReservationRejected` in `Application/Events/Rejected/`.

**External events consumed:** `customer_created` (`Application/Events/External/CustomerCreated.cs`) and `vehicle_deleted` (`Application/Events/External/VehicleDeleted.cs`), each with a handler alongside.

Wire names are confirmed against the platform catalogue at `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. Payload fields for the classes not quoted above follow the same one-line constructor style; the exact serialised shape on the wire is **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`, dispatcher endpoints):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `` (root) | returns the configured application name |
| `GET` | `resources` | `GetResources` → `IEnumerable<ResourceDto>` |
| `GET` | `resources/{resourceId}` | `GetResource` → `ResourceDto` |
| `POST` | `resources` | `AddResource`, responds `Created` at `resources/{cmd.ResourceId}` |
| `POST` | `resources/{resourceId}/reservations/{dateTime}` | `ReserveResource` |
| `DELETE` | `resources/{resourceId}/reservations/{dateTime}` | `ReleaseResourceReservation` |
| `DELETE` | `resources/{resourceId}` | `DeleteResource` |

Consumed: `customers-service` through `Services/Clients/CustomersServiceClient.cs`, returning `CustomerStateDto`.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.availability`, published `5001:80` in `hianshul100_Pacco/compose/services.yml`, `restart: unless-stopped`, network `pacco`. Consul registration on port `5001`. Fabio provides load balancing for outbound service calls (`httpClient.type: fabio`).

CI: `.travis.yml` runs `./scripts/build.sh` and then `./scripts/dockerize.sh`.

## 10. Security / auth clues

- JWT validation using the certificate at `certs/localhost.cer`, `validIssuer: pacco`.
- `security.certificate.header: Certificate` — the service reads a client certificate from that header.
- Vault: KV path `availability-service/settings`; PKI role `availability-service` with common name `availability-service.pacco.io`; a `lease` entry issuing dynamic MongoDB credentials.
- `Convey.WebApi.Security` and `Convey.Security` are referenced.

## 11. Observability / logging / tracing

- Jaeger tracing, `serviceName: availability`, including RabbitMQ span propagation (`Convey.Tracing.Jaeger.RabbitMQ`) and a bespoke `Jaeger/JaegerCommandHandlerDecorator.cs`.
- Structured logging through `Convey.Logging` and `Convey.Logging.CQRS`, with `Logging/MessageToLogTemplateMapper.cs` mapping each message type to a log template.
- Metrics through `Convey.Metrics.AppMetrics` with `prometheusEnabled: true`, plus `Metrics/CustomMetricsMiddleware.cs` and `Metrics/MetricsJob.cs` — the only service in the platform with bespoke metrics code.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Availability.Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs` and `OutboxEventHandlerDecorator.cs` — the reliable-messaging decision.
- `src/Pacco.Services.Availability.Infrastructure/Services/EventMapper.cs`, `EventProcessor.cs`, `MessageBroker.cs` — how domain events become integration events.
- `src/Pacco.Services.Availability.Infrastructure/Exceptions/ExceptionToMessageMapper.cs` — how failures become rejected events.
- `src/Pacco.Services.Availability.Core/Entities/Resource.cs` — the reservation and expropriation rules.
- `src/Pacco.Services.Availability.Api/appsettings.json` — the full infrastructure contract.

**Feature-flag system: none.** No flag provider is referenced. The configuration contains boolean switches per integration (`consul.enabled`, `fabio.enabled`, `jaeger.enabled`, `metrics.enabled`, `outbox.enabled`, `redis`, `vault.enabled`, `swagger`), which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` and `tests/` contain only C# projects. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes an availability and reservations service on .NET Core 3.1 built with Convey | Confirmed by the project files, `Program.cs` and `appsettings.json` | Confirmed |
| `.travis.yml` in the sibling repositories runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` | This repository's `.travis.yml` **omits the `./scripts/test.sh` step**, even though it is the only repository with unit, integration, end-to-end and performance test projects | Needs validation — the richest test suite in the platform is not run in continuous integration |
| Vault is documented as supplying dynamic database credentials | Confirmed: the `vault.lease` block for MongoDB is present here and matches the Vault database secrets engine setup in `hianshul100_Pacco/docker-images.txt` | Confirmed |

**Docs-only claims:** none identified.
**Disk-only components:** `Metrics/CustomMetricsMiddleware.cs`, `Metrics/MetricsJob.cs`, the NBomber performance test project and the `MongoDbFixture` / `RabbitMqFixture` / `PaccoApplicationFactory` test harness — present on disk, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | This service is the reference example for the whole platform, and the other services are simplified copies of it. | It is the only one with a full test suite, bespoke metrics code and a complete Vault setup, and its structure is repeated elsewhere. |
| A2 | Message names on the wire are the snake-case forms listed in the shared catalogue file, not the C# class names. | The service is configured to convert names to snake case, and the shared catalogue lists exactly those forms. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | The automated tests in this repository are never executed by the build, so nobody finds out when they break. | **[ACTION NOW]** Recorded here for the requesting team; this stage does not change build files. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | What exactly does each published message contain? We can read the field names from the code, but not the final format sent over the network. | **[handled later by the ADR authoring stage]** Capture one real message per event from a running system if exact shapes are needed. |
| Q2 | The service reads a client certificate from a request header. Which caller is expected to supply it, and is it enforced? | **[ACTION NOW]** Confirm with the requesting team, because it affects how service-to-service trust is described. |
