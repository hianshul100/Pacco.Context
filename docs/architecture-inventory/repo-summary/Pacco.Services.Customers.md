# Repository summary — `Pacco.Services.Customers`

**Repository:** `Pacco.Services.Customers` (workspace clone path: `hianshul100_Pacco.Services.Customers`)
**Deployable:** `customers-service` (also known as: Customers Service, `Pacco.Services.Customers.Api`, image `devmentors/pacco.services.customers`). **Repository: `Pacco.Services.Customers`, path: `src/Pacco.Services.Customers.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Customers
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Owns the **customer profile and customer lifecycle state**. It completes a customer's registration after Identity signs a user up, tracks their state (`incomplete` → `valid` → `invalid`/`locked`), counts completed orders, and applies the **VIP policy**. It is also the platform's authority on "may this customer transact", queried by `availability-service` before a reservation is accepted.

**Evidence:** `src/Pacco.Services.Customers.Core/Entities/Customer.cs`, `src/Pacco.Services.Customers.Core/Services/VipPolicy.cs`, `src/Pacco.Services.Customers.Api/Program.cs`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice plus RabbitMQ consumer, in one process, using the canonical four-project clean-architecture layering (`.Api`, `.Application`, `.Core`, `.Infrastructure`) on Convey.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Customers.Api/Program.cs` — `AddConvey().AddWebApi().AddApplication().AddInfrastructure()`, then `UseInfrastructure()` + `UseDispatcherEndpoints(...)` |
| RabbitMQ subscriptions | `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` → `UseInfrastructure` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Customers.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Customers.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Customers.Api` | Host, endpoint map, `appsettings.json` |
| `src/Pacco.Services.Customers.Application` | `Commands/` (`CompleteCustomerRegistration`, `ChangeCustomerState`) + handlers; `Queries/` (`GetCustomer`, `GetCustomers`, `GetCustomerState`) + handlers; `Events/`, `Events/External/`, `Events/Rejected/`; `DTO/` |
| `src/Pacco.Services.Customers.Core` | `Entities/Customer.cs`, `Entities/AggregateRoot.cs`, `Services/VipPolicy.cs`, `Events/`, `Exceptions/`, `Repositories/ICustomerRepository` |
| `src/Pacco.Services.Customers.Infrastructure` | `Mongo/Documents/CustomerDocument.cs`, `Mongo/Repositories/CustomerMongoRepository.cs`, `Decorators/` (outbox), `Contexts/`, `Exceptions/ExceptionToResponseMapper.cs` + `ExceptionToMessageMapper.cs`, `Logging/`, `Extensions.cs` |

**No test projects exist in this repository.**

Convey packages match the platform set (`Convey`, `Convey.CQRS.*`, `Convey.Discovery.Consul`, `Convey.HTTP`, `Convey.LoadBalancing.Fabio`, `Convey.Logging`, `Convey.Logging.CQRS`, `Convey.MessageBrokers.CQRS`, `Convey.MessageBrokers.Outbox.Mongo`, `Convey.MessageBrokers.RabbitMQ`, `Convey.Metrics.AppMetrics`, `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis`, `Convey.Secrets.Vault`, `Convey.Security`, `Convey.Tracing.Jaeger`, `Convey.WebApi.*`), all `0.4.*`.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| RabbitMQ | in + out | exchange `customers`, topic |
| MongoDB | out | database `customers-service` |
| Redis | out | instance prefix `customers:` |
| Consul | out | registers `customers-service` on port `5002` |
| Fabio | out | `http://localhost:9999` |
| Vault | out | KV v2 `kv/customers-service/settings`; PKI role `customers-service`, CN `customers-service.pacco.io`; dynamic Mongo credentials |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

`httpClient.services` is **empty** — this service makes **no outbound HTTP calls to other services**. It is a pure sink and source: it consumes events, exposes queries, and publishes events. That makes it the least coupled of the domain services in the outbound direction, and the most depended-upon in the inbound direction.

**Evidence:** `src/Pacco.Services.Customers.Api/appsettings.json`; no `Services/Clients/` directory exists in `Application` or `Infrastructure`.

## 6. Data stores / state handling

- **Store:** MongoDB, database `customers-service`.
- **Collections:** `customers` (`AddMongoRepository<CustomerDocument, Guid>("customers")`), plus `inbox` and `outbox`.
- **Access mechanism:** Convey `IMongoRepository<>` over `MongoDB.Driver`. **No ORM.**
- **Migration tool: none.** No Flyway, Liquibase, Alembic, or EF Core migrations.
- **Document shape** (`Infrastructure/Mongo/Documents/CustomerDocument.cs`): the full customer aggregate — id, email, name, address, state, VIP flag, completed-order identifiers, timestamps.
- **Cross-domain coupling — important:** this service is the **origin** of the `customer_created` event that causes `orders-service`, `parcels-service`, and `availability-service` to each record the customer locally. `orders-service` and `parcels-service` each maintain their own `customers` **collection** holding an id-only replica document. There is no database-level foreign key anywhere (MongoDB, separate logical databases), but there is a real logical dependency: three services hold customer identity that is only ever correct if the `customer_created` event was delivered. A missed event leaves those services unable to accept work for that customer.
- **Outbox:** enabled, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: inbox`, `outboxCollection: outbox`, `disableTransactions: true`.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `customers`; `conventionsCasing: snakeCase`; queue template `customers-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — commands:**

