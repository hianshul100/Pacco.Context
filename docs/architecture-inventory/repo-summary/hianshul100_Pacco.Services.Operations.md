# Repository Summary — `hianshul100_Pacco.Services.Operations`

**Primary name:** `operations-service` (aliases used in this file: `Pacco.Services.Operations.Api` — the .NET project and assembly name; `devmentors/pacco.services.operations` — the published Docker image name). A second, non-deployed project in the same repository is named `Pacco.Services.Operations.GrpcClient`.

**Repository:** `hianshul100_Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

The platform's operation-tracking and client-notification service. Because every mutation in Pacco is fired asynchronously through RabbitMQ, the caller gets no direct answer. This service listens to *every* command, event and rejection on *every* exchange, records the resulting operation state, and pushes it back to the originating user in real time over SignalR. It is the mechanism that makes an asynchronous platform feel synchronous to a client.

It is the most architecturally unusual service in the workspace and the only one with three separate transport surfaces: HTTP, SignalR and gRPC.

## 2. Main runtime / service type

ASP.NET Core 3.1 host (`netcoreapp3.1`) on **Convey** `0.4.*`, serving simultaneously:

- an HTTP API,
- a SignalR hub with a Redis backplane,
- a gRPC service (unary and server-streaming),
- static files for a small browser test page,
- a RabbitMQ consumer whose subscriptions are generated at runtime by reflection.

It does **not** use the `.Api` / `.Application` / `.Core` / `.Infrastructure` layering that the other domain services use — everything lives in `Pacco.Services.Operations.Api`. It has no domain layer, because it has no domain.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Operations.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Operations.Api/Infrastructure/Extensions.cs` |
| Runtime subscription generator | `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` |
| Subscription manifest | `src/Pacco.Services.Operations.Api/messages.json` |
| gRPC contract | `src/Pacco.Services.Operations.Api/Operations.proto` |
| gRPC host | `src/Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs` |
| SignalR hub | `src/Pacco.Services.Operations.Api/Hubs/` (`PaccoHub`) |
| Browser test page | `src/Pacco.Services.Operations.Api/wwwroot/ui/index.html` |
| Standalone gRPC console client | `src/Pacco.Services.Operations.GrpcClient/Program.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Operations.Api.dll` |

`Program.cs` maps three things: `GET operations/{operationId}` (`404` when `IOperationsService.GetAsync` returns null), `endpoints.MapHub<PaccoHub>("/pacco")`, and `endpoints.MapGrpcService<GrpcServiceHost>()`.

## 4. Important modules / packages

Two source projects, enumerated from `Pacco.Services.Operations.sln`:

- `src/Pacco.Services.Operations.Api` — the deployable.
- `src/Pacco.Services.Operations.GrpcClient` — a console application that exercises the gRPC surface. Not containerised, not in `compose/services.yml`, not published as an image. A developer tool.

The most significant single file is **`src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs`**. It reads `messages.json` at start-up and, for every message name listed, uses `System.Reflection.Emit` (`AssemblyBuilder`, `ModuleBuilder`, `TypeBuilder`) to *generate a .NET type at runtime* named after the message string and decorated with `MessageAttribute(exchange, null, null, true)`. It then reflectively invokes `IBusSubscriber.Subscribe` for each, routing everything to three catch-all handlers: `GenericCommandHandler`, `GenericEventHandler` and `GenericRejectedEventHandler` (`src/Pacco.Services.Operations.Api/Handlers/`). This is how one service subscribes to the platform's entire message catalogue without referencing any other service's contracts.

Notable NuGet references beyond the standard Convey stack (`Pacco.Services.Operations.Api.csproj`): `Google.Protobuf 3.11.4`, `Grpc.AspNetCore 2.28.0`, `Grpc.Tools 2.28.1`, `Microsoft.AspNetCore.SignalR 1.1.0`, `Microsoft.AspNetCore.SignalR.Redis 1.1.5`, plus `Convey.Auth`. `Pacco.Services.Operations.GrpcClient.csproj` references `Google.Protobuf 3.11.4`, `Grpc.Net.Client 2.28.0`, `Grpc.Tools 2.28.1`, `Newtonsoft.Json 12.0.3`.

## 5. External integrations

- **RabbitMQ** — subscribes to all eight service exchanges.
- **Redis** — both as the operation store and as the SignalR backplane.
- **MongoDB**, **Consul**, **Fabio**, **Jaeger**, **Prometheus**, **Seq** — through Convey extensions.
- **No Vault.** Unlike the eight other domain services, `operations-service` has no `vault` block. It does not need dynamic database credentials, because it does not persist to MongoDB (see below).
- No outbound HTTP calls to peer services: `httpClient.services` is empty.

