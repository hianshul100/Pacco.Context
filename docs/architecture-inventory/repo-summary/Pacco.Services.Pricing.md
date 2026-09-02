# Repository summary — `Pacco.Services.Pricing`

**Repository:** `Pacco.Services.Pricing` (workspace clone path: `hianshul100_Pacco.Services.Pricing`)
**Deployable:** `pricing-service` (also known as: Pricing Service, `Pacco.Services.Pricing.Api`, image `devmentors/pacco.services.pricing`). **Repository: `Pacco.Services.Pricing`, path: `src/Pacco.Services.Pricing.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Pricing
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

Calculates the **discounted price of an order** for a given customer. It is a **pure calculation service**: it holds no data of its own, publishes nothing, subscribes to nothing, and persists nothing. Given a customer id and an order price it fetches the customer's order history and VIP status from `customers-service`, applies a discount table, and returns the result.

**Evidence:** `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs`, `src/Pacco.Services.Pricing.Api/Program.cs`, `appsettings.json`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice — **and nothing else**. It is the platform's **only service with no messaging, no database, and no cache**. It is also a **single-project service**: everything lives in `src/Pacco.Services.Pricing.Api`, with `Core/` and other folders nested inside that one project rather than split into `.Application` / `.Core` / `.Infrastructure`. `Program.cs` does not call `.AddApplication()`, because there is no separate application layer to add.

This is the platform's clearest instance of the root `Pacco` README's stated principle — clean architecture "or another style that is the best fit". A stateless calculator with one endpoint does not need four projects, and it does not have them.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Pricing.Api/Program.cs` — `AddConvey().AddWebApi().AddInfrastructure()`, then `UseInfrastructure()` + `UseDispatcherEndpoints(...)` |
| The single route | `Program.cs` — `GET pricing` → `GetOrderPricing` query |
| Discount rules | `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Pricing.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Pricing.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Pricing.Api` | The entire service: `Program.cs`, `Queries/GetOrderPricing.cs` + handler, `Core/Services/CustomerDiscountsService.cs`, `Services/Clients/CustomersServiceClient.cs`, `DTO/`, `Exceptions/`, `appsettings.json` |

**No test projects exist in this repository** — notable for the only service whose entire purpose is a calculation.

The Convey package set is correspondingly **smaller than every other service's**: no `Convey.Persistence.MongoDB`, no `Convey.Persistence.Redis`, no `Convey.MessageBrokers.*`, no outbox packages.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| `customers-service` | outbound HTTP | `GET {customers-service}/customers/{customerId}` → `CustomerDto` (`Services/Clients/CustomersServiceClient.cs`; `httpClient.services.customers: customers-service`) |
| Consul | out | registers `pricing-service` on port `5008` |
| Fabio | out | `http://localhost:9999`, `httpClient.type: "fabio"` |
| Vault | out | KV v2 `kv/pricing-service/settings`; PKI role `pricing-service`, CN `pricing-service.pacco.io` — **no `lease.mongo` block**, since there is no database to get credentials for |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

**Absent, uniquely:** no `mongo` block, no `rabbitMq` block, no `redis` block, no `outbox` block in `appsettings.json`. Every other service has at least three of the four.

**Single point of dependency:** the one outbound call to `customers-service` is unauthenticated (no client certificate, unlike `availability-service`'s call to the same service) and has no cache or fallback. If `customers-service` is unavailable, pricing cannot be calculated, which in turn blocks order creation in `orders-service`.

## 6. Data stores / state handling

- **No data store of any kind.** No MongoDB, no Redis, no file persistence, no in-memory cache.
- **ORM: none. Query mechanism: none. Migration tool: none. Table or collection names: none.** These are not gaps to be filled — the service is stateless by design.
- **Cross-domain coupling:** it reads customer data (completed order count, VIP status) from `customers-service` at request time and keeps no copy. This makes it the **least coupled service at the data layer** and, at the same time, the one with the hardest runtime dependency: every request requires a live call to another service.

## 7. Messaging / async / event mechanisms

**None.** `pricing-service` publishes no messages and subscribes to none. It has no `rabbitMq` configuration block and no `Convey.MessageBrokers.*` packages.

Confirming this from the other direction: **`pricing-service` is the only one of the ten services absent from `messages.json`** (`Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`), the platform's authoritative message catalogue. It has no exchange, no commands, no events, and no rejection events.

Consequences worth stating:
- Pricing is **invisible to `operations-service`**. A caller tracking an asynchronous order sees operations for every step except the pricing call, which happens synchronously inside `orders-service`'s handler.
- There is no way to react to a price change asynchronously; pricing is always pull, never push.
- Because there is no `Saga` header path through this service, a pricing failure during saga-driven order creation surfaces only as a failure of the `create_order` command.

Per the task's grounding rules: **no event or topic names exist for this service — this is a confirmed absence, not "unknown — requires runtime capture".**

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, `UseDispatcherEndpoints`; base URL `http://localhost:5008`, container port `80`):

