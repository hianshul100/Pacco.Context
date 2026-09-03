# Repository summary — `Pacco.Services.Operations`

**Repository:** `Pacco.Services.Operations` (workspace clone path: `hianshul100_Pacco.Services.Operations`)
**Deployable:** `operations-service` (also known as: Operations Service, `Pacco.Services.Operations.Api`, image `devmentors/pacco.services.operations`). **Repository: `Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.Api`.**
**Second artefact in this repository:** `Pacco.Services.Operations.GrpcClient` — a console gRPC client, **not a deployable service**. **Repository: `Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.GrpcClient`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Operations
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

The platform's **asynchronous operation tracker** — the piece that makes the fire-and-forget gateway usable. Because `api-gateway` runs the asynchronous profile and returns immediately after publishing a command, callers need somewhere to learn what happened. `operations-service` subscribes to **every message on every exchange**, records each one as an operation with a state (`pending`, `completed`, `rejected`), exposes it over HTTP and gRPC, and pushes it live to browsers over SignalR.

**Evidence:** `src/Pacco.Services.Operations.Api/messages.json`, `Infrastructure/Subscriptions.cs`, `Hubs/PaccoHub.cs`, `Operations.proto`, `Program.cs`, `Services/OperationsService.cs`.

**Correction recorded in this pass:** an earlier reading of this repository described the operation store as a MongoDB collection named `operations`. **The code does not support that.** `Services/OperationsService.cs` writes every operation through `IDistributedCache` — that is, **Redis** — under the key `requests:{id}` with a **sliding expiry** taken from `requests.expirySeconds`, which is **`300`**. `Infrastructure/Extensions.cs` calls `.AddMongo()` but registers **no** `AddMongoRepository<>` and no repository type, and `grep -rn "Mongo" src` returns exactly two hits, both in `Extensions.cs` (the `using` and the `.AddMongo()` call). There is no `operations` collection and no document type anywhere in the repository. See §6.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) process serving **four protocols at once**: HTTP/REST, **gRPC**, **SignalR (WebSockets)**, and RabbitMQ consumption. It is the only multi-protocol service in Pacco and the only one that serves a browser UI.

**Structurally it is the platform's biggest departure from the norm.** There is no `.Application` / `.Core` / `.Infrastructure` split: everything lives in the single `Pacco.Services.Operations.Api` project. It has no domain model — it stores what it observes rather than enforcing invariants — so the clean-architecture layering used by the eight domain services would be empty ceremony here. The root `Pacco` README's hedge ("clean architecture + DDD … or another style that is the best fit") is exactly this case.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Operations.Api/Program.cs` — `UseEndpoints` mapping `GET operations/{operationId}`, **`MapHub<PaccoHub>("/pacco")`**, and **`MapGrpcService<GrpcServiceHost>()`** |
| Dynamic RabbitMQ subscription | `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` |
| Message catalogue | `src/Pacco.Services.Operations.Api/messages.json` — read at runtime; **the platform's authoritative event/command catalogue** |
| SignalR hub | `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs` |
| gRPC contract | `src/Pacco.Services.Operations.Api/Operations.proto` — at the **project root**, not under a `Protos/` directory (a duplicate copy exists at `src/Pacco.Services.Operations.GrpcClient/Operations.proto`) |
| Operation state store | `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` — Redis via `IDistributedCache`, key `requests:{id}` |
| Browser UI | `src/Pacco.Services.Operations.Api/wwwroot/ui/index.html` |
| gRPC sample client | `src/Pacco.Services.Operations.GrpcClient/Program.cs` |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Operations.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh`, `scripts/proto/lin-compile.sh`, `scripts/proto/mac-compile.sh`, `scripts/proto/win-compile.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Operations.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Operations.Api` | The entire service — hub, gRPC host, dynamic subscriber, Redis-backed operation store, message catalogue, `wwwroot/ui/`. **34 files, no layering:** `DTO/`, `Handlers/` (3 generic handlers + `Extensions.cs`), `Hubs/`, `Infrastructure/` (6 files), `Queries/`, `Services/` (7 files), `Types/` (6 files), `certs/localhost.cer`, `messages.json`, `Operations.proto`, `wwwroot/ui/` |
| `src/Pacco.Services.Operations.GrpcClient` | A console application demonstrating the gRPC contract (`Program.cs` + its own copy of `Operations.proto`). Not deployed, not containerised, not in any compose file. |

