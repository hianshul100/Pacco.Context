# Repository: `Pacco.Services.Customers`

`customers-service` (also known as: Customers Service, `Pacco.Services.Customers`, Docker image
`devmentors/pacco.services.customers`) owns the customer profile, the customer state machine, and
VIP status.

- **Repository:** `Pacco.Services.Customers`, path: `src/Pacco.Services.Customers.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5002`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- The four-project clean-architecture split.
- `Core/Entities/Customer.cs` and `Core/Entities/State.cs` — the customer state machine.
- All five HTTP endpoints and the RabbitMQ `customers` exchange with its nine messages.
- MongoDB database `customers-service`, collection `customers`.
- **The certificate ACL** granting `availability-service` the `customers:read` permission — the
  only inbound authorisation rule of its kind on the platform, and undocumented.
- That this service is the platform's most depended-upon service: two synchronous callers and
  three event subscribers.

**Stale doc:** none identified.

**Unknown:** the business meaning of the customer states and the rule that promotes a customer to
VIP — `customer_became_vip` is published but the trigger condition is only visible in code, and its
consumers are none.

---

## 1. Primary purpose

Hold the customer record created in response to a sign-up elsewhere, complete customer
registration with profile details, expose customer state to other services, and announce state
transitions — including promotion to VIP — as events.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process, using Convey CQRS dispatcher
endpoints.

## 3. Key entrypoints

- `src/Pacco.Services.Customers.Api/Program.cs` — composition root and route table.
- `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` — DI composition root.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Customers.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Project | Role |
|---|---|
| `Pacco.Services.Customers.Api` | Host, route table, configuration, `certs/` |
| `Pacco.Services.Customers.Application` | Commands (`CompleteCustomerRegistration`, `ChangeCustomerState`), queries, events, DTOs, handlers |
| `Pacco.Services.Customers.Core` | `Entities/Customer.cs`, `Entities/State.cs`, `Repositories/ICustomerRepository.cs`, domain exceptions |
| `Pacco.Services.Customers.Infrastructure` | Mongo documents and repository, RabbitMQ broker, decorators, contexts, logging |

**Key packages:** `Convey`, `Convey.CQRS.Commands/.Events/.Queries`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Security`, `.WebApi.Swagger`.

## 5. External integrations

Consul (registration, `pingEndpoint: ping`), Fabio, RabbitMQ (exchange `customers`), MongoDB
(database `customers-service`), Redis (prefix `customers:`), Vault (kv v2 path
`customers-service/settings`, PKI role `customers-service`, common name
`customers-service.pacco.io`, MongoDB dynamic credentials), Jaeger, Seq, Prometheus.

**It calls no other service.** `httpClient.services` is empty — this is a leaf in the synchronous
call graph and a source, not a consumer, of platform data.

## 6. Data stores / state

- **Store:** MongoDB, database `customers-service`.
- **Query mechanism:** Convey `IMongoRepository<CustomerDocument, Guid>` over the MongoDB .NET
  driver. **Not a relational ORM.**
- **Registration:** `AddMongoRepository<CustomerDocument, Guid>("customers")` in
  `src/Pacco.Services.Customers.Infrastructure/Extensions.cs`.
- **Collection for the primary domain:** **`customers`**, mapped by
  `Infrastructure/Mongo/Documents/CustomerDocument.cs`.
- **Framework collections:** `inbox`, `outbox` (`type: sequential`, `disableTransactions: true`).
- **Migration tool:** **none.** No migration files or tooling in the repository.
- **Cross-domain coupling — the important one on this platform.** This service is the *owner* of
  the `customers` collection, but two other services keep their own copy of it:
  `orders-service` (`Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs`, collection
  `customers`) and `parcels-service` (`Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs`,
  collection `customers`). MongoDB has no foreign keys, so the coupling is **event-carried state
  replication**: both replicas are populated by the `customer_created` event and by nothing else.
  Neither replica consumes `customer_state_changed` or `customer_became_vip`, so both drift from
  this service as soon as a customer's state changes. **Needs validation** — see Open Questions.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `customers`, type `topic`, durable.
- **Conventions:** `snakeCase`; queue template `customers-service/{{exchange}}.{{message}}`;
  headers `message_context` and `span_context`.
- **Outbox:** enabled (`AddMessageOutbox(o => o.AddMongo())`) with outbox decorators on handlers.

**Commands consumed:** `change_customer_state`, `complete_customer_registration`.

**Events published:**

| Event | Observable payload fields |
|---|---|
| `customer_created` | `CustomerId`, `Email` |
| `customer_state_changed` | `CustomerId`, `State` |
| `customer_became_vip` | `CustomerId` |

**Rejected events published:** `change_customer_state_rejected`,
`complete_customer_registration_rejected` — each with `Reason` and `Code`.

**External events consumed** (`Application/Events/External/Handlers/`):

| Event | Source exchange | Effect |
|---|---|---|
| `signed_up` | `identity` | Creates the customer record — this is how a customer comes into existence |
| `order_completed` | `orders` | Feeds the VIP promotion rule |

**Consumers of this service's events:** `customer_created` → `availability-service`,
`orders-service`, `parcels-service`. `customer_state_changed` and `customer_became_vip` → **no
domain consumer anywhere in the workspace**; only `operations-service` observes them.

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Customers.Api/Program.cs`, verbatim):

