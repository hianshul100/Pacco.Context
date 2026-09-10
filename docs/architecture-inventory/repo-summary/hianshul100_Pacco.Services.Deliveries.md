# Repository Summary — `hianshul100_Pacco.Services.Deliveries`

**Primary name:** `deliveries-service` (aliases used in this file: `Pacco.Services.Deliveries.Api` — the .NET project and assembly name; `devmentors/pacco.services.deliveries` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Deliveries`, path: `src/Pacco.Services.Deliveries.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Tracks the physical delivery of an order from start to completion or failure, and records intermediate *delivery registrations* — checkpoint entries logged as the delivery progresses. It is the operational tail of the order lifecycle.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`, with an in-process RabbitMQ consumer. Layered `.Api` / `.Application` / `.Core` / `.Infrastructure`.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Deliveries.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Deliveries.sln`: `src/Pacco.Services.Deliveries.Api`, `.Application`, `.Core`, `.Infrastructure`. `.Core` has no NuGet references. `.Infrastructure` carries the standard Convey stack (Vault, Consul, Fabio, HTTP, message brokers with Mongo outbox, AppMetrics, MongoDB, Redis, Security, Jaeger, Swagger).

## 5. External integrations

- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — all through Convey extensions.
- No outbound HTTP calls to peer services: `httpClient.services` in `src/Pacco.Services.Deliveries.Api/appsettings.json` is empty.

## 6. Data stores & state

- **MongoDB.** Database `deliveries-service`, connection string `mongodb://localhost:27017`, `seed: false`.
- **Collection:** `deliveries` — `.AddMongoRepository<DeliveryDocument, Guid>("deliveries")`.
- **Outbox collections:** `outbox` and `inbox` (`outbox.enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `connectionString: localhost`, key prefix `deliveries:`.
- **Query mechanism:** Convey `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver. **No ORM.**
- **Migration tool:** none.
- **Cross-domain coupling:** a delivery is created for an order, so a delivery document holds an order identifier that belongs to the `orders-service` domain. There is no foreign key — MongoDB does not enforce them, and the two live in separate databases. Consistency depends on the `delivery_started` / `delivery_completed` / `delivery_failed` events reaching `orders-service`.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `deliveries`, queue template `deliveries-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Mongo-backed outbox; Jaeger plugin registered on the broker.

**Consumed** (`src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs`, lines 87–90) — all four are commands, all published by `api-gateway` in async mode:

| Kind | Message | Wire name |
|---|---|---|
| Command | `StartDelivery` | `start_delivery` |
| Command | `CompleteDelivery` | `complete_delivery` |
| Command | `FailDelivery` | `fail_delivery` |
| Command | `AddDeliveryRegistration` | `add_delivery_registration` |

This service subscribes to **no events from other services**. It is driven entirely by commands.

**Published**, per the `deliveries-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: `delivery_completed`, `delivery_failed`, `delivery_started`, `registration_added_to_delivery`; rejection events `complete_delivery_rejected`, `fail_delivery_rejected`, `start_delivery_rejected`.

Note the asymmetry: `add_delivery_registration` has no matching `*_rejected` event, unlike the other three commands.

`orders-service` subscribes to `delivery_completed`, `delivery_failed` and `delivery_started`, so this service drives order state changes.

**Payload key fields:** the API gateway binds `deliveryId` on the read route. Message bodies carry at minimum a delivery identifier; the complete field set is **unknown — requires runtime capture**.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.Deliveries.Api/Program.cs`:

| Method | Path | Dispatched to |
|---|---|---|
| GET | `deliveries/{deliveryId}` | `GetDelivery` |
| POST | `deliveries` | `StartDelivery` |
| POST | `deliveries/{deliveryId}/fail` | `FailDelivery` |
| POST | `deliveries/{deliveryId}/complete` | `CompleteDelivery` |
| POST | `deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Consumed by:** `api-gateway`, which exposes only `GET deliveries/{deliveryId}` publicly for reads and routes the four mutations through RabbitMQ.

**Consumes:** nothing over HTTP.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5003`.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.deliveries`.

## 10. Security & auth clues

- No `security` access-control list and no certificate authentication — unlike `customers-service` and `availability-service`, this service does not participate in the certificate-based service-to-service scheme.
- JWT validated against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- Vault: `enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, key-value path `deliveries-service/settings`, PKI `roleName: deliveries-service`, `commonName: deliveries-service.pacco.io`, plus a `lease.mongo` block issuing dynamic MongoDB credentials with `autoRenewal: true`.
- **Checked-in credentials:** `vault.token: "secret"`, RabbitMQ `guest`/`guest`, Seq `apiKey: secret` in `src/Pacco.Services.Deliveries.Api/appsettings.json`.
- `.UsePublicContracts<ContractAttribute>()` exposes the message contract surface.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: deliveries`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker.
- **Correlation:** `Correlation-Context` header on inbound requests; the `Saga` header is forwarded on outbound messages.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` (daily) + Seq at `http://localhost:5341`. ELK configured at `http://localhost:9200` but disabled. `excludePaths: ["/", "/ping", "/metrics"]`; `excludeProperties` redacts `api_key`, `access_key`, `ApiKey`, `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`, `Password`, `Email`, `Login`, `Secret`, `Token`. `.AddHandlersLogging()` logs each command and event handler.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Deliveries.Infrastructure/Extensions.cs` — the composition root and the four-command subscription list.
- `src/Pacco.Services.Deliveries.Api/Program.cs` — the route table, including the decision to model state transitions as `POST` sub-resources (`/fail`, `/complete`, `/registrations`) rather than a `PATCH` on the delivery.
- `src/Pacco.Services.Deliveries.Api/appsettings.json` — the configuration contract.
- `Pacco.Services.Deliveries.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- `add_delivery_registration` has no `*_rejected` counterpart in `messages.json`, so a failed registration is invisible to `operations-service` and therefore to any client watching the operations feed. Whether that is deliberate is **Unknown**. **Needs validation.**
- The delivery state machine — which transitions are legal from which state — lives in `src/Pacco.Services.Deliveries.Core/` and was not enumerated. **Unknown.**
- What a "delivery registration" contains, and whether registrations are appended to the delivery document or stored separately, is **Unknown**; only one collection (`deliveries`) is registered, which suggests they are embedded.
- Full payload field sets are **unknown — requires runtime capture**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Deliveries.Api/`, `src/Pacco.Services.Deliveries.Application/`, `src/Pacco.Services.Deliveries.Core/`, `src/Pacco.Services.Deliveries.Infrastructure/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Deliveries` directory". | No such directory. The runnable project is `src/Pacco.Services.Deliveries.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5003`." | `appsettings.json` sets the Consul port to `5003`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.deliveries`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Deliveries.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README says nothing about messaging or the outbox. | Four command subscriptions and four published events drive the whole service; only one of its five operations is reachable over public HTTP. | **Docs gap.** The README is the shared platform template with the service name substituted, and it materially understates how the service is actually driven. |

**On disk but not mentioned in `README.md`:** the RabbitMQ subscriptions, the outbox pattern, the Vault integration, and the coupling to `orders-service` through delivery events.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | A delivery document holds an order identifier linking it to `orders-service`. | `orders-service` subscribes to `delivery_started`, `delivery_completed` and `delivery_failed` and changes order state from them, which requires the delivery to know its order. | The relationship between the delivery and order domains would be misdescribed. | Read `DeliveryDocument` under `src/Pacco.Services.Deliveries.Infrastructure/Mongo/`. |
| A2 | Delivery registrations are embedded in the delivery document rather than stored in their own collection. | Only one Mongo repository is registered, for `deliveries`. | A second collection would be missing from the platform data inventory. | Read `DeliveryDocument` and the `AddDeliveryRegistration` handler. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[handled later by the platform inventory review]** Should `add_delivery_registration` publish a rejection event like the other three commands do? | Without one, a failed registration never reaches the operations feed, so the caller is left waiting with no signal. | Yes — it looks like an omission rather than a decision. | Service owner |
| Q2 | **[handled later by the platform inventory review]** What are the legal delivery state transitions? | The route table implies a state machine (`start` → `complete` or `fail`) but nothing documents which transitions are rejected. | Read from `src/Pacco.Services.Deliveries.Core/`. | Service owner |
