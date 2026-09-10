# Repository Summary — `hianshul100_Pacco.Services.Availability`

**Primary name:** `availability-service` (aliases used in this file: `Pacco.Services.Availability.Api` — the .NET project and assembly name; `devmentors/pacco.services.availability` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Availability`, path: `src/Pacco.Services.Availability.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Owns *resources* and their reservations. A resource is a bookable thing with a set of tags and a calendar of reservations; the service adds resources, reserves them for a given date, releases reservations and deletes resources. It is the service the platform's "limited availability" domain concept lives in.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) built on **Convey** `0.4.*`, plus a RabbitMQ message consumer running in the same process. Layered as Clean Architecture / DDD: `.Api`, `.Application`, `.Core`, `.Infrastructure`.

It is the most fully developed service in the platform: it is the only one with a complete test pyramid (`Tests.Unit`, `Tests.Integration`, `Tests.EndToEnd`, `Tests.Performance`, `Tests.Shared`).

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Availability.Api/Program.cs` |
| Test-facing host builder | `CreateWebHostBuilder` in the same file, exposed so the end-to-end tests can boot the host |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Availability.Api.dll` |
| Local run script | `scripts/start.sh` |
| Dependency wiring | `src/Pacco.Services.Availability.Infrastructure/Extensions.cs` |

`Program.cs` chains `AddConvey().AddWebApi().AddApplication().AddInfrastructure()` and `.UseLogging().UseVault()`, then declares its routes with `UseDispatcherEndpoints`.

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Availability.sln`:

- `src/Pacco.Services.Availability.Api` — host, route table, composition root.
- `src/Pacco.Services.Availability.Application` — commands, queries, events, handlers, DTOs. References `Convey.CQRS.Commands`, `.Events`, `.Queries`, `Convey.MessageBrokers`, `Microsoft.Extensions.Logging 3.1.3`.
- `src/Pacco.Services.Availability.Core` — the domain model. **No NuGet package references at all** — a deliberately dependency-free domain layer.
- `src/Pacco.Services.Availability.Infrastructure` — the full Convey stack, Mongo documents and repositories, HTTP clients, decorators, exception mappers.

Five test projects: `tests/Pacco.Services.Availability.Tests.Unit`, `.Tests.Integration`, `.Tests.EndToEnd`, `.Tests.Performance`, `.Tests.Shared`.

## 5. External integrations

- **`customers-service`** over HTTP through Fabio. `ICustomersServiceClient` → `CustomersServiceClient`, registered in `src/Pacco.Services.Availability.Infrastructure/Extensions.cs`; the address comes from `httpClient.services` in `appsettings.json`, which maps `customers` → `customers-service`.
- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — all through Convey extensions.

## 6. Data stores & state

- **MongoDB.** Database `availability-service`, connection string `mongodb://localhost:27017` in `src/Pacco.Services.Availability.Api/appsettings.json`, `seed: false`.
- **Collection:** `resources`, declared by `.AddMongoRepository<ResourceDocument, Guid>("resources")` in `Infrastructure/Extensions.cs`. Document type `ResourceDocument`, identifier type `Guid`.
- **Outbox collections:** `outbox` and `inbox` in the same database, from the `outbox` block in `appsettings.json` (`enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `redis.connectionString: localhost`, key prefix `availability:`. Used as a cache; no schema.
- **Query mechanism:** Convey's `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver, implemented as `ResourcesMongoRepository` behind the domain interface `IResourcesRepository`. **There is no ORM.**
- **Migration tool:** none. MongoDB collections are created on first write; there is no Flyway, Liquibase, Entity Framework migration or equivalent anywhere in this repository.
- **Cross-domain coupling:** none in the data layer. This service stores no copy of another domain's entities. It reads customer data synchronously over HTTP instead.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange, via `Convey.MessageBrokers.RabbitMQ`. Settings in `appsettings.json`: exchange `availability`, `conventionsCasing: snakeCase`, durable, queue template `availability-service/{{exchange}}.{{message}}`, vhost `/`, port `5672`, credentials `guest`/`guest`, `context.header: message_context`, `spanContextHeader: span_context`. Wired with `.AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())` and a Mongo-backed outbox, `.AddMessageOutbox(o => o.AddMongo())`.

**Consumed** (`UseInfrastructure()` in `src/Pacco.Services.Availability.Infrastructure/Extensions.cs`, lines 107–112):

| Kind | Message | Wire name |
|---|---|---|
| Command | `AddResource` | `add_resource` |
| Command | `DeleteResource` | `delete_resource` |
| Command | `ReleaseResourceReservation` | `release_resource` |
| Command | `ReserveResource` | `reserve_resource` |
| Event | `CustomerCreated` | `customer_created` (published by `customers-service`) |
| Event | `VehicleDeleted` | `vehicle_deleted` (published by `vehicles-service`) |

**Published**, per the `availability-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: `resource_added`, `resource_deleted`, `resource_reservation_released`, `resource_reservation_canceled`, `resource_reserved`; rejection events `add_resource_rejected`, `delete_resource_rejected`, `release_resource_rejected`, `reserve_resource_rejected`.

**Payload key fields:** the identifiers bound by the API gateway for this exchange are `resourceId` and `dateTime` (`ntrada-async.yml`). The complete field set of each message body is defined by the command and event classes under `src/Pacco.Services.Availability.Application/`; those classes were not enumerated field by field for this inventory, so the full payload shape is **unknown — requires runtime capture** or a per-class read.

Reliability: outbox and inbox decorators are applied to every handler — `TryDecorate(ICommandHandler<>, OutboxCommandHandlerDecorator<>)` and `TryDecorate(IEventHandler<>, OutboxEventHandlerDecorator<>)`.

## 8. APIs exposed & consumed

**Exposed** — declared with `UseDispatcherEndpoints` in `src/Pacco.Services.Availability.Api/Program.cs`:

| Method | Path | Dispatched to | Response |
|---|---|---|---|
| GET | `""` | — | returns `AppOptions.Name` |
| GET | `resources` | `GetResources` | `IEnumerable<ResourceDto>` |
| GET | `resources/{resourceId}` | `GetResource` | `ResourceDto` |
| POST | `resources` | `AddResource` | `201 Created`, `Location: resources/{cmd.ResourceId}` |
| POST | `resources/{resourceId}/reservations/{dateTime}` | `ReserveResource` | — |
| DELETE | `resources/{resourceId}/reservations/{dateTime}` | `ReleaseResourceReservation` | — |
| DELETE | `resources/{resourceId}` | `DeleteResource` | — |

Swagger UI at `docs` (`swagger.routePrefix: docs`, `includeSecurity: true`). Health/registration endpoint `ping` for Consul.

**Consumed:** `customers-service` over HTTP, through Fabio, with 3 retries and request masking (`maskTemplate: "*****"`).

## 9. Deployment & runtime clues

- `Dockerfile`: `mcr.microsoft.com/dotnet/core/sdk:3.1` build stage running `dotnet publish src/Pacco.Services.Availability.Api -c release -o out`, then `mcr.microsoft.com/dotnet/core/aspnet:3.1` runtime. `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh` and `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5001` (`app.service: availability-service`, Consul port `5001`).
- Service discovery: Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping endpoint `ping`, interval `3`. Load balancing: Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.availability`, started from `hianshul100_Pacco/compose/services.yml`.

## 10. Security & auth clues

- `.AddCertificateAuthentication()` and `.UseCertificateAuthentication()` — this service authenticates **to** peers with a client certificate. Its `security` block sets `certificate.header: "Certificate"`.
- Vault supplies both the certificate and the database credentials: `vault.enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, key-value path `availability-service/settings`, PKI `roleName: availability-service` with `commonName: availability-service.pacco.io`, and a `lease.mongo` block issuing dynamic MongoDB credentials with `autoRenewal: true`.
- On the receiving side, `customers-service` grants `availability-service` the permission `customers:read` in its own access-control list.
- JWT validation configured against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- **Checked-in credentials:** `vault.token: "secret"` and RabbitMQ `guest`/`guest` are literal values in `src/Pacco.Services.Availability.Api/appsettings.json`.
- `.UsePublicContracts<ContractAttribute>()` publishes the message contract surface at runtime.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: availability`, UDP `localhost:6831`, `sampler: const`. `.AddJaeger().AddJaegerDecorators()` wraps handlers in spans, and `AddJaegerRabbitMqPlugin()` carries the span across the broker via the `span_context` header.
- **Correlation:** `GetCorrelationContext` reads the `Correlation-Context` header; `GetHeadersToForward` forwards the `Saga` header — the same header the OrderMaker saga sets.
- **Logging:** Serilog-style Convey logging at level `information`, console + rolling file `logs/logs.txt` (daily) + Seq at `http://localhost:5341` with `apiKey: secret`. ELK configured but disabled (`http://localhost:9200`). `excludePaths: ["/", "/ping", "/metrics"]`. A long `excludeProperties` list redacts `api_key`, `access_key`, `ApiKey`, `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`, `Password`, `Email`, `Login`, `Secret`, `Token`. `.AddHandlersLogging()` logs every command and event handler.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`. This service adds two things no other service has: a background `MetricsJob` (`AddHostedService<MetricsJob>()`) and a `CustomMetricsMiddleware`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Availability.Infrastructure/Extensions.cs` — the canonical composition root for the whole platform. Every other service's equivalent file is a variation on this one. It encodes the decisions to use CQRS dispatchers, a Mongo-backed outbox, Consul + Fabio, certificate authentication, and Jaeger decorators.
- `src/Pacco.Services.Availability.Api/appsettings.json` — the per-service configuration contract.
- `src/Pacco.Services.Availability.Api/Program.cs` — the route table and the decision to expose `CreateWebHostBuilder` for tests.
- `Pacco.Services.Availability.rest` — worked examples of the API.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- `Program.cs` calls `.UseVault()` but `UseInfrastructure()` does not appear in the `Program.cs` chain in the same place as it does for other services; the exact ordering of Vault initialisation relative to configuration binding is **Needs validation**.
- The `Tests.Performance` project's target and thresholds were not read; what it asserts is **Unknown**.
- The full field list of each published event is **unknown — requires runtime capture**.
- This service is the only one with `MetricsJob` and `CustomMetricsMiddleware`. Whether that is a pilot intended to spread to the other services, or a one-off, is **Unknown**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Availability.Api/`, `src/Pacco.Services.Availability.Application/`, `src/Pacco.Services.Availability.Core/`, `src/Pacco.Services.Availability.Infrastructure/`, `tests/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "Service can be started locally via `dotnet run` command (executed in the `/src/Pacco.Services.Availability` directory)". | No such directory exists. The runnable project is `src/Pacco.Services.Availability.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5001`." | `appsettings.json` sets the Consul port to `5001`. | **Confirmed.** |
| `./scripts/start.sh` starts the service. | `scripts/start.sh` exists. | **Confirmed.** |
| `docker build -t pacco.services.availability .` or `docker pull devmentors/pacco.services.availability`. | `Dockerfile` is at the root; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests are listed in `Pacco.Services.Availability.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README describes the service only as "the microservice being part of Pacco solution". | The service owns resources and reservations, consumes six message types, publishes nine, calls `customers-service`, and carries a five-project test suite. | **Docs gap.** The README is a shared template and says nothing service-specific. |

**On disk but not mentioned in `README.md`:** the entire messaging surface, the outbox pattern, certificate authentication, the Vault integration, and all five test projects.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The events this service publishes are exactly those listed for `availability-service` in `messages.json`. | `messages.json` is read at runtime by `operations-service` to build its subscriptions, so it must match reality or the operations feed breaks. | A published event could be missed from the platform event catalogue. | Read the event classes under `src/Pacco.Services.Availability.Application/Events/` and compare. |
| A2 | `appsettings.json` values such as `vault.token: "secret"` and `guest`/`guest` are development defaults overridden at deployment. | The same literals appear identically in all ten services, which is the signature of a shared local-development template. | Real credentials would be committed and identical everywhere. | Check the deployment pipeline and the Vault key-value path `availability-service/settings`. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[handled later by the platform inventory review]** Are `MetricsJob` and `CustomMetricsMiddleware` meant to exist in every service, or only here? | Determines whether the platform has uniform metrics or a single instrumented service. | Looks like a pilot that was never rolled out. | Platform architect |
| Q2 | **[ACTION NOW]** Should the README's `dotnet run` instruction be corrected to `src/Pacco.Services.Availability.Api`? | The documented command does not work, and the same wrong pattern appears in every service README. | Yes — correct the path in all ten service READMEs. | Repository owner |
