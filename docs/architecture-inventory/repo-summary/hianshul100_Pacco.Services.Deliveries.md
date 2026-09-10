# Repository summary — `hianshul100_Pacco.Services.Deliveries`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Deliveries` (also known as: Pacco.Services.Deliveries, the Deliveries service)
**Deployable:** `Pacco.Services.Deliveries.Api` (also known as: `deliveries-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.deliveries`, its published image). Repository: `hianshul100_Pacco.Services.Deliveries`, path: `src/Pacco.Services.Deliveries.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

Tracks the physical delivery of an order: when it started, the registrations recorded along the way (a courier's notes with timestamps), and whether it ended in success or failure. It is the last stage of the order lifecycle and the source of the events that close an order.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, Convey-based, serving HTTP dispatcher endpoints and consuming RabbitMQ commands in the same process.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Deliveries.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Deliveries.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Deliveries.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Dispatches to |
|---|---|---|
| GET | `deliveries/{deliveryId}` | `GetDelivery` |
| POST | `deliveries` | `StartDelivery` |
| POST | `deliveries/{deliveryId}/fail` | `FailDelivery` |
| POST | `deliveries/{deliveryId}/complete` | `CompleteDelivery` |
| POST | `deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` |

## 4. Important modules / packages

Four projects: `src/Pacco.Services.Deliveries.Api`, `.Application`, `.Core`, `.Infrastructure`.

- **Core** — `Entities/Delivery.cs`, `Entities/DeliveryStatus.cs`, `Entities/AggregateRoot.cs`, `Entities/AggregateId.cs`; domain events `DeliveryStateChanged`, `DeliveryRegistrationAdded`.
- **Application** — commands `StartDelivery`, `CompleteDelivery`, `FailDelivery`, `AddDeliveryRegistration` with handlers; integration events `DeliveryStarted`, `DeliveryCompleted`, `DeliveryFailed`, `RegistrationAddedToDelivery`; rejected events `StartDeliveryRejected`, `CompleteDeliveryRejected`, `FailDeliveryRejected`, `AddDeliveryRegistrationRejected`; query `GetDelivery`.
- **Infrastructure** — `Mongo/Documents/DeliveryDocument.cs`, `Mongo/Documents/DeliveryRegistrationDocument.cs`, the Mongo repository and the `GetDelivery` handler, `Services/EventMapper.cs`, `Services/MessageBroker.cs`, outbox decorators, exception mappers, `Logging/MessageToLogTemplateMapper.cs`.

Packages are the platform's standard Convey `0.4.*` set, matching the other services.

## 5. External integrations

MongoDB (database `deliveries-service`), Redis (instance prefix `deliveries:`), RabbitMQ (exchange `deliveries`), Consul (service `deliveries-service`, port 5003, ping endpoint `ping`), Fabio (`http://localhost:9999`), Vault (kv v2 mount `kv`, path `deliveries-service/settings`; PKI common name `deliveries.pacco.io`; dynamic MongoDB credentials with auto-renewal), Jaeger (service name `deliveries`, UDP `localhost:6831`, `sampler: const`), Prometheus, Seq.

`httpClient.services` is **empty** — no outbound service-to-service HTTP.

## 6. Data stores and state handling

**Store:** MongoDB, database `deliveries-service`. **Query mechanism:** Convey MongoDB repositories over the MongoDB .NET driver. **No ORM. No migration tool.**

| Collection | Fields |
|---|---|
| `deliveries` | `Id` (Guid), `OrderId` (Guid), `Status` (enum `DeliveryStatus`), `Notes` (string), `Registrations` (embedded `DeliveryRegistrationDocument` collection) |
| embedded `DeliveryRegistrationDocument` | `Description` (string), `DateTime` (DateTime) |
| `inbox` | Convey inbox, de-duplication |
| `outbox` | Convey outbox, pending publishes |

Outbox: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Cross-domain coupling.** `DeliveryDocument.OrderId` points at an order owned by `orders-service`. There is no database foreign key and no replicated order read model here — the delivery simply carries the identifier and Orders reacts to the delivery events. A delivery can therefore be created for an order identifier that does not exist, and nothing in this service would notice.

`Registrations` is an unbounded embedded collection; a long-running delivery with many registrations grows the document indefinitely.

Redis is registered in the composition but no cache call appears in this service's own code, so what it holds is **unknown — requires runtime capture**.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Mongo-backed transactional outbox and the Jaeger RabbitMQ plugin.

**Broker settings:** exchange `deliveries`, `conventionsCasing: snakeCase`, queue name template `deliveries-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `deliveries`:**

- Events: `delivery_started`, `delivery_completed`, `delivery_failed`, `registration_added_to_delivery`
- Rejected events: `start_delivery_rejected`, `complete_delivery_rejected`, `fail_delivery_rejected`, and `add_delivery_registration_rejected` (on disk but missing from the platform catalogue — see *README vs repository*)
- Commands accepted on the same exchange: `start_delivery`, `complete_delivery`, `fail_delivery`, `add_delivery_registration`

Payload key fields: the delivery events carry the delivery identifier and the order identifier; `registration_added_to_delivery` additionally carries the registration description and timestamp. Exact field lists per event are **unknown — requires runtime capture** beyond what the command signatures imply, because no `[Contract]`-annotated payload was read for these classes.

**Consumed:**

| Kind | Message | Origin exchange |
|---|---|---|
| Command | `StartDelivery` | `deliveries` |
| Command | `CompleteDelivery` | `deliveries` |
| Command | `FailDelivery` | `deliveries` |
| Command | `AddDeliveryRegistration` | `deliveries` |

**This service subscribes to no events from any other service.** It is driven entirely by commands that arrive from the gateway (async profile) or by direct HTTP (sync profile). Nothing in the platform automatically starts a delivery when an order is approved — a caller has to do it.

`delivery_started`, `delivery_completed` and `delivery_failed` are consumed by `orders-service`, each with its own local copy of the class.

## 8. APIs exposed and consumed

**Exposed:** the five HTTP routes in section 3, Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5003`; port 80 in the container.

