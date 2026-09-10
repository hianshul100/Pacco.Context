# Pacco platform — repository inventory

**Project:** Common Architecture
**Scope:** every repository cloned into this workspace except the artifact repository that holds this document.
**Source of truth:** the files on disk in each repository. Where a README, a diagram or any other document disagrees with the code, the code is recorded as the truth and the disagreement is stated rather than reconciled away.
**Per-repository detail:** `docs/architecture-inventory/repo-summary/<repo-name>.md`, one file per repository listed below.

---

## Repositories in scope

| Repository | Role | Deployables | Detail file |
|---|---|---|---|
| `hianshul100_Pacco` | Platform orchestration: the umbrella solution, Docker Compose stacks, process lists, scripts, assets | none of its own | [hianshul100_Pacco.md](repo-summary/hianshul100_Pacco.md) |
| `hianshul100_Pacco.APIGateway` | Edge — Ntrada-configured API gateway | `Pacco.APIGateway` (`api-gateway`) | [hianshul100_Pacco.APIGateway.md](repo-summary/hianshul100_Pacco.APIGateway.md) |
| `hianshul100_Pacco.Services.Availability` | Resource availability and reservations | `Pacco.Services.Availability.Api` (`availability-service`) | [hianshul100_Pacco.Services.Availability.md](repo-summary/hianshul100_Pacco.Services.Availability.md) |
| `hianshul100_Pacco.Services.Customers` | Customer profiles, VIP status | `Pacco.Services.Customers.Api` (`customers-service`) | [hianshul100_Pacco.Services.Customers.md](repo-summary/hianshul100_Pacco.Services.Customers.md) |
| `hianshul100_Pacco.Services.Deliveries` | Delivery records and registrations | `Pacco.Services.Deliveries.Api` (`deliveries-service`) | [hianshul100_Pacco.Services.Deliveries.md](repo-summary/hianshul100_Pacco.Services.Deliveries.md) |
| `hianshul100_Pacco.Services.Identity` | Sign-up, sign-in, JWT issue and refresh | `Pacco.Services.Identity.Api` (`identity-service`) | [hianshul100_Pacco.Services.Identity.md](repo-summary/hianshul100_Pacco.Services.Identity.md) |
| `hianshul100_Pacco.Services.Operations` | Cross-cutting operation tracking, SignalR and gRPC push | `Pacco.Services.Operations.Api` (`operations-service`) | [hianshul100_Pacco.Services.Operations.md](repo-summary/hianshul100_Pacco.Services.Operations.md) |
| `hianshul100_Pacco.Services.OrderMaker` | Saga orchestration of the end-to-end ordering flow | `Pacco.Services.OrderMaker` (`ordermaker-service`) | [hianshul100_Pacco.Services.OrderMaker.md](repo-summary/hianshul100_Pacco.Services.OrderMaker.md) |
| `hianshul100_Pacco.Services.Orders` | Orders, parcels on orders, vehicle assignment, totals | `Pacco.Services.Orders.Api` (`orders-service`) | [hianshul100_Pacco.Services.Orders.md](repo-summary/hianshul100_Pacco.Services.Orders.md) |
| `hianshul100_Pacco.Services.Parcels` | Parcel catalogue | `Pacco.Services.Parcels.Api` (`parcels-service`) | [hianshul100_Pacco.Services.Parcels.md](repo-summary/hianshul100_Pacco.Services.Parcels.md) |
| `hianshul100_Pacco.Services.Pricing` | Discount and order price calculation | `Pacco.Services.Pricing.Api` (`pricing-service`) | [hianshul100_Pacco.Services.Pricing.md](repo-summary/hianshul100_Pacco.Services.Pricing.md) |
| `hianshul100_Pacco.Services.Vehicles` | Vehicle catalogue | `Pacco.Services.Vehicles.Api` (`vehicles-service`) | [hianshul100_Pacco.Services.Vehicles.md](repo-summary/hianshul100_Pacco.Services.Vehicles.md) |
| `hianshul100_Pacco.Web` | **Unverifiable — Missing Source Evidence.** Contains one 11-byte README and nothing else | none found | [hianshul100_Pacco.Web.md](repo-summary/hianshul100_Pacco.Web.md) |

`hianshul100_Pacco.Context` is the repository this document is written into and is deliberately excluded from the inventory.

Aliases used throughout: each service is named by its deployable project name on first mention, followed by its Consul/container name in brackets. The two are used interchangeably below and both are copied character-for-character from the source.

---

## Consolidated inventory by dimension

### 1. Primary purpose