| Method | Route | Dispatches |
|---|---|---|
| `GET` | `customers` | query — customer list |
| `GET` | `customers/{customerId}` | query — single customer |
| `GET` | `customers/{customerId}/state` | query — customer state only |
| `POST` | `customers` | `CompleteCustomerRegistration` |
| `PUT` | `customers/{customerId}/state/{state}` | `ChangeCustomerState` → `204 No Content` |

Swagger UI at route prefix `docs`.

**Consumed:** none.

**Inbound synchronous callers:** `availability-service`
(`GET /customers/{customerId}/state`) and `pricing-service` (`GET /customers/{customerId}`).

**Upstream:** the gateway module `customers` maps `GET /` and
`PUT /{customerId}/state/{state}` behind the `admin` role, and rewrites `GET /me` to
`customers-service/customers/@user_id`.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Customers.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.customers`.
- Port `5002` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5002:80`), and the
  gateway `localUrl`.
- Consul service name `customers-service`; `httpClient.type: fabio`.
- Environments: `appsettings.json`, `.local.json`, `.docker.json`.

## 10. Security / auth clues

- **JWT bearer** with `certs/localhost.cer`, `validIssuer: pacco`.
- **Inbound certificate ACL — unique on this platform.**
  `src/Pacco.Services.Customers.Api/appsettings.json` declares `security.certificate` with:
  `allowedDomains: ["pacco.io"]`, `allowSubdomains: true`, and an `acl` entry granting
  `availability-service` (with `validIssuer` `localhost`) the permission **`customers:read`**.
  This is the receiving half of the client-certificate call path implemented in
  `availability-service`'s `CustomersServiceClient`.
- **Admin-gated at the edge:** the gateway requires `role: admin` for `GET /customers` and for the
  state-change route; this service itself does not appear to re-check the role. **Needs validation.**
- **Vault:** kv v2 settings, PKI role `customers-service`, MongoDB dynamic credentials with lease
  auto-renewal.
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: customers`, UDP `6831`, `const` sampler rate 1, with the
  `Convey.Tracing.Jaeger.RabbitMQ` plugin propagating `span_context` across AMQP.
- **Logging:** console, file and Seq sinks enabled; ELK sink present but `enabled: false`.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Customers.sln` | Four-project clean-architecture split |
| `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` | Capability chain and outbox decorators on every command and event handler |
| `src/Pacco.Services.Customers.Core/Entities/Customer.cs`, `Entities/State.cs` | That customer status is a state machine with explicit transitions, and that VIP is a derived status rather than a stored flag set externally |
| `src/Pacco.Services.Customers.Application/Events/External/Handlers/` | That customers are created **only** in reaction to `signed_up` — there is no create-customer command |
| `src/Pacco.Services.Customers.Api/appsettings.json` | The `customers:read` certificate ACL; Vault PKI; outbox with `disableTransactions: true` |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. Why `customer_state_changed` and `customer_became_vip` have no domain consumer, while two
   services hold customer replicas that would need them.
2. Whether the `customers:read` ACL is enforced by the framework or is advisory configuration.
3. Whether this service re-checks the `admin` role, or trusts the gateway entirely.
4. What business rule promotes a customer to VIP — visible in `Core/Entities/Customer.cs` but not
   stated anywhere as a policy.
