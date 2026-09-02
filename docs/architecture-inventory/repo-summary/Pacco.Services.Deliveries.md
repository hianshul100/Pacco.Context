# Repository summary — `Pacco.Services.Deliveries`

**Repository:** `Pacco.Services.Deliveries` (workspace clone path: `hianshul100_Pacco.Services.Deliveries`)
**Deployable:** `deliveries-service` (also known as: Deliveries Service, `Pacco.Services.Deliveries.Api`, image `devmentors/pacco.services.deliveries`). **Repository: `Pacco.Services.Deliveries`, path: `src/Pacco.Services.Deliveries.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Deliveries
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Owns the **physical delivery execution** of an approved order: starting a delivery, appending progress registrations (scan events / status notes along the route), and closing it as completed or failed. It is the last stage of the order lifecycle and the source of the events that let `orders-service` move an order to delivering, completed, or cancelled.

**Evidence:** `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs`, `Entities/DeliveryRegistration.cs`, `src/Pacco.Services.Deliveries.Api/Program.cs`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice plus RabbitMQ consumer in one process, using the canonical four-project clean-architecture layering (`.Api`, `.Application`, `.Core`, `.Infrastructure`) on Convey.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Deliveries.Api/Program.cs` — `AddConvey().AddWebApi().AddApplication().AddInfrastructure()`, then `UseInfrastructure()` + `UseDispatcherEndpoints(...)` |
| RabbitMQ subscriptions | `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` → `UseInfrastructure` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Deliveries.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Deliveries.Api` | Host, endpoint map, `appsettings.json` |
| `src/Pacco.Services.Deliveries.Application` | `Commands/` (`StartDelivery`, `CompleteDelivery`, `FailDelivery`, `AddDeliveryRegistration`) + handlers; `Queries/` (`GetDelivery`) + handler; `Events/`, `Events/Rejected/`; `DTO/` |
| `src/Pacco.Services.Deliveries.Core` | `Entities/Delivery.cs`, `Entities/DeliveryRegistration.cs`, `Entities/AggregateRoot.cs`, `Events/`, `Exceptions/`, `Repositories/IDeliveriesRepository` |
| `src/Pacco.Services.Deliveries.Infrastructure` | `Mongo/Documents/DeliveryDocument.cs`, `Mongo/Repositories/DeliveriesMongoRepository.cs`, `Decorators/` (outbox), `Contexts/`, `Exceptions/ExceptionToResponseMapper.cs` + `ExceptionToMessageMapper.cs`, `Logging/`, `Extensions.cs` |

**No test projects exist in this repository.**

Convey package set matches the platform standard (`Convey`, `Convey.CQRS.*`, `Convey.Discovery.Consul`, `Convey.HTTP`, `Convey.LoadBalancing.Fabio`, `Convey.Logging`, `Convey.Logging.CQRS`, `Convey.MessageBrokers.CQRS`, `Convey.MessageBrokers.Outbox.Mongo`, `Convey.MessageBrokers.RabbitMQ`, `Convey.Metrics.AppMetrics`, `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis`, `Convey.Secrets.Vault`, `Convey.Security`, `Convey.Tracing.Jaeger`, `Convey.WebApi.*`), all `0.4.*`.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| RabbitMQ | in + out | exchange `deliveries`, topic |
| MongoDB | out | database `deliveries-service` |
| Redis | out | instance prefix `deliveries:` |
| Consul | out | registers `deliveries-service` on port `5003` |
| Fabio | out | `http://localhost:9999` |
| Vault | out | KV v2 `kv/deliveries-service/settings`; PKI role `deliveries-service`, CN `deliveries-service.pacco.io`; dynamic Mongo credentials |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

`httpClient.services` is **empty** — no outbound HTTP calls to other services. All integration is via the broker.

**No external carrier, tracking, mapping, geocoding, or notification integration exists.** For a delivery-execution service this is notable: there is no courier API, no address validation, no route optimisation, and no customer notification channel. Deliveries are advanced entirely by commands arriving from the API or the broker.

**Evidence:** `src/Pacco.Services.Deliveries.Api/appsettings.json`; no `Services/Clients/` directory in any project.

## 6. Data stores / state handling

