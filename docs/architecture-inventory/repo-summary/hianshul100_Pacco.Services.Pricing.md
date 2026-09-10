# Repository summary — `hianshul100_Pacco.Services.Pricing`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Pricing` (also known as: Pacco.Services.Pricing, the Pricing service)
**Deployable:** `Pacco.Services.Pricing.Api` (also known as: `pricing-service` — its Consul service name, container name and compose service name — and `devmentors/pacco.services.pricing`, its published image). Repository: `hianshul100_Pacco.Services.Pricing`, path: `src/Pacco.Services.Pricing.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

It works out what an order should cost. Given a customer and a base price, it fetches that customer's record, applies a loyalty and VIP discount, and returns the base price, the discount rate and the discounted price. It owns one business rule and nothing else.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, built with Convey and the CQRS dispatcher. It is **stateless** — the smallest service in the platform, and the only one with no database, no cache and no message broker of any kind.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Pricing.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Pricing.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Pricing.Api.dll` |
| Composition | `src/Pacco.Services.Pricing.Api/Infrastructure/Extensions.cs` |
| The business rule | `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` |
| Request collection | `Pacco.Services.Pricing.rest` |

HTTP routes, registered through `UseDispatcherEndpoints`:

| Method | Route | Behaviour |
|---|---|---|
| GET | `` (root) | writes the configured `app.name` |
| GET | `pricing` | dispatches `GetOrderPricing` and returns `OrderPricingDto` |

## 4. Important modules / packages

One project in `Pacco.Services.Pricing.sln`: `src/Pacco.Services.Pricing.Api/Pacco.Services.Pricing.Api.csproj`. There is no Api/Application/Core/Infrastructure split — the folders `Core/`, `DTO/`, `Queries/`, `Services/`, `Exceptions/`, `Infrastructure/` all live inside the one project, the same shape as OrderMaker and Operations.

- `Queries/GetOrderPricing.cs` — `{ Guid CustomerId, decimal OrderPrice }`.
- `Queries/Handlers/GetOrderPricingHandler.cs` — fetch the customer, throw `CustomerNotFoundException` if absent, compute, log, return.
- `Core/Services/CustomerDiscountsService.cs` — the rule.
- `Core/Entities/Customer.cs` — `{ Guid Id, bool IsVip, int CompletedOrdersNumber }`.
- `DTO/CustomerDto.cs` — `{ Guid Id, bool IsVip, IEnumerable<Guid> CompletedOrders }`; `DTO/Extensions.cs` maps it to the entity by counting `CompletedOrders`.
- `DTO/OrderPricingDto.cs` — `{ decimal OrderPrice, decimal CustomerDiscount, decimal OrderDiscountPrice }`.
- `Services/Clients/CustomersServiceClient.cs` — the one outbound call.
- `Exceptions/{AppException,CustomerNotFoundException,ExceptionToResponseMapper}.cs`.

Packages: Convey `0.4.*` — `Convey`, `Convey.Secrets.Vault`, `Convey.CQRS.Queries`, `Convey.Discovery.Consul`, `Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`, `Convey.Security`, `Convey.Tracing.Jaeger`, `Convey.Tracing.Jaeger.RabbitMQ`, `Convey.WebApi.CQRS`, `Convey.WebApi.Swagger`. Plus `<Content Include="certs\**" CopyToPublishDirectory="Always" />`.

**No Mongo, no Redis, no RabbitMQ, no outbox, no `Convey.Auth`.**

Composition from `Infrastructure/Extensions.cs` → `AddInfrastructure()`: registers `ICustomersServiceClient → CustomersServiceClient` and `ICustomerDiscountsService → CustomerDiscountsService`, then `.AddErrorHandler<ExceptionToResponseMapper>().AddQueryHandlers().AddInMemoryQueryDispatcher().AddHttpClient().AddConsul().AddFabio().AddMetrics().AddJaeger().AddWebApiSwaggerDocs().AddSecurity()`. `UseInfrastructure()`: `.UseErrorHandler().UseSwaggerDocs().UseJaeger().UseConvey().UseMetrics()`.

