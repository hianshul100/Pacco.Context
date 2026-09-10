# Repository summary — `hianshul100_Pacco.Services.Availability`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Availability` (also known as: Pacco.Services.Availability, the Availability service)
**Deployable:** `Pacco.Services.Availability.Api` (also known as: `availability-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.availability`, its published image). Repository: `hianshul100_Pacco.Services.Availability`, path: `src/Pacco.Services.Availability.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

Owns *resources* and their reservations. A resource is anything with limited availability that an order can consume; the service records the resource's tags and the set of reservations held against it on particular dates. This is the domain the whole platform is named around, and it is also the reference implementation the other services copy: it is the only repository here with a full test pyramid.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, built with the Convey framework. HTTP endpoints are wired through `Convey.WebApi.CQRS` dispatcher endpoints rather than MVC controllers, and the same service also runs as a RabbitMQ consumer in the same process.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Availability.Api/Program.cs` |
| Local run | `scripts/start.sh` — sets `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Availability.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Availability.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Availability.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Availability.rest` |

HTTP routes registered in `Program.cs` via `UseDispatcherEndpoints`:

| Method | Route | Dispatches to |
|---|---|---|
| GET | `` (root) | service name banner |
| GET | `resources` | query returning resources |
| GET | `resources/{resourceId}` | query returning one resource |
| POST | `resources` | `AddResource`, responds Created at `resources/{id}` |
| POST | `resources/{resourceId}/reservations/{dateTime}` | `ReserveResource` |
| DELETE | `resources/{resourceId}/reservations/{dateTime}` | `ReleaseResourceReservation` |
| DELETE | `resources/{resourceId}` | `DeleteResource` |

`Program.cs` also calls `.UseLogging()` and `.UseVault()`.

## 4. Important modules / packages

Four source projects plus five test projects, all listed in `Pacco.Services.Availability.sln`:

| Project | Path | Role |
|---|---|---|
| `Pacco.Services.Availability.Api` | `src/Pacco.Services.Availability.Api` | host, endpoint registration, configuration |
| `Pacco.Services.Availability.Application` | `src/Pacco.Services.Availability.Application` | commands, events, queries, DTOs, handlers, service clients interface |
| `Pacco.Services.Availability.Core` | `src/Pacco.Services.Availability.Core` | `Resource` aggregate, `AggregateRoot`, `AggregateId`, domain events, domain exceptions, repository interface |
| `Pacco.Services.Availability.Infrastructure` | `src/Pacco.Services.Availability.Infrastructure` | Mongo documents and repositories, RabbitMQ broker, decorators, Jaeger, logging, metrics, HTTP clients, contexts |
| `Pacco.Services.Availability.Tests.Unit` | `tests/` | NSubstitute 4.2.1, Shouldly 3.0.2, coverlet.collector 1.2.1 |
| `Pacco.Services.Availability.Tests.Integration` | `tests/` | xunit 2.4.1, Mvc.Testing 3.1.3, TestHost |
| `Pacco.Services.Availability.Tests.EndToEnd` | `tests/` | xunit, Mvc.Testing, TestHost |
| `Pacco.Services.Availability.Tests.Performance` | `tests/` | **NBomber 0.16.0**, NBomber.Http 0.16.0 |
| `Pacco.Services.Availability.Tests.Shared` | `tests/` | Convey.MessageBrokers.RabbitMQ, Convey.Persistence.MongoDB, Mvc.Testing — factories, fixtures, helpers |

Infrastructure sub-folders worth naming: `Infrastructure/Contexts`, `Decorators`, `Exceptions`, `Jaeger`, `Logging`, `Metrics`, `Mongo/{Documents,Queries/Handlers,Repositories}`, `Services/Clients`, `Tests`.