**No test projects exist in this repository.**

**Distinctive NuGet packages** (beyond the standard Convey set): `Google.Protobuf 3.11.4`, `Grpc.AspNetCore 2.28.0`, `Grpc.Tools 2.28.1`, `Microsoft.AspNetCore.SignalR 1.1.0`, `Microsoft.AspNetCore.SignalR.Redis 1.1.5`.

`Microsoft.AspNetCore.SignalR.Redis` matters architecturally: it is the **Redis backplane** that lets SignalR broadcast across multiple instances of the service. Redis is configured in every service, but this is the only one with an identified functional use for it.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| RabbitMQ | in | subscribes to **every exchange** listed in `messages.json` |
| MongoDB | out | **configured but not used.** `appsettings.json` declares `mongo.connectionString: mongodb://localhost:27017`, `mongo.database: operations-service`, `mongo.seed: false`, and `Infrastructure/Extensions.cs` calls `.AddMongo()` — but no repository, document type, or collection is registered anywhere. Vault is even configured to issue **dynamic MongoDB credentials** for it (`vault.lease.mongo.roleName: operations-service`, `autoRenewal: true`) |
| Redis | out | instance prefix `operations:`; **the actual operation store** (`IDistributedCache`, key `requests:{id}`) **and** the SignalR backplane |
| Consul | out | registers `operations-service` on port `5005` |
| Fabio | out | `http://localhost:9999` |
| Vault | out | KV v2 `kv/operations-service/settings`; PKI role `operations-service` |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |
| Browsers | in | SignalR over WebSockets at `/pacco` |
| gRPC clients | in | `GrpcOperationsService` |

`httpClient.services` is **empty** — no outbound HTTP calls. Notably, this service depends on **all eight message-producing services** at the broker level, and `Pacco/compose/services.yml` gives it the platform's only `depends_on` list: availability, customers, deliveries, identity, orders, ordermaker, parcels, vehicles.

## 6. Data stores / state handling

- **Store: Redis, and only Redis.** `Services/OperationsService.cs` takes an `IDistributedCache` and a `RequestsOptions`, and is registered as `AddSingleton<IOperationsService, OperationsService>()` in `Infrastructure/Extensions.cs`. Every read is `_cache.GetStringAsync(GetKey(id))` and every write is `_cache.SetStringAsync(...)`. The key is built by `private static string GetKey(string id) => $"requests:{id}"`, and `redis.instance` is `operations:`, so the effective Redis key is **`operations:requests:{operationId}`**.
- **Access mechanism:** `Microsoft.Extensions.Caching.Distributed.IDistributedCache` backed by `Convey.Persistence.Redis` (`.AddRedis()`). The value is the `OperationDto` serialised with `Newtonsoft.Json`. **No ORM, no query language, no repository abstraction** — operations can only be fetched by exact id, never listed, searched, or aggregated.
- **Migration tool: none.** None is needed; there is no schema.
- **MongoDB is configured but unused.** See §5. This is a live discrepancy: the service holds a Mongo connection string, a database name, and a Vault dynamic-credential lease for a database it never writes to.
- **Record shape** (`DTO/OperationDto.cs`): `Id`, `UserId`, `Name`, `State` (`OperationState` — `Pending` / `Completed` / `Rejected`), `Code`, `Reason` — matching the gRPC `GetOperationResponse` field set exactly. **There is no timestamp field.**
- **Expiry — the defining property of this store.** Each write sets `new DistributedCacheEntryOptions { SlidingExpiration = TimeSpan.FromSeconds(_options.ExpirySeconds) }`, and `appsettings.json` sets `requests.expirySeconds: 300`. **An operation record therefore disappears 5 minutes after it was last touched.** This is a *sliding* expiry, so the clock restarts on every read and every state change; a record that nobody reads and that receives no further messages is gone 5 minutes after it was created. There is no archive, no durable copy, and no way to recover it — the record is not written anywhere else.
- **Terminal-state guard:** `TrySetAsync` refuses to modify an operation already in `Completed` or `Rejected` state, returning `(false, operation)`. Late or duplicate messages cannot flip a finished operation back to `pending`, which makes the write **idempotent with respect to terminal states** — but only for as long as the record still exists.
- **Cross-domain coupling:** this service holds a **projection of activity from every other domain**. It does not own any aggregate and enforces no invariant; its records are derived entirely from observed messages. There is no foreign key of any kind, but it is coupled to every service's message contract at once — a change to any exchange or message name affects it.
- **No outbox.** `appsettings.json` has **no `outbox` block**, unlike the seven services that do. This service consumes and never publishes, so it needs no outbox; but it also therefore has **no inbox deduplication**, so a redelivered message is processed again (mitigated for terminal states by the guard above).
- **Duplicate registration:** `Infrastructure/Extensions.cs` calls **`.AddRedis()` twice** in the same builder chain (once between `.AddMongo()` and `.AddMetrics()`, once between `.AddJaeger()` and `.AddSignalR()`). Whether this is harmless or duplicates a registration is **Needs validation** — it is recorded here because it is a visible copy-paste artefact in the platform's most unusual composition root.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ. This service's subscription model is unique in the platform and worth stating precisely.