| Message | Wire name | Key payload fields |
|---|---|---|
| `CompleteCustomerRegistration` | `complete_customer_registration` | `CustomerId`, `Name`, `Address` |
| `ChangeCustomerState` | `change_customer_state` | `CustomerId`, `State` |

**Consumed — external events:**

| Message | Wire name | Origin | Effect |
|---|---|---|---|
| `SignedUp` | `signed_up` | `identity-service` | creates the customer record (state `incomplete`) |
| `OrderCompleted` | `order_completed` | `orders-service` | records the completed order and re-evaluates the VIP policy |

**Published — events:**

| Event | Wire name | Key payload fields |
|---|---|---|
| `CustomerCreated` | `customer_created` | `CustomerId` |
| `CustomerStateChanged` | `customer_state_changed` | `CustomerId`, `CurrentState`, `PreviousState` |
| `CustomerBecameVip` | `customer_became_vip` | `CustomerId` |

**Published — rejection events:** `complete_customer_registration_rejected`, `change_customer_state_rejected`, each with `Reason` and `Code`, produced by `Infrastructure/Exceptions/ExceptionToMessageMapper.cs`.

**Reliability:** outbox/inbox decorators wrap every command and event handler.

**Fan-out significance:** `customer_created` is the most widely consumed event in the platform — `availability-service`, `orders-service`, and `parcels-service` all subscribe to it.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5002`, container port `80`):

| Method | Path | Maps to | Gateway exposure |
|---|---|---|---|
| GET | `customers` | `GetCustomers` | `/customers` — requires claim `role: admin` |
| GET | `customers/{customerId}` | `GetCustomer` | `/customers/{customerId}` (admin) and `/customers/me` (self, bound to `@user_id`) |
| GET | `customers/{customerId}/state` | `GetCustomerState` | `/customers/{customerId}/state` (admin) |
| POST | `customers` | `CompleteCustomerRegistration` | `/customers`, binds `customerId: @user_id`, JSON-schema validated at the gateway (`create_customer.schema`) |
| PUT | `customers/{customerId}/state/{state}` | `ChangeCustomerState` → `204 No Content` | `/customers/{customerId}/state/{state}` (admin) |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**Consumed:** none over HTTP.

**Called by:** `availability-service` → `GET /customers/{customerId}/state` (with a client certificate in the `Certificate` header); `pricing-service` → `GET /customers/{customerId}`.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Customers.Api.dll`.
- Composed as `customers-service` on `5002:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5002`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**

## 10. Security/auth clues