Convey composition, from `Infrastructure/Extensions.cs` → `AddInfrastructure()`: `.AddErrorHandler<ExceptionToResponseMapper>().AddQueryHandlers().AddInMemoryQueryDispatcher().AddHttpClient().AddConsul().AddFabio().AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin()).AddMessageOutbox(o => o.AddMongo()).AddExceptionToMessageMapper<ExceptionToMessageMapper>().AddMongo().AddRedis().AddMetrics().AddJaeger().AddJaegerDecorators().AddHandlersLogging().AddMongoRepository<ResourceDocument, Guid>("resources").AddWebApiSwaggerDocs().AddCertificateAuthentication().AddSecurity()`. It also registers `IEventMapper`, `IMessageBroker`, `IResourcesRepository → ResourcesMongoRepository`, `ICustomersServiceClient → CustomersServiceClient`, `IDateTimeProvider`, `IAppContextFactory`, `IEventProcessor`, a hosted `MetricsJob`, `CustomMetricsMiddleware`, and decorates every `ICommandHandler<>` and `IEventHandler<>` with the outbox decorators.

`UseInfrastructure()`: `.UseErrorHandler().UseSwaggerDocs().UseJaeger().UseConvey().UsePublicContracts<ContractAttribute>().UseMetrics().UseMiddleware<CustomMetricsMiddleware>().UseCertificateAuthentication().UseRabbitMq()` plus the subscriptions listed below.

## 5. External integrations

| Integration | Detail |
|---|---|
| MongoDB | database `availability-service` |
| Redis | instance prefix `availability:` |
| RabbitMQ | exchange `availability` |
| Consul | service `availability-service`, port 5001, ping endpoint `ping` |
| Fabio | `http://localhost:9999`, used as the HTTP client type with 3 retries |
| Vault | kv v2 mount `kv`, path `availability-service/settings`; PKI role `availability-service`, common name `availability.pacco.io`; dynamic MongoDB credentials with auto-renewal |
| Jaeger | service name `availability`, UDP `localhost:6831`, `sampler: const` |
| Prometheus / Seq | metrics `prometheus: true`, `influx: false`, database `pacco`; logger console + file + Seq enabled, ELK disabled |
| `customers-service` | declared in `httpClient.services` as `customers: customers-service` |

## 6. Data stores and state handling

**Store:** MongoDB, database `availability-service`.
**Query mechanism:** Convey's MongoDB repository abstraction over the official MongoDB .NET driver. **There is no ORM.** Documents are plain C# classes mapped by the driver's conventions.
**Migration tool: none.** No Alembic, Flyway, Liquibase, EF Core migrations or equivalent exists in this repository or anywhere in the platform.

**Collections:**

| Collection | Registered at | Fields |
|---|---|---|
| `resources` | `AddMongoRepository<ResourceDocument, Guid>("resources")` | `Id` (Guid), `Version` (int), `Tags` (string collection), `Reservations` (embedded `ReservationDocument` collection) |
| embedded `ReservationDocument` | `Infrastructure/Mongo/Documents/ReservationDocument.cs` | `TimeStamp` (int), `Priority` (int) |
| `inbox` | outbox settings | Convey inbox, message de-duplication |
| `outbox` | outbox settings | Convey outbox, pending publishes |

Outbox settings: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, **`disableTransactions: true`**. With transactions disabled the state write and the outbox write are not atomic, so exactly-once delivery is not guaranteed; the inbox de-duplicates on the consuming side instead.

**Cross-domain foreign-key coupling.** There is no foreign key in the database sense — MongoDB does not enforce them. The logical couplings are the `Guid` identifiers stored in `ReservationDocument`'s owning aggregate, which point at customers owned by `customers-service`. Availability does **not** keep a replicated customer read model, unlike Orders and Parcels; it asks `customers-service` over HTTP instead. Redis is present in the composition but no cache key scheme is declared in code, so what is cached is **unknown — requires runtime capture**.

Concurrency: `ResourceDocument.Version` implies optimistic concurrency on the `Resource` aggregate.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ, topic exchanges, through `Convey.MessageBrokers.RabbitMQ` with the transactional outbox in `Convey.MessageBrokers.Outbox.Mongo`, plus the Jaeger RabbitMQ plugin for trace propagation.