`Infrastructure/Subscriptions.cs` reads `messages.json` at runtime and **emits .NET types dynamically** using `AssemblyBuilder` / `ModuleBuilder` / `TypeBuilder`. For each message name it builds a type, decorates it with `MessageAttribute(exchange, null, null, true)`, and then reflectively invokes `IBusSubscriber.Subscribe<T>` for it. Consequently:

- There are **no `SubscribeCommand<>` / `SubscribeEvent<>` calls in this repository** — searching the source for them finds nothing, even though the service subscribes to more messages than any other.
- The subscription set is **data, not code**: adding an entry to `messages.json` adds a subscription with no recompilation of handler code.
- The service is coupled to every other service's wire contract, but only by name.

**`messages.json` is the platform's authoritative message catalogue.** Reproduced verbatim by service and exchange:

| Service | Exchange | Commands | Events | Rejected |
|---|---|---|---|---|
| `availability-service` | `availability` | `add_resource`, `delete_resource`, `release_resource`, `reserve_resource` | `resource_added`, `resource_deleted`, `resource_reservation_released`, `resource_reservation_canceled`, `resource_reserved` | `add_resource_rejected`, `delete_resource_rejected`, `release_resource_rejected`, `reserve_resource_rejected` |
| `customers-service` | `customers` | `change_customer_state`, `complete_customer_registration` | `customer_created`, `customer_became_vip`, `customer_state_changed` | `change_customer_state_rejected`, `complete_customer_registration_rejected` |
| `deliveries-service` | `deliveries` | `add_delivery_registration`, `complete_delivery`, `fail_delivery`, `start_delivery` | `delivery_completed`, `delivery_failed`, `delivery_started`, `registration_added_to_delivery` | `complete_delivery_rejected`, `fail_delivery_rejected`, `start_delivery_rejected` |
| `identity-service` | `identity` | `sign_in`, `sign_up` | `signed_up`, `signed_in` | `sign_in_rejected`, `sign_up_rejected` |
| `ordermaker-service` | `ordermaker` | — | `make_order_completed` | `make_order_rejected` |
| `orders-service` | `orders` | `add_parcel_to_order`, `approve_order`, `assign_vehicle_to_order`, `cancel_order`, `create_order`, `delete_order`, `delete_parcel_from_order` | `order_approved`, `order_canceled`, `order_completed`, `order_created`, `order_deleted`, `order_delivering`, `parcel_added_to_order`, `parcel_deleted_from_order`, `vehicle_assigned_to_order` | `add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`, `cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`, `delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found` |
| `parcels-service` | `parcels` | `add_parcel`, `delete_parcel` | `parcel_added`, `parcel_deleted` | `add_parcel_rejected`, `delete_parcel_rejected` |
| `vehicles-service` | `vehicles` | `add_vehicle`, `delete_vehicle`, `update_vehicle` | `vehicle_added`, `vehicle_deleted`, `vehicle_updated` | `add_vehicle_rejected`, `delete_vehicle_rejected`, `update_vehicle_rejected` |