- **Store:** MongoDB, database `deliveries-service`.
- **Collections:** `deliveries` (`AddMongoRepository<DeliveryDocument, Guid>("deliveries")`), plus `inbox` and `outbox`.
- **Access mechanism:** Convey `IMongoRepository<>` over `MongoDB.Driver`. **No ORM.**
- **Migration tool: none.** No Flyway, Liquibase, Alembic, or EF Core migrations.
- **Document shape** (`Infrastructure/Mongo/Documents/DeliveryDocument.cs`): the delivery aggregate — delivery id, the associated `OrderId`, status, and the embedded collection of registrations.
- **Cross-domain coupling:** the delivery document carries an `OrderId` referring to an aggregate owned by `orders-service`. It is a **logical reference only** — no database foreign key exists (separate MongoDB logical databases, no relational constraints), and this service stores no copy of the order. Unlike `orders-service` and `parcels-service`, it keeps **no local `customers` replica** and does not subscribe to `customer_created`.
- **Outbox:** enabled, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: inbox`, `outboxCollection: outbox`, `disableTransactions: true`.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `deliveries`; `conventionsCasing: snakeCase`; queue template `deliveries-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — commands** (from `Infrastructure/Extensions.cs` → `UseInfrastructure`):

| Message | Wire name | Key payload fields |
|---|---|---|
| `StartDelivery` | `start_delivery` | `DeliveryId`, `OrderId` |
| `AddDeliveryRegistration` | `add_delivery_registration` | `DeliveryId`, `Message` |
| `CompleteDelivery` | `complete_delivery` | `DeliveryId` |
| `FailDelivery` | `fail_delivery` | `DeliveryId`, `Reason` |

**Consumed — external events: none.** This service subscribes to **no events from other services** — the only service besides `vehicles-service` with no event subscriptions at all. It is driven purely by commands.

**Published — events:**

| Event | Wire name | Key payload fields |
|---|---|---|
| `DeliveryStarted` | `delivery_started` | `DeliveryId`, `OrderId` |
| `DeliveryCompleted` | `delivery_completed` | `DeliveryId`, `OrderId` |
| `DeliveryFailed` | `delivery_failed` | `DeliveryId`, `OrderId`, `Reason` |
| `RegistrationAddedToDelivery` | `registration_added_to_delivery` | `DeliveryId`, `OrderId`, `Message` |

**Published — rejection events:** `start_delivery_rejected`, `complete_delivery_rejected`, `fail_delivery_rejected`, each with `Reason` and `Code`, produced by `Infrastructure/Exceptions/ExceptionToMessageMapper.cs`. Note that `messages.json` lists **no** `add_delivery_registration_rejected` — the registration command has no declared rejection event.

**Downstream effect:** `orders-service` subscribes to `delivery_started`, `delivery_completed`, and `delivery_failed`, using them to drive the order through `order_delivering` → `order_completed` / `order_canceled`. This is the closing link of the platform's order lifecycle.