**The business rule**, verbatim from `CustomerDiscountsService.CalculateDiscount`:

| Completed orders | Discount |
|---|---|
| 10 or more | 0.10 |
| 4 to 9 | 0.05 |
| 1 to 3 | 0.02 |
| 0 | 0.00 |

and if the customer is VIP, a further `0.10` is added. The maximum is therefore `0.20`. The handler then computes `orderDiscountPrice = orderPrice - discount * orderPrice`, and returns `orderPrice` unchanged if that result is not greater than zero.

## 5. External integrations

Consul (`pricing-service`, port 5008, ping endpoint `ping`), Fabio, Vault, Jaeger (service name `pricing`), Prometheus, Seq. One downstream service: `customers-service`, declared as `httpClient.services.customers`.

No broker, no database, no cache.

## 6. Data stores and state handling

**There is no data store.** No database, no cache, no file, no in-memory collection that outlives a request. **ORM:** none. **Migration tool:** none. **Tables/collections:** none.

Every request rebuilds its inputs from `customers-service`. The discount tiers are compiled into `CustomerDiscountsService`, not configured.

**Cross-domain coupling.** There is no foreign key and no replicated data. The coupling is a **runtime read dependency**: `GET {customers-service}/customers/{id}` on every priced order. `pricing-service` therefore cannot serve a request while `customers-service` is down, and `orders-service` in turn cannot assign a vehicle while `pricing-service` is down — a three-service synchronous chain in an otherwise event-driven platform.

There is one shape coupling worth naming: `CustomerDto.CompletedOrders` is deserialised as the full `IEnumerable<Guid>` from the Customers service purely so that `.Count()` can be taken. The list grows without bound over a customer's lifetime, and every pricing call transfers all of it.

## 7. Messaging, async and event mechanisms

**None.** This service publishes no message, subscribes to no message, declares no exchange and has no outbox. There is no `rabbitMq` block in any of its four configuration files.

`Convey.Tracing.Jaeger.RabbitMQ` is referenced in the csproj, but `Infrastructure/Extensions.cs` calls plain `.AddJaeger()` with no RabbitMQ plugin and there is no broker to plug into — the package is an unused leftover.

Event/topic names and payload key fields: **not applicable — this service does not participate in messaging.**

## 8. APIs exposed and consumed

**Exposed:** the two routes in section 3, plus Swagger at route prefix `docs` and `ping` for Consul. Local base URL `http://localhost:5008`; port 80 in the container.

`GET /pricing?customerId={guid}&orderPrice={decimal}` returns:

```
{ "orderPrice": <decimal>, "customerDiscount": <decimal>, "orderDiscountPrice": <decimal> }
```

Errors map through `ExceptionToResponseMapper`: an `AppException` becomes `{code, reason}` with 400, everything else becomes `{code: "error", reason: "There was an error."}` with 400. The one declared code is `customer_not_found`.

**Through the gateway:** all four Ntrada profiles expose `pricing`, `GET /` → `downstream: pricing-service/pricing?customerId=@user_id`, with `auth: true`. The gateway substitutes the caller's own identifier, so a signed-in user can only price for themselves.

**Consumed:** one call, through Fabio — `CustomersServiceClient.GetAsync(id)` → `GET {customers-service}/customers/{id}`.

**Called by:** `orders-service`. `Pacco.Services.Orders.Infrastructure/Services/Clients/PricingServiceClient.cs` calls `GET {pricing-service}/pricing?customerId={customerId}&orderPrice={orderPrice}`, and `AssignVehicleToOrderHandler` passes the assigned vehicle's `PricePerService` as `orderPrice`, then stores `pricing.OrderDiscountPrice` as the order total. That is the only production caller.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Pricing.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `pricing-service` on host port `5008:80` in `hianshul100_Pacco/compose/services.yml`; also listed in `services.yml` and `prod-services.yml`.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.pricing`.
- Being stateless, this is the one service that can be scaled horizontally with no further work.

