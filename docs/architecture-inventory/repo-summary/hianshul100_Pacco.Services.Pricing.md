# Repository Summary — `hianshul100_Pacco.Services.Pricing`

**Primary name:** `pricing-service` (aliases used in this file: `Pacco.Services.Pricing.Api` — the .NET project and assembly name; `devmentors/pacco.services.pricing` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Pricing`, path: `src/Pacco.Services.Pricing.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Calculates the price of an order, applying a customer discount. It holds no data of its own: it takes an order's details, asks `customers-service` about the customer, applies a discount rule, and returns a figure. It is a pure calculation service.

It is the simplest deployable in the platform and the only one that is stateless in every sense — no database, no cache, no message broker.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`. Query-side only: `Program.cs` chains `AddConvey().AddWebApi().AddInfrastructure()` — it does **not** call `AddApplication()`, because there is no application layer.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Pricing.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Pricing.Api/Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Pricing.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

**One source project**, enumerated from `Pacco.Services.Pricing.sln`: `src/Pacco.Services.Pricing.Api`. It is the only domain service without the four-project `.Api` / `.Application` / `.Core` / `.Infrastructure` split. The domain lives in a nested folder, `src/Pacco.Services.Pricing.Api/Core/`, inside the API project rather than in its own assembly, and the single project carries the full infrastructure dependency set itself.

Types registered in `src/Pacco.Services.Pricing.Api/Infrastructure/Extensions.cs`:

- `ICustomersServiceClient` → `CustomersServiceClient` (`src/Pacco.Services.Pricing.Api/Services/Clients/`)
- `ICustomerDiscountsService` → `CustomerDiscountsService` (`src/Pacco.Services.Pricing.Api/Core/Services/`)

`AddInfrastructure()` chains only `.AddErrorHandler<ExceptionToResponseMapper>().AddQueryHandlers().AddInMemoryQueryDispatcher().AddHttpClient().AddConsul().AddFabio().AddMetrics().AddJaeger().AddWebApiSwaggerDocs().AddSecurity()`. `UseInfrastructure()` chains only `.UseErrorHandler().UseSwaggerDocs().UseJaeger().UseConvey().UseMetrics()`.

**A JetBrains Rider settings folder is committed** at `src/Pacco.Services.Pricing.Api/.idea/`. It is the only editor-configuration directory checked in anywhere in the workspace.

## 5. External integrations

- **`customers-service`** over HTTP through Fabio. `httpClient.services` in `src/Pacco.Services.Pricing.Api/appsettings.json` maps `customers` → `customers-service`. This is the service's only dependency.
- **Consul**, **Fabio**, **Jaeger**, **Prometheus**, **Seq** — through Convey extensions.
- **Vault** — key-value and PKI engines, configured and used. See dimension 10.
- **No RabbitMQ, no MongoDB, no Redis.** (Verification method: no `rabbitMq`, `mongo` or `redis` key in `src/Pacco.Services.Pricing.Api/appsettings.json`; no `AddRabbitMq`, `AddMongo` or `AddRedis` match under `src/`.)

## 6. Data stores & state

**None.** This is the only service in the platform with no persistence at all:

- No MongoDB — no `mongo` block in `appsettings.json`, no `.AddMongo()` call, no repositories, no collections.
- No Redis — no `redis` block, no `.AddRedis()` call.
- No outbox or inbox collections.
- **No ORM, no query mechanism against any store, no migration tool, no table or collection names.**
- **Cross-domain coupling:** it reads customer data from `customers-service` synchronously on every request but stores nothing. There are no foreign keys because there is no data. It is the only service that could be restarted, replaced or scaled with no state consequences whatsoever.

## 7. Messaging / async / events

**None.** `pricing-service` neither publishes nor consumes messages:

- No `rabbitMq` block in `appsettings.json`.
- No `.AddRabbitMq()` call in `Infrastructure/Extensions.cs`.
- No exchange, no queue template, no subscriptions, no outbox.
- **It is the only service absent from `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`**, which lists all eight message-carrying services. Its absence from that file is therefore correct and consistent with the code, not an omission.

There are no event or topic names and no payloads to record for this service.

## 8. APIs exposed & consumed

**Exposed** — `UseDispatcherEndpoints` in `src/Pacco.Services.Pricing.Api/Program.cs`. A single route:

| Method | Path | Dispatched to | Response |
|---|---|---|---|
| GET | `pricing` | `GetOrderPricing` | `OrderPricingDto` |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Consumed by:**

- `api-gateway` — exposes `GET pricing?customerId=@user_id` publicly, binding the customer identifier from the JWT.
- `orders-service` — over HTTP through Fabio.

**Consumes over HTTP:** `customers-service`, through Fabio with 3 retries and request masking (`maskTemplate: "*****"`).

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5008`.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.pricing`, present in `hianshul100_Pacco/compose/services.yml`.
- **This repository has no `LICENSE` file**, unlike the eight other service repositories and the gateway. `hianshul100_Pacco.Services.Vehicles` is the only other repository missing one.

## 10. Security & auth clues

- `.AddSecurity()` is called, but there is no `security` access-control list and no certificate authentication.
- **Vault is configured and used.** `src/Pacco.Services.Pricing.Api/appsettings.json` carries a `vault` block with `enabled: true`, `url: http://localhost:8200`, `authType: token`, `token: "secret"`, `kv.path: pricing-service/settings` (engine version 2, mount point `kv`) and `pki.roleName: pricing-service`, `pki.commonName: pricing-service.pacco.io`. `src/Pacco.Services.Pricing.Api/Program.cs:33` calls `.UseVault()`, with `using Convey.Secrets.Vault;` in the file header.
- **It is the only Vault user on the platform with no `lease` block.** The eight other Vault-enabled services declare `lease.mongo` for dynamic MongoDB credentials; this service has no database, so it leases nothing and uses Vault for key-value settings and PKI only. That is the one Vault difference here — not an absence.
- JWT validated against `certs/localhost.cer`, `validIssuer: pacco`, `validateAudience: false`, `validateIssuer: true`, `validateLifetime: true`.
- Customer scoping is applied by `api-gateway` through the `customerId:@user_id` binding, not by this service.
- **A gap worth flagging:** this service calls `customers-service` over HTTP, but `customers-service` grants `customers:read` only to `availability-service` in its access-control list. `pricing-service` is not on that list, and unlike `availability-service` it does not call `.AddCertificateAuthentication()`, so it presents no client certificate. See open questions.
- **Checked-in credentials:** in `src/Pacco.Services.Pricing.Api/appsettings.json` — `vault.token: "secret"`, `vault.username: user`, `vault.password: secret`, and Seq `apiKey: secret`. A public certificate, `src/Pacco.Services.Pricing.Api/certs/localhost.cer`, is committed for JWT validation; unlike `identity-service` this repository holds no private key material, and unlike `identity-service` and `operations-service` it does not carry the shared `jwt.issuerSigningKey` literal. It carries no RabbitMQ `guest`/`guest` pair, because it has no broker.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: pricing`, UDP `localhost:6831`, `sampler: const`. No RabbitMQ Jaeger plugin, because there is no broker — its traces are HTTP-only.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]` and the standard `excludeProperties` redaction list.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.
- **No handler logging** — `.AddHandlersLogging()` is not called here, unlike the messaging services.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Pricing.Api/Infrastructure/Extensions.cs` — a short file whose value is what it *omits*. It is the clearest statement in the workspace that the Convey stack is adopted piecemeal rather than wholesale.
- `src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` — the discount rule, which is the only business logic in the repository.
- `src/Pacco.Services.Pricing.Api/Program.cs` — the single-route surface and the decision to skip `AddApplication()`.
- `Pacco.Services.Pricing.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository. Given that this service computes discounts, it is the most likely candidate in the platform to have wanted one; it has none, so discount rules are changed only by redeploying.

## 13. Open questions & ambiguities

- **Does the call to `customers-service` succeed?** `customers-service` defines an access-control list granting `customers:read` to `availability-service` only, and this service presents no client certificate. Whether the list is enforced, and whether `pricing-service` calls are being rejected, is **Unknown**. **Needs validation** — this is the sharpest open item in the repository.
- The discount rule itself was not read; what tiers exist and how they map to customer state is **Unknown**.
- `GET pricing` takes no path parameters. Which query parameters it requires beyond `customerId`, and how it learns the order to price, is **Unknown** from the route table alone. **Needs validation.**
- Why this service alone was built as a single project rather than the four-layer structure is **Unknown**. It may be deliberate — the service has no domain worth isolating — or it may predate the convention.
- The committed `.idea/` folder suggests the repository has not been tidied; whether other editor artefacts are present elsewhere is **Unknown**.
- This repository and `hianshul100_Pacco.Services.Vehicles` are the only two without a `LICENSE` file. Whether that is deliberate is **Unknown**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Pricing.Api/`, `src/Pacco.Services.Pricing.Api/Core/`, `src/Pacco.Services.Pricing.Api/Infrastructure/`, `src/Pacco.Services.Pricing.Api/Services/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Pricing` directory". | No such directory. The runnable project is `src/Pacco.Services.Pricing.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5008`." | `appsettings.json` sets the Consul port to `5008`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.pricing`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Pricing.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README is the shared platform template: "Pacco.Services.Pricing is the microservice being part of Pacco solution", with the same wording as the nine other services. | This service shares only part of the platform's architecture. It has no database, no broker and no layering, and exposes one route — though it does register with Consul, Fabio, Jaeger, Prometheus and Seq and does use Vault, like the rest. | **Docs conflict.** The template implies a uniformity that holds for the cross-cutting stack but not for the domain architecture. |
| `hianshul100_Pacco/assets/pacco_overview.png` draws **Pricing Service with both a red "Command" and a green "Integration Event" dashed connector to RabbitMQ**, identically to the six other domain services. | This service has no `rabbitMq` block, no `.AddRabbitMq()` call, no exchange, no subscriptions and no outbox, and is the only service absent from `messages.json`. | **Docs conflict — the diagram overstates the code.** The platform's only architecture diagram shows a messaging integration this service does not have. |

**On disk but not mentioned in `README.md`:** the dependency on `customers-service`, the absence of persistence and messaging, the nested `Core/` folder, and the committed `.idea/` directory.

**Docs-only claims:** none beyond the `dotnet run` path.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory, and no `LICENSE`** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `pricing-service` genuinely holds no state and can be restarted or scaled freely. | No database, cache, broker or outbox is configured or registered anywhere in the repository. | Any statement about the platform's stateless components would be wrong. | Confirmed by reading `Infrastructure/Extensions.cs` and `appsettings.json` in full; no further check needed unless the service changes. |
| A2 | The `customers-service` access-control list is not blocking this service's calls today. | The platform is documented as working end to end, and `pricing` is exposed through the gateway as a live route. | Order pricing would be silently failing in every environment. | Call `GET pricing` against a running platform and check the `customers-service` logs. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is `pricing-service` authorised to call `customers-service`, given it is absent from that service's access-control list and presents no client certificate? | If the list is enforced, pricing is broken for every order. If it is not enforced, the platform's only service-to-service authorisation rule is decorative. | The list is probably only checked when a certificate is presented, so unauthenticated calls pass through. Either way the model is inconsistent. | Security owner |
| Q2 | **[handled later by the platform inventory review]** Should discount rules be configurable rather than compiled in? | Changing a discount currently requires a code change and a redeploy of a service that has no other reason to change. | Move the rule to configuration or introduce the platform's first feature-flag mechanism. | Product owner |
| Q3 | **[handled later by the platform inventory review]** Should the committed `src/Pacco.Services.Pricing.Api/.idea/` folder be removed and added to `.gitignore`? | It is an editor artefact that does not belong in a shared repository. | Yes. | Repository owner |
