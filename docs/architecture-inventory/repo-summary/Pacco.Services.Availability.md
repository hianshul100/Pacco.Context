# Repository: `Pacco.Services.Availability`

`availability-service` (also known as: Availability Service, `Pacco.Services.Availability`, Docker
image `devmentors/pacco.services.availability`) owns resources and their date-based reservations —
the "limited resources availability" concept the platform is built around.

- **Repository:** `Pacco.Services.Availability`, path: `src/Pacco.Services.Availability.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5001`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency
belonging to this service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- The four-project clean-architecture split (`.Api`, `.Application`, `.Core`, `.Infrastructure`).
- The `Resource` aggregate and its `Reservation` value object.
- All six HTTP endpoints and the RabbitMQ `availability` exchange with its 13 messages.
- The synchronous dependency on `customers-service`.
- **Vault PKI client-certificate authentication** — the most distinctive feature of this
  repository and entirely undocumented.
- MongoDB database `availability-service` and collection `resources`.
- The **five test projects**, including an NBomber performance suite — the most thorough test
  arrangement in the workspace, mentioned nowhere.

**Stale doc:** none identified — the README makes no claim specific enough to be contradicted.

**Unknown:** whether the NBomber suite is run in CI (`.travis.yml` calls `scripts/build.sh` only,
not `scripts/test.sh`).

---

## 1. Primary purpose

Maintain a catalogue of reservable resources, each carrying tags and a set of date-keyed
reservations, and arbitrate reservation and release requests — including a priority mechanism that
allows a higher-priority reservation to displace a lower-priority one.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process. Uses Convey's CQRS dispatcher
endpoints (`UseDispatcherEndpoints`), so HTTP routes map directly onto command and query objects.

## 3. Key entrypoints

- `src/Pacco.Services.Availability.Api/Program.cs` — composition root and route table. It exposes
  a `CreateWebHostBuilder` seam so the end-to-end and integration test projects can host the
  application in-process.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Availability.Api.dll`.
- `scripts/start.sh` — `ASPNETCORE_ENVIRONMENT=local`, `dotnet run` from the `.Api` project.
- `src/Pacco.Services.Availability.Infrastructure/Extensions.cs` — the DI composition root where
  every framework capability is switched on.

## 4. Important modules / packages

**Project structure** (`Pacco.Services.Availability.sln`):

| Project | Role |
|---|---|
| `Pacco.Services.Availability.Api` | Host, route table, configuration |
| `Pacco.Services.Availability.Application` | Commands, queries, events, DTOs, handlers, service interfaces |
| `Pacco.Services.Availability.Core` | Domain: `Entities/Resource.cs` (aggregate root), `Entities/Reservation.cs`, `Repositories/IResourceRepository.cs`, domain exceptions |
| `Pacco.Services.Availability.Infrastructure` | Mongo documents and repository, RabbitMQ message broker, HTTP clients, decorators, contexts, logging |

**Test projects:** `Pacco.Services.Availability.Tests.Unit`, `.Tests.Integration`,
`.Tests.EndToEnd`, `.Tests.Performance` (NBomber `0.16.0`), `.Tests.Shared` — xUnit, Shouldly,
NSubstitute.

**Key packages:** `Convey`, `Convey.CQRS.Commands`, `.CQRS.Events`, `.CQRS.Queries`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Security`, `.WebApi.Swagger`.

## 5. External integrations

| Integration | How |
|---|---|
| `customers-service` | HTTP `GET {customers}/customers/{id}/state` via `Convey.HTTP` `IHttpClient`; `httpClient.services` maps `customers` |
| HashiCorp Vault | `Convey.Secrets.Vault` — kv v2 settings at `availability-service/settings`, **PKI role `availability-service`** issuing a client certificate with common name `availability-service.pacco.io`, and MongoDB dynamic database credentials with lease auto-renewal |
| Consul | Service registration, `pingEndpoint: ping` |
| Fabio | `httpClient.type: fabio` — outbound calls resolve through the load balancer |
| RabbitMQ | Exchange `availability` |
| MongoDB | Database `availability-service` |
| Redis | Instance prefix `availability:` |
| Jaeger, Seq, Prometheus | Tracing, logs, metrics |

## 6. Data stores / state

