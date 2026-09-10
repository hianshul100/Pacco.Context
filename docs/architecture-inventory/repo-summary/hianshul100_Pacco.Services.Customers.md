# Repository summary — `hianshul100_Pacco.Services.Customers`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Customers` (also known as: Pacco.Services.Customers, the Customers service)
**Deployable:** `Pacco.Services.Customers.Api` (also known as: `customers-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.customers`, its published image). Repository: `hianshul100_Pacco.Services.Customers`, path: `src/Pacco.Services.Customers.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

Owns the customer profile: the identity that Identity created becomes a customer here once registration is completed with a name and address. It also owns the VIP decision — `Core/Services/VipPolicy.cs` promotes a customer to VIP based on their completed orders, and that promotion is what Pricing later turns into a discount.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, Convey-based, serving HTTP dispatcher endpoints and consuming RabbitMQ messages in the same process.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Customers.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Customers.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Customers.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Customers.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Dispatches to |
|---|---|---|
| GET | `customers` | `GetCustomers` |
| GET | `customers/{customerId}` | `GetCustomer` |
| GET | `customers/{customerId}/state` | `GetCustomerState` |
| POST | `customers` | `CompleteCustomerRegistration` |
| PUT | `customers/{customerId}/state/{state}` | `ChangeCustomerState`, responds No Content |

## 4. Important modules / packages

Four projects: `src/Pacco.Services.Customers.Api`, `.Application`, `.Core`, `.Infrastructure`.

- **Core** — `Entities/Customer.cs`, `Entities/State.cs`, `Entities/AggregateRoot.cs`, `Entities/AggregateId.cs`; domain events `CustomerRegistrationCompleted`, `CustomerStateChanged`, `CustomerBecameVip`; `Services/VipPolicy.cs` behind `IVipPolicy`; domain exceptions including `CannotChangeCustomerStateException`, `InvalidCustomerAddressException`, `InvalidCustomerFullNameException`; `Repositories/ICustomerRepository.cs`.
- **Application** — commands `CompleteCustomerRegistration`, `ChangeCustomerState` with handlers; integration events `CustomerCreated`, `CustomerStateChanged`, `CustomerBecameVip`; rejected events `CompleteCustomerRegistrationRejected`, `ChangeCustomerStateRejected`; external events `SignedUp`, `OrderCompleted` with handlers; queries `GetCustomer`, `GetCustomers`, `GetCustomerState`; DTOs `CustomerDto`, `CustomerDetailsDto`, `CustomerStateDto`; `ContractAttribute.cs`.
- **Infrastructure** — `Mongo/Documents/CustomerDocument.cs`, `Mongo/Repositories/CustomerMongoRepository.cs`, three query handlers, `Services/EventMapper.cs`, `Services/MessageBroker.cs`, `Services/DateTimeProvider.cs`, `Contexts/{AppContext,AppContextFactory,CorrelationContext,IdentityContext}.cs`, `Decorators/{OutboxCommandHandlerDecorator,OutboxEventHandlerDecorator}.cs`, `Exceptions/{ExceptionToMessageMapper,ExceptionToResponseMapper}.cs`, `Logging/MessageToLogTemplateMapper.cs`.

Packages follow the platform's standard Convey set: `Convey`, `Convey.CQRS.*`, `Convey.MessageBrokers.RabbitMQ`, `Convey.MessageBrokers.Outbox.Mongo`, `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis`, `Convey.Discovery.Consul`, `Convey.LoadBalancing.Fabio`, `Convey.Tracing.Jaeger(.RabbitMQ)`, `Convey.Metrics.AppMetrics`, `Convey.Secrets.Vault`, `Convey.Auth`, `Convey.Security`, `Convey.WebApi.CQRS`, `Convey.WebApi.Swagger`, `Convey.Logging`, all pinned to `0.4.*`.

## 5. External integrations

MongoDB (database `customers-service`), Redis (instance prefix `customers:`), RabbitMQ (exchange `customers`), Consul (service `customers-service`, port 5002, ping endpoint `ping`), Fabio (`http://localhost:9999`), Vault (kv v2 mount `kv`, path `customers-service/settings`; PKI common name `customers.pacco.io`; dynamic MongoDB credentials with auto-renewal), Jaeger (service name `customers`, UDP `localhost:6831`, `sampler: const`), Prometheus (`prometheus: true`, `influx: false`, database `pacco`), Seq.