- **JWT bearer** validation via `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateLifetime: true`.
- **This service is the platform's only certificate-authentication *verifier*.** `appsettings.json` → `security.certificate`:
  - `enabled: true`
  - `header: "Certificate"`
  - `skipRevocationCheck: false`
  - `allowedDomains: ["pacco.io"]`, `allowSubdomains: true`, `allowedHosts: ["localhost"]`
  - **`acl`**: `{ "availability-service": { "validIssuer": "localhost", "permissions": ["customers:read"] } }`

  This is the only explicit **service-to-service authorisation policy** in the entire platform — a named caller granted a named permission. Every other cross-service HTTP call in Pacco is unauthenticated.
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- Log redaction via `logger.excludeProperties` (api keys, secrets, connection strings, passwords, email, login, token).
- Role-based access (`admin`) for customer listing and state changes is enforced **at the gateway** (`ntrada.yml` `claims: role: admin`), not in this service.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: customers-service`, UDP `localhost:6831`, `sampler: const`), with the RabbitMQ Jaeger plugin so broker hops stay in the trace.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink configured but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`. Handler-level logging via `.AddHandlersLogging()`.
- **Correlation:** `Correlation-Context` header read in `Infrastructure/Extensions.cs`; `Saga` header forwarded.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics (unlike `availability-service`).

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Customers.Core/Services/VipPolicy.cs` | **The VIP business rule, hard-coded:** `ApplyVipStatusIfEligible` returns early if the customer is already VIP, returns if `customer.CompletedOrders.Count() < 20`, otherwise calls `customer.SetVip()`. The threshold **20** is a literal in source with no configuration binding and no feature flag. |
| `src/Pacco.Services.Customers.Core/Entities/Customer.cs` | Customer states and the legal transitions between them |
| `src/Pacco.Services.Customers.Api/appsettings.json` | The `security.certificate.acl` — the platform's only service-to-service authorisation policy |
| `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` | Composition: Consul, Fabio, RabbitMQ + outbox, Mongo, Redis, metrics, Jaeger, handler logging |
| `src/Pacco.Services.Customers.Infrastructure/Exceptions/ExceptionToMessageMapper.cs` | Async error contract (rejection events) |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration exists. The only switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `security.certificate.enabled`, `logger.*.enabled`). Business rules — most notably the VIP threshold of 20 completed orders — are **not** configurable; changing them requires a code change and a redeploy.

## 13. Open questions / ambiguities

- **The VIP threshold (20 completed orders) is a magic number** with no config binding. Whether the business expects to tune it is **Unknown**.
- **`customer_became_vip` has no subscriber.** No service in the workspace subscribes to it (`pricing-service` reads the VIP flag over HTTP instead, via `GET /customers/{customerId}`). The event may be published for future use or for external consumers. **Needs validation.**
- **No tests exist** in this repository, despite it owning the VIP policy and the customer state machine — the two pieces of logic most likely to be got wrong.
- The `acl` grants `availability-service` `customers:read`, but whether the permission is actually enforced per-endpoint (as opposed to being a coarse allow-list) was not verified in Convey's source. **Needs validation.**
- The exact customer state vocabulary and which transitions are legal were read from `Customer.cs` but not cross-checked against gateway or client expectations. **Needs validation.**

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (four C# projects only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS files. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Customers service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5002`. — **Confirmed** (`appsettings.json` `consul.port: 5002`, `Pacco/compose/services.yml` `5002:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Customers` directory"**; the actual host project is **`/src/Pacco.Services.Customers.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image all reference the upstream `devmentors` organisation, not the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **The certificate-authentication ACL** — the platform's only service-to-service authorisation policy, and the reason `availability-service` can read customer state. Entirely undocumented.
- **The VIP policy** (`VipPolicy.cs`, threshold 20) — the service's most significant business rule, undocumented.
- The customer state machine and the `PUT /customers/{customerId}/state/{state}` admin endpoint.
- The message contracts: two commands consumed, two external events consumed, three events published, two rejection events.
- The transactional outbox/inbox and the handler decorators.
- `scripts/` (`build.sh`, `test.sh`, `dockerize.sh`).

**Unknown (neither pass yielded proof):**
- Whether the absence of tests is deliberate (the service is considered simple) or an oversight.
- Whether `customer_became_vip` is consumed by anything outside this workspace.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The `customers` collections held by `orders-service` and `parcels-service` are read-only id-only replicas kept in step by the `customer_created` event, not independent sources of truth. | Both `CustomerDocument` classes in those services contain only `public Guid Id { get; set; }`, and both services subscribe to `customer_created`. | If either service writes customer data of its own, there would be three competing customer records and no defined reconciliation. | Read the write paths in both services' Mongo repositories and confirm the only insert is the event handler. |
| A2 | The VIP threshold of 20 completed orders is a product rule that the business is content to change by code deployment. | It is a bare literal in `VipPolicy.cs` with no configuration binding, and the platform has no feature-flag system at all. | A routine business request to tune the threshold would require a code change, review, build, and redeploy of a service — which the business may not expect. | Ask the product owner how often the threshold is expected to change. |
| A3 | The `security.certificate.acl` entry granting `availability-service` the `customers:read` permission is enforced by Convey at request time. | The configuration is explicit and structured, and `availability-service` demonstrably sends the matching header. | Certificate presence might be checked while the named permission is ignored, so the authorisation would be coarser than it looks. | Read the Convey certificate-authentication middleware, or test with a certificate that has no matching ACL entry. |

### Blockers

_None identified for this repository._

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should the VIP threshold and the customer state vocabulary be configurable rather than compiled in? | They are the service's core business rules; today changing either needs a full deployment cycle. | Move the threshold to configuration if the business expects to tune it; otherwise document it as fixed. | Product owner |
| Q2 | **[handled later by architecture_evolution_generation]** Does anything consume `customer_became_vip`? No service in these repositories subscribes to it, and `pricing-service` reads the VIP flag over HTTP instead. | Either the event is dead weight, or there is a consumer outside the workspace that the inventory is missing. | Likely published for future or external use; unverified. | Architecture team |
| Q3 | **[ACTION NOW]** Is the complete absence of tests in this repository acceptable, given it owns the VIP policy and the customer state machine? | These are the two behaviours most likely to break silently, and a break shows up as wrong pricing or blocked reservations. | No — at minimum the VIP policy and state transitions warrant unit tests. | Service owner |
| Q4 | **[ACTION NOW]** What are the legal customer state transitions, and which of them may an administrator force through `PUT /customers/{customerId}/state/{state}`? | An admin endpoint that can set an arbitrary state can put a customer into a state the domain never intended. | Read from `Customer.cs`, but the endpoint's validation was not confirmed. | Product owner |