## 6. Data stores & state

This is the clearest configuration-versus-code discrepancy inside a single service in the workspace.

- **Redis is the actual store.** `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` takes `IDistributedCache` and `RequestsOptions` in its constructor. `appsettings.json` sets `requests.expirySeconds: 300`, so **operation records live in Redis with a five-minute time to live and are then gone**. Redis key prefix `operations:`.
- **MongoDB is configured but unused.** `appsettings.json` declares `mongo.connectionString: mongodb://localhost:27017`, `mongo.database: operations-service`, `seed: false`, and `AddInfrastructure()` calls `.AddMongo()` — but **no `AddMongoRepository<>` call exists anywhere in the repository**, so no collection is registered and nothing is written. There are no table or collection names to record for this service.
- **No outbox.** Unlike every other messaging service, `operations-service` has no `outbox` configuration and no outbox decorators. It only consumes; it never publishes.
- **Query mechanism:** `IDistributedCache` get/set against Redis. **No ORM, no repository pattern, no migration tool.**
- **Cross-domain coupling:** total, by design. This service holds transient state keyed by identifiers from every other domain (order, parcel, vehicle, resource, delivery, customer, user) and understands none of them. Records are grouped per user via the `ToUserGroup` helper in `Infrastructure/Extensions.cs`, which formats a user identifier as `users:{userId}`. There are no foreign keys.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchanges, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`, registered with `.AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`. Own exchange `operations`, queue template `operations-service/{{exchange}}.{{message}}` — although this service publishes nothing.

**Consumed:** everything in `src/Pacco.Services.Operations.Api/messages.json`. That file is the platform's de facto message catalogue and is reproduced here in full, verbatim.

| Service key | Exchange | Commands | Events | Rejected events |
|---|---|---|---|---|
| `availability-service` | `availability` | `add_resource`, `delete_resource`, `release_resource`, `reserve_resource` | `resource_added`, `resource_deleted`, `resource_reservation_released`, `resource_reservation_canceled`, `resource_reserved` | `add_resource_rejected`, `delete_resource_rejected`, `release_resource_rejected`, `reserve_resource_rejected` |
| `customers-service` | `customers` | `change_customer_state`, `complete_customer_registration` | `customer_created`, `customer_became_vip`, `customer_state_changed` | `change_customer_state_rejected`, `complete_customer_registration_rejected` |
| `deliveries-service` | `deliveries` | `add_delivery_registration`, `complete_delivery`, `fail_delivery`, `start_delivery` | `delivery_completed`, `delivery_failed`, `delivery_started`, `registration_added_to_delivery` | `complete_delivery_rejected`, `fail_delivery_rejected`, `start_delivery_rejected` |
| `identity-service` | `identity` | `sign_in`, `sign_up` | `signed_up`, `signed_in` | `sign_in_rejected`, `sign_up_rejected` |
| `ordermaker-service` | `ordermaker` | — | `make_order_completed` | `make_order_rejected` |
| `orders-service` | `orders` | `add_parcel_to_order`, `approve_order`, `assign_vehicle_to_order`, `cancel_order`, `create_order`, `delete_order`, `delete_parcel_from_order` | `order_approved`, `order_canceled`, `order_completed`, `order_created`, `order_deleted`, `order_delivering`, `parcel_added_to_order`, `parcel_deleted_from_order`, `vehicle_assigned_to_order` | `add_parcel_to_order_rejected`, `approve_order_rejected`, `assign_vehicle_to_order_rejected`, `cancel_order_rejected`, `create_order_rejected`, `delete_order_rejected`, `delete_parcel_from_order_rejected`, `delivering_order_rejected`, `order_for_delivery_not_found`, `order_for_reserved_vehicle_not_found` |
| `parcels-service` | `parcels` | `add_parcel`, `delete_parcel` | `parcel_added`, `parcel_deleted` | `add_parcel_rejected`, `delete_parcel_rejected` |
| `vehicles-service` | `vehicles` | `add_vehicle`, `delete_vehicle`, `update_vehicle` | `vehicle_added`, `vehicle_deleted`, `vehicle_updated` | `add_vehicle_rejected`, `delete_vehicle_rejected`, `update_vehicle_rejected` |

`pricing-service` is absent from `messages.json` — it neither publishes nor consumes messages.

**Published:** none.

**Payload key fields:** this service deliberately does not know them. The generated types are empty shells; the handlers read only the message context (correlation identifier, user identifier, message name) and never the body. The bodies themselves are **unknown — requires runtime capture**.

## 8. APIs exposed & consumed

**HTTP** (`src/Pacco.Services.Operations.Api/Program.cs`):

| Method | Path | Behaviour |
|---|---|---|
| GET | `operations/{operationId}` | `IOperationsService.GetAsync`; `404` when null |

Exposed publicly through `api-gateway` as `GET operations/{operationId}` with **`auth: false`** — the only read route on the platform that requires no token.

**SignalR** — hub at `/pacco` (`endpoints.MapHub<PaccoHub>("/pacco")`), Redis backplane selected by `signalR.backplane: redis` in `appsettings.json` and wired by the private `AddSignalR` helper in `Infrastructure/Extensions.cs`, which falls back to an in-memory hub if the backplane is anything other than `redis`. Client-visible surface, from `src/Pacco.Services.Operations.Api/wwwroot/ui/js/app.js`:

- invoked by the client: `initializeAsync(jwt)`
- pushed to the client: `connected`, `disconnected`, `operation_pending`, `operation_completed`

**gRPC** (`src/Pacco.Services.Operations.Api/Operations.proto`, `syntax = "proto3"`, `package Services.Operations`):

```
service GrpcOperationsService {
  rpc GetOperation(GetOperationRequest) returns (GetOperationResponse);
  rpc SubscribeOperations(Empty) returns (stream GetOperationResponse);
}
```

`GetOperationResponse` fields: `id` (1), `userId` (2), `name` (3), `state` (4), `code` (5), `reason` (6). These six fields are the closest thing the platform has to a documented payload shape.

**Consumes:** nothing over HTTP.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5005`. The browser page hardcodes `http://localhost:5005/pacco` as the hub address.
- Consul at `http://localhost:8500`, ping interval `3`; Fabio at `http://localhost:9999`.
- `UseStaticFiles()` is called, so `wwwroot/` is served.
- Published image `devmentors/pacco.services.operations`.
- Because SignalR uses a Redis backplane, this service can be scaled horizontally without losing client connections — the only service where horizontal scaling is explicitly designed for.