**`pricing-service` is absent from `messages.json`** — it participates in no messaging at all.

**Operation state derivation:** a command message produces a `pending` operation; a matching event marks it `completed`; a `*_rejected` message marks it `rejected` and carries `Reason` and `Code`. The **`Saga` header** (`SagaStates.Pending` / `Completed` / `Rejected`, set by `AIOrderMakingSaga` in `ordermaker-service`) also flows through these messages.

**Published: nothing.** This service is purely a consumer.

## 8. APIs exposed or consumed

**HTTP** (base URL `http://localhost:5005`, container port `80`):

| Method | Path | Behaviour | Gateway exposure |
|---|---|---|---|
| GET | `` (root) | writes `AppOptions.Name` — the literal string `Pacco Operations Service` | not routed publicly |
| GET | `operations/{operationId}` | `IOperationsService.GetAsync(id)`; **`404 Not Found` when the record is null** — which includes the case of a real operation whose 5-minute sliding expiry has elapsed. A caller cannot distinguish "never existed" from "expired" | `/operations/{operationId}` — **`auth: false`** |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**gRPC** (`Operations.proto` at the project root, `package Services.Operations`):

```proto
service GrpcOperationsService {
    rpc GetOperation (GetOperationRequest) returns (GetOperationResponse) {}
    rpc SubscribeOperations (Empty) returns (stream GetOperationResponse) {}
}
message GetOperationRequest { string id = 1; }
message GetOperationResponse { string id = 1; string userId = 2; string name = 3; string state = 4; string code = 5; string reason = 6; }
```

`SubscribeOperations` is a **server-streaming** RPC delivering every operation as it happens. It takes no filter argument, so a connected client receives operations for **all users**, including their `userId`. The gRPC endpoint is not routed through `api-gateway` — it is reachable only by direct network access to the service.

**SignalR** (`Hubs/PaccoHub.cs`, hub path `/pacco`):
- Client calls `InitializeAsync(string token)`; the hub parses the JWT with `IJwtHandler.GetTokenPayload(token)`, takes `Guid.Parse(payload.Subject)`, converts it with `ToUserGroup()`, and adds the connection to that group.
- Server-sent messages: `connected`, `disconnected`, and per-operation `operation_pending`, `operation_completed`, `operation_rejected`.
- Because messages are sent to a per-user group, **SignalR delivery is correctly scoped to the owning user** — unlike the gRPC stream and unlike the anonymous HTTP endpoint.

**Consumed:** nothing over HTTP or gRPC.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Operations.Api.dll`.
- Composed as `operations-service` on `5005:80` with the platform's only `depends_on` list (availability, customers, deliveries, identity, orders, ordermaker, parcels, vehicles) — `Pacco/compose/services.yml`.
- Present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5005`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- `scripts/proto/{lin,mac,win}-compile.sh` regenerate the gRPC stubs per platform — a manual step with no CI equivalent.
- **Multi-instance readiness:** the SignalR Redis backplane means this service *can* scale horizontally, but gRPC server streaming and the absence of inbox deduplication mean concurrent instances would each process every message. **Needs validation.**
- **HTTP/2 requirement:** hosting gRPC on the same Kestrel endpoint as HTTP/1.1 REST and WebSockets constrains what proxy can sit in front of it; Fabio's suitability for gRPC is **Unknown**.

## 10. Security/auth clues