## 10. Security and auth clues

- `Convey.Security` is composed and `certs/localhost.cer` ships with the image.
- Vault is configured in `appsettings.json` (kv v2 at `pricing-service/settings`, PKI role `pricing-service`, common name `pricing-service.pacco.io`) and `Program.cs` calls `.UseVault()`. There is **no dynamic database lease**, consistent with having no database. `appsettings.docker.json` disables Vault.
- **The service performs no authentication or authorisation of its own.** `appsettings.json` has a `jwt` block (`certificate.location: certs/localhost.cer`, `validIssuer: pacco`, `validateIssuer: true`), but `Convey.Auth` is not referenced in the csproj and `AddJwt()` is never called. The block is inert configuration. Protection comes entirely from the gateway's `auth: true` and from the service not being published outside the Docker network.
- Anything that can reach `pricing-service` directly can price for any `customerId`, which discloses whether a customer is VIP and roughly how many orders they have completed (the discount rate reveals the tier).
- `logger.excludeProperties` masks secret-like values; `httpClient.requestMasking` is enabled with `maskTemplate: "*****"`.

## 11. Observability, logging and tracing clues

Jaeger enabled, service name `pricing`, constant sampler, `excludePaths` `/`, `/ping`, `/metrics`. Prometheus metrics via `Convey.Metrics.AppMetrics`. Structured logs to console, file and Seq; ELK configured but disabled.

`GetOrderPricingHandler` logs every calculation at information level, including `CustomerId`, base price, discount and final price. Two notes on that line: it labels the discount `$` when the value is a rate, not an amount; and it writes a customer identifier alongside a derived loyalty signal into Seq on every request.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` is the only written statement of the discount policy anywhere in the platform — the tiers exist nowhere else, in no document and no configuration. `src/Pacco.Services.Pricing.Api/Queries/Handlers/GetOrderPricingHandler.cs` holds the composition of that rule with the base price and the floor at zero. `src/Pacco.Services.Pricing.Api/Infrastructure/Extensions.cs` records the decision to keep the service stateless.

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the Convey `enabled` booleans on Consul, Fabio, Jaeger, metrics, Swagger, Vault and the log sinks.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Pricing.Api/` and its `Core/`, `DTO/`, `Queries/`, `Services/`, `Exceptions/`, `Infrastructure/`, `Properties/`, `certs/` subdirectories, and `scripts/`. No `package.json`, no `wwwroot`, no template, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5008`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.pricing`; `Pacco.Services.Pricing.rest` lists the HTTP requests — it lists exactly one, the `GET /pricing` call.

**Present on disk but absent from the README:** the discount policy itself. The README is the shared template and never states the tiers, the VIP bonus, the dependency on `customers-service`, or that this service holds no data. The rule that decides what every Pacco customer pays is documented nowhere outside one C# file.

**Conflicts to surface:**