| Repository | Primary purpose |
|---|---|
| `hianshul100_Pacco` | Ties the platform together: `Pacco.sln` spanning all sibling repositories, Compose stacks for infrastructure and services, process lists, build and dockerize scripts, logo assets |
| `hianshul100_Pacco.APIGateway` | Single external entry point; translates HTTP into either a proxied call or a published command |
| `…Services.Availability` | Tracks bookable resources and their reservations; the platform's scarcity model |
| `…Services.Customers` | Customer profile, address, VIP status, completed-order tally |
| `…Services.Deliveries` | Delivery lifecycle and the notes registered against it |
| `…Services.Identity` | Accounts, passwords, roles, access and refresh tokens |
| `…Services.Operations` | Answers "did my asynchronous command succeed?" and pushes the answer to the user |
| `…Services.OrderMaker` | Runs the whole ordering process as one saga |
| `…Services.Orders` | The order aggregate: parcels, vehicle, status, total price |
| `…Services.Parcels` | Parcel catalogue and its link to an order |
| `…Services.Pricing` | One rule: how much discount a customer gets |
| `…Services.Vehicles` | Vehicle catalogue with capacity and price per service |
| `…Web` | **Unverifiable — Missing Source Evidence** |

### 2. Main runtime / service type

Every deployable is an ASP.NET Core application on `netcoreapp3.1`, built on **Convey `0.4.*`**. Beyond that they differ:

| Repository | Runtime shape |
|---|---|
| `hianshul100_Pacco` | Not a runtime — Docker Compose and shell scripts only |
| `…APIGateway` | **Ntrada `0.4.*`** — the whole service is 20 lines of `Program.cs` plus YAML |
| `…Availability`, `…Customers`, `…Deliveries`, `…Identity`, `…Orders`, `…Parcels`, `…Vehicles` | Four-project Clean Architecture: `.Api` / `.Application` / `.Core` / `.Infrastructure` |
| `…Operations` | Single project; HTTP + **SignalR** + **gRPC** + static files at once |
| `…OrderMaker` | Single project; Convey plus **Chronicle_ 3.2.1** saga coordination |
| `…Pricing` | Single project; stateless, no broker, no store |
| `…Web` | **Unverifiable — Missing Source Evidence** |

### 3. Key entrypoints

Every service follows the same three: `src/<project>/Program.cs`, `scripts/start.sh` (sets `ASPNETCORE_ENVIRONMENT=local` and runs `dotnet run` in the project directory), and a `Dockerfile` whose `ENTRYPOINT` runs the published DLL. Each also ships a `.rest` request collection at the repository root.

The service-specific entry points that matter architecturally:

| Repository | Entrypoint worth knowing |
|---|---|
| `…APIGateway` | `src/Pacco.APIGateway/ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` — the four routing profiles, selected by the `NTRADA_CONFIG` environment variable |
| `…Operations` | `src/Pacco.Services.Operations.Api/messages.json` — the platform message catalogue that drives subscriptions; `Infrastructure/Subscriptions.cs` |
| `…OrderMaker` | `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` — the only written form of the end-to-end process |
| all layered services | `src/<service>.Infrastructure/Extensions.cs` — composition and `SubscribeCommand<T>()` / `SubscribeEvent<T>()` list |
| `hianshul100_Pacco` | `compose/infrastructure.yml`, `compose/services.yml`, `services.yml`, `prod-services.yml` |

### 4. Important modules / packages

| Concern | Package |
|---|---|
| Framework and CQRS | `Convey`, `Convey.CQRS.Commands/Queries/Events`, `Convey.WebApi`, `Convey.WebApi.CQRS`, `Convey.WebApi.Swagger` |
| Messaging | `Convey.MessageBrokers`, `.CQRS`, `.RabbitMQ`, `.Outbox.Mongo` |
| Persistence | `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis` |
| Discovery and routing | `Convey.Discovery.Consul`, `Convey.LoadBalancing.Fabio`, `Convey.HTTP` |
| Security | `Convey.Auth`, `Convey.Security`, `Convey.Secrets.Vault` |
| Observability | `Convey.Tracing.Jaeger`, `.Jaeger.RabbitMQ`, `Convey.Metrics.AppMetrics`, `Convey.Logging` |
| Gateway | `Ntrada`, `Ntrada.Extensions.Cors/CustomErrors/Jwt/RabbitMq/Swagger/Tracing` |
| Saga | `Chronicle_ 3.2.1` (OrderMaker only) |
| Real-time and RPC | `Microsoft.AspNetCore.SignalR 1.1.0`, `.SignalR.Redis 1.1.5`, `Grpc.AspNetCore 2.28.0`, `Grpc.Tools 2.28.1`, `Google.Protobuf 3.11.4` (Operations only) |
| Testing | `xunit 2.4.1`, `Shouldly 3.0.2`, `NSubstitute 4.2.1`, `Microsoft.AspNetCore.Mvc.Testing 3.1.3`, `coverlet.collector 1.2.1`, `Pactify 1.1.0`, `NBomber 0.16.0` |

### 5. External integrations

| Repository | RabbitMQ | MongoDB | Redis | Consul | Fabio | Vault | Jaeger | Other |
|---|---|---|---|---|---|---|---|---|
| `…APIGateway` | yes (publishes) | no | no | yes | yes | no | yes | — |
| `…Availability` | yes | yes | no | yes | yes | yes | yes | — |
| `…Customers` | yes | yes | no | yes | yes | yes | yes | — |
| `…Deliveries` | yes | yes | no | yes | yes | yes | yes | — |
| `…Identity` | yes | yes | no | yes | yes | yes | yes | — |
| `…Operations` | yes (consumes only) | configured, **unused** | yes | yes | yes | yes | yes | SignalR, gRPC |
| `…OrderMaker` | yes | **no** | yes (registered, no call found) | yes | yes | **no** | **no** | Chronicle |
| `…Orders` | yes | yes | no | yes | yes | yes | yes | — |
| `…Parcels` | yes | yes | no | yes | yes | yes | yes | — |
| `…Pricing` | **no** | **no** | **no** | yes | yes | yes | yes | — |
| `…Vehicles` | yes | yes | no | yes | yes | yes | yes | — |