**Reliability:** outbox/inbox decorators wrap every command and event handler.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5003`, container port `80`):

| Method | Path | Maps to | Gateway exposure |
|---|---|---|---|
| GET | `deliveries/{deliveryId}` | `GetDelivery` | `/deliveries/{deliveryId}` |
| POST | `deliveries` | `StartDelivery` | `/deliveries` — gateway generates `deliveryId` (`resourceId.property: deliveryId, generate: true`) |
| POST | `deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` | `/deliveries/{deliveryId}/registrations` |
| POST | `deliveries/{deliveryId}/complete` | `CompleteDelivery` | `/deliveries/{deliveryId}/complete` |
| POST | `deliveries/{deliveryId}/fail` | `FailDelivery` | `/deliveries/{deliveryId}/fail` |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**Consumed:** none over HTTP.

**Called by:** nothing — no other service holds an HTTP client for `deliveries-service`.

**Access control note:** all `/deliveries/*` gateway routes are `auth: true` but carry **no role claim requirement**. Any authenticated user — including an ordinary customer — can start, complete, or fail a delivery, or append registrations to one, for any delivery id. There is no ownership check binding a delivery to the calling customer (contrast `/orders` and `/parcels`, which bind `customerId: @user_id`).

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll`.
- Composed as `deliveries-service` on `5003:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5003`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- Consul health check `pingEndpoint: ping`, interval `3` seconds.

## 10. Security/auth clues

- **JWT bearer** validation via `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- `.AddSecurity()` is registered, but **no `security.certificate` block** exists in `appsettings.json` — this service neither presents nor verifies client certificates.
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token); PKI role `deliveries-service`, CN `deliveries-service.pacco.io`.
- Log redaction via `logger.excludeProperties`.
- **No authorisation inside the service and no role gate at the gateway** — see the access-control note in §8. This is the widest-open write surface in the platform.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: deliveries-service`, UDP `localhost:6831`, `sampler: const`) with the RabbitMQ Jaeger plugin.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`. Handler logging via `.AddHandlersLogging()`.
- **Correlation:** `Correlation-Context` header; `Saga` header forwarded.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs` | The delivery state machine and its invariants (which transitions are legal, when registrations may be added) |
| `src/Pacco.Services.Deliveries.Core/Entities/DeliveryRegistration.cs` | Registrations are modelled as an append-only log embedded in the delivery aggregate rather than a separate collection |
| `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` | Composition, and the decision to subscribe to commands only — this service has no event-driven inputs |
| `src/Pacco.Services.Deliveries.Infrastructure/Exceptions/ExceptionToMessageMapper.cs` | Async error contract (three rejection events; none for `add_delivery_registration`) |
| `src/Pacco.Services.Deliveries.Api/appsettings.json` | Outbox with `disableTransactions: true`; no certificate security block |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. The only switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`). No business behaviour is gated.

## 13. Open questions / ambiguities

- **Who starts a delivery?** `POST /deliveries` is a public gateway route and `start_delivery` is a broker command, but **no service in the workspace publishes `start_delivery`**. `orders-service` reacts to `delivery_started` rather than causing it. The actual trigger — a courier app, an operator console, a manual call — is **Unknown**.
- **No ownership or role check** on any delivery write endpoint. Whether this is deliberate (deliveries are assumed to be driven only by trusted internal operators over a private network) is **Unknown**.
- **`add_delivery_registration` has no rejection event**, unlike every other command in the platform. Whether registration failures are simply swallowed is **Needs validation**.
- **No tests** exist in this repository, despite it owning a state machine with terminal states.
- **No carrier, tracking, geocoding, or notification integration.** Whether delivery execution is intended to remain manual is **Unknown**.
- The delivery status vocabulary was read from `Delivery.cs` but not cross-checked against any client expectation. **Needs validation.**

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (four C# projects only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS files. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Deliveries service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5003`. — **Confirmed** (`appsettings.json` `consul.port: 5003`, `Pacco/compose/services.yml` `5003:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Deliveries` directory"**; the actual host project is **`/src/Pacco.Services.Deliveries.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- The delivery lifecycle itself — start, registrations, complete, fail — and the four events it publishes, which are what close the platform's order lifecycle. Undocumented.
- The fact that this service consumes **commands only** and subscribes to no events.
- The transactional outbox/inbox and the handler decorators.
- The absence of any rejection event for `add_delivery_registration`.
- `scripts/` (`build.sh`, `test.sh`, `dockerize.sh`).

**Unknown (neither pass yielded proof):**
- Which actor is expected to call `POST /deliveries`; neither the README nor any code in the workspace names one.
- Whether the missing ownership check is a deliberate trust assumption.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Deliveries are driven by an actor outside these repositories — a courier application, an operator console, or a manual API call. | `POST /deliveries` is exposed publicly and `start_delivery` is a declared broker command, yet nothing in the thirteen clones publishes it or calls that route. | The delivery leg of the order lifecycle would be unreachable, meaning orders can never complete, and the platform's happy path would be broken rather than merely undocumented. | Ask the product owner who operates deliveries; check whether an operator client exists outside this workspace. |
| A2 | The `OrderId` stored on a delivery is a logical reference that this service never validates against `orders-service`. | The document carries the id, but there is no HTTP client for `orders-service` and no subscription to any order event. | A delivery could be started for an order that does not exist or is already cancelled, and nothing would detect it. | Read `StartDeliveryHandler` and confirm whether any existence check occurs. |
| A3 | Delivery registrations are an append-only audit log rather than mutable records. | They are modelled as an embedded collection on the aggregate with add-only commands and no update or delete command. | Any downstream design treating registrations as editable would conflict with the aggregate's actual behaviour. | Confirm in `Delivery.cs` that no removal or mutation path exists. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Every delivery write endpoint is reachable by any authenticated user, with no role requirement and no check that the caller is connected to the delivery. Any customer can complete or fail any delivery if they know its id. | Security sign-off on the public API surface; any decision to expose the gateway beyond a trusted network. | Security owner / service owner | Decide who may operate deliveries, then add a role claim on the `/deliveries` gateway routes and an ownership or operator check in the command handlers. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Who or what is meant to start a delivery? Nothing in the workspace publishes `start_delivery` or calls `POST /deliveries`. | Without an answer, the order lifecycle has no proven path from an approved order to a completed one. | Likely an external courier or operator client; unproven. | Product owner |
| Q2 | **[ACTION NOW]** Why does `add_delivery_registration` have no rejection event when every other command in the platform has one? | Callers using the asynchronous path get no failure signal, so a rejected registration is invisible. | Probably an omission rather than a decision. | Service owner |
| Q3 | **[handled later by architecture_evolution_generation]** Is delivery execution intended to stay entirely manual, with no carrier, tracking, geocoding, or customer-notification integration? | It is a large functional gap for a parcel-delivery product and would shape any future integration design. | Unknown from the repositories. | Product owner |
| Q4 | **[ACTION NOW]** Should a delivery verify that its `OrderId` refers to a real, approved order before starting? | Without it, the delivery and order domains can drift apart with no detection. | Add a check, or accept the looseness deliberately and record why. | Service owner |