- **A symmetric `issuerSigningKey` literal is committed in `appsettings.json`** — the same value that appears in `Pacco.Services.Identity`'s settings and in both of the gateway's Ntrada configurations. The value is not reproduced here. This service needs it because `PaccoHub` validates the token itself rather than relying on the gateway.
- **`jwt.validateIssuer: false`.** This service and `identity-service` are the **only two** of the ten that disable issuer validation (`grep -rn '"validateIssuer"' */src/*/appsettings.json` — the other eight are `true`). Any token signed with the shared key is accepted here regardless of who issued it. An earlier reading of the platform described this setting as unique to `identity-service`; it is not.
- `jwt.allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` is present here, copied verbatim from Identity — **neither path exists in this service**, so the entry has no effect. A copied configuration block.
- **`jwt.certificate.location: certs/localhost.cer`** is set *and* a symmetric `issuerSigningKey` literal is present; `certs/localhost.cer` is committed in the repository.
- **`GET /operations/{operationId}` is `auth: false` at the gateway.** Anyone who can reach the gateway and guess or observe an operation id can read that operation's `userId`, `name`, `state`, `code`, and `reason`.
- **`SubscribeOperations` (gRPC) has no authentication and no per-user filter.** Any client with network access to the service receives a live stream of every operation for every user. There is no `[Authorize]` attribute on the gRPC service and no interceptor in the repository.
- **SignalR is the one correctly scoped channel:** `PaccoHub.InitializeAsync` validates the JWT and joins a per-user group, so broadcasts reach only the owning user. Note the token is passed as a **hub method argument**, not as a connection-level bearer credential, so it is subject to whatever logging the hub pipeline applies to method arguments. **Needs validation.**
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- **The UI loads Bootstrap 4.0.0 from `maxcdn.bootstrapcdn.com`** with subresource-integrity hashes. The SRI hashes mitigate tampering, but the page still depends on a third-party CDN at runtime. Bootstrap 4.0.0 is a January 2018 release.
- Log redaction via `logger.excludeProperties`.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`jaeger.serviceName: "operations"` — the short form, without the `-service` suffix used in `consul.service` and `app.service`, UDP `localhost:6831`, `sampler: const`) with the RabbitMQ Jaeger plugin. **`.AddJaegerDecorators()` is not called here**, unlike in `availability-service`, so command and event handlers are not individually spanned. `.AddHandlersLogging()` is also absent. Whether spans propagate into gRPC and SignalR calls is **Unknown**.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics — in particular there is no gauge for SignalR connection count or gRPC stream count, which is what one would want for a service that holds long-lived connections.
- **This service is itself an observability tool.** It gives the platform a per-user, per-operation view of asynchronous work that Jaeger (per-trace) and Prometheus (aggregate) do not provide. That makes its own instrumentation gaps more consequential: if `operations-service` is down, callers of the asynchronous gateway have no way to learn the outcome of anything.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Operations.Api/messages.json` | **The platform's message catalogue.** It defines what this service watches and is the only single-file inventory of every command, event, and rejection across all eight messaging services. |
| `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` | The decision to generate message types at runtime by reflection rather than referencing each service's contracts — trading compile-time safety for zero coupling to other services' assemblies |
| `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs` | Per-user group scoping as the delivery model for live updates; token validated in the hub |
| `src/Pacco.Services.Operations.Api/Operations.proto` | The gRPC contract, including the unfiltered server-streaming subscription |
| `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` | **The decision that operation records are cache entries, not durable records** — Redis under `requests:{id}`, sliding expiry, plus the terminal-state guard that makes updates idempotent |
| `src/Pacco.Services.Operations.Api/Infrastructure/RequestsOptions.cs` + `appsettings.json` `requests.expirySeconds: 300` | **The platform's asynchronous outcome-retention window: 5 minutes.** This single number bounds how long any caller has to discover the result of any fire-and-forget write |
| `src/Pacco.Services.Operations.Api/Program.cs` | Three protocols on one host: REST, gRPC, SignalR; `UseVault()` |
| `src/Pacco.Services.Operations.Api/appsettings.json` | No outbox; committed signing key; `validateIssuer: false`; Redis backplane configuration; an unused Mongo configuration and Mongo Vault lease |
| `src/Pacco.Services.Operations.Api/wwwroot/ui/` | The decision to ship a demonstration UI inside a backend service, with a vendored script bundle and no build tooling |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`). The closest thing to a runtime-configurable behaviour anywhere in the platform is `messages.json`, which changes the subscription set without a code change — but it is read at start-up, not reloaded, and it ships inside the image.