## 10. Security & auth clues

- `.AddJwt()` and `Convey.Auth` are referenced. The `jwt` block in `appsettings.json` carries the **same shared `issuerSigningKey` literal** found in `identity-service` and `api-gateway`, with `issuer: pacco`, `expiryMinutes: 60`, `validateAudience: false`, `validateIssuer: false`, `validateLifetime: true`, `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` (those two paths do not exist on this service — copied template values).
- `certs/localhost.cer` is committed.
- The SignalR hub authenticates by having the client pass a JWT as an argument to `initializeAsync`, not through the standard `Authorization` header or SignalR's built-in `[Authorize]` attribute. The token is typed into a text box on the test page.
- **`GET operations/{operationId}` is exposed with `auth: false` at the gateway.** Operation identifiers are the only protection. See open questions.
- No `security` access-control list, no certificate authentication, **no Vault**.
- **Checked-in credentials:** the JWT signing key and RabbitMQ `guest`/`guest` in `src/Pacco.Services.Operations.Api/appsettings.json`.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: operations`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker. This service sees every message on the platform, so its traces are the broadest.
- **Correlation:** `GetCorrelationContext` in `Infrastructure/Extensions.cs` reads `ICorrelationContextAccessor` and round-trips it through JSON. The correlation identifier is what ties a client's operation record to the message that produced it.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]` and the standard `excludeProperties` redaction list.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` — the decision to generate message types at runtime by reflection rather than share contract assemblies. This single choice is why no service in the platform depends on another service's code.
- `src/Pacco.Services.Operations.Api/messages.json` — the platform's message catalogue, and a runtime dependency, not just documentation. Editing it changes what the service subscribes to.
- `src/Pacco.Services.Operations.Api/Operations.proto` — the gRPC contract.
- `src/Pacco.Services.Operations.Api/Infrastructure/Extensions.cs` — the Redis-backplane SignalR decision and the catch-all handler registrations.
- `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` with `requests.expirySeconds: 300` — the decision that operation state is transient.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- **MongoDB is configured and `.AddMongo()` is called, but nothing uses it.** Whether persistence was planned and dropped in favour of Redis, or Redis is the cache in front of an unimplemented store, is **Unknown**. **Needs validation.**
- **Operation records expire after 300 seconds.** A client disconnected for more than five minutes can never learn the outcome of its request. Whether that is an accepted trade-off is **Unknown**.
- `GET operations/{operationId}` is unauthenticated at the gateway while `GetOperationResponse` includes `userId`. Whether operation identifiers are unguessable enough to be the sole control is **Unknown**. **Needs validation.**
- `src/Pacco.Services.Operations.GrpcClient` is not deployed anywhere. Whether it is a test harness or an abandoned client is **Unknown**.
- `messages.json` is maintained by hand. Nothing verifies it against the messages the other services actually publish, so drift is undetectable. Whether any process keeps it in step is **Unknown**.
- The `MessageAttribute(exchange, null, null, true)` arguments were not decoded against the Convey source; the meaning of the third and fourth positions is **Unknown**.

## 14. Frontend stack

**Frontend assets are present** — the only frontend in the entire workspace.

**Location:** `src/Pacco.Services.Operations.Api/wwwroot/ui/`, served by `UseStaticFiles()`.

| File | What it is |
|---|---|
| `wwwroot/ui/index.html` | Single page, title `Pacoo SignalR` (spelling as in the file). Bootstrap `4.0.0` loaded from the maxcdn CDN. A text input for a JWT and a Connect button. |
| `wwwroot/ui/js/app.js` | Hand-written vanilla JavaScript in an immediately-invoked function expression. `new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco')`, then `connection.invoke('initializeAsync', jwt)`, then listeners for `connected`, `disconnected`, `operation_pending`, `operation_completed`. |
| `wwwroot/ui/js/signalr.js` | A vendored webpack UMD bundle of the ASP.NET Core SignalR JavaScript client, `VERSION = "1.1.0"`. |

