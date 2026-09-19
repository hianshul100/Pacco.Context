# Repository: `Pacco.Services.Deliveries`

`deliveries-service` (also known as: Deliveries Service, `Pacco.Services.Deliveries`, Docker image
`devmentors/pacco.services.deliveries`) owns the delivery lifecycle and the registration scans
recorded against a delivery in transit.

- **Repository:** `Pacco.Services.Deliveries`, path: `src/Pacco.Services.Deliveries.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5003`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- The four-project clean-architecture split.
- `Core/Entities/Delivery.cs` and `Core/Entities/DeliveryStatus.cs`.
- All five HTTP endpoints and the RabbitMQ `deliveries` exchange with its 11 messages.
- MongoDB database `deliveries-service`, collection `deliveries`, and the nested delivery
  registration documents.
- That this service subscribes to **no** external event — it is driven only by direct commands.

**Stale doc:** none identified.

**Unknown:** what starts a delivery in practice. No service publishes a `deliveries` command and
this service subscribes to nothing, so the only observable trigger is a caller hitting
`POST /deliveries` through the gateway.

---

## 1. Primary purpose

Track a delivery from start to a terminal outcome (completed or failed), and accumulate
registration events — the scan-style records made against a delivery as it progresses — announcing
each transition to the rest of the platform.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process, using Convey CQRS dispatcher
endpoints.

## 3. Key entrypoints

- `src/Pacco.Services.Deliveries.Api/Program.cs` — composition root and route table.
- `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` — DI composition root.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Project | Role |
|---|---|
| `Pacco.Services.Deliveries.Api` | Host, route table, configuration, `certs/` |
| `Pacco.Services.Deliveries.Application` | Commands (`StartDelivery`, `CompleteDelivery`, `FailDelivery`, `AddDeliveryRegistration`), queries, events, DTOs, handlers |
| `Pacco.Services.Deliveries.Core` | `Entities/Delivery.cs`, `Entities/DeliveryStatus.cs`, `Repositories/IDeliveryRepository.cs`, domain exceptions |
| `Pacco.Services.Deliveries.Infrastructure` | Mongo documents and repository, RabbitMQ broker, decorators, contexts, logging |

**Key packages:** `Convey`, `Convey.CQRS.Commands/.Events/.Queries`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Swagger`.

**Note:** unlike `availability-service` and `customers-service`, this repository does **not**
reference `Convey.WebApi.Security`, even though its `appsettings.json` carries a
`security.certificate` section. **Needs validation** — the certificate configuration may be inert
here.

## 5. External integrations

Consul (registration, `pingEndpoint: ping`), Fabio, RabbitMQ (exchange `deliveries`), MongoDB
(database `deliveries-service`), Redis (prefix `deliveries:`), Vault (kv v2 path
`deliveries-service/settings`, PKI role `deliveries-service`, common name
`deliveries-service.pacco.io`, MongoDB dynamic credentials), Jaeger, Seq, Prometheus.

**It calls no other service.** `httpClient.services` is empty — a leaf in the synchronous call
graph.

## 6. Data stores / state

- **Store:** MongoDB, database `deliveries-service`.
- **Query mechanism:** Convey `IMongoRepository<DeliveryDocument, Guid>` over the MongoDB .NET
  driver. **Not a relational ORM.**
- **Registration:** `AddMongoRepository<DeliveryDocument, Guid>("deliveries")` in
  `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs`.
- **Collection for the primary domain:** **`deliveries`**, mapped by
  `Infrastructure/Mongo/Documents/DeliveryDocument.cs`, with `DeliveryRegistrationDocument`
  embedded as a nested collection — registrations are part of the delivery document, not a
  separate collection, so the delivery is the consistency boundary.
- **Framework collections:** `inbox`, `outbox` (`type: sequential`, `disableTransactions: true`).
- **Migration tool:** **none.** No migration files or tooling in the repository.
- **Cross-domain coupling:** `DeliveryDocument` carries an `OrderId` originating from
  `orders-service`. MongoDB has no foreign keys, so this is an identifier reference with no
  enforcement: nothing validates that the order exists, and this service neither calls
  `orders-service` nor subscribes to any `orders` event. The relationship is one-way and
  unverified at the data layer. Unlike `orders-service` and `parcels-service`, this service keeps
  **no replicated `customers` collection**.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `deliveries`, type `topic`, durable.
- **Conventions:** `snakeCase`; queue template `deliveries-service/{{exchange}}.{{message}}`;
  headers `message_context` and `span_context`.
- **Outbox:** enabled (`AddMessageOutbox(o => o.AddMongo())`) with outbox decorators on handlers.

**Commands consumed:** `add_delivery_registration`, `complete_delivery`, `fail_delivery`,
`start_delivery`.

**Events published:**

| Event | Observable payload fields |
|---|---|
| `delivery_started` | `DeliveryId`, `OrderId` |
| `delivery_completed` | `DeliveryId`, `OrderId` |
| `delivery_failed` | `DeliveryId`, `OrderId`, `Reason` |
| `registration_added_to_delivery` | `DeliveryId` and the registration details |

**Rejected events published:** `complete_delivery_rejected`, `fail_delivery_rejected`,
`start_delivery_rejected` — each with `Reason` and `Code`.
**Note:** `messages.json` declares no `add_delivery_registration_rejected`, so a failed
registration has no rejection message. **Needs validation.**

**External events consumed:** **none.** There is no `Events/External/Handlers/` content in this
repository — the only service in the platform with a broker connection and no inbound event
subscriptions.

**Consumers of this service's events:** all three lifecycle events (`delivery_started`,
`delivery_completed`, `delivery_failed`) are consumed by `orders-service`;
`registration_added_to_delivery` has **no domain consumer** anywhere in the workspace.
`operations-service` observes all of them.

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Deliveries.Api/Program.cs`, verbatim):

