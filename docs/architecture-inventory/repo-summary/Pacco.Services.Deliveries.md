---
title: "Repository Summary — Pacco.Services.Deliveries"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Deliveries"
status: "evidence-based inventory"
---

# Pacco.Services.Deliveries

**Primary name:** `Pacco.Services.Deliveries` (aliases used in this file: `deliveries-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `deliveries` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Deliveries`, path: `hianshul100_Pacco.Services.Deliveries/`

---

## 1. Primary purpose

Tracks the physical delivery of an order through its lifecycle: created, started, registrations added along the route, then completed or failed.

Evidence: `src/Pacco.Services.Deliveries.Application/Commands/`, `src/Pacco.Services.Deliveries.Application/Events/`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service using Convey dispatcher endpoints, plus a RabbitMQ subscriber. Listens on `5003`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` | `src/Pacco.Services.Deliveries.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

Four source projects: `Pacco.Services.Deliveries.Api`, `.Application`, `.Core`, `.Infrastructure`. **No test project exists in this repository.**

- **Core:** delivery aggregate with `AggregateId` / `AggregateRoot` base types and a repository interface.
- **Application:** commands `AddDeliveryRegistration`, `CompleteDelivery`, `FailDelivery`, `StartDelivery` with handlers; integration events `DeliveryCompleted`, `DeliveryFailed`, `DeliveryStarted`, `RegistrationAddedToDelivery`; rejected events `AddDeliveryRegistrationRejected`, `CompleteDeliveryRejected`, `FailDeliveryRejected`, `StartDeliveryRejected`; DTOs `DeliveryDto`, `DeliveryRegistrationDto`; exceptions `DeliveryAlreadyStartedException`, `DeliveryNotFoundException`.
- **Infrastructure:** MongoDB documents, repositories and query handlers, plus the outbox decorators, event mapper, message broker and exception mappers shared across the platform.

Convey `0.4.*` packages as used across the platform.

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus. No outbound HTTP service clients are defined.

## 6. Data stores / state

- **Store:** MongoDB, database `deliveries-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with explicit document classes and hand-written repositories.
- **Collections:** a deliveries collection derived from the delivery document type, plus `inbox` and `outbox`.
- **Migration tool:** none anywhere in the repository.
- **Cross-domain coupling:** the delivery document carries the order identifier it belongs to. There is no foreign key; the link is an identifier copied from an order event.
- **Cache:** Redis.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `deliveries`, snake-case naming, queue template `deliveries-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`. Transactional outbox and inbox on MongoDB (`inbox`, `outbox`).

**Commands consumed:** `add_delivery_registration`, `complete_delivery`, `fail_delivery`, `start_delivery`.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `delivery_started` | `Application/Events/DeliveryStarted.cs` | `DeliveryId` (Guid), `OrderId` (Guid) — read directly from the class |
| `delivery_completed` | `Application/Events/DeliveryCompleted.cs` | delivery identifier, order identifier |
| `delivery_failed` | `Application/Events/DeliveryFailed.cs` | delivery identifier, order identifier, reason |
| `registration_added_to_delivery` | `Application/Events/RegistrationAddedToDelivery.cs` | delivery identifier, registration details |

**Rejected events published:** `complete_delivery_rejected`, `fail_delivery_rejected`, `start_delivery_rejected`.

**Note on the catalogue:** the shared catalogue at `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` lists `add_delivery_registration` as a command for this service but lists **no matching `add_delivery_registration_rejected` entry**, even though the class `AddDeliveryRegistrationRejected` exists in this repository. **Needs validation.**

**External events consumed:** none are declared in this repository. Deliveries are driven by commands published to its exchange rather than by reacting to other services' events, so it is the terminal service in the order flow: it emits delivery outcomes, and `Pacco.Services.Orders` consumes them.

Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `deliveries/{deliveryId}` | delivery query |
| `POST` | `deliveries` | `AddDelivery`-style creation, responds `Created` |
| `POST` | `deliveries/{deliveryId}/fail` | `FailDelivery` |
| `POST` | `deliveries/{deliveryId}/complete` | `CompleteDelivery` |
| `POST` | `deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` |

Consumed by: `Pacco.APIGateway` (module `deliveries`; the gateway's `POST /` route generates the delivery identifier with `resourceId: {property: deliveryId, generate: true}`).

This service consumes no other service's HTTP API.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.deliveries`, published `5003:80` in `hianshul100_Pacco/compose/services.yml`, `restart: unless-stopped`, network `pacco`. Consul registration on port `5003`.

CI: `.travis.yml` runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` on success.

## 10. Security / auth clues

- JWT validation following the platform pattern, `validIssuer: pacco`.
- Vault: KV path `deliveries-service/settings`.
- **This is the only service in the platform with no `security` block in `appsettings.json`** — it defines neither a certificate header nor an access-control list. It therefore has no service-to-service certificate checking at all.

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: deliveries`, including RabbitMQ span propagation; structured logging via `Convey.Logging` and `Convey.Logging.CQRS` with a message-to-log-template mapper; Prometheus metrics via `Convey.Metrics.AppMetrics`.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Deliveries.Application/Commands/Handlers/` — the delivery state transitions and the guard implemented by `DeliveryAlreadyStartedException`.
- `src/Pacco.Services.Deliveries.Infrastructure/Decorators/` — the outbox decorators.
- `src/Pacco.Services.Deliveries.Infrastructure/Exceptions/ExceptionToMessageMapper.cs` — how failures become rejected events.
- `src/Pacco.Services.Deliveries.Api/appsettings.json` — the infrastructure contract and the absent security block.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json`, which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` contains only the four C# projects. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes a deliveries service on .NET Core 3.1 built with Convey, following clean architecture | Confirmed: four layered projects, Convey `0.4.*`, `netcoreapp3.1` | Confirmed |
| The platform README presents certificate-based service security as part of the design | This service has no `security` block at all, unlike its siblings | Stale doc — the security design is not applied here |
| The shared message catalogue is the platform's message contract | The catalogue omits the rejected event for adding a delivery registration, which exists as a class in this repository | Needs validation — the catalogue is incomplete for this service |
| The build script chain includes `./scripts/test.sh` | There is no test project in this repository, so the step has nothing to execute | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the delivery registration concept (`DeliveryRegistrationDto`, `AddDeliveryRegistration`) — present in code, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | A delivery is always created for an existing order, and the order identifier is copied in from the order side. | The delivery events carry an order identifier, and the orders service is the only place orders are created. |
| A2 | This service does not listen to other services' events; it only receives commands and reports outcomes. | No external event folder or handler exists in the repository. |

### Blockers

_None._

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Why does this service alone have no service-to-service security settings? | **[ACTION NOW]** Confirm with the requesting team whether this is deliberate or an omission, because it affects the platform security picture. |
| Q2 | Who creates a delivery, and when? Nothing in this workspace shows an automatic trigger from an order. | **[handled later by the ADR authoring stage]** Trace the end-to-end order flow and record where delivery creation is triggered. |
| Q3 | The shared message catalogue is missing one of this service's rejected messages. Which one is authoritative, the catalogue or the code? | **[ACTION NOW]** Confirm with the requesting team, since the catalogue drives what the operations service can report on. |