| Method | Path | Maps to | Gateway exposure |
|---|---|---|---|
| GET | `pricing` | `GetOrderPricing` query — query-string parameters `customerId` and `orderPrice` | `/pricing` → `pricing-service/pricing?customerId=@user_id` |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**One endpoint — the smallest public surface in the platform.**

**Consumed:** `GET {customers-service}/customers/{customerId}` → `CustomerDto`.

**Called by:** `orders-service` → `GET {pricing-service}/pricing?customerId={customerId}&orderPrice={orderPrice}` → `OrderPricingDto` (`Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Services/PricingServiceClient.cs`).

**Note on the gateway route:** the gateway binds `customerId: @user_id`, so a customer can only price for themselves. But `orders-service` calls the service **directly**, passing whatever `customerId` it holds — so the gateway's protection does not apply to the internal path, and the service itself performs no check.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Pricing.Api.dll`.
- Composed as `pricing-service` on `5008:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5008`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- **This service is trivially horizontally scalable** — no state, no broker subscription, no saga. It is the only service in the platform for which that is unambiguously true.
- **`LICENSE` is missing** from this repository. Eleven of the thirteen repositories carry one; `Pacco.Services.Pricing` and `Pacco.Services.Vehicles` do not.
- **A committed JetBrains Rider `.idea/` directory** exists at `src/Pacco.Services.Pricing.Api/.idea/`. It is IDE-local state that does not belong in version control and appears in no other repository in the workspace.

## 10. Security/auth clues