`httpClient.services` is **empty** — this service calls no other service over HTTP. It is a pure downstream consumer of events and an upstream source for everyone else.

## 6. Data stores and state handling

**Store:** MongoDB, database `customers-service`. **Query mechanism:** Convey MongoDB repositories over the MongoDB .NET driver. **No ORM. No migration tool.**

| Collection | Fields |
|---|---|
| `customers` | `Id` (Guid), `Email` (string), `FullName` (string), `Address` (string), `IsVip` (bool), `State` (enum), `CreatedAt` (DateTime), `CompletedOrders` (collection of Guid) |
| `inbox` | Convey inbox, de-duplication |
| `outbox` | Convey outbox, pending publishes |

Outbox: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Cross-domain coupling.** `CustomerDocument.Id` is the same `Guid` as the Identity service's user identifier — the `SignedUp` handler creates the customer with the identity's `UserId`. `CompletedOrders` is a growing list of order identifiers owned by `orders-service`, appended whenever an `OrderCompleted` event arrives. Neither is a database foreign key; both are logical references maintained by event handlers, so drift between services is possible and nothing detects it. `CompletedOrders` is also unbounded — a long-lived customer document grows without limit.

Redis is registered in the composition but no cache key or cache call appears in this service's own code, so what it holds is **unknown — requires runtime capture**.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Mongo-backed transactional outbox and the Jaeger RabbitMQ plugin.

**Broker settings:** exchange `customers`, `conventionsCasing: snakeCase`, queue name template `customers-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `customers`:**

- Events: `customer_created`, `customer_became_vip`, `customer_state_changed`
- Rejected events: `change_customer_state_rejected`, `complete_customer_registration_rejected`
- Commands accepted on the same exchange: `change_customer_state`, `complete_customer_registration`

Observable payload: `CustomerCreated { Guid CustomerId }`. `customer_became_vip` and `customer_state_changed` follow the same identifier-carrying shape; the state change also carries the new state. Full payload field lists beyond `CustomerCreated` are **unknown — requires runtime capture** for the two rejected events, which carry `Reason` and `Code`.

**Consumed:**

| Kind | Message | Origin exchange | Declared in |
|---|---|---|---|
| Command | `CompleteCustomerRegistration` | `customers` | own commands |
| Command | `ChangeCustomerState` | `customers` | own commands |
| Event | `SignedUp` | `identity` | `Application/Events/External/SignedUp.cs`, `[Message("identity")]` |
| Event | `OrderCompleted` | `orders` | `Application/Events/External/OrderCompleted.cs`, `[Message("orders")]` |

`SignedUp { Guid UserId, string Email, string Role }` is what creates the customer record. `OrderCompleted { Guid OrderId, Guid CustomerId }` is what feeds `CompletedOrders` and therefore the VIP policy.

`customer_created` is one of the most widely consumed events in the platform: `availability-service`, `orders-service` and `parcels-service` all subscribe to it, each with its own local copy of the class.

## 8. APIs exposed and consumed

**Exposed:** the five HTTP routes in section 3, Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5002`; port 80 in the container.

Two of these routes are consumed by other services rather than by end users:

- `GET /customers/{customerId}/state` — called by `availability-service`.
- `GET /customers/{customerId}` — called by `pricing-service`.

Through the gateway, `GET /customers`, `GET /customers/{customerId}` and `GET /customers/{customerId}/state` all require the `admin` role, while `GET /customers/me` rewrites to `customers-service/customers/@user_id` so a caller can only read their own record.