**Stack summary:** no framework, no bundler, no package manager, no TypeScript. There is **no `package.json` anywhere in the workspace**, no npm or yarn lockfile, no webpack or Vite configuration of its own, and no micro-frontend or module-federation setup. The hub URL is hardcoded to `localhost:5005`, so the page only works against a local run. This is a developer test harness, not a product user interface.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Operations` directory". | No such directory. The runnable project is `src/Pacco.Services.Operations.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5005`." | `appsettings.json` sets the Consul port to `5005`, and `wwwroot/ui/js/app.js` hardcodes the same. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.operations`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Operations.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README is the shared platform template. It describes an HTTP microservice and mentions nothing else. | The service also runs a SignalR hub, a gRPC service and a static web page, generates its RabbitMQ subscriptions by reflection, and stores state in Redis for five minutes. | **Docs gap — severe.** Four of the five things that make this service architecturally interesting are undocumented. |

**Disk-only components with no documentation anywhere:** the SignalR hub and its browser client, the gRPC service and `Operations.GrpcClient`, the reflection-based subscription generator, and `messages.json` as a runtime input.

**Docs-only claims:** none — the README makes no claim this repository fails to satisfy beyond the `dotnet run` path.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `messages.json` is an accurate list of the messages the other eight services actually publish. | It is read at runtime to build subscriptions, so a wrong entry would silently drop notifications; it is the only place a full catalogue exists. | The platform event catalogue in the consolidated inventory would inherit any drift in this file. | Compare against the event classes in each service's `.Application/Events/` directory. |
| A2 | Operation records are stored only in Redis, not MongoDB. | `OperationsService` takes `IDistributedCache`, and no `AddMongoRepository<>` call exists in the repository. | Any statement about where operation history lives, and for how long, would be wrong. | Read `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` in full and check Redis at runtime. |
| A3 | `src/Pacco.Services.Operations.GrpcClient` is a developer tool, not a deployed component. | It is a console application, is absent from `compose/services.yml`, and has no `Dockerfile` of its own. | A deployable would be missing from the platform inventory. | Confirm with the service owner. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is a 300-second expiry on operation records acceptable, given this is the only way a client learns whether its request succeeded? | Every mutation on the platform is asynchronous. If the client is away for more than five minutes, the outcome is unrecoverable. | It is likely a deliberate trade-off for a demonstration platform, but it should be a recorded decision rather than a default. | Platform architect |
| Q2 | **[ACTION NOW]** Should `GET operations/{operationId}` stay unauthenticated at the gateway when the response includes `userId`? | It is the only public read route with `auth: false`, and it returns data attributable to a user. | Guard it with the same JWT check as every other read route. | Security owner |
| Q3 | **[handled later by the platform inventory review]** Why is MongoDB configured and initialised here but never used? | It makes the service look like it persists data when it does not, which misleads anyone reading the configuration. | Leftover from an earlier design; the `mongo` block and `.AddMongo()` call could be removed. | Service owner |
| Q4 | **[handled later by the platform inventory review]** What keeps `messages.json` in step with the events the other services actually publish? | Nothing detects drift today. A renamed event silently stops reaching clients. | A build-time check against each service's event classes. | Platform architect |