## 13. Open questions / ambiguities

- **Operation records expire after 5 minutes of inactivity and are stored nowhere else.** Whether `requests.expirySeconds: 300` is a deliberate retention decision or an unexamined default is **Unknown** — treated as a blocker below. It is the opposite of the unbounded-growth risk an earlier reading recorded.
- **`SubscribeOperations` streams every user's operations to any caller** with no authentication and no filter. Whether it is meant to be an internal operator tool is **Unknown** — treated as a blocker below.
- **MongoDB is configured, credentialled through Vault, and never used.** Whether a Mongo-backed store was intended, removed, or never built is **Unknown**.
- **No inbox deduplication** (no outbox block), so a redelivered message creates or updates an operation again. The `TrySetAsync` terminal-state guard makes this safe once an operation has completed or been rejected; a duplicate arriving while the operation is still `pending` simply rewrites the same values and **restarts the 5-minute sliding clock**. **Confirmed by reading `Services/OperationsService.cs`** — this replaces the earlier "Needs validation" on idempotency.
- **`.AddRedis()` is called twice** in `Infrastructure/Extensions.cs`. **Needs validation.**
- **`Pacco.Services.Operations.GrpcClient` has no stated purpose.** It is not deployed, not containerised, and not referenced by any compose file or script. Whether it is a sample, a test harness, or an internal tool is **Unknown**.
- **`messages.json` must be kept in step by hand** with eight other repositories. Nothing validates it, and a renamed message would silently stop being tracked. **Needs validation.**
- **Behaviour when `operations-service` is unavailable** is undefined: the asynchronous gateway keeps accepting writes, but nothing records or reports them. **Unknown.**
- **How the browser UI obtains its JWT** is **Unknown** — `app.js` reads a token from the page, but there is no login flow in `wwwroot/ui/`.
- Whether Jaeger tracing covers the gRPC and SignalR paths is **Unknown**.

## 14. Frontend stack

**Frontend assets are present — the only ones in the entire Pacco workspace.** Located at `src/Pacco.Services.Operations.Api/wwwroot/ui/`.

| File | What it is |
|---|---|
| `wwwroot/ui/index.html` | Plain HTML page, title "Pacoo SignalR" (sic — the title is misspelled in the source). Loads **Bootstrap 4.0.0** from `maxcdn.bootstrapcdn.com` with subresource-integrity hashes, then `js/signalr.js`, then `js/app.js`. |
| `wwwroot/ui/js/app.js` | Hand-written vanilla JavaScript in an IIFE (ES5/ES6, no modules, no transpilation). Builds `new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco')`, calls `connection.invoke('initializeAsync', jwt)`, and listens for `connected`, `disconnected`, `operation_pending`, `operation_completed`, `operation_rejected`. |
| `wwwroot/ui/js/signalr.js` | A **vendored, pre-built webpack UMD bundle** of `@microsoft/signalr` (180,968 bytes, `webpackUniversalModuleDefinition` header). Committed as a binary-like artefact. |

**Stack characterisation:**
- **Framework: none.** No React, Vue, Angular, Svelte, or Blazor.
- **Build tooling: none.** There is **no `package.json` anywhere in the Pacco workspace** — not in this repository and not in any of the other twelve. No npm, yarn, webpack, Vite, or Rollup configuration; the only webpack output present was produced elsewhere and checked in.
- **No micro-frontend architecture.** No module federation, no `single-spa`, no import maps, no web components. The UI is one static page inside one backend service.
- **Styling:** Bootstrap 4.0.0 via CDN only; no local CSS file, no preprocessor.
- **Hard-coded endpoint:** `app.js` connects to `http://localhost:5005/pacco`, so the page only works against a local run — it is a **development demonstration, not a production client**.
- **Checked and absent:** `public/`, `public/js/`, `resources/js/`, `static/`, `assets/`, `web/`, and view templates (`*.cshtml`, `*.razor`, `*.blade.php`, `*.twig`, `*.erb`). Only `wwwroot/` exists.