| Method | Route | Dispatches |
|---|---|---|
| `GET` | `deliveries/{deliveryId}` | query — single delivery |
| `POST` | `deliveries` | `StartDelivery` |
| `POST` | `deliveries/{deliveryId}/fail` | `FailDelivery` |
| `POST` | `deliveries/{deliveryId}/complete` | `CompleteDelivery` |
| `POST` | `deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` |

Swagger UI at route prefix `docs`.

**Consumed:** none.

**Inbound synchronous callers:** none.

**Upstream:** the gateway module `deliveries` fronts all five routes; in async mode the four write
routes arrive as RabbitMQ commands on the `deliveries` exchange instead.

**Note:** there is no route to list deliveries — only fetch-by-id. A caller that does not already
hold a `deliveryId` has no way to discover one through this API. **Needs validation.**

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.deliveries`.
- Port `5003` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5003:80`), and the
  gateway `localUrl`.
- Consul service name `deliveries-service`; `httpClient.type: fabio` (configured but unused, since
  the service calls nothing).
- Environments: `appsettings.json`, `.local.json`, `.docker.json`.

## 10. Security / auth clues

- **JWT bearer** with `certs/localhost.cer`, `validIssuer: pacco`.
- `security.certificate` section present in `appsettings.json` with
  `header: Certificate`, but **`Convey.WebApi.Security` is not referenced** by any project in this
  repository. Whether certificate checking is active here is **Unknown — needs validation**.
- **No certificate ACL** — nothing grants another service named access, and no service calls this
  one synchronously, so no ACL is currently needed.
- **Vault:** kv v2 settings, PKI role `deliveries-service`, MongoDB dynamic credentials with lease
  auto-renewal.
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties.
- **No role gate at the edge.** The gateway's `deliveries` module applies no `claims` restriction,
  so any authenticated caller can start, fail or complete any delivery by id.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: deliveries`, UDP `6831`, `const` sampler rate 1, with the
  `Convey.Tracing.Jaeger.RabbitMQ` plugin propagating `span_context` across AMQP.
- **Logging:** console, file and Seq sinks enabled; ELK sink present but `enabled: false`.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Deliveries.sln` | Four-project clean-architecture split |
| `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` | Capability chain and outbox decorators on every command and event handler |
| `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs`, `Entities/DeliveryStatus.cs` | That a delivery is a status machine owning its registrations, making the delivery document the consistency boundary |
| `src/Pacco.Services.Deliveries.Api/Program.cs` | That delivery transitions are separate `POST` sub-resources (`/fail`, `/complete`, `/registrations`) rather than a state field update — an explicitly command-shaped API |
| `src/Pacco.Services.Deliveries.Api/appsettings.json` | Exchange, outbox with `disableTransactions: true`, Vault PKI, and the (possibly inert) certificate section |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. What triggers `start_delivery` — no service publishes it and this service subscribes to nothing.
2. Whether the `security.certificate` configuration does anything without
   `Convey.WebApi.Security`.
3. Why `registration_added_to_delivery` has no consumer.
4. Why there is no `add_delivery_registration_rejected` message when the other three commands each
   have a rejection.