- **JWT bearer** validation via `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- `.AddSecurity()` is registered; there is **no `security.certificate` block**, so the outbound call to `customers-service` presents no client certificate — unlike `availability-service`'s call to the same endpoint family, which does. The platform is inconsistent here: `customers-service` maintains an ACL granting `availability-service` `customers:read`, and `pricing-service` reads customer data with no equivalent grant.
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- Log redaction via `logger.excludeProperties`.
- **No authorisation inside the service.** The `customerId` is taken from the query string and used as given. Anyone who can reach port `5008` can price for any customer, which reveals that customer's discount tier and therefore their approximate order history and VIP status.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: pricing-service`, UDP `localhost:6831`, `sampler: const`). No RabbitMQ Jaeger plugin, because there is no broker.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`.
- **Handler logging:** `.AddHandlersLogging()` is registered for the query handler.
- **Correlation:** `Correlation-Context` header read from the incoming request.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics — no counter or histogram over discounts applied, which for a pricing service is the one business metric worth having.
- **Blind spot:** because this service emits no messages, it is absent from `operations-service`'s view entirely. Jaeger is the only place a pricing call appears.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` | **The entire pricing policy, hard-coded.** `CalculateDiscount` applies: `>= 10` completed orders → `0.1`; `> 3 && < 10` → `0.05`; `<= 3 && > 0` → `0.02`; then `+ 0.1` if the customer `IsVip`. So the maximum discount is 20% and a VIP with no completed orders still receives 10%. Every threshold and rate is a literal in source with no configuration binding, no feature flag, and no tests. |
| `src/Pacco.Services.Pricing.Api/Program.cs` | Single-project structure; no `.AddApplication()`; one endpoint |
| `src/Pacco.Services.Pricing.Api/Services/Clients/CustomersServiceClient.cs` | Customer data is fetched synchronously per request rather than replicated by event — the opposite choice from `orders-service` and `parcels-service` |
| `src/Pacco.Services.Pricing.Api/appsettings.json` | No Mongo, no RabbitMQ, no Redis, no outbox — the decision to be a pure calculator |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`).

This matters more here than anywhere else in the platform. Discount rates are exactly the kind of value a business changes for a promotion, a season, or a segment — and changing any of them requires a code edit, a build, an image push, and a redeploy of the service. There is no runtime override, no configuration binding, and no A/B capability. **Together with the VIP threshold of 20 completed orders in `Pacco.Services.Customers`, this is the platform's commercial policy, and all of it is compiled in.**

## 13. Open questions / ambiguities

- **The discount table is hard-coded and untested.** Whether the business expects to tune rates without a deployment is **Unknown**.
- **The VIP bonus stacks additively** with the volume discount, so a VIP with ten or more completed orders receives 20%. Whether that ceiling is intended is **Needs validation**.
- **Boundary at exactly 3 orders:** the rules are `<= 3 && > 0` → `0.02` and `> 3 && < 10` → `0.05`, so exactly 3 orders yields 2% and exactly 4 yields 5%. A customer with **0** completed orders receives no volume discount. These boundaries look deliberate but are unverified and untested. **Needs validation.**
- **No tests at all** for the platform's only pricing logic.
- **Hard dependency on `customers-service`** with no cache, no fallback, and no circuit breaker beyond Convey's two HTTP retries. If Customers is down, pricing fails, and order creation fails with it. **Needs validation** of the actual failure behaviour.
- **No authorisation on `customerId`** — see §10.
- **Absent from `messages.json`**, so pricing is invisible to `operations-service`. Whether that is intentional is **Unknown**.
- **The committed `.idea/` directory** and the **missing `LICENSE`** are repository-hygiene issues with no stated explanation.
- Whether `orderPrice` is validated (for negative or absurd values) before the discount is applied is **Needs validation**.

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (one C# project only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Pricing service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5008`. — **Confirmed** (`appsettings.json` `consul.port: 5008`, `Pacco/compose/services.yml` `5008:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Pricing` directory"**; the actual host project is **`/src/Pacco.Services.Pricing.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **The discount policy itself** — the thresholds, the rates, the VIP bonus, and the 20% ceiling. This is the service's entire reason to exist and the platform's commercial policy, and it is documented nowhere in any of the thirteen repositories.
- **The dependency on `customers-service`**, and what happens to pricing — and therefore to order creation — when it is unavailable.
- **The deliberate absence of a database, a cache, and messaging**, and the single-project structure that follows from it. A reader comparing this repository to its siblings would reasonably assume something was missing rather than deliberately omitted.
- That the service is absent from `messages.json` and therefore invisible to `operations-service`.
- `scripts/` (`build.sh`, `test.sh`, `dockerize.sh`).

**Also present on disk and worth noting:**
- `src/Pacco.Services.Pricing.Api/.idea/` — a committed JetBrains Rider settings directory, present in no other repository.
- **No `LICENSE` file**, unlike eleven of the thirteen repositories.

**Unknown (neither pass yielded proof):**
- Whether the discount table is considered stable business policy or something expected to change often.
- Whether the missing licence is deliberate.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The absence of a database, a cache, and any messaging is a deliberate design choice, not an unfinished service. | The service is a pure function of its inputs, its package set was trimmed to match, and the root `Pacco` README explicitly allows a simpler style where it fits. | Someone would "complete" the service by adding persistence and events it does not need, and the platform would gain a store with no owner. | Confirm with the platform owner that pricing is intended to stay stateless. |
| A2 | The discount table in `CustomerDiscountsService.CalculateDiscount` is the platform's complete and current pricing policy. | It is the only discount computation in the workspace, and `orders-service` calls this service for every order price. | Orders would be priced by a rule the business does not recognise, and nothing would surface the discrepancy. | Have the product owner read and confirm the four rules and the VIP bonus. |
| A3 | A VIP customer's 10% bonus is meant to stack on top of the volume discount, giving a 20% maximum. | The code adds `0.1` to whichever volume tier applies, rather than taking the larger of the two. | The platform could be discounting twice as much as intended on its most valuable customers' orders. | Product owner confirms the intended maximum discount. |

### Blockers

_None identified for this repository._

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should discount rates and thresholds be configurable rather than compiled into the service? | Every rate change — a promotion, a seasonal offer, a segment adjustment — currently needs a code change, a build, an image push, and a redeploy. | Move them to configuration if the business expects to tune them; otherwise document them as fixed policy. | Product owner |
| Q2 | **[ACTION NOW]** Should the platform's only pricing logic have tests? It has none, and the tier boundaries at exactly 3 and exactly 10 completed orders are easy to get wrong. | A silent error here mis-prices every order, and nothing in the platform would detect it. | Yes — the rules are pure and trivially testable. | Service owner |
| Q3 | **[ACTION NOW]** What should happen when `customers-service` is unavailable? Pricing has no cache and no fallback, and `orders-service` depends on pricing to create an order. | A single service being down would stop all order creation, and no fallback behaviour is defined anywhere. | Undefined in the repositories. | Service owner |
| Q4 | **[handled later by architecture_evolution_generation]** Should a caller be able to price for a customer other than themselves? The service reads `customerId` from the query string with no check. | It exposes the target customer's discount tier, and therefore their approximate order history and VIP status, to anyone who can reach the service. | The gateway binds the customer id to the token, but `orders-service` calls the service directly, so the service itself is unprotected. | Security owner |
| Q5 | **[handled later by architecture_evolution_generation]** Should pricing emit events so `operations-service` can see it? It is the only service absent from `messages.json`. | Callers tracking an asynchronous order see every step except pricing, so a slow or failing price lookup is invisible to them. | Probably acceptable, since pricing is synchronous by design — but the gap is not recorded anywhere. | Architecture team |