**This is the platform's entire frontend.** The repository nominally intended to hold a web client, `Pacco.Web`, contains only a one-line README (see `repo-summary/Pacco.Web.md`).

---

## README vs repository

**What the README claims:**
- Operations service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5005`. — **Confirmed** (`appsettings.json` `consul.port: 5005`, `Pacco/compose/services.yml` `5005:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Operations` directory"**; the actual host project is **`/src/Pacco.Services.Operations.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories. Here it is more confusing than elsewhere, because `src/` really does contain a second project (`Pacco.Services.Operations.GrpcClient`), so a reader could reasonably think the missing directory is one they simply need to create.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README — the largest documentation gap in the platform:**
- **The gRPC API.** An entire second protocol, a `.proto` contract, three platform-specific compile scripts, and a companion client project — none of it mentioned.
- **The SignalR hub.** The platform's only real-time channel, its per-user group model, and its five client-facing message names — undocumented.
- **The browser UI** at `wwwroot/ui/` — the only frontend in the workspace, undocumented.
- **`messages.json`** — the authoritative catalogue of every message in the platform lives in this repository and is not mentioned in its README, nor in the root `Pacco` README.
- **The dynamic subscription mechanism** in `Subscriptions.cs`, which is the single most surprising implementation choice in the platform.
- **The `Pacco.Services.Operations.GrpcClient` project** — present in the solution, absent from the documentation.
- **The service's role in the asynchronous flow.** Nothing states that this is where callers of the fire-and-forget gateway learn their outcome, which is the reason the service exists.
- The Redis SignalR backplane; the absence of an outbox; `scripts/proto/`.
- **The 5-minute retention window.** `requests.expirySeconds: 300` is the single most consequential setting in this repository — it bounds how long any caller of the asynchronous gateway has to learn any outcome — and it is documented nowhere, in this README or any other.
- **That MongoDB is configured but unused**, including a Vault dynamic-credential lease for a database the service never opens.