5. Whether the absence of a delivery-list endpoint is deliberate.
6. Whether `outbox.disableTransactions: true` is deliberate.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Deliveries.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Deliveries.Application/`,
`src/Pacco.Services.Deliveries.Core/`, `src/Pacco.Services.Deliveries.Infrastructure/`, and the
repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor). The only browser-facing surface is the Convey Swagger UI at
`/docs`, generated by `Convey.WebApi.Swagger`.

---

## Evidence

| Fact | File |
|---|---|
| Route table and host composition | `src/Pacco.Services.Deliveries.Api/Program.cs` |
| DI composition, Mongo collection registration, outbox decorators | `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` |
| Delivery status machine | `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs`, `Entities/DeliveryStatus.cs` |
| Persistence documents, nested registrations | `src/Pacco.Services.Deliveries.Infrastructure/Mongo/Documents/DeliveryDocument.cs`, `DeliveryRegistrationDocument.cs` |
| Commands | `src/Pacco.Services.Deliveries.Application/Commands/StartDelivery.cs`, `CompleteDelivery.cs`, `FailDelivery.cs`, `AddDeliveryRegistration.cs` |
| Published events and payloads | `src/Pacco.Services.Deliveries.Application/Events/*.cs` |
| Rejected events | `src/Pacco.Services.Deliveries.Application/Events/Rejected/*.cs` |
| Absence of external event subscriptions | `src/Pacco.Services.Deliveries.Application/Events/` (no `External/Handlers` content) |
| Exchange, outbox, Vault, JWT, logging, metrics, tracing configuration | `src/Pacco.Services.Deliveries.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Package set, absence of `Convey.WebApi.Security` | `src/Pacco.Services.Deliveries.Infrastructure/Pacco.Services.Deliveries.Infrastructure.csproj`, `src/Pacco.Services.Deliveries.Api/Pacco.Services.Deliveries.Api.csproj` |
| Project list | `Pacco.Services.Deliveries.sln` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Consumers of delivery events | `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Events/External/Handlers/` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Gateway routes and absence of a role gate | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Deliveries are started by a caller going through the gateway, not by an automated reaction to an order | This service subscribes to no event and no other service publishes a `deliveries` command; the only route in is `POST /deliveries` | If an external dispatch system is meant to drive deliveries, that integration is entirely missing from this inventory | Ask the Deliveries domain owner what calls `POST /deliveries` in practice |
| A2 | Delivery registrations live inside the delivery document rather than in their own collection | Only `deliveries` is registered as a repository, and `DeliveryRegistrationDocument` appears as a nested type | Statements about the delivery consistency boundary would be wrong, and registration volume per delivery would have different scaling behaviour | Inspect a delivery document in a running MongoDB instance |
| A3 | The `security.certificate` configuration in this service is inert | `Convey.WebApi.Security` is not referenced by any project here, unlike in `availability-service` and `customers-service` | If certificate checking is active, this service has an access control mechanism this document does not describe | Read the Convey 0.4 security wiring, or call an endpoint without a certificate against a running instance |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Who or what starts a delivery? | This is the only broker-connected service that reacts to nothing. An order can be approved and a vehicle assigned, and no delivery is created — somebody or something outside the code has to call `POST /deliveries`. If that actor is a system we have not seen, a whole integration is undiscovered | Either a human operator drives it, or an intended automatic reaction to `order_approved` was never built | Domain owner for Deliveries |
| Q2 | **[ACTION NOW]** Should any service consume `registration_added_to_delivery`? | Registration scans are published to the broker and nobody listens. A customer tracking a parcel has no service that turns those scans into visible progress — only `operations-service` sees them, and only as raw operation traffic | Either a tracking consumer is missing, or registrations are meant purely as an internal audit trail | Domain owner for Deliveries |
| Q3 | **[ACTION NOW]** Should the delivery write routes be restricted to specific roles? | The gateway applies no role gate to `deliveries`, so any authenticated user who knows a delivery id can mark that delivery failed or completed. Vehicle and customer-state routes are admin-gated; these are not | Likely an oversight relative to the other admin-gated routes, but the intent is not stated anywhere | Whoever owns Pacco authentication |
| Q4 | **[handled later by HLD]** Why does `add_delivery_registration` have no matching rejected event? | The other three commands each publish a rejection when they fail, which is how a caller in async mode learns the outcome. A failed registration is silent — the caller waits for a result that never arrives | Add the missing rejected event, or confirm registrations cannot fail | Domain owner for Deliveries |
| Q5 | **[handled later by HLD]** Is `outbox.disableTransactions: true` the intended setting? | Without transactions, the delivery write and the outbox write are not atomic, so a crash between them can complete a delivery that `orders-service` never hears about — leaving the order stuck in a delivering state | Likely a single-node MongoDB constraint in development; confirm the production topology | Platform architect |