- **Store:** MongoDB, database `availability-service`.
- **Query mechanism:** Convey `Convey.Persistence.MongoDB` — `IMongoRepository<TDocument, Guid>`,
  a thin document-mapping repository over the MongoDB .NET driver. **This is not a relational ORM**
  and there is no LINQ-to-SQL-style translation beyond the driver's own `IQueryable` support.
- **Registration:** `AddMongoRepository<ResourceDocument, Guid>("resources")` in
  `src/Pacco.Services.Availability.Infrastructure/Extensions.cs`.
- **Collections for the primary domain:** **`resources`** — documents mapped by
  `Infrastructure/Mongo/Documents/ResourceDocument.cs` with an embedded
  `ReservationDocument` collection. Reservations are **not** a separate collection; they are nested
  inside the resource document, which makes the resource the transactional boundary.
- **Framework collections:** `inbox` and `outbox` (Convey message outbox, `type: sequential`,
  `disableTransactions: true`).
- **Migration tool:** **none.** No EF Core migrations, no Flyway, Liquibase or equivalent, and no
  index-creation code in this repository. Schema evolution relies on document tolerance.
  **Needs validation.**
- **Cross-domain coupling:** MongoDB has no foreign keys. The equivalent coupling here is that
  `ResourceDocument` stores a `CustomerId` originating from `customers-service`, and the service
  additionally calls that service synchronously to read customer state before allowing a
  reservation. It does **not** replicate a `customers` collection locally — unlike `orders-service`
  and `parcels-service`.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `availability`, type `topic`, durable.
- **Conventions:** `conventionsCasing: snakeCase`; queue template
  `availability-service/{{exchange}}.{{message}}`; context header `message_context`; span header
  `span_context`.
- **Transactional outbox:** enabled — `AddMessageOutbox(o => o.AddMongo())`, collections `inbox`
  and `outbox`, with `OutboxCommandHandlerDecorator<>` and `OutboxEventHandlerDecorator<>`
  wrapping every handler.

**Commands consumed** (names verbatim): `add_resource`, `delete_resource`, `release_resource`,
`reserve_resource`.

**Events published** (names verbatim, with observable payload fields from the classes in
`Application/Events/`):

| Event | Observable payload fields |
|---|---|
| `resource_added` | `ResourceId`, `Tags` |
| `resource_deleted` | `ResourceId` |
| `resource_reserved` | `ResourceId`, `CustomerId`, `DateTime` |
| `resource_reservation_released` | `ResourceId`, `CustomerId`, `DateTime` |
| `resource_reservation_canceled` | `ResourceId`, `CustomerId`, `DateTime` |

**Rejected events published:** `add_resource_rejected`, `delete_resource_rejected`,
`release_resource_rejected`, `reserve_resource_rejected` — each carrying a `Reason` and `Code`.

**External events consumed** (`Application/Events/External/Handlers/`): `customer_created` (from
`customers-service`), `vehicle_deleted` (from `vehicles-service`, which removes the corresponding
resource).

**Naming discrepancy to flag.** `messages.json` in `operations-service` declares the command
`release_resource` and the rejected event `release_resource_rejected`, but this repository's
classes are named `ReleaseResourceReservation` and `ReleaseResourceReservationRejected`. Under
`snakeCase` conventions those serialise to `release_resource_reservation` and
`release_resource_reservation_rejected`, which do **not** match the catalogue or the gateway's
`routing_key: release_resource`. **Needs validation** — either Convey's `[Message]` attribute
overrides the derived name, or the release route does not bind. Code is authoritative and the code
shows the longer class names; the discrepancy is recorded rather than reconciled.

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Availability.Api/Program.cs`, verbatim route templates):

| Method | Route | Dispatches |
|---|---|---|
| `GET` | `resources` | query — resource list |
| `GET` | `resources/{resourceId}` | query — single resource |
| `POST` | `resources` | `AddResource` → `201 Created` |
| `POST` | `resources/{resourceId}/reservations/{dateTime}` | `ReserveResource` |
| `DELETE` | `resources/{resourceId}/reservations/{dateTime}` | `ReleaseResourceReservation` |
| `DELETE` | `resources/{resourceId}` | `DeleteResource` |

Swagger UI at route prefix `docs`.

**Consumed:** `GET {customers}/customers/{customerId}/state` on `customers-service`, via
`src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs`.

**Upstream:** the API gateway module `availability` fronts all six routes (see
`Pacco.APIGateway.md`); in async mode the four write routes arrive as RabbitMQ commands instead.

## 9. Deployment / runtime clues

- `Dockerfile`: `mcr.microsoft.com/dotnet/core/sdk:3.1` build →
  `mcr.microsoft.com/dotnet/core/aspnet:3.1` runtime; `dotnet publish src/Pacco.Services.Availability.Api -c release -o out`;
  `ASPNETCORE_URLS http://*:80`; `ASPNETCORE_ENVIRONMENT docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`, then
  `./scripts/dockerize.sh` on success.