**Unknown (neither pass yielded proof):**
- Whether the demonstration UI is intended to survive into a real deployment.
- Whether the gRPC surface is meant for internal operators or is vestigial.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `messages.json` is an accurate and current catalogue of every message the platform emits. | It is the only single-file inventory in the workspace, and every message name in it was cross-checked against the subscribing and publishing code in the other repositories without contradiction. | Any message renamed or added elsewhere without updating this file would silently stop being tracked, so callers would wait forever for an operation that is never recorded. | Add a build-time check that compares each service's declared message contracts against `messages.json`. |
| A2 | The browser UI under `wwwroot/ui/` is a development demonstration, not a shipped product surface. | It hard-codes `http://localhost:5005/pacco`, has no build step, has no login flow, and its page title is misspelled. | It would be a production client with a hard-coded local endpoint, an unmanaged vendored dependency, and no way to obtain a token. | Ask the product owner whether any real user is expected to open this page. |
| A3 | Redis is genuinely required here — both as the SignalR backplane and as the operation store — unlike in the other nine services where no functional use was found. | `Microsoft.AspNetCore.SignalR.Redis 1.1.5` is referenced only in this project, and `Services/OperationsService.cs` reads and writes every operation through `IDistributedCache`. | Running multiple instances would silently deliver live updates to only whichever instance holds the connection; and losing Redis would lose every in-flight operation record at once, not just the cache. | Confirm the backplane is registered at start-up, test live updates against two instances, and confirm what happens to callers when Redis restarts. |
| A4 | `requests.expirySeconds: 300` is the platform's intended retention window for asynchronous outcomes, not an unexamined default. | It is the only tunable in the `requests` block, it is read into a dedicated `RequestsOptions` type, and it is applied as a sliding expiry on every write. | If it is an accidental default, every asynchronous write silently becomes unobservable after five idle minutes, and no one has decided that. | Ask the service owner what retention the asynchronous flow requires, then set the value deliberately. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** The gRPC `SubscribeOperations` stream has no authentication and no per-user filter: any client that can reach the service receives a live feed of every operation for every user, including user identifiers and rejection reasons. | Security sign-off; any deployment where the service port is reachable by anything other than trusted infrastructure. | Security owner | Decide whether the stream is an internal operator tool (then restrict it at the network layer and say so) or a client-facing API (then authenticate it and filter by the caller's user id). | TBD |
| B2 | **[ACTION NOW]** Operation records live only in Redis with a **5-minute sliding expiry** (`requests.expirySeconds: 300`) and are written nowhere else. Five idle minutes after a write, `GET /operations/{operationId}` returns `404` and the outcome of that write is unrecoverable — indistinguishable from an operation that never existed. Because the gateway runs the asynchronous profile, this is the **only** channel by which a caller learns whether their command succeeded. | Any claim that the asynchronous API is usable; any client that polls, retries, or reconnects more than five minutes after submitting a write; any post-incident investigation of a failed operation. | Service owner / API owner | Decide the required retention window, then either raise `requests.expirySeconds`, persist operations durably (the Mongo configuration and its Vault lease already exist but are unused — see §6), or publish the limit as part of the API contract so clients are built to it. | TBD |
| B3 | **[ACTION NOW]** A single Redis instance now holds both the SignalR backplane and every in-flight operation record, with no durable copy anywhere. A Redis restart loses the outcome of every operation currently in flight across the whole platform. | Production readiness; any availability target for the asynchronous write path. | Platform owner | Decide whether Redis is a cache or a system of record here, and configure persistence, replication, or a durable store accordingly. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should `GET /operations/{operationId}` remain reachable without a token? | It returns the operation's user id, state, and rejection reason to anyone who can guess or observe an id. | It may be intentional so a client can poll before it holds a token, but that reasoning is not written down anywhere. | Security owner |
| Q2 | **[ACTION NOW]** What happens to callers of the asynchronous gateway when this service is unavailable? | Writes keep being accepted and published, but nothing records them, so callers have no way to learn any outcome. | Undefined in the repositories. | Platform owner |
| Q3 | **[handled later by architecture_evolution_generation]** Is processing each message more than once safe? This service has no outbox or inbox, so redelivery is not deduplicated. | Duplicate operation records, or an operation flipping back to `pending` after completing, would mislead every client watching it. | **Answered from source this pass:** yes for terminal states — `TrySetAsync` refuses to modify an operation already `Completed` or `Rejected`. A duplicate arriving while still `pending` rewrites identical values and restarts the expiry clock. | Service owner |
| Q4 | **[ACTION NOW]** Why is MongoDB configured — connection string, database name, and a Vault dynamic-credential lease with auto-renewal — for a service that never opens it? | It implies either a removed feature, an unbuilt one, or a copied configuration block; and it makes the service appear to depend on a datastore it does not need, which distorts capacity and failure-mode planning. | Unknown. The likeliest reading is a configuration block copied from another service, since `allowAnonymousEndpoints` was demonstrably copied the same way. | Service owner |
| Q5 | **[handled later by architecture_evolution_generation]** Should `.AddRedis()` be called twice in `Infrastructure/Extensions.cs`? | It is a visible duplicate registration in the composition root of the platform's most unusual service; harmless or not, it signals that this file is not being read carefully when changed. | Almost certainly an editing artefact. | Service owner |
| Q6 | **[handled later by architecture_evolution_generation]** What is `Pacco.Services.Operations.GrpcClient` for? | It is a full project in the solution with no deployment, no documentation, and no caller. | Probably a demonstration of the gRPC contract. | Service owner |
| Q7 | **[ACTION NOW]** Should the vendored `signalr.js` bundle and the CDN-hosted Bootstrap 4.0.0 be replaced with managed dependencies? | Both are unpinned to any manifest — there is no `package.json` in the workspace — so neither can be patched, audited, or updated through a normal dependency process. | Introduce a dependency manifest for the UI, or delete the UI if it is only a demonstration. | Service owner |
