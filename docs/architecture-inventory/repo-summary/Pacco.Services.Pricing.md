# Repository: `Pacco.Services.Pricing`

`pricing-service` (also known as: Pricing Service, `Pacco.Services.Pricing`, Docker image
`devmentors/pacco.services.pricing`) computes an order's final price by applying a
customer-dependent discount. It is the platform's only fully stateless service.

- **Repository:** `Pacco.Services.Pricing`, path: `src/Pacco.Services.Pricing.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5008`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, dependency or discount rule.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Claimed at platform level, not true of this repository.** `Pacco/README.md` describes the
platform as built with "clean architecture and DDD". This service is a **single project** with
`Core/`, `DTO/`, `Queries/` and `Services/Clients/` folders inside the `.Api` project — no
`.Application`, `.Core` or `.Infrastructure` assemblies. The platform README does qualify the
claim with "or another style that is the best fit", so this is a documented-as-permitted deviation
rather than a contradiction. **Code is authoritative: this service does not follow the four-project
split.**

**Present on disk, absent from README (disk-only):**

- That this service has **no database, no cache and no message broker** — the only such service.
- `Core/Services/CustomerDiscountsService.cs`, which holds the entire pricing rule set.
- The synchronous dependency on `customers-service`.
- The single `GET /pricing` endpoint and its query parameters.

**Present in every sibling repository, absent here:** a `LICENSE` file. Every other service
repository has one; this one and `Pacco.Services.Vehicles` do not. **Recorded as an observation.**

**Stale doc:** none identified.

**Unknown:** whether the discount tiers in `CustomerDiscountsService.cs` are business-approved
values or placeholders.

---

## 1. Primary purpose

Given a customer and a raw order price, return the price the customer should actually pay, applying
whatever discount their standing earns. It is a pure calculation service with one input dependency.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API — **query-only**. It is the **only service on the platform with no
RabbitMQ connection at all**: no `Convey.MessageBrokers.*` package, no `rabbitMq` configuration
section, no exchange, and no entry in `messages.json`. It neither publishes nor consumes a single
message.

## 3. Key entrypoints

- `src/Pacco.Services.Pricing.Api/Program.cs` — composition root and the single route
  (`GET pricing` → `GetOrderPricing` → `OrderPricingDto`).
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Pricing.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

**Single project:** `src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj`, with
internal folders:

| Folder / file | Role |
|---|---|
| `Core/Services/CustomerDiscountsService.cs` | The discount rules — the whole business logic of the service |
| `Queries/GetOrderPricing.cs`, `Queries/Handlers/GetOrderPricingHandler.cs` | The single query and its handler |
| `DTO/OrderPricingDto.cs` | The response shape returned to callers |
| `Services/Clients/CustomersServiceClient.cs` | The one outbound dependency |

**Key packages:** `Convey`, `Convey.CQRS.Queries`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `Convey.Security`, `Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Swagger`.

**Packages notably absent:** `Convey.CQRS.Commands`, `Convey.CQRS.Events`,
`Convey.MessageBrokers.RabbitMQ`, `Convey.MessageBrokers.Outbox`, `Convey.Persistence.MongoDB`,
`Convey.Persistence.Redis`, `Convey.Secrets.Vault`. The package list is the clearest statement of
what this service is: a query endpoint and an HTTP client, nothing more.

## 5. External integrations

| Integration | How |
|---|---|
| `customers-service` | HTTP `GET {customers}/customers/{customerId}` via `Convey.HTTP` `IHttpClient`; `httpClient.services` maps `customers` |
| Consul | Registration on port 5008, `pingEndpoint: ping` |
| Fabio | `httpClient.type: fabio` |
| Jaeger, Seq, Prometheus | Tracing, logs, metrics |

**No RabbitMQ. No MongoDB. No Redis. No Vault.** Its entire external surface is one HTTP call and
the platform's discovery and observability infrastructure.

## 6. Data stores / state

- **None.** There is no `mongo` or `redis` section in `appsettings.json` and no persistence package
  is referenced. The service holds no state between requests.
- **ORM / query mechanism:** none — there is no data access layer of any kind.
- **Migration tool:** none, and none needed.
- **Tables / collections:** none.
- **Cross-domain coupling:** no data-layer coupling, since there is no data layer. The coupling is
  behavioural and synchronous: every pricing calculation requires a live call to
  `customers-service`, so this service cannot answer at all while that service is unreachable. It
  also means the discount is always computed from *current* customer data rather than from a
  replica — the opposite trade-off from `orders-service` and `parcels-service`, which hold stale
  customer copies. **This is worth noting as the platform's one place where customer state is read
  fresh.**

## 7. Messaging / async / events

**None.** This service participates in no messaging at all:

- No `Convey.MessageBrokers.RabbitMQ` package and no `rabbitMq` configuration section.
- No exchange, no queue, no commands, no events, no rejected events.
- It has **no entry in `messages.json`** — the platform message catalogue lists eight exchanges and
  `pricing` is not among them.
- It publishes nothing, so no other service can react to a price being calculated, and it consumes
  nothing, so it cannot react to customer changes.

There is consequently **no transactional outbox** here and none is needed.

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Pricing.Api/Program.cs`, verbatim):

| Method | Route | Dispatches | Returns |
|---|---|---|---|
| `GET` | `pricing` | `GetOrderPricing` | `OrderPricingDto` |

Query parameters, as used by the caller in `orders-service`: `customerId` and `orderPrice`.
Swagger UI at route prefix `docs`.

**Consumed:** `GET {customers}/customers/{customerId}` on `customers-service`, via
`src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs`.

**Inbound synchronous callers:** `orders-service`, via
`Orders.Infrastructure/Services/Clients/PricingServiceClient.cs`, which calls
`GET {pricing}/pricing?customerId={customerId}&orderPrice={orderPrice}`.

**Upstream:** the gateway module `pricing` exposes `GET /` and rewrites it to
`pricing-service/pricing?customerId=@user_id`. Note that the gateway route binds `customerId` from
the caller's own token, whereas `orders-service` passes an arbitrary `customerId` on the internal
call. **Needs validation** — the two callers reach the same endpoint with different trust
assumptions.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Pricing.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.pricing`.
- Port `5008` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5008:80`), and the
  gateway `localUrl`.
- Consul service name `pricing-service`; `httpClient.type: fabio`.
- **Scale-out characteristics:** being stateless with no broker connection, this service is the
  simplest on the platform to run at any instance count — nothing to shard, no saga state, no
  queue affinity.
- **Runtime dependency note:** it sits on `orders-service`'s order-creation critical path, and it
  in turn depends on `customers-service`. A `customers-service` outage therefore breaks order
  pricing through two hops.
- A `.idea/` directory is present in the source tree — IDE metadata committed to the repository.

## 10. Security / auth clues

- **JWT bearer** via `Convey.Security`, `validIssuer: pacco`, certificate `certs/localhost.cer`.
- **No certificate ACL** and no `security.certificate` enforcement — although `orders-service`
  calls this service synchronously, the call carries no client certificate. `customers-service`
  protects its equivalent inbound call with an ACL; this service does not. **Needs validation.**
- **No Vault** — the only secrets-bearing configuration is the JWT certificate path, and there is
  no dynamic-credential or PKI integration, because the service has no database to get credentials
  for.
- **No role gate at the edge** for the `pricing` module.
- **Trust asymmetry to flag:** the gateway forces `customerId=@user_id`, so an external caller can
  only price for themselves. `orders-service` supplies any `customerId`. If this service does not
  itself verify the caller, the internal path is the weaker one. **Needs validation.**
- **Log masking:** `logger.excludeProperties` removes api key, password and token properties.

## 11. Observability / logging / tracing

- **Tracing:** `Convey.Tracing.Jaeger`, `serviceName: pricing`, UDP `6831`, `const` sampler.
  Note there is **no `Convey.Tracing.Jaeger.RabbitMQ` plugin** — correctly so, since the service
  has no broker connection.
- **Logging:** `Convey.Logging` with console, file and Seq sinks; ELK sink present but
  `enabled: false`.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` | **The discount rules** — the only business policy in the repository, expressed directly in code with no configuration or data source behind it |
| `src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj` | The single-project layout and, by omission, the decision that this service has no persistence, no broker and no Vault — a deliberate departure from the platform template, permitted by the platform README's "or another style that is the best fit" |
| `src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs` | That customer data is read **live** on every request rather than replicated locally — the opposite choice from `orders-service` and `parcels-service` |
| `src/Pacco.Services.Pricing.Api/Program.cs` | That the service exposes exactly one query and takes part in no command or event flow |
| `src/Pacco.Services.Pricing.Api/appsettings.json` | By omission: no `mongo`, no `redis`, no `rabbitMq`, no `vault` sections |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**. The discount tiers in
`CustomerDiscountsService.cs` are hard-coded in C# rather than driven by configuration or flags —
changing a discount requires a code change and a redeploy.

## 13. Open questions / ambiguities

1. Whether the discount tiers are business-approved values or placeholders.
2. Whether discounts should be configurable without a redeploy.
3. Whether this service verifies the caller, given `orders-service` passes an arbitrary
   `customerId` while the gateway forces the caller's own.
4. Why the inbound call from `orders-service` carries no client certificate, when the equivalent
   Availability → Customers call does.