**Broker settings:** exchange `availability`, `conventionsCasing: snakeCase`, queue name template `availability-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `availability`** (names from `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`, the platform's message catalogue):

- Events: `resource_added`, `resource_deleted`, `resource_reservation_released`, `resource_reservation_canceled`, `resource_reserved`
- Rejected events: `add_resource_rejected`, `delete_resource_rejected`, `release_resource_rejected`, `reserve_resource_rejected`

Observable payload: `ResourceReserved { Guid ResourceId, DateTime DateTime }`. The remaining event payloads follow the same identifier-only shape; rejected events carry `Reason` and `Code` per the platform's `IRejectedEvent` contract.

**Consumed** (from `UseInfrastructure()`):

| Kind | Message | Origin exchange |
|---|---|---|
| Command | `AddResource` | `availability` |
| Command | `DeleteResource` | `availability` |
| Command | `ReleaseResourceReservation` | `availability` |
| Command | `ReserveResource` | `availability` |
| Event | `CustomerCreated` | `customers` — `Events/External/CustomerCreated.cs`, `[Message("customers")]` |
| Event | `VehicleDeleted` | `vehicles` — `Events/External/VehicleDeleted.cs`, `[Message("vehicles")]` |

Each consumed external event is redeclared as a local class in this repository. There is no shared contracts package anywhere in the platform, so every consumer keeps its own copy of every producer's contract.

`GetCorrelationContext` reads the `Correlation-Context` header; `GetHeadersToForward` forwards the `Saga` header, which is how the OrderMaker saga's messages stay tagged as they pass through.

## 8. APIs exposed and consumed

**Exposed:** the seven HTTP routes in section 3, plus Swagger at route prefix `docs` and a `ping` health endpoint used by Consul. Base URL locally `http://localhost:5001`; port 80 in the container.

**Consumed:** `GET {customers-service}/customers/{customerId}/state`, from `Infrastructure/Services/Clients/CustomersServiceClient.cs`, resolved through Fabio. That call sets a certificate header before sending — see section 10.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; `dotnet publish src/Pacco.Services.Availability.Api -c release -o out`; `ASPNETCORE_URLS http://*:80`; `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `availability-service` on host port 5001.
- Configuration files: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `script: ./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`. **`./scripts/test.sh` is not run**, even though the file exists and five test projects do too. See *README vs repository*.

## 10. Security and auth clues

- `Convey.Security` plus `AddCertificateAuthentication()` / `UseCertificateAuthentication()`. Configuration adds `security.certificate.header: "Certificate"` — this is the only service in the platform that authenticates callers by certificate, and `CustomersServiceClient` sets that same header on its outbound calls.
- JWT validation configured against `certs/localhost.cer` with `validIssuer: pacco`. The certificate file is committed to the repository.
- `logger.excludeProperties` lists secret-like property names so they are kept out of log output.
- `httpClient.requestMasking` is configured, which masks sensitive request values in logs.
- Vault supplies both settings and dynamic MongoDB credentials, so database passwords are not meant to live in configuration — but the RabbitMQ credentials in `appsettings.json` are the plain defaults.

## 11. Observability, logging and tracing clues

Jaeger traces under service name `availability`, with `AddJaegerDecorators()` wrapping handlers and `AddJaegerRabbitMqPlugin()` continuing traces across message hops. Prometheus metrics through `Convey.Metrics.AppMetrics`, plus a hosted `MetricsJob` and a `CustomMetricsMiddleware` — this service emits custom business metrics that the others do not. Structured logging via `Convey.Logging` to console, file and Seq, with `/`, `/ping` and `/metrics` excluded from request logging. `AddHandlersLogging()` logs every command and event handler invocation.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Availability.Infrastructure/Extensions.cs` is the decision record for this service: it names every cross-cutting technology and every subscription in one place. `src/Pacco.Services.Availability.Api/Program.cs` fixes the HTTP surface. `src/Pacco.Services.Availability.Api/appsettings.json` fixes the integration endpoints. `src/Pacco.Services.Availability.Core/Entities/Resource.cs` holds the domain rules.

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only conditional behaviour is environment selection through `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on individual Convey integrations (`consul.enabled`, `fabio.enabled`, `metrics.enabled`, `swagger.enabled`, `logger.seq.enabled`, `logger.elk.enabled`, `vault.enabled`). Those are integration switches, not product feature flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Availability.Api/`, `src/Pacco.Services.Availability.Application/`, `src/Pacco.Services.Availability.Core/`, `src/Pacco.Services.Availability.Infrastructure/`, `tests/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** the service is part of Pacco; it runs via `dotnet run` or `./scripts/start.sh`; it listens on `http://localhost:5001`; it builds from the local `Dockerfile` or pulls as `devmentors/pacco.services.availability`; `Pacco.Services.Availability.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the five test projects and the whole test strategy, including the NBomber performance suite; the certificate authentication model; the Vault integration and dynamic database credentials; the transactional outbox; the entire messaging surface — the README describes an HTTP service and never mentions that the service consumes commands and publishes events.