- `scripts/dockerize.sh` pushes `$DOCKER_USERNAME/pacco.services.availability` (`master`→`latest`,
  `develop`→`dev`).
- Port `5001` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5001:80`) and the
  gateway's `localUrl`.
- Consul registration at `availability-service`; `httpClient.type: fabio`.
- Configuration environments: `appsettings.json`, `.local.json`, `.docker.json`.

## 10. Security / auth clues

- **JWT bearer** validation using `certs/localhost.cer`, `validIssuer: pacco`.
- **Client-certificate service-to-service authentication (mTLS-style, application-level).**
  `security.certificate` is configured and `Convey.WebApi.Security` is referenced.
  `CustomersServiceClient` obtains a certificate from Vault's PKI backend when
  `vaultOptions.Pki.Enabled` is true and attaches it to outbound requests in the **`Certificate`**
  header. `customers-service` holds the matching ACL granting `availability-service` the
  `customers:read` permission. This is the only proven certificate-authenticated call path on the
  platform.
- **Vault**: kv v2 mount `kv`, path `availability-service/settings`; PKI role
  `availability-service`, common name `availability-service.pacco.io`; MongoDB dynamic credentials
  with lease auto-renewal.
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties from
  log output.

## 11. Observability / logging / tracing

- **Tracing:** `Convey.Tracing.Jaeger` + `Convey.Tracing.Jaeger.RabbitMQ`; `serviceName:
  availability`; UDP `6831`; `const` sampler at rate 1 — every request is sampled.
  The RabbitMQ plugin propagates `span_context` so a trace spans the HTTP and AMQP hops.
- **Logging:** `Convey.Logging` — console, file and Seq sinks enabled; ELK sink present but
  `enabled: false`.
- **Metrics:** `Convey.Metrics.AppMetrics`, `prometheusEnabled: true`, `influxEnabled: false`,
  database `pacco`, endpoints `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Availability.sln` | The four-project clean-architecture split, with `Core` depending on nothing |
| `src/Pacco.Services.Availability.Infrastructure/Extensions.cs` | The full capability chain: Consul, Fabio, RabbitMQ with the Jaeger plugin, Mongo outbox, Redis, metrics, tracing, Swagger, security — and the outbox decorators applied to every handler |
| `src/Pacco.Services.Availability.Core/Entities/Resource.cs` | That the resource aggregate owns its reservations, making the resource the consistency boundary for reservation conflicts |
| `src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` | That cross-service calls carry a Vault-issued client certificate |
| `src/Pacco.Services.Availability.Api/appsettings.json` | Outbox with `disableTransactions: true`; Vault PKI and dynamic Mongo credentials |
| `tests/Pacco.Services.Availability.Tests.Performance/` | That this service has an explicit load-testing posture (NBomber) that no other service has |

**Feature flag system:** **none detected.** No flag library and no in-house toggle mechanism
appears in the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. Whether `release_resource` (catalogue/gateway) and `ReleaseResourceReservation` (code) bind to
   the same message name.
2. Whether `outbox.disableTransactions: true` is deliberate — with transactions disabled the
   outbox write and the domain write are not atomic, so a crash between them can lose an event.
3. Whether the Vault PKI path is exercised in any deployed environment or only configured.
4. Whether the NBomber performance suite runs anywhere — `.travis.yml` does not invoke
   `scripts/test.sh`.
5. What the reservation priority rules are meant to guarantee when a higher-priority reservation
   displaces an existing one — the displaced customer receives `resource_reservation_canceled`,
   but nothing states the intended compensation.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Availability.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Availability.Application/`,
`src/Pacco.Services.Availability.Core/`, `src/Pacco.Services.Availability.Infrastructure/`,
`tests/`, and the repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`,
`assets/`, `resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no
view templates (`.cshtml`, `.html`, Razor). The only browser-facing surface is the Convey Swagger
UI at `/docs`, which is generated by `Convey.WebApi.Swagger` and contains no first-party frontend
code.

