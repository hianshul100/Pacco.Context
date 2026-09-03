---
title: "Repository Summary — Pacco.Services.Customers"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Customers"
status: "evidence-based inventory"
---

# Pacco.Services.Customers

**Primary name:** `Pacco.Services.Customers` (aliases used in this file: `customers-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `customers` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Customers`, path: `hianshul100_Pacco.Services.Customers/`

---

## 1. Primary purpose

Owns the customer profile: registration completion, customer state, and the VIP policy that decides when a customer is promoted.

Evidence: `src/Pacco.Services.Customers.Core/Entities/Customer.cs`, `src/Pacco.Services.Customers.Core/Entities/State.cs`, `src/Pacco.Services.Customers.Core/Services/VipPolicy.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service using Convey dispatcher endpoints, plus a RabbitMQ subscriber. Listens on `5002`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` | `src/Pacco.Services.Customers.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

Four source projects: `Pacco.Services.Customers.Api`, `.Application`, `.Core`, `.Infrastructure`. **No test project exists in this repository.**

- **Core:** `Entities/Customer.cs`, `Entities/State.cs`, `Entities/AggregateId.cs`, `Entities/AggregateRoot.cs`, `Services/IVipPolicy.cs`, `Services/VipPolicy.cs`, `Repositories/ICustomerRepository.cs`, domain events `CustomerBecameVip`, `CustomerRegistrationCompleted`, `CustomerStateChanged`.
- **Application:** commands `ChangeCustomerState`, `CompleteCustomerRegistration` with handlers; integration events `CustomerBecameVip`, `CustomerCreated`, `CustomerStateChanged`; external events `OrderCompleted`, `SignedUp` with handlers; two rejected events; queries `GetCustomer`, `GetCustomerState`, `GetCustomers`.
- **Infrastructure:** MongoDB documents, repositories and query handlers, plus the outbox decorators, event mapper, message broker and exception mappers that every service in this platform shares.

Convey `0.4.*` packages as used across the platform.

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus. No outbound HTTP service clients are defined; this service is called by others rather than calling them.

## 6. Data stores / state

- **Store:** MongoDB, database `customers-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with explicit document classes and hand-written repositories.
- **Collections:** a customers collection derived from the customer document type, plus `inbox` and `outbox`.
- **Migration tool:** none anywhere in the repository.
- **Cross-domain coupling:** none inbound. Note that `Pacco.Services.Orders` and `Pacco.Services.Parcels` each keep their own `CustomerDocument` copy, so customer identity is replicated across services by event rather than joined by a foreign key.
- **Cache:** Redis.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `customers`, snake-case naming conventions, queue template `customers-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`. Transactional outbox and inbox on MongoDB (`inbox`, `outbox`).

**Commands consumed:** `change_customer_state`, `complete_customer_registration`.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `customer_created` | `Application/Events/CustomerCreated.cs` | `CustomerId` (Guid) — read directly from the class |
| `customer_became_vip` | `Application/Events/CustomerBecameVip.cs` | customer identifier |
| `customer_state_changed` | `Application/Events/CustomerStateChanged.cs` | customer identifier, state |

**Rejected events published:** `change_customer_state_rejected`, `complete_customer_registration_rejected`.

**External events consumed:** `signed_up` from `Pacco.Services.Identity` (creates the customer record) and `order_completed` from `Pacco.Services.Orders` (feeds the VIP policy). Handlers live in `Application/Events/External/Handlers/`.

Wire names are confirmed against `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`):

| Method | Path | Dispatched type |
|---|---|---|
| `GET` | `customers` | `GetCustomers` |
| `GET` | `customers/{customerId}` | `GetCustomer` |
| `GET` | `customers/{customerId}/state` | `GetCustomerState` |
| `POST` | `customers` | `CompleteCustomerRegistration`, responds `Created` at `customers/{cmd.CustomerId}` |
| `PUT` | `customers/{customerId}/state/{state}` | `ChangeCustomerState`, responds `NoContent` |

Consumed by: `Pacco.APIGateway` (module `customers`), `Pacco.Services.Availability` (`CustomersServiceClient`), `Pacco.Services.Pricing` (`CustomersServiceClient`).

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.customers`, published `5002:80` in `hianshul100_Pacco/compose/services.yml`, `restart: unless-stopped`, network `pacco`. Consul registration on port `5002`.

CI: `.travis.yml` runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` on success.

## 10. Security / auth clues

This is the **only service in the platform that defines a caller access-control list**. `appsettings.json` contains:

```
security.certificate:
  allowedDomains: ["pacco.io"]
  allowedHosts:   ["localhost"]
  acl:
    availability-service:
      validIssuer: "localhost"
      permissions: ["customers:read"]
```

So certificate-based service-to-service authorisation is configured, granting `availability-service` the `customers:read` permission only. Vault supplies the certificate: KV path `customers-service/settings`, PKI common name `customers-service.pacco.io`.

JWT validation follows the platform pattern with `validIssuer: pacco`.

**Observation, not a defect:** `Pacco.Services.Pricing` also calls this service, but it is not listed in the access-control list. **Needs validation.**

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: customers`, including RabbitMQ span propagation; structured logging via `Convey.Logging` and `Convey.Logging.CQRS` with a message-to-log-template mapper; Prometheus metrics via `Convey.Metrics.AppMetrics`.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Customers.Core/Services/VipPolicy.cs` — the promotion rule that gives this service its business meaning.
- `src/Pacco.Services.Customers.Application/Events/External/Handlers/SignedUpHandler.cs` — the decision that a customer record is created by reacting to an identity event rather than by a direct call.
- `src/Pacco.Services.Customers.Api/appsettings.json` — the access-control list, the only one of its kind in the platform.
- The outbox decorators in `Pacco.Services.Customers.Infrastructure/Decorators/`.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans in `appsettings.json`, which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` contains only the four C# projects. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes a customers service on .NET Core 3.1 built with Convey, following clean architecture | Confirmed: four layered projects, Convey `0.4.*` throughout, `netcoreapp3.1` | Confirmed |
| The build script chain includes `./scripts/test.sh` | There is no test project in this repository, so the step has nothing to execute | Needs validation |
| The platform README presents service-to-service security as a uniform concern | Only this service defines an access-control list; `Pacco.Services.Deliveries` has no security block at all | Stale doc — security configuration is not uniform across services |
| The access list grants `availability-service` read access | `Pacco.Services.Pricing` also calls this service but is not in the list | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the VIP policy and the customer state machine — present in code, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | A customer record is created only in response to a sign-up event from the identity service, never by a direct call. | The only creation path in the code is the handler for that event; the write endpoint completes an existing registration. |
| A2 | The access-control list is not enforced in day-to-day local development. | It depends on certificates issued by the secrets manager, which is started in development mode in the shared stack. |

### Blockers

_None._

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Should the pricing service be added to the caller access list, or does it reach this service by a path the list does not cover? | **[ACTION NOW]** Confirm with the requesting team; the answer changes how service-to-service trust is described for the whole platform. |
| Q2 | What rule decides when a customer becomes a VIP, in business terms? | **[handled later by the ADR authoring stage]** Read the policy class with a business owner present and record the rule. |
| Q3 | Why does this service have no automated tests when the availability service has four test projects? | **[handled later by the ADR authoring stage]** Record the testing approach for the platform as a whole. |
