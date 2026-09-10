# Repository Summary — `hianshul100_Pacco.Services.Customers`

**Primary name:** `customers-service` (aliases used in this file: `Pacco.Services.Customers.Api` — the .NET project and assembly name; `devmentors/pacco.services.customers` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Customers`, path: `src/Pacco.Services.Customers.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Owns the customer profile and the customer *state* lifecycle. It completes a customer's registration after `identity-service` has signed them up, exposes customer details and state to other services, and promotes customers to VIP. It is the platform's reference data source for "who is this customer and what may they do".

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`, with an in-process RabbitMQ consumer. Layered `.Api` / `.Application` / `.Core` / `.Infrastructure`.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Customers.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Customers.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Customers.sln`: `src/Pacco.Services.Customers.Api`, `.Application`, `.Core`, `.Infrastructure`. `.Core` has no NuGet references. `.Infrastructure` carries the full Convey stack and additionally references `Convey.Auth` and `Convey.WebApi.Security` — only this service and `availability-service` reference the latter.

## 5. External integrations

- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — all through Convey extensions.
- No outbound HTTP calls to peer services: `httpClient.services` in `src/Pacco.Services.Customers.Api/appsettings.json` is empty. This service is called, it does not call.

## 6. Data stores & state

- **MongoDB.** Database `customers-service`, connection string `mongodb://localhost:27017`, `seed: false`.
- **Collection:** `customers` — `.AddMongoRepository<CustomerDocument, Guid>("customers")`.
- **Outbox collections:** `outbox` and `inbox` in the same database (`outbox.enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `connectionString: localhost`, key prefix `customers:`.
- **Query mechanism:** Convey `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver. **No ORM.**
- **Migration tool:** none.
- **Cross-domain coupling:** this service is the *source* of a coupling rather than a victim of one. `orders-service` and `parcels-service` each keep their own local `customers` collection, populated from the `customer_created` event this service publishes. There are no foreign keys — MongoDB does not enforce them — but the customer identifier is duplicated into two other databases and kept in step only by eventual consistency.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `customers`, queue template `customers-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Mongo-backed outbox via `.AddMessageOutbox(o => o.AddMongo())`, Jaeger plugin via `.AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`.

**Consumed** (`src/Pacco.Services.Customers.Infrastructure/Extensions.cs`, lines 93–96):

| Kind | Message | Wire name | Origin |
|---|---|---|---|
| Command | `CompleteCustomerRegistration` | `complete_customer_registration` | `api-gateway` async mode |
| Command | `ChangeCustomerState` | `change_customer_state` | `api-gateway` async mode |
| Event | `SignedUp` | `signed_up` | `identity-service` |
| Event | `OrderCompleted` | `order_completed` | `orders-service` |

**Published**, per the `customers-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: `customer_created`, `customer_became_vip`, `customer_state_changed`; rejection events `change_customer_state_rejected`, `complete_customer_registration_rejected`.

**Payload key fields:** `ntrada-async.yml` names a payload and schema pair `complete_customer_registration` / `complete_customer_registration.schema` for this exchange, and binds `customerId:@user_id`. `customer_created` is known to carry at least a customer identifier, because `orders-service`, `parcels-service` and `availability-service` all key their local records on it. The complete field set of each message is **unknown — requires runtime capture**.

Note the loop worth flagging: `orders-service` publishes `order_completed`, this service consumes it and may publish `customer_became_vip`. Customer status is therefore derived from order history asynchronously.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.Customers.Api/Program.cs`:

| Method | Path | Dispatched to | Response |
|---|---|---|---|
| GET | `customers` | `GetCustomers` | collection of customer DTOs |
| GET | `customers/{customerId}` | `GetCustomer` | `CustomerDetailsDto` |
| GET | `customers/{customerId}/state` | `GetCustomerState` | `CustomerStateDto` |
| POST | `customers` | `CompleteCustomerRegistration` | — |
| PUT | `customers/{customerId}/state/{state}` | `ChangeCustomerState` | `204 No Content` |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Consumed by:** `api-gateway` (all five routes; the three `GET` routes require `role: admin` at the gateway), `availability-service` over HTTP, `pricing-service` over HTTP.

**Consumes:** nothing over HTTP.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5002`.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.customers`.

## 10. Security & auth clues

This service has the richest security configuration in the platform. Its `security` block in `src/Pacco.Services.Customers.Api/appsettings.json` is the only one that defines an access-control list:

- `certificate.enabled: true`, `allowedDomains: ["pacco.io"]`, `allowSubdomains: true`, `allowedHosts: ["localhost"]`.
- An ACL entry granting `availability-service` (with `validIssuer: localhost`) the permission `customers:read`. This is the platform's only declarative service-to-service authorisation rule.
- Matching Vault PKI setup exists in `hianshul100_Pacco/docker-images.txt`: a `customers-service` PKI role with `allowed_domains=pacco.io`.
- Vault: `enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, key-value path `customers-service/settings`, PKI `roleName: customers-service`, `commonName: customers-service.pacco.io`, and a `lease.mongo` block for dynamic MongoDB credentials with `autoRenewal: true`.
- JWT validated against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`.
- **Checked-in credentials:** `vault.token: "secret"`, RabbitMQ `guest`/`guest`, Seq `apiKey: secret`.
- `.UsePublicContracts<ContractAttribute>()` exposes the message contract surface.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: customers`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker.
- **Correlation:** `Correlation-Context` header read on inbound requests; the `Saga` header is forwarded.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]`; `excludeProperties` redacts `api_key`, `access_key`, `ApiKey`, `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`, `Password`, `Email`, `Login`, `Secret`, `Token`. `.AddHandlersLogging()` logs each handler.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Customers.Api/appsettings.json` — the `security` ACL block is a genuine architectural decision recorded nowhere else: service-to-service authorisation is done with client certificates plus a named permission, not with tokens.
- `src/Pacco.Services.Customers.Infrastructure/Extensions.cs` — the composition root and subscription list.
- `src/Pacco.Services.Customers.Api/Program.cs` — the route table.
- `Pacco.Services.Customers.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- The ACL grants `availability-service` `customers:read`, yet `pricing-service` also calls this service over HTTP and is **not** in the ACL. Whether `pricing-service` calls are currently rejected, or whether the ACL is only advisory, is **Unknown**. **Needs validation** — this is the sharpest inconsistency found in this repository.
- The set of valid customer *states* is defined in `src/Pacco.Services.Customers.Core/` and was not enumerated; the accepted values of the `{state}` route parameter are **Unknown**.
- The condition under which `customer_became_vip` fires is **Unknown** from configuration alone.
- Full payload field sets are **unknown — requires runtime capture**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Customers.Api/`, `src/Pacco.Services.Customers.Application/`, `src/Pacco.Services.Customers.Core/`, `src/Pacco.Services.Customers.Infrastructure/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Customers` directory". | No such directory. The runnable project is `src/Pacco.Services.Customers.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5002`." | `appsettings.json` sets the Consul port to `5002`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.customers`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Customers.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README says nothing about messaging, the ACL, or certificate authentication. | All three are present and architecturally significant. | **Docs gap.** The README is the shared platform template with the service name substituted. |

**On disk but not mentioned in `README.md`:** the four message subscriptions, the outbox, the `security` ACL, the Vault integration, and the fact that two other services keep replicas of this service's data.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `customer_created` carries the customer identifier and enough profile data for `orders-service` and `parcels-service` to build their local replicas. | Both services subscribe to it and both maintain a `customers` collection keyed by that identifier. | The replica-building logic in two other services would be misdescribed. | Read the `CustomerCreated` event class and the consuming handlers. |
| A2 | The events published by this service are exactly those listed under `customers-service` in `messages.json`. | `operations-service` reads that file at runtime to build subscriptions, so it must match reality. | An event could be missing from the platform catalogue. | Read the event classes under `src/Pacco.Services.Customers.Application/Events/`. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Why is `pricing-service` absent from this service's access-control list when it calls this service over HTTP? | Either `pricing-service` is silently failing its customer lookups, or the ACL is not actually enforced. Both change how the platform's authorisation model should be described. | The ACL may only be enforced when a client certificate is presented, and `pricing-service` may not present one. | Service owner |
| Q2 | **[handled later by the platform inventory review]** Is the certificate-plus-ACL model intended to become the platform-wide standard for service-to-service authorisation? | Right now exactly one service defines an ACL and exactly one other authenticates with a certificate. That is either a pilot or an oversight. | Looks like a pilot on the `availability-service` → `customers-service` call only. | Platform architect |
| Q3 | **[handled later by the platform inventory review]** What triggers `customer_became_vip`? | It is a business rule embedded in code with no documentation. | Derived from `order_completed` events, but the threshold is not visible in configuration. | Product owner |
