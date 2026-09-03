---
title: "Repository Summary — Pacco.Services.Pricing"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Pricing"
status: "evidence-based inventory"
---

# Pacco.Services.Pricing

**Primary name:** `Pacco.Services.Pricing` (aliases used in this file: `pricing-service` — the value of `app.service`, the Consul registration name and the Compose service name; `pricing` — the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Pricing`, path: `hianshul100_Pacco.Services.Pricing/`
Deployable project: `Pacco.Services.Pricing.Api`, path: `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj`

---

## 1. Primary purpose

Calculates the price of an order, applying customer discounts. It is the platform's only pure calculation service: it stores nothing and publishes nothing.

Evidence: `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs`, `Queries/Handlers/GetOrderPricingHandler.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service exposing a single read endpoint through the Convey dispatcher. **It is not a message consumer and not a message publisher.** Listens on `5008`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` — a single `GET pricing` route | `src/Pacco.Services.Pricing.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

**One flat project**, `Pacco.Services.Pricing.Api`. There is no `.Application`, `.Core` or `.Infrastructure` project; the layer names appear as folders inside the single project instead.

Folders and files: `Core/Entities/Customer.cs`; `Core/Services/ICustomerDiscountsService.cs`, `CustomerDiscountsService.cs`; `DTO/CustomerDto.cs`, `OrderPricingDto.cs`, `Extensions.cs`; `Queries/GetOrderPricing.cs`; `Queries/Handlers/GetOrderPricingHandler.cs`; `Services/Clients/ICustomersServiceClient.cs`, `CustomersServiceClient.cs`; `Infrastructure/Extensions.cs`; `Exceptions/`.

Packages: the Convey `0.4.*` set **without** `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis`, `Convey.MessageBrokers.RabbitMQ` or the outbox packages. It does reference `Convey.Tracing.Jaeger.RabbitMQ`, which has no purpose in a service with no broker connection. **Needs validation.**

**Repository hygiene:** JetBrains Rider IDE files are committed under `src/Pacco.Services.Pricing.Api/.idea/`.

## 5. External integrations

Consul, Fabio, Vault, Jaeger, Prometheus, and one service over HTTP: `httpClient.services: {customers: customers-service}`.

## 6. Data stores / state

- **Store: none.** `appsettings.json` contains **no `mongo` block, no `redis` block and no `rabbitMq` block**. The service is fully stateless.
- **Access mechanism:** not applicable — no ORM, no repository classes, no document types.
- **Collections / tables:** none. No `inbox` or `outbox` either.
- **Migration tool:** none.
- **Cross-domain coupling:** it reads customer data live from `Pacco.Services.Customers` for each request through `Services/Clients/CustomersServiceClient.cs`, mapping it into its own `Core/Entities/Customer.cs` and `DTO/CustomerDto.cs`. It is the only service that does **not** keep a local copy of customer data, which makes it the only one whose customer view can never go stale — and the only one that fails when the customers service is unavailable.

## 7. Messaging / async / events

**None.** This service does not connect to RabbitMQ. It has no exchange, publishes no commands, no events and no rejected events, and subscribes to nothing.

Confirmed by the platform message catalogue at `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`, which contains **no entry for `pricing-service`** — the only service absent from that file.

There is therefore nothing to mark as requiring runtime capture for this dimension.

## 8. APIs exposed / consumed

Exposed (`Program.cs`):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `pricing` | `GetOrderPricing` → `OrderPricingDto` |

That is the entire public surface: one route, one query, one response type.

Consumed: `customers-service` through `Services/Clients/CustomersServiceClient.cs`.

Called by: `Pacco.APIGateway` (module `pricing`; the route is rewritten to `pricing-service/pricing?customerId=@user_id`) and by `Pacco.Services.Orders` through its `PricingServiceClient`.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.pricing`, `5008:80` per the platform port map, network `pacco`. Consul registration on port `5008`. Outbound calls go through Fabio (`httpClient.type: fabio`).

CI: `.travis.yml` present with the standard build and dockerize chain.

**Repository completeness:** this repository has **no `LICENSE` file**, unlike most of its siblings.

## 10. Security / auth clues

- JWT validation following the platform pattern.
- Vault: KV path `pricing-service/settings` and a PKI role, but **no `lease` block for database credentials** — correctly so, since there is no database.
- **This service is not listed in the caller access-control list defined by `Pacco.Services.Customers`**, even though it calls that service on every request. That list currently grants only `availability-service` the `customers:read` permission. **Needs validation.**

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: pricing`; structured logging via `Convey.Logging`; Prometheus metrics via `Convey.Metrics.AppMetrics`. The RabbitMQ tracing package is referenced but has no broker to trace.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` — the discount rules, and the reason this service exists as a separate deployable at all.
- `src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs` — the decision to read customer data live rather than replicate it, which is the opposite of the choice made by the orders and parcels services.
- `src/Pacco.Services.Pricing.Api/appsettings.json` — the absence of a database and of a broker, which is what makes this service stateless.
- `src/Pacco.Services.Pricing.Api/Program.cs` — the single-endpoint design.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json`, which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` contains only the single C# project, alongside a committed `.idea/` IDE settings directory. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| The platform README describes clean architecture with four projects per service | This service is a single flat project with layer names used as folders | Stale doc |
| The platform README describes an event-driven platform where services communicate through messages | This service has no broker connection at all and is missing from the platform message catalogue | Stale doc — it is a synchronous read service, and the documentation does not say so |
| The platform README describes services owning their own data | This service owns no data and depends on a live call to the customers service for every request | Stale doc |
| Every service is registered with the shared secrets manager for database credentials | Correct here that no database lease exists, since there is no database | Confirmed |
| The customers service access list defines who may read customer data | This service is not on that list but reads customer data on every request | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the discount rules and the committed IDE settings directory — present on disk, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | This service is intentionally stateless, not simply unfinished. | It has no database settings, no database packages and no repository code; the omission is consistent everywhere. |
| A2 | Pricing is always calculated fresh on request rather than stored with the order. | There is no storage here, and the orders service calls this service rather than reading a stored price. |
| A3 | The message-broker tracing package is a leftover from copying another service's project file. | The service has no broker connection for it to act on. |

### Blockers

_None._

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Should this service be added to the customers service access list? It reads customer data on every request but is not listed as an allowed caller. | **[ACTION NOW]** Confirm with the requesting team; this and the same question in the customers summary are the same issue seen from both sides. |
| Q2 | What happens to pricing when the customers service is unavailable? There is no local copy and no cache to fall back on. | **[handled later by the ADR authoring stage]** Record the intended behaviour when a dependency is down. |
| Q3 | What are the discount rules in business terms? | **[handled later by the ADR authoring stage]** Read the discount service with a business owner present and record the rules. |
| Q4 | Why does this repository have no licence file when most of its siblings do? | **[handled later by the ADR authoring stage]** Confirm the intended licence for the platform and note the gap. |