Through the gateway, all five routes require authentication and none requires a role. `POST /deliveries` generates the `deliveryId` at the gateway (`resourceId: {property: deliveryId, generate: true}`) and returns it in the `Resource-ID` header.

**Consumed:** no outbound HTTP calls and no event subscriptions.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Deliveries.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `deliveries-service` on host port 5003.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.deliveries`.

## 10. Security and auth clues

- JWT validated against the committed `certs/localhost.cer`, `validIssuer: pacco`.
- `Convey.Security` is composed; no certificate authentication here.
- No role check anywhere in the service — any authenticated caller who reaches the gateway can start, complete or fail any delivery, and anything on the internal network can do so without a token. There is no check that the caller owns the order being delivered.
- `logger.excludeProperties` keeps secret-like values out of logs; `httpClient.requestMasking` is configured.
- Vault supplies settings and dynamic MongoDB credentials; RabbitMQ credentials remain the plain defaults in `appsettings.json`.

## 11. Observability, logging and tracing clues

Jaeger under service name `deliveries`; Prometheus metrics via `Convey.Metrics.AppMetrics` (`metrics.database: pacco`, interval 5); structured logs to console, file and Seq with `/`, `/ping` and `/metrics` excluded. `Infrastructure/Logging/MessageToLogTemplateMapper.cs` gives each command and event its own log template.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` (technology choices and subscriptions), `src/Pacco.Services.Deliveries.Api/Program.cs` (HTTP surface), `src/Pacco.Services.Deliveries.Api/appsettings.json` (integration endpoints), `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs` and `DeliveryStatus.cs` (the delivery lifecycle).

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Deliveries.Api/`, `src/Pacco.Services.Deliveries.Application/`, `src/Pacco.Services.Deliveries.Core/`, `src/Pacco.Services.Deliveries.Infrastructure/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5003`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.deliveries`; `Pacco.Services.Deliveries.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the four consumed commands and four published events; the fact that Orders depends on this service's events to close an order; the Vault integration; the transactional outbox.

**Conflicts to surface:**

- **Stale doc.** The README instructs running `dotnet run` in `/src/Pacco.Services.Deliveries`. That directory does not exist — the host project is `src/Pacco.Services.Deliveries.Api`. `scripts/start.sh` has the correct path.
- **Catalogue conflict.** `Application/Events/Rejected/AddDeliveryRegistrationRejected.cs` exists on disk, but the platform's message catalogue (`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`) lists only three rejected events for `deliveries-service` and omits `add_delivery_registration_rejected`. The Operations dashboard builds its subscriptions from that catalogue, so a failed registration is published but never surfaced to the user who triggered it.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`, but this repository contains **no test project at all**. The step passes vacuously.
- **Design gap, not a doc conflict.** Nothing subscribes an order-approval event to start a delivery. The README describes an event-driven platform, and this service is the one place where the chain is broken and a human or an external caller must intervene.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | Deliveries are started by an operator or an external system, not automatically by an order event. | The service has no event subscriptions, so nothing inside the platform can start one. |
| A2 | `messages.json` in the Operations repository is meant to be the complete catalogue of platform messages. | Operations builds live subscriptions from it at start-up. |
| A3 | The `Notes` field on a delivery is free text with no business meaning. | It is a plain string with no validation in the document or the command. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** Any authenticated caller can complete or fail any delivery; there is no ownership or role check. | A customer could mark someone else's delivery as complete. | Deliveries service maintainer, with the security-architecture owner. |
| B2 | **[ACTION NOW]** `add_delivery_registration_rejected` is published but absent from the message catalogue. | Rejections of that command never reach the user-facing Operations feed. | Deliveries service maintainer: add it to `messages.json`. |
| B3 | **[ACTION NOW]** CI runs `dotnet test` against a repository with no test projects. | The build reports success while testing nothing. | Deliveries service maintainer. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[handled later by the domain-model stage]** Should an approved order start a delivery automatically? | This is the one break in the platform's otherwise end-to-end event chain. | Domain-model stage. |
| Q2 | **[handled later by the API-contract stage]** What fields does each of the four delivery events carry? | Orders reacts to three of them and its handling depends on the fields present. | API-contract stage. |
| Q3 | **[handled later by the data-architecture stage]** What is stored in Redis under the `deliveries:` prefix? | Not answerable from source; determines whether Redis is a hard dependency here. | Data-architecture stage, by inspecting a running instance. |
| Q4 | **[ACTION NOW]** Is a delivery validated against a real order before it is started? | Nothing in the service checks the order identifier, so orphan deliveries are possible. | Deliveries service maintainer. |