- **Stale doc.** The README says to run `dotnet run` in `/src/Pacco.Services.Pricing`. That directory does not exist; the project is `src/Pacco.Services.Pricing.Api`, as `scripts/start.sh` correctly shows.
- **Configuration conflict.** `appsettings.json` carries a `jwt` block, but no JWT handling is composed — `Convey.Auth` is not referenced and `AddJwt()` is not called. A reader would reasonably conclude the endpoint validates tokens; it does not.
- **Package conflict.** `Convey.Tracing.Jaeger.RabbitMQ` is referenced while the service has no broker.
- **Code defect — needs validation.** `Core/Entities/Customer.cs` takes `id` as its first constructor parameter and never assigns `Id`, so `Customer.Id` is always the empty GUID. Nothing downstream reads it today, so the effect is latent rather than live.
- **Gateway behaviour — needs validation.** The gateway route is written as `downstream: pricing-service/pricing?customerId=@user_id`, naming only one of the two query parameters the endpoint needs. Whether Ntrada also forwards the caller's `orderPrice` is not determinable from these files; if it does not, a gateway-originated call prices against `orderPrice = 0`. This does not affect the `orders-service` path, which calls the service directly.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which runs `dotnet test`, but this repository contains **no test project** — `Pacco.Services.Pricing.sln` lists only `Pacco.Services.Pricing.Api.csproj`. The platform's only pricing rule has no automated test.
- **Repository hygiene.** `src/Pacco.Services.Pricing.Api/.idea/` is committed, including `workspace.xml`. This is a developer's local IDE state, not source.
- **Unknown.** This repository has **no LICENSE file**, unlike eight of the ten service repositories. `hianshul100_Pacco.Services.Vehicles` is the other one missing it. The licence that applies to this code is not determinable from the repository.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next. The untested discount rule (B1) is the item that matters most: it decides revenue and nothing verifies it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | `CustomerDiscountsService` is the complete and only pricing policy. | It is the sole calculation in the repository, and the handler applies nothing else. |
| A2 | `orders-service` is the only production caller. | It is the one service in the workspace with a `pricing` entry in `httpClient.services`; the gateway route exists but serves a read-only preview. |
| A3 | The service is meant to be stateless by design rather than by omission. | The Vault configuration deliberately omits the dynamic database lease every other stateful service has. |
| A4 | `orderPrice` is a vehicle's `PricePerService`, not an order total. | `AssignVehicleToOrderHandler` passes `vehicle.PricePerService` and stores the result as the order total. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** The discount rule has no automated test, while CI runs `scripts/test.sh` against a solution with no test project. | A change to the tier boundaries or the VIP bonus would reach production unverified, and it decides what every customer is charged. | Pricing service maintainer. |
| B2 | **[ACTION NOW]** The pricing endpoint performs no authentication of its own, and the `jwt` configuration block implies it does. | Anything on the internal network can price for any customer and infer their VIP status and order-count tier from the discount. | Pricing service maintainer, with the security-architecture owner. |
| B3 | **[ACTION NOW]** `Customer.Id` is never assigned in the constructor. | The field is silently always empty; any future rule that reads it will be wrong in a way that is hard to spot. | Pricing service maintainer. |
| B4 | **[ACTION NOW]** No LICENSE file. | The terms under which this code may be used are unstated, unlike the rest of the platform. | Repository owner. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Does the gateway forward `orderPrice`, or does a gateway-originated pricing call always see zero? | If it does not forward it, the customer-facing pricing preview returns a meaningless number. | Gateway maintainer with the Pricing service maintainer. |
| Q2 | **[ACTION NOW]** Should the discount tiers be configurable rather than compiled in? | Changing a rate today needs a code change, a build and a redeploy of a service that has no tests. | Pricing service maintainer with the business owner of pricing. |
| Q3 | **[ACTION NOW]** Should the Customers service expose a completed-order **count** instead of the full list? | Every pricing call transfers a list that grows for the lifetime of the customer, only to take its length. | Customers service maintainer with the Pricing service maintainer. |
| Q4 | **[handled later by the integration-architecture stage]** Is a synchronous three-service chain acceptable for assigning a vehicle to an order? | Orders depends on Pricing which depends on Customers; either being down stops vehicle assignment. | Integration-architecture stage. |
| Q5 | **[handled later by the domain-model stage]** What should happen when the calculated price would be zero or negative? | The handler quietly falls back to the undiscounted price, which hides the condition rather than reporting it. | Domain-model stage. |
| Q6 | **[handled later by the observability stage]** Should the per-request pricing log line be kept as it is? | It writes a customer identifier together with a derived loyalty signal to Seq on every call, and labels a rate as a currency amount. | Observability stage. |