Shared infrastructure comes up from `hianshul100_Pacco/compose/infrastructure.yml`: MongoDB, Redis, RabbitMQ, Consul, Fabio, Vault, Jaeger, Prometheus, Grafana, InfluxDB, Seq.

### 6. Data stores and state handling

**Across the whole platform there is no ORM and no migration tool.** Persistence is MongoDB accessed through Convey's `IMongoRepository<TDocument, TIdentifiable>`; collections are created implicitly on first write. Nothing versions a schema.

| Repository | Store | Collections | Document key fields |
|---|---|---|---|
| `…Availability` | Mongo `availability-service` | `resources` | `ResourceDocument { Id, Version, Tags, Reservations }`, embedded `ReservationDocument { TimeStamp, Priority }` |
| `…Customers` | Mongo `customers-service` | `customers` | `{ Id, Email, FullName, Address, IsVip, State, CreatedAt, CompletedOrders }` |
| `…Deliveries` | Mongo `deliveries-service` | `deliveries` | `{ Id, OrderId, Status, Notes, Registrations }`, embedded `DeliveryRegistrationDocument { Description, DateTime }` |
| `…Identity` | Mongo `identity-service` | `users`, `refreshTokens` | `UserDocument { Id, Email, Role, Password, CreatedAt, Permissions }`; `RefreshTokenDocument { Id, UserId, Token, CreatedAt, RevokedAt }` |
| `…Operations` | **Redis only** | key `requests:{correlationId}` under instance prefix `operations:` | `OperationDto { Id, UserId, Name, State, Code, Reason }`, sliding expiry 300 s |
| `…OrderMaker` | **none** | — | saga state in process memory (`AIMakingOrderData`) |
| `…Orders` | Mongo `orders-service` | `orders`, `customers` | `OrderDocument { Id, CustomerId, VehicleId, Status, CreatedAt, DeliveryDate, TotalPrice, Parcels[{Id,Name,Variant,Size}] }`; replicated `CustomerDocument { Id }` |
| `…Parcels` | Mongo `parcels-service` | `parcels`, `customers` | `ParcelDocument { Id, CustomerId, Variant, Size, Name, Description, CreatedAt, OrderId, AddedToOrder }`; replicated `CustomerDocument { Id }` |
| `…Pricing` | **none** | — | stateless |
| `…Vehicles` | Mongo `vehicles-service` | `vehicles` | `{ Id, Brand, Model, Description, PayloadCapacity, LoadingCapacity, PricePerService, Variants }` |
| `…APIGateway` | none | — | stateless proxy |