**Consumed:** no outbound HTTP calls.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Customers.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `customers-service` on host port 5002.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.customers`.

## 10. Security and auth clues

- JWT validated against the committed `certs/localhost.cer`, `validIssuer: pacco`.
- `Convey.Security` is composed; there is no certificate authentication here (only Availability has that).
- Authorisation is enforced at the gateway by role claim, not in this service — the service's own endpoints do not check roles. Anything that can reach `customers-service` directly on port 5002 can read every customer record without a token check for role.
- `logger.excludeProperties` keeps secret-like values out of logs; `httpClient.requestMasking` is configured.
- Vault supplies settings and dynamic MongoDB credentials; RabbitMQ credentials remain the plain defaults in `appsettings.json`.
- The service holds personal data — email, full name, address — with no field-level encryption and no retention rule in code.

## 11. Observability, logging and tracing clues

Jaeger under service name `customers`; Prometheus metrics via `Convey.Metrics.AppMetrics` (`metrics.database: pacco`, interval 5); structured logs to console, file and Seq, with `/`, `/ping` and `/metrics` excluded. `Infrastructure/Logging/MessageToLogTemplateMapper.cs` gives every command and event its own log message template, which is how message flow stays readable in Seq.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Customers.Infrastructure/Extensions.cs` (technology choices and subscriptions), `src/Pacco.Services.Customers.Api/Program.cs` (HTTP surface), `src/Pacco.Services.Customers.Api/appsettings.json` (integration endpoints), `src/Pacco.Services.Customers.Core/Services/VipPolicy.cs` (the VIP rule, a business decision embedded in code), `src/Pacco.Services.Customers.Core/Entities/Customer.cs` and `State.cs` (the customer lifecycle).

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations (`consul.enabled`, `fabio.enabled`, `metrics.enabled`, `swagger.enabled`, `logger.seq.enabled`, `logger.elk.enabled`, `vault.enabled`) — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Customers.Api/`, `src/Pacco.Services.Customers.Application/`, `src/Pacco.Services.Customers.Core/`, `src/Pacco.Services.Customers.Infrastructure/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5002`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.customers`; `Pacco.Services.Customers.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the entire messaging surface (two consumed commands, two consumed external events, three published events, two rejected events); the VIP policy, which is the service's most consequential business rule; the Vault integration and dynamic database credentials; the transactional outbox; the fact that two other services call this one synchronously.

**Conflicts to surface:**

- **Stale doc.** The README instructs running `dotnet run` in `/src/Pacco.Services.Customers`. That directory does not exist — the host project is `src/Pacco.Services.Customers.Api`. `scripts/start.sh` has the correct path.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`, but this repository contains **no test project at all**. The test step passes because there is nothing to run, which reads as a green build with zero coverage.
- **Gateway conflict.** The synchronous gateway profile posts to `customers-service/customers` with `payload: create_customer` and `schema: create_customer.schema`, while the asynchronous profile publishes routing key `complete_customer_registration` with `payload: complete_customer_registration`. The two profiles disagree on the name of the customer-creation payload, and neither payload file exists in the gateway repository.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The customer identifier is the same value as the Identity user identifier. | The `SignedUp` handler creates the customer from the identity event, and the gateway binds `customerId:@user_id` on customer routes. |
| A2 | `messages.json` in the Operations repository accurately names this service's published messages. | It is the platform's only message catalogue and Operations builds live subscriptions from it. |
| A3 | Role-based access to customer data is enforced only at the gateway. | No role check appears in this service's endpoint registration or handlers. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** The service stores personal data (email, full name, address) with no encryption at rest configured, no retention rule and no deletion path in code. | Likely a data-protection compliance gap. | Customers service maintainer, with the platform's data-protection owner. |
| B2 | **[ACTION NOW]** `CompletedOrders` grows without bound on every completed order. | A high-volume customer document will eventually hit MongoDB's document size limit and the service will start failing writes for that customer. | Customers service maintainer: store a count, or move completed orders out of the document. |
| B3 | **[ACTION NOW]** CI runs `dotnet test` against a repository with no test projects. | The build reports success while testing nothing. | Customers service maintainer: add tests, or stop implying they exist. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Which payload name is correct for creating a customer — `create_customer` or `complete_customer_registration`? | The two gateway profiles disagree, so one of the two write paths is likely broken. | Gateway maintainer with the Customers service maintainer. |
| Q2 | **[handled later by the security-architecture stage]** Should the service enforce the `admin` role itself rather than trusting the gateway? | Anything on the internal network can currently read every customer record. | Security-architecture stage. |
| Q3 | **[handled later by the data-architecture stage]** What is stored in Redis under the `customers:` prefix? | Not answerable from source; determines whether Redis is a hard dependency here. | Data-architecture stage, by inspecting a running instance. |
| Q4 | **[handled later by the domain-model stage]** What are the valid values of `State` and the allowed transitions between them? | The gateway exposes `PUT /customers/{customerId}/state/{state}` as a free-form path segment, so the accepted values are part of the public API. | Domain-model stage. |