**Conflicts to surface:**

- **Stale doc.** The README says to run `dotnet run` in `/src/Pacco.Services.Availability`. No such directory exists; the host project is `src/Pacco.Services.Availability.Api`. `scripts/start.sh` has the correct path, so following the script works and following the prose does not.
- **CI conflict.** `.travis.yml` runs only `./scripts/build.sh`. The repository contains `scripts/test.sh` and five test projects, none of which run in CI. The most thoroughly tested service in the platform is the one whose tests are never executed automatically.
- **Catalogue conflict.** `Application/Events/Rejected/ReleaseResourceReservationRejected.cs` exists on disk but is not listed in `messages.json`, while `messages.json` lists `reserve_resource_rejected` for which no matching class exists in this repository (`Events/Rejected/` holds `AddResourceRejected`, `DeleteResourceRejected`, `ReleaseResourceRejected` and `ReleaseResourceReservationRejected` only). The catalogue and the code disagree in both directions, so a client relying on `messages.json` will wait for a rejection that is never published and will not recognise the one that is.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | `messages.json` in the Operations repository is an accurate catalogue of what this service publishes. | It is the only enumeration of message names in the platform, and Operations builds real subscriptions from it at start-up. |
| A2 | The `Reservations` collection embedded in `ResourceDocument` carries the customer identifier that links a reservation to `customers-service`. | The `ReserveResource` command binds `customerId` at the gateway, so the value reaches the aggregate. |
| A3 | Redis is used by Convey internals rather than by this service's own code. | No cache key or cache call appears in the service's own source. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** Five test projects exist but CI never runs them. | The platform's only real test suite gives no protection against regressions. | Availability service maintainer: add `./scripts/test.sh` to `.travis.yml`. |
| B2 | **[ACTION NOW]** The outbox runs with `disableTransactions: true`. | A crash between the state write and the outbox write loses or duplicates an event, and nothing detects it. | Availability service maintainer, with the platform architect: confirm the deployment supports MongoDB transactions and turn them on, or document duplicate delivery as accepted behaviour. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Is the rejected event published as `reserve_resource_rejected` or as `release_resource_reservation_rejected`? | The message catalogue and the source disagree, so a consumer listening for a rejection may never hear one. | Availability service maintainer. |
| Q2 | **[handled later by the security-architecture stage]** Is certificate authentication meant to be adopted by the other services, or is it an experiment left in one place? | Right now exactly one of ten services authenticates its callers, which makes the platform's internal trust model inconsistent. | Security-architecture stage. |
| Q3 | **[handled later by the data-architecture stage]** What is actually stored in Redis under the `availability:` prefix? | Cannot be answered from source; it determines whether Redis is a hard dependency or an optimisation. | Data-architecture stage, by inspecting a running instance. |
| Q4 | **[handled later by the API-contract stage]** What are the full payloads of the four rejected events? | Consumers, including the Operations dashboard, display them to end users. | API-contract stage. |