Each service that owns messages also holds `inbox` and `outbox` collections in its own database, created by `Convey.MessageBrokers.Outbox.Mongo` (`expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).

**Cross-domain coupling.** There is no foreign key anywhere — each service has its own database and Mongo enforces nothing across them. Coupling takes three forms:

1. **Replicated read models.** `orders-service` and `parcels-service` each keep a local `customers` collection holding only `{ Id }`, populated from the `CustomerCreated` event on the `customers` exchange. Both use it as an existence check before accepting work.
2. **Denormalised snapshots.** An `OrderDocument` embeds a copy of each parcel's `{ Id, Name, Variant, Size }`. A parcel renamed in `parcels-service` does not change the copy inside an existing order.
3. **Shared identifier spaces by convention.** `ordermaker-service` passes a **vehicle** identifier to `availability-service` as a **resource** identifier. Nothing in either service states that contract, and nothing enforces it.

### 7. Messaging, async and event mechanisms

**System:** RabbitMQ **topic exchanges** through `Convey.MessageBrokers.RabbitMQ`, one exchange per service, with `conventionsCasing: snakeCase` and a queue template of `<service-name>/{{exchange}}.{{message}}`. Message identity travels in headers: `message_context` (correlation context), `span_context` (Jaeger), and `Saga` (the OrderMaker state, read by `operations-service`). Publishing services use the Mongo transactional outbox; `api-gateway`, `ordermaker-service` and `operations-service` do not.

**Exchanges and volumes** (counts from `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`, the platform catalogue):

| Exchange | Owning service | Commands | Events | Rejected events |
|---|---|---|---|---|
| `availability` | `availability-service` | 4 | 5 | 4 |
| `customers` | `customers-service` | 2 | 3 | 2 |
| `deliveries` | `deliveries-service` | 4 | 4 | 3 |
| `identity` | `identity-service` | 2 | 2 | 2 |
| `ordermaker` | `ordermaker-service` | 0 | 1 | 1 |
| `orders` | `orders-service` | 7 | 9 | 10 |
| `parcels` | `parcels-service` | 2 | 2 | 2 |
| `vehicles` | `vehicles-service` | 3 | 3 | 3 |
| `operations` | `operations-service` | — | — | — (declares an exchange, publishes nothing) |

**Cross-service event subscriptions actually wired in code** (each is `SubscribeEvent<T>()` in the subscriber's `Extensions.cs`):

| Event | Published on exchange | Consumed by | Payload key fields |
|---|---|---|---|
| `SignedUp` | `identity` | `customers-service` | `UserId`, `Email`, `Role` |
| `CustomerCreated` | `customers` | `orders-service`, `parcels-service` | `CustomerId` |
| `OrderCompleted` | `orders` | `customers-service` | `OrderId`, `CustomerId` |
| `OrderCreated` | `orders` | `ordermaker-service` | `OrderId`, `CustomerId` |
| `OrderApproved` | `orders` | `ordermaker-service` | `OrderId`, `CustomerId` |
| `ParcelAddedToOrder` | `orders` | `ordermaker-service` | `OrderId`, `ParcelId` |
| `VehicleAssignedToOrder` | `orders` | `ordermaker-service` | `OrderId`, `VehicleId` |
| `ResourceReserved` | `availability` | `ordermaker-service` (subscribed, **no saga action** — dead path) | `ResourceId`, `DateTime` |
| `ParcelDeleted` | see the conflict below | `orders-service` | `ParcelId` |
| every catalogued message | all eight exchanges | `operations-service` | envelope only — the generated types carry **no properties** |

**Conflict — needs validation.** `orders-service` declares its `ParcelDeleted` subscription with `[Message("deliveries")]`, while `parcels-service` publishes `ParcelDeleted` on the `parcels` exchange. On the evidence on disk the binding does not match the publisher, so deleted parcels would not be removed from open orders. Recorded in full in the Orders summary.

**Not observable from source and marked accordingly:** actual queue bindings and dead-letter behaviour at runtime, message volumes, retry outcomes, and whether any queue is currently accumulating — all **unknown — requires runtime capture**.

### 8. APIs exposed and consumed

**Ports** (host ports from `hianshul100_Pacco/compose/services.yml`; every container listens on 80 internally):

| Service | Host port |
|---|---|
| `api-gateway` | 5000 |
| `availability-service` | 5001 |
| `customers-service` | 5002 |
| `deliveries-service` | 5003 |
| `identity-service` | 5004 |
| `operations-service` | 5005 |
| `orders-service` | 5006 |
| `parcels-service` | 5007 |
| `pricing-service` | 5008 |
| `vehicles-service` | 5009 |
| `ordermaker-service` | 5015 |

Every service exposes Swagger at route prefix `docs` and a `ping` endpoint for Consul health checks. All but Identity and Operations register their routes through Convey's CQRS dispatcher (`UseDispatcherEndpoints`); those two use plain `UseEndpoints`.

**External surface.** The gateway is the only intended entry point. Its two profile families behave differently: `ntrada.yml` proxies everything downstream over HTTP, while `ntrada-async.yml` publishes write operations to RabbitMQ and returns a correlation identifier. The composed stack runs the **async** profile. Twenty routing keys are bound across the seven exchanges the gateway publishes to; the full route tables are in the gateway summary.

**Synchronous service-to-service calls** — all through Fabio, all declared in `httpClient.services`:

| Caller | Callee | Call |
|---|---|---|
| `orders-service` | `parcels-service` | `GET /parcels/{id}` |
| `orders-service` | `pricing-service` | `GET /pricing?customerId={id}&orderPrice={price}` |
| `orders-service` | `vehicles-service` | `GET /vehicles/{id}` |
| `pricing-service` | `customers-service` | `GET /customers/{id}` |
| `ordermaker-service` | `vehicles-service` | `GET /vehicles` then takes the first item |
| `ordermaker-service` | `availability-service` | `GET /resources/{resourceId}` |

**Not reachable through the gateway:** `ordermaker-service` entirely (no module in any profile), the Operations SignalR hub at `/pacco`, and the Operations gRPC service.

### 9. Deployment and runtime clues

- **Containers.** Every service has the same two-stage `Dockerfile` (`sdk:3.1` → `aspnet:3.1`), sets `ASPNETCORE_URLS http://*:80` and `ASPNETCORE_ENVIRONMENT docker`, and publishes as `devmentors/pacco.services.<name>` (`devmentors/pacco.apigateway` for the gateway).
- **Compose.** `hianshul100_Pacco/compose/infrastructure.yml` brings up the eleven infrastructure containers; `compose/services.yml` and `compose/services-local.yml` bring up the eleven application containers. Both attach to an **external** network `pacco-network`, which must exist before either stack starts.
- **Process lists.** `hianshul100_Pacco/services.yml` and `prod-services.yml` describe running the services as processes rather than containers. **Neither lists `ordermaker`.**
- **CI.** Every service repository carries the same `.travis.yml`: dotnet 3.1.100, branches `master` and `develop`, `./scripts/build.sh`, then `./scripts/test.sh` in most repositories, then `./scripts/dockerize.sh` on success.
- **Environments.** Four configuration files per service — `appsettings.json`, `.development.json`, `.docker.json`, `.local.json` — selected by `ASPNETCORE_ENVIRONMENT`.

### 10. Security and auth clues

- **Authentication** is JWT, issued by `identity-service` and validated at the gateway (`Ntrada.Extensions.Jwt`). The gateway sets `auth.global: false`, so each route opts in with `auth: true`; the admin claim is checked by claim type `role` with value `admin` on selected routes.
- **The same 80-character JWT signing key is committed in plain text** in `hianshul100_Pacco.Services.Identity`, `hianshul100_Pacco.Services.Operations` and `hianshul100_Pacco.APIGateway`. Rotating it requires all three to change together.
- **A signing certificate and its password are committed** in `hianshul100_Pacco.Services.Identity` (`certs/localhost.pfx`, password `"test"`). Every other service ships the matching public `certs/localhost.cer`.
- **Vault** is configured in every service except OrderMaker: kv v2 at `<service>/settings`, PKI role `<service>` with common name `<service>.pacco.io`, and for the Mongo-backed services a dynamic database credential lease with auto-renewal. **Every `appsettings.docker.json` disables Vault**, so the composed stack runs on committed values.
- **Certificate authentication** is available in `availability-service` (`security.certificate.header: "Certificate"`) and is the only service-to-service authentication mechanism found anywhere.
- **Service-to-service calls carry no credentials.** Nothing on the internal network is authenticated between services, and several endpoints that take a customer identifier accept it from the request body or query string rather than from a token.
- Every service masks secret-like values in logs via `logger.excludeProperties`, and most enable `httpClient.requestMasking` with `maskTemplate: "*****"`.

### 11. Observability, logging and tracing clues

- **Tracing:** Jaeger via `Convey.Tracing.Jaeger`, constant sampler, UDP to port 6831, with `Convey.Tracing.Jaeger.RabbitMQ` propagating `span_context` across the broker. **`ordermaker-service` has no Jaeger integration at all** — the trace breaks at the orchestrator.
- **Metrics:** Prometheus through `Convey.Metrics.AppMetrics`, scraped into Grafana; InfluxDB configured but disabled everywhere. `availability-service` additionally ships a `MetricsJob` and a `CustomMetricsMiddleware`.
- **Logging:** Serilog through `Convey.Logging` to console, rolling file and **Seq**; ELK configured but disabled in every service. `logger.excludePaths` and `jaeger.excludePaths` both drop `/`, `/ping` and `/metrics`.
- **User-visible operation status:** `operations-service` over SignalR — `connected`, `disconnected`, `operation_pending`, `operation_completed`, `operation_rejected` — retained for 300 seconds.

### 12. Files holding major architecture decisions; feature flags

No repository contains an ADR directory, a decision log, or an architecture document. The decisions are embedded in code and configuration:

| Decision | Where it lives |
|---|---|
| External routing, sync vs async per route, auth per route | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada*.yml` |
| Technology composition per service | `src/<service>.Infrastructure/Extensions.cs` (or `Infrastructure/Extensions.cs` in single-project services) |
| Which messages a service consumes | the `SubscribeCommand<T>()` / `SubscribeEvent<T>()` chain in the same file |
| The end-to-end ordering process | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` |
| The platform message catalogue | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Operation state semantics and retention | `…Operations.Api/Handlers/GenericEventHandler.cs`, `Services/OperationsService.cs` |
| The discount policy | `hianshul100_Pacco.Services.Pricing/src/Pacco.Services.Pricing.Api/Core/Services/CustomerDiscountsService.cs` |
| Deployment topology | `hianshul100_Pacco/compose/*.yml`, `services.yml`, `prod-services.yml` |

**Feature flag system: none, anywhere in the platform.** No flag library is referenced in any `csproj`, no flag store is configured, and no flag key exists. The only runtime switches found are:

| Switch | Values seen | Effect |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | `local`, `development`, `docker` | selects the `appsettings.*.json` overlay |
| `NTRADA_CONFIG` | `ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` | selects the gateway's whole routing and sync/async behaviour |
| Convey `enabled` booleans | `true` / `false` per integration | turns Consul, Fabio, Vault, Jaeger, metrics, Swagger and each log sink on or off |
| `signalR.backplane` | `redis` in every file | Redis backplane, or in-process SignalR for any other value |

These are deployment switches, not product feature flags.

### 13. Open questions and ambiguities

Consolidated in *Assumptions, Blockers & Open Questions* at the end of this document, and per repository in each summary file.

### 14. Frontend stack

**Frontend assets exist in exactly one place in the workspace:** `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/` — `index.html` (plain HTML, **Bootstrap 4.0.0** from `maxcdn.bootstrapcdn.com`), `js/app.js` (45 lines, no framework, hub URL hard-coded to `http://localhost:5005/pacco`), and `js/signalr.js` (the SignalR JavaScript client, vendored unminified). No `package.json`, no bundler, no transpiler, no lock file, anywhere.

For every other repository: **No frontend assets detected — checked:** each repository root, every `src/<project>/` directory and its subdirectories, `tests/` where present, `scripts/`, and `compose/` and `assets/` in `hianshul100_Pacco`. No `package.json`, no `wwwroot`, no `index.html`, no template, bundler configuration or static asset directory was found in any of them. `hianshul100_Pacco/assets/` holds only the project logo images used by the READMEs.

`hianshul100_Pacco.Web`, the one repository whose name implies a front end, contains no frontend assets — see its summary.

---

## Cross-repository relationships

**Asynchronous.** Nine exchanges, one per service plus `operations`. `identity` feeds `customers`; `customers` feeds `orders` and `parcels`; `orders` feeds `customers` and `ordermaker`; `availability` feeds `ordermaker`; `parcels` feeds `orders`. `operations-service` subscribes to all eight publishing exchanges and publishes nothing.

**Synchronous.** Six calls, listed in dimension 8. Two chains matter: `orders-service → pricing-service → customers-service` (three services deep, on the vehicle-assignment path) and `ordermaker-service → vehicles-service` / `→ availability-service` (inside a saga step with no compensation).

**Shared code.** None. There is no shared library, no NuGet package produced by this platform, and no common contracts assembly. Message contracts are **re-declared in each repository that uses them** — `ordermaker-service` keeps its own copies of five other services' commands and five of their events, and `operations-service` generates types at runtime from `messages.json`. The contract is the wire name and nothing else.

**Shared configuration.** The gateway, Identity and Operations share one committed JWT signing key. Every service shares one committed `localhost.cer`. All eleven services share one `pacco-network` and one instance each of MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus and Seq.

**Build-time.** `hianshul100_Pacco/Pacco.sln` references 41 project files by relative path into sibling repository directories. The solution only opens if all sibling repositories are cloned next to it under their upstream names.

---

## Suspected subsystems

Grouped from the observed message flow and call graph, not from any document.

| Subsystem | Members | What holds it together |
|---|---|---|
| **Ordering and fulfilment** | `orders-service`, `parcels-service`, `deliveries-service`, `ordermaker-service` | The order aggregate and the saga that drives it; the `orders` exchange carries the most traffic in the platform (7 commands, 9 events, 10 rejected events) |
| **Resource and fleet** | `availability-service`, `vehicles-service` | Both model scarce physical capacity; a vehicle identifier is used as an availability resource identifier |
| **Identity and customer** | `identity-service`, `customers-service`, `pricing-service` | `SignedUp` creates a customer; the customer's VIP status and completed-order count drive the discount |
| **Edge** | `api-gateway` | The only external entry point; decides sync vs async per route |
| **Operations and observability** | `operations-service` and the shared Jaeger/Prometheus/Grafana/Seq stack | Cross-cutting; listens to everything, owns no domain |
| **Platform** | `hianshul100_Pacco` | Compose stacks, process lists, umbrella solution, build scripts |
| **Unassigned** | `hianshul100_Pacco.Web` | No evidence on which to place it |

`deliveries-service` sits oddly in the first group: it subscribes to **no events at all** and is driven only by commands, so nothing in the platform automatically starts a delivery when an order completes.

---

## Platform documentation versus the repository tree

- **Correct across the board:** service names, ports, Docker image names, and the instruction to run `./scripts/start.sh`.
- **Stale doc, repeated in every service README:** the `dotnet run` path is given as `/src/Pacco.Services.<Name>` while the project directory is `src/Pacco.Services.<Name>.Api`. Each repository's own `scripts/start.sh` has the right path. OrderMaker and Pricing are the exceptions in different directions — OrderMaker's path is correct because it has no `.Api` suffix; Pricing's is wrong like the rest.
- **Stale doc:** `hianshul100_Pacco/README.md` lists the repositories to clone and omits both `Pacco.Web` and `Pacco.Context`, the two repositories in this workspace with no platform code.
- **Stale doc:** nothing in the platform README mentions that the composed stack runs the gateway's **async** profile, which changes the behaviour of every write route.
- **Conflict:** `Pacco.sln` references `Pacco.APIGateway.Ocelot.csproj`, which exists in no repository in this workspace. The solution cannot be loaded as committed.
- **Conflict:** `ordermaker` is in both Compose files and in neither process list.
- **Conflict:** most service repositories run `./scripts/test.sh` in CI while containing no test project. Only Availability (5 test projects), Orders (Pact consumer) and Parcels (Pact provider) have any tests at all, and Availability's own `.travis.yml` runs **only** `build.sh`, so its five test projects never run in CI.
- **Undocumented anywhere:** the saga, the outbox, the dynamic subscription mechanism, the discount tiers, the 300-second operation retention, and the fact that no service is authenticated to any other.

---

## Gaps and unknowns

| Gap | Status |
|---|---|
| Runtime queue bindings, dead-letter behaviour, message volumes, retry outcomes | **unknown — requires runtime capture** |
| What `ordermaker-service` uses Redis for | **unknown — requires runtime capture**; registered in composition, no call found in source |
| Whether the gateway forwards `orderPrice` to `pricing-service` | **unknown**; the route names only `customerId` |
| Whether `ParcelDeleted` is delivered to `orders-service` | **needs validation**; the subscriber's exchange annotation does not match the publisher's exchange |
| Whether `order_approved` is ever published after a resource is reserved | **unknown**; the saga's final step waits for it and no publisher was found on that path |
| The licence for `hianshul100_Pacco.Services.Pricing` and `hianshul100_Pacco.Services.Vehicles` | **unknown**; neither repository has a LICENSE file |
| `Pacco.APIGateway.Ocelot` | **Unverifiable — Missing Source Evidence**; referenced by `Pacco.sln`, present in no repository |
| The Pacco web client | **Unverifiable — Missing Source Evidence**; `hianshul100_Pacco.Web` holds only a README |
| Vault unseal keys and root token committed in `hianshul100_Pacco/docker-images.txt` | recorded as a blocker in the platform summary; the values are deliberately not reproduced in these documents |
| Production topology — replicas, scaling, resource limits, secrets delivery | **unknown**; `prod-services.yml` is a process list, not a deployment description |

---

## Coverage

Authoritative project lists were enumerated per repository from `*.sln` and `*.csproj` on disk. Every enumerated project is either documented in the corresponding summary or excluded below with a reason.

| Repository | Enumerated | Documented | Excluded (with reason) |
|---|---|---|---|
| `hianshul100_Pacco` | 0 projects of its own; `Pacco.sln` references 41 project files in sibling repositories | 40 of the 41, each within its own repository's summary | 1 — `Pacco.APIGateway.Ocelot.csproj`: referenced by `Pacco.sln` but present in no repository in this workspace; recorded as **Unverifiable — Missing Source Evidence** |
| `hianshul100_Pacco.APIGateway` | 1 — `src/Pacco.APIGateway/Pacco.APIGateway.csproj` | 1 | 0 |
| `…Services.Availability` | 9 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure`; `tests/…Tests.EndToEnd`, `…Tests.Integration`, `…Tests.Performance`, `…Tests.Shared`, `…Tests.Unit` | 9 | 0 |
| `…Services.Customers` | 4 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure` | 4 | 0 |
| `…Services.Deliveries` | 4 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure` | 4 | 0 |
| `…Services.Identity` | 4 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure` | 4 | 0 |
| `…Services.Operations` | 2 — `src/Pacco.Services.Operations.Api`, `src/Pacco.Services.Operations.GrpcClient` | 2 (the GrpcClient documented as a developer tool, not a deployable) | 0 |
| `…Services.OrderMaker` | 1 — `src/Pacco.Services.OrderMaker` | 1 | 0 |
| `…Services.Orders` | 5 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure`; `tests/Pacco.Services.Orders.PactConsumerTests` | 5 | 0 |
| `…Services.Parcels` | 5 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure`; `tests/Pacco.Services.Parcels.PactProviderTests` | 5 | 0 |
| `…Services.Pricing` | 1 — `src/Pacco.Services.Pricing.Api` | 1 | 0 |
| `…Services.Vehicles` | 4 — `src/…Api`, `…Application`, `…Core`, `…Infrastructure` | 4 | 0 |
| `…Web` | 0 — no manifest of any kind on disk | 0 | 0; the repository itself is documented as **Unverifiable — Missing Source Evidence** |
| `hianshul100_Pacco.Context` | — | — | Excluded from the inventory: this is the artifact repository these documents are written into, not a platform source repository |

**Totals:** 40 projects enumerated across the twelve source repositories, 40 documented, 0 excluded on their own account, plus 1 solution reference (`Pacco.APIGateway.Ocelot`) excluded because it has no source in this workspace.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This is the platform-level view. Each repository's own summary carries its specific items, and everything raised there that affects more than one repository is repeated here. The three committed-secret blockers (B1, B2, B3) are the items that should be dealt with first.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The twelve repositories with source plus `hianshul100_Pacco.Web` are the whole platform. | They are what the workspace contains; `Pacco.sln` references nothing outside them except the missing Ocelot project. |
| A2 | The `ntrada-async` profile is the intended production behaviour. | It is what `compose/services.yml` sets via `NTRADA_CONFIG`, which is the only complete runnable description of the platform. |
| A3 | `messages.json` in `…Services.Operations` is meant to be the authoritative platform message catalogue. | It is the only file listing every message across all services, and `operations-service` builds live subscriptions from it. |
| A4 | Each service owns its database exclusively. | Each configures its own Mongo database name and no connection string is shared between services. |
| A5 | Where a README and the code disagree, the code describes the running system. | The task's own rule, and it is borne out — for example every `scripts/start.sh` contradicts its README's path and matches the directory that exists. |
| A6 | This platform is the devmentors.io Pacco reference implementation. | Every README links to `github.com/devmentors/Pacco` and every image is published under `devmentors/`. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** One JWT signing key is committed in plain text in three repositories — `…Services.Identity`, `…Services.Operations` and `…APIGateway` — and is identical in all three. | Anyone with read access to any of the three can forge a token that the whole platform accepts. | Security-architecture owner, coordinating the Identity, Operations and gateway maintainers; the key must be rotated everywhere at once. |
| B2 | **[ACTION NOW]** A signing certificate and its password are committed in `…Services.Identity` (`certs/localhost.pfx`, password `"test"`). | The private key backing the platform's certificate story is public to anyone with the repository. | Identity service maintainer with the security-architecture owner. |
| B3 | **[ACTION NOW]** `hianshul100_Pacco/docker-images.txt` contains Vault unseal keys and a root token. | Anyone with the repository can unseal and read every secret Vault holds. The values are not reproduced in these documents. | Platform owner with the security-architecture owner: rotate the Vault instance and purge the values from history. |
| B4 | **[ACTION NOW]** Services are not authenticated to one another, and several endpoints accept a customer identifier from the request rather than from a token. | Anything on the `pacco-network` can act as any customer against `orders-service`, `pricing-service` or `ordermaker-service`. | Security-architecture owner with every service maintainer. |
| B5 | **[ACTION NOW]** `Pacco.sln` references `Pacco.APIGateway.Ocelot.csproj`, which does not exist in this workspace. | The umbrella solution cannot be opened or built as committed. | Platform owner: restore the project or remove the reference. |
| B6 | **[ACTION NOW]** Eight of the eleven services have no test project while their CI runs `scripts/test.sh`, and Availability's five test projects are not run by its own CI. | The platform's build is green without verifying anything, including the discount rule and the saga. | Each service maintainer, with the platform owner setting the expectation. |
| B7 | **[ACTION NOW]** `ordermaker-service` holds saga state in process memory with no persistence configured. | A restart abandons every in-flight order mid-flight, and the service cannot be scaled beyond one instance. | OrderMaker service maintainer. |
| B8 | **[ACTION NOW]** `orders-service` binds its `ParcelDeleted` subscription to the `deliveries` exchange while `parcels-service` publishes it on `parcels`. | Deleting a parcel would leave it attached to open orders. Marked **needs validation** — confirm against a running broker before changing code. | Orders service maintainer with the Parcels service maintainer. |
| B9 | **[ACTION NOW]** Message contracts are re-declared by name in each repository, with no shared package and no schema. | A rename in one service silently breaks every other service and the Operations catalogue, with no build error anywhere. | Integration owner with all service maintainers. |
| B10 | **[ACTION NOW]** The backlog spreadsheet supplied as an attachment for this stage is empty and contains no rows. | No requirement, feature or ticket from that input could be used, so nothing in this inventory is traced to a backlog item. | Whoever supplied the attachment: re-supply it with content, or confirm that no backlog input applies to this stage. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Should `ordermaker-service` be reachable through the API gateway, and should it appear in `services.yml` and `prod-services.yml`? | Today the platform's only orchestrated flow cannot be started externally and is missing from two of the four ways to run the system. | Platform owner with the gateway and OrderMaker maintainers. |
| Q2 | **[ACTION NOW]** What publishes `order_approved` after a resource is reserved? | The saga's final step waits for it; if nothing publishes it, no saga ever completes. | Orders service maintainer with the OrderMaker service maintainer. |
| Q3 | **[ACTION NOW]** What starts a delivery? | `deliveries-service` subscribes to no events, so nothing in the platform creates a delivery when an order completes. | Deliveries service maintainer with the Orders service maintainer. |
| Q4 | **[ACTION NOW]** Who keeps `messages.json` in step with the services it mirrors? | Several entries already disagree with the code on disk, and a stale entry silently stops an operation from ever completing. | Operations service maintainer with each service maintainer. |
| Q5 | **[ACTION NOW]** Under what licence are `…Services.Pricing` and `…Services.Vehicles` published? | They are the only two repositories with no LICENSE file. | Repository owner. |
| Q6 | **[handled later by the integration-architecture stage]** Should the synchronous chains be broken up? | `orders-service → pricing-service → customers-service` fails as a unit, and the saga's HTTP steps have no compensation. | Integration-architecture stage. |
| Q7 | **[handled later by the domain-model stage]** Is a vehicle the same thing as an availability resource? | `ordermaker-service` passes a vehicle identifier where a resource identifier is expected, and no service states that contract. | Domain-model stage. |
| Q8 | **[handled later by the domain-model stage]** Should replicated `customers` read models and embedded parcel snapshots be kept as they are? | They are the platform's only cross-domain coupling and nothing keeps them consistent after the initial event. | Domain-model stage. |
| Q9 | **[handled later by the deployment-architecture stage]** What is the intended production topology? | `prod-services.yml` is a process list; there is no description of replicas, scaling, resource limits or how secrets reach a running service with Vault disabled in the Docker profile. | Deployment-architecture stage. |
| Q10 | **[handled later by the observability stage]** Should `ordermaker-service` be brought into the tracing story? | It is the only service with no Jaeger integration and the only one that spans five services, so traces break exactly where they are most useful. | Observability stage. |
| Q11 | **[handled later by the ADR-authoring stage]** Which of the platform's implicit decisions should be written up as ADRs? | The saga, the outbox, the sync-versus-async gateway split, the no-shared-contracts choice and the no-migrations choice are all undocumented and all consequential. | ADR-authoring stage. |