5. Why this repository has no `LICENSE` file when nine sibling service repositories do.
6. Whether the two-hop dependency (Orders → Pricing → Customers) on the order-creation path is
   acceptable.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Pricing.Api/` and all its
subdirectories (`Core/`, `Core/Services/`, `DTO/`, `Queries/`, `Queries/Handlers/`, `Services/`,
`Services/Clients/`, `certs/`, `Properties/`, `.idea/`), and the repository root. There is no
`wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`, `resources/js/`, or `web/` directory; no
`package.json` or bundler configuration; and no view templates (`.cshtml`, `.html`, Razor). The
only browser-facing surface is the Convey Swagger UI at `/docs`, generated by
`Convey.WebApi.Swagger`.

---

## Evidence

| Fact | File |
|---|---|
| Single route, host composition, absence of broker wiring | `src/Pacco.Services.Pricing.Api/Program.cs` |
| Discount rules | `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` |
| Query and handler | `src/Pacco.Services.Pricing.Api/Queries/GetOrderPricing.cs`, `Queries/Handlers/GetOrderPricingHandler.cs` |
| Response shape | `src/Pacco.Services.Pricing.Api/DTO/OrderPricingDto.cs` |
| The one outbound dependency | `src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs` |
| Package set; absence of persistence, broker, outbox and Vault packages | `src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj` |
| Absence of `mongo`, `redis`, `rabbitMq` and `vault` sections; Consul port 5008; HTTP service map; JWT; logging; metrics; tracing | `src/Pacco.Services.Pricing.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Single-project layout | `Pacco.Services.Pricing.sln` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Absence of a `LICENSE` file | repository root |
| Inbound caller and its parameters | `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Services/Clients/PricingServiceClient.cs` |
| Gateway route binding `customerId` to the caller's token | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |
| Absence from the platform message catalogue | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Downstream dependency target | `../hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/Program.cs` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | This service is genuinely stateless and caches nothing between requests | There is no persistence package, no cache package, and no `mongo` or `redis` configuration section | If any in-memory caching exists, discounts could be served from stale customer data despite the live-read design | Read `CustomerDiscountsService.cs` and the query handler for any static or memoised state |
| A2 | The discount rules in `CustomerDiscountsService.cs` are the platform's only pricing policy | No other repository contains pricing logic, and `orders-service` obtains the final price from this service | If pricing rules also exist elsewhere, two systems could disagree about what a customer pays | Search the platform for any other price adjustment before an order is stored |
| A3 | The `customerId` query parameter identifies the customer whose discount applies, and the raw `orderPrice` is supplied by the caller | This matches how `orders-service` builds the call and how the gateway rewrites the public route | If the service recomputes or validates the price itself, the trust model differs from what is described here | Read `GetOrderPricingHandler.cs` for any independent price derivation |

### Blockers

*(none identified for this repository)*

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does this service verify who is asking, or does it price for whatever `customerId` it is given? | The gateway forces `customerId` to the caller's own id, but `orders-service` passes any id on the internal call. If the service itself does not check, anything that can reach it on the network can obtain another customer's discount — and the price is a real business value | Read the handler for a caller check and confirm against a running instance | Whoever owns Pacco authentication |
| Q2 | **[ACTION NOW]** Are the discount tiers in `CustomerDiscountsService.cs` real, approved values? | They are hard-coded in C# with no configuration behind them, so whatever is written there is what customers are charged. If they were placeholder numbers, nobody would notice from outside the code | Confirm the values with whoever owns pricing policy | Domain owner for Pricing |
| Q3 | **[handled later by HLD]** Should discounts be changeable without a code change? | Any pricing adjustment currently requires editing C#, rebuilding, and redeploying the service. That is a slow path for a value that commercial teams typically want to move | Move the tiers into configuration or a data source, or accept redeployment as the change mechanism | Domain owner for Pricing |
| Q4 | **[ACTION NOW]** Should the inbound call from `orders-service` be certificate-authenticated? | `customers-service` requires a Vault-issued client certificate from its one synchronous caller and holds a matching ACL. This service is called the same way with no such protection, on a path that determines what a customer pays | Either add the ACL and certificate as Availability/Customers do, or record why this call is treated differently | Whoever owns Pacco authentication |
| Q5 | **[handled later by HLD]** Is the two-hop synchronous chain on the order-creation path acceptable? | Creating an order calls this service, which calls `customers-service`. A `customers-service` outage stops order pricing and therefore order creation, two hops away from the failure | Confirm the intended availability behaviour, including whether a default price should apply when the customer lookup fails | Platform architect |
| Q6 | **[ACTION NOW]** Why does this repository have no `LICENSE` file? | Nine sibling service repositories carry one and this one does not, which leaves the terms for this code unstated | Add the platform licence, or state deliberately that it differs | Platform owner |