---

## Evidence

| Fact | File |
|---|---|
| Route table, host builder test seam | `src/Pacco.Services.Availability.Api/Program.cs` |
| DI composition, Mongo collection registration, outbox decorators, capability chain | `src/Pacco.Services.Availability.Infrastructure/Extensions.cs` |
| Aggregate and reservation model | `src/Pacco.Services.Availability.Core/Entities/Resource.cs`, `Entities/Reservation.cs` |
| Persistence documents | `src/Pacco.Services.Availability.Infrastructure/Mongo/Documents/ResourceDocument.cs`, `ReservationDocument.cs` |
| Certificate-authenticated cross-service call | `src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` |
| Published events and payload fields | `src/Pacco.Services.Availability.Application/Events/*.cs` |
| Rejected events | `src/Pacco.Services.Availability.Application/Events/Rejected/*.cs` |
| Consumed external events | `src/Pacco.Services.Availability.Application/Events/External/Handlers/*.cs` |
| Commands | `src/Pacco.Services.Availability.Application/Commands/*.cs` |
| Exchange, outbox, Vault, JWT, logging, metrics, tracing configuration | `src/Pacco.Services.Availability.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Package set | `src/Pacco.Services.Availability.Infrastructure/Pacco.Services.Availability.Infrastructure.csproj`, `src/Pacco.Services.Availability.Api/Pacco.Services.Availability.Api.csproj` |
| Project list | `Pacco.Services.Availability.sln` |
| Test posture | `tests/Pacco.Services.Availability.Tests.Unit/`, `.Tests.Integration/`, `.Tests.EndToEnd/`, `.Tests.Performance/`, `.Tests.Shared/` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Certificate ACL counterpart | `../hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Convey's `snakeCase` message convention derives each wire name from its class name | This is the only naming rule visible in configuration, and it correctly predicts the names in `messages.json` for every other message in this service | The `release_resource` naming discrepancy would have a different explanation, and the analysis of which routes bind would be wrong | Read the Convey 0.4 naming implementation, or observe the exchange bindings on a running broker |
| A2 | Reservations live inside the resource document rather than in their own collection | Only `resources` is registered as a repository, and `ReservationDocument` appears as a nested type on `ResourceDocument` | Statements about the reservation consistency boundary would be wrong | Inspect a resource document in a running MongoDB instance |
| A3 | The Vault PKI certificate path is the intended production mechanism for calling `customers-service` | The client attaches a certificate when `vaultOptions.Pki.Enabled` is set, and `customers-service` carries a matching ACL entry | If the PKI path is never enabled, cross-service calls are unauthenticated in practice while appearing protected | Check whether `pki.enabled` is true in the deployed configuration |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does the release-reservation path actually work end to end? | The gateway publishes `release_resource` and the catalogue lists `release_resource`, but the handling class is `ReleaseResourceReservation`. If the derived wire name is the longer one, released reservations published through the gateway are never consumed and the resource stays blocked | Either a `[Message]` attribute overrides the name, or the binding is broken. This needs to be checked against a running broker, not read | Domain owner for Availability |
| Q2 | **[ACTION NOW]** Is `outbox.disableTransactions: true` deliberate? | With transactions off, the domain write and the outbox write are not atomic, so a crash between them silently loses an event — a reservation could succeed while nobody downstream ever hears about it | It is likely set because the development MongoDB runs as a single node without a replica set, which is a requirement for transactions. Whether production shares that constraint is unknown | Platform architect |
| Q3 | **[handled later by HLD]** What is meant to happen to a customer whose reservation is displaced by a higher-priority one? | The code publishes `resource_reservation_canceled`, but no service consumes it beyond `orders-service` cancelling the related order. Whether the customer is told, refunded or offered an alternative is undefined | Define the compensation behaviour for displaced reservations | Domain owner for Availability |
| Q4 | **[ACTION NOW]** Are the five test projects, including the NBomber load suite, run anywhere? | `.travis.yml` runs `scripts/build.sh` only; `scripts/test.sh` exists but is never invoked by CI. The most thorough test arrangement in the workspace may not be executing at all | Wire `scripts/test.sh` into CI, or confirm the tests run elsewhere | Whoever owns the Availability build |