5. Whether `outbox.disableTransactions: true` is deliberate.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Customers.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Customers.Application/`,
`src/Pacco.Services.Customers.Core/`, `src/Pacco.Services.Customers.Infrastructure/`, and the
repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor). The only browser-facing surface is the Convey Swagger UI at
`/docs`, generated by `Convey.WebApi.Swagger`.

---

## Evidence

| Fact | File |
|---|---|
| Route table and host composition | `src/Pacco.Services.Customers.Api/Program.cs` |
| DI composition, Mongo collection registration, outbox decorators | `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` |
| Customer state machine and VIP rule | `src/Pacco.Services.Customers.Core/Entities/Customer.cs`, `Entities/State.cs` |
| Persistence document | `src/Pacco.Services.Customers.Infrastructure/Mongo/Documents/CustomerDocument.cs` |
| Commands | `src/Pacco.Services.Customers.Application/Commands/CompleteCustomerRegistration.cs`, `ChangeCustomerState.cs` |
| Published events and payloads | `src/Pacco.Services.Customers.Application/Events/*.cs` |
| Rejected events | `src/Pacco.Services.Customers.Application/Events/Rejected/*.cs` |
| Consumed external events | `src/Pacco.Services.Customers.Application/Events/External/Handlers/*.cs` |
| Certificate ACL, exchange, Vault, JWT, logging, metrics, tracing configuration | `src/Pacco.Services.Customers.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Package set | `src/Pacco.Services.Customers.Infrastructure/Pacco.Services.Customers.Infrastructure.csproj`, `src/Pacco.Services.Customers.Api/Pacco.Services.Customers.Api.csproj` |
| Project list | `Pacco.Services.Customers.sln` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Replicated customer copies elsewhere | `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Mongo/Documents/CustomerDocument.cs`, `../hianshul100_Pacco.Services.Parcels/src/Pacco.Services.Parcels.Infrastructure/Mongo/Documents/CustomerDocument.cs` |
| Certificate-authenticated caller | `../hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Services/Clients/CustomersServiceClient.cs` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | This service is the single owner of customer data, and the copies in `orders-service` and `parcels-service` are read-only projections | Only this service exposes customer write endpoints and publishes customer events; the other two only subscribe | If either replica is written independently, there are three sources of truth for a customer and no reconciliation anywhere | Read the write paths in `orders-service` and `parcels-service` for their `customers` collections |
| A2 | Customers are created only in reaction to the `signed_up` event | There is no create-customer command; `CompleteCustomerRegistration` updates an existing record | A second creation path would bypass the event that populates the two downstream replicas, leaving them permanently missing that customer | Confirm with the Identity and Customers domain owners |
| A3 | The `customers:read` ACL is enforced by `Convey.WebApi.Security` at request time | The ACL is declared in configuration and the calling service attaches the matching certificate, which only makes sense if something checks it | Cross-service calls would be unauthenticated while appearing protected by configuration | Read the Convey 0.4 security implementation, or call the endpoint without a certificate against a running instance |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should `orders-service` and `parcels-service` consume `customer_state_changed` and `customer_became_vip`? | Both services keep a customer copy populated once at creation and never updated. Every state change — including VIP promotion — leaves those copies wrong, and nothing anywhere notices. If either service ever bases a decision on its local copy, customers get treated according to stale status | Either subscribe both services to the state events, or state explicitly that only the creation snapshot matters to them | Domain owners for Customers, Orders and Parcels |
| Q2 | **[ACTION NOW]** What rule promotes a customer to VIP, and who is supposed to act on it? | `customer_became_vip` is published but no service consumes it, so becoming a VIP currently changes nothing outside this service. Meanwhile `pricing-service` calculates discounts from customer data it fetches synchronously — so the two mechanisms may be solving the same problem twice | Confirm whether VIP status is meant to drive pricing, and if so which path is authoritative | Domain owner for Customers |
| Q3 | **[ACTION NOW]** Does this service verify the `admin` role itself, or rely entirely on the gateway? | `GET /customers` returns every customer and `PUT /customers/{id}/state/{state}` changes any customer's status. Both are gated at the gateway only, as far as the code shows. Anything that reaches this service without passing the gateway is unrestricted | Confirm whether direct service-to-service access is possible in the deployed network, and whether the role check should also live here | Whoever owns Pacco authentication |
| Q4 | **[handled later by HLD]** Is `outbox.disableTransactions: true` the intended setting? | Without transactions the customer write and the outbox write are not atomic, so a crash between them can create a customer that no downstream service ever hears about — leaving both replicas permanently missing that record | Likely a single-node MongoDB constraint in development; confirm the production topology | Platform architect |
