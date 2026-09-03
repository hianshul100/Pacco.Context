---
title: "Repository Summary — Pacco.Services.Operations"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Operations"
status: "evidence-based inventory"
---

# Pacco.Services.Operations

**Primary name:** `Pacco.Services.Operations` (aliases used in this file: `operations-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `operations` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Operations`, path: `hianshul100_Pacco.Services.Operations/`
Deployable projects: `Pacco.Services.Operations.Api` (`src/Pacco.Services.Operations.Api/`) and `Pacco.Services.Operations.GrpcClient` (`src/Pacco.Services.Operations.GrpcClient/`).

---

## 1. Primary purpose

The platform's operation tracker and push-notification hub. Because every write in Pacco is asynchronous, a client that submits a command has no immediate result. This service subscribes to **every** command, event and rejected event across the platform, records each as an operation with a state, and pushes state changes to connected browsers over SignalR. It also exposes the same data over gRPC.

Evidence: `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs`, `src/Pacco.Services.Operations.Api/messages.json`, `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs`, `src/Pacco.Services.Operations.Api/Operations.proto`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` process serving **three protocols at once**: HTTP (Convey dispatcher endpoint), SignalR WebSocket hub, and gRPC. It is also a RabbitMQ subscriber and serves static browser files. Listens on `5005`.

`Pacco.Services.Operations.GrpcClient` is a separate console application used to exercise the gRPC surface.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` — dispatcher endpoint plus a second `UseEndpoints` block mapping `MapHub<PaccoHub>("/pacco")` and `MapGrpcService<GrpcServiceHost>()` | `src/Pacco.Services.Operations.Api/Program.cs` |
| gRPC console client `Main` | `src/Pacco.Services.Operations.GrpcClient/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |
| Protocol buffer compilation helpers | `src/Pacco.Services.Operations.Api/scripts/proto/lin-compile.sh`, `mac-compile.sh`, `win-compile.sh` |

## 4. Modules / packages

**Only two projects, and no clean-architecture split** — this repository does not follow the `.Api` / `.Application` / `.Core` / `.Infrastructure` layering used by the other services.

`Pacco.Services.Operations.Api` folders: `Hubs/PaccoHub.cs`; `Infrastructure/` (`CorrelationContext.cs`, `ExceptionToResponseMapper.cs`, `Extensions.cs`, `GrpcServiceHost.cs`, `RequestsOptions.cs`, `Subscriptions.cs`); `Handlers/` (`Extensions.cs`, `GenericCommandHandler.cs`, `GenericEventHandler.cs`, `GenericRejectedEventHandler.cs`); `Services/` (`HubService.cs`, `HubWrapper.cs`, `IHubService.cs`, `IHubWrapper.cs`, `IOperationsService.cs`, `OperationsService.cs`, `OperationUpdatedEventArgs.cs`); `Types/` (`Command.cs`, `Event.cs`, `IMessage.cs`, `OperationState.cs`, `RejectedEvent.cs`, `SignalrOptions.cs`); `Queries/GetOperation.cs`; `DTO/OperationDto.cs`; `Operations.proto`; `messages.json`; `wwwroot/ui/`.

Packages beyond the platform's Convey set: `Google.Protobuf 3.11.4`, `Grpc.AspNetCore 2.28.0`, `Grpc.Tools 2.28.1`, `Microsoft.AspNetCore.SignalR 1.1.0`, `Microsoft.AspNetCore.SignalR.Redis 1.1.5`. The gRPC client project uses `Grpc.Net.Client 2.28.0` and `Newtonsoft.Json 12.0.3`.

**Notable absence:** this service references no `Convey.MessageBrokers.Outbox` package. It is the only RabbitMQ subscriber in the platform without the outbox and inbox pattern.

## 5. External integrations

MongoDB, RabbitMQ, Redis (both as cache and as the SignalR backplane), Consul, Fabio, Vault, Jaeger, Prometheus. It subscribes to the exchanges of every other service.

## 6. Data stores / state

- **Store:** MongoDB, database `operations-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`.
- **Collections:** an operations collection holding one record per tracked message. No `inbox` or `outbox` collections, because the outbox package is not referenced.
- **Migration tool:** none anywhere in the repository.
- **Cross-domain coupling:** by design this service stores identifiers belonging to every other domain — user identifier, operation name, state, code and reason. There are no foreign keys; the coupling is to the *message names* in `messages.json`, which must stay in step with the other repositories.
- **Redis:** used twice — as the general cache and as the SignalR backplane (`signalR: {"backplane": "redis"}`), which is what allows more than one instance of this service to serve WebSocket clients.

## 7. Messaging / async / events

**System:** RabbitMQ. Own topic exchange `operations`, message context header `message_context`, span context header `span_context`.

**What makes this service unusual:** `Infrastructure/Subscriptions.cs` reads `messages.json` at start-up and **generates .NET types at runtime** using `AssemblyBuilder`, `ModuleBuilder` and `TypeBuilder`, applying `MessageAttribute(exchange, null, null, true)` to each generated type, then calls `subscriber.Subscribe<T>` for every command, event and rejected event. It therefore has no compiled message classes of its own and needs no code change when another service adds a message — only a `messages.json` edit.

**`messages.json` is the platform's message catalogue.** It is the single most valuable architecture artefact in the workspace. Contents, verbatim:

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

`pricing-service` has **no entry**, because it is query-only and publishes nothing.

**Payload key fields:** the generated types carry only what `Subscriptions.cs` needs to correlate an operation — the correlation context and the message name. The originating services own the real payload fields, listed in their own summaries. The exact serialised shape received here is **unknown — requires runtime capture**.

**Events pushed to browsers** (`wwwroot/ui/js/app.js`): `connected`, `disconnected`, `operation_pending`, `operation_completed`, `operation_rejected`.

## 8. APIs exposed / consumed

**HTTP:** `GET operations/{operationId}` → `GetOperation` / `OperationDto`. The gateway exposes this route with `auth: false`.

**SignalR:** hub at `/pacco`; the client calls the hub method `initializeAsync` passing a JWT.

**gRPC** (`Operations.proto`, package `Services.Operations`, service `GrpcOperationsService`):

```protobuf
rpc GetOperation (GetOperationRequest) returns (GetOperationResponse) {}
rpc SubscribeOperations (Empty) returns (stream GetOperationResponse) {}
```

`GetOperationRequest` carries `string id = 1`. `GetOperationResponse` carries `string id = 1`, `string userId = 2`, `string name = 3`, `string state = 4`, `string code = 5`, `string reason = 6`.

**Consumed:** the RabbitMQ exchanges of every other service. No outbound HTTP service clients are defined.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.operations`, published `5005:80` in `hianshul100_Pacco/compose/services.yml`, `restart: unless-stopped`, network `pacco`, and it is the only service with a `depends_on: availability-service` entry. Consul registration on port `5005`. The gRPC client defaults to `localhost:50050`.

CI: `.travis.yml` runs the build and dockerize scripts.

## 10. Security / auth clues

- JWT settings: `expiryMinutes: 60`, `issuer: pacco`, `validateIssuer: false`, `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`. **Those two endpoints do not exist in this service** — the block is a copied artefact from `Pacco.Services.Identity`. **Stale doc / copied configuration.**
- The SignalR hub authenticates by receiving a JWT through the `initializeAsync` hub call rather than through a standard header.
- Vault KV path `operations-service/settings`.

**Security findings** (reported, not remediated in this stage):

1. `Pacco.Services.Operations.GrpcClient` uses `HttpClientHandler.DangerousAcceptAnyServerCertificateValidator`, disabling server certificate checking. The source comment states it is for local development only.
2. The `GET operations/{operationId}` route is exposed publicly with `auth: false` at the gateway, so operation status — including a user identifier and a failure reason — is readable without a token.
3. The browser page under `wwwroot/ui/` asks the user to paste a raw JWT into a text box.

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: operations`, including RabbitMQ span propagation; structured logging via `Convey.Logging`; Prometheus metrics via `Convey.Metrics.AppMetrics`; `Infrastructure/CorrelationContext.cs` carries the correlation identifier that ties an operation back to the originating request.

In practice this service is itself an observability tool: it turns the platform's message flow into a queryable, pushable operation record.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` — runtime type generation driven by a JSON file; the single most consequential design decision in this repository.
- `src/Pacco.Services.Operations.Api/messages.json` — the platform-wide message contract.
- `src/Pacco.Services.Operations.Api/Handlers/GenericCommandHandler.cs`, `GenericEventHandler.cs`, `GenericRejectedEventHandler.cs` — one handler per message category instead of one per message.
- `src/Pacco.Services.Operations.Api/Operations.proto` — the gRPC contract.
- `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs` and `Services/HubService.cs` — the push model.
- `src/Pacco.Services.Operations.Api/appsettings.json` — the Redis backplane choice, which is what makes the hub horizontally scalable.

**Feature-flag system: none.** No flag provider package is referenced. `messages.json` acts as a configuration-driven subscription list, which is close to a switchboard but is a message registry, not a feature-flag store. The remaining switches are per-integration `enabled` booleans. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

**This is the only repository in the entire platform that contains frontend assets.**

Location: `src/Pacco.Services.Operations.Api/wwwroot/ui/`.

| File | What it is |
|---|---|
| `wwwroot/ui/index.html` | Static page titled "Pacoo SignalR". Styling from Bootstrap 4.0.0 loaded over a CDN. Contains a JWT text input, a Connect button and a `<ul id="messages">` list. Loads `js/signalr.js` then `js/app.js`. |
| `wwwroot/ui/js/app.js` | Hand-written vanilla JavaScript in an immediately-invoked function. Builds `new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco')`, calls `connection.invoke('initializeAsync', $jwt.value)`, and listens for `connected`, `disconnected`, `operation_pending`, `operation_completed`, `operation_rejected`. |
| `wwwroot/ui/js/signalr.js` | A vendored, pre-built webpack bundle of the SignalR JavaScript client (`webpackUniversalModuleDefinition`), 180,968 bytes, committed as-is. |

**Stack assessment:** no framework (no React, Angular, Vue or Svelte), no `package.json`, no bundler or build step, no TypeScript, no test setup, no module federation or micro-frontend arrangement. The hub URL is hard-coded to `http://localhost:5005`, so the page works only when the service is reached on that address. This is a developer diagnostic page, not a product user interface.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| The platform README describes clean architecture with four layers per service | This service has two projects and no layering at all | Stale doc |
| The platform README describes reliable messaging with the outbox pattern | This service subscribes to every exchange without the outbox or inbox packages | Needs validation — the one service that must not miss a message has the weakest delivery guarantee |
| The platform is described as event-driven with services publishing to their own exchange | Confirmed, and this service is the only cross-cutting subscriber | Confirmed |
| The token settings list anonymous endpoints `/sign-in` and `/sign-up` | Neither endpoint exists in this service | Stale doc — copied configuration |
| The gateway exposes operation status | Confirmed, but with authentication switched off | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the gRPC contract and its console client, the browser diagnostic page under `wwwroot/ui/`, and the runtime type generation in `Subscriptions.cs` — all present on disk and none described in the platform README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The message list file is the platform's agreed contract, and each service is expected to keep it up to date. | It is the only place where every service's messages are written down together, and this service subscribes from it directly. |
| A2 | The browser page under the static files folder is a developer aid, not a product screen. | It has no build tooling, a hard-coded local address, and it asks the user to paste a raw token. |
| A3 | Redis is required, not optional, for this service. | Without the shared backplane, two instances could not push updates to the same connected browsers. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | The message list file is already out of step with the code: one delivery rejection message exists in the deliveries service but is missing here, so that failure would never be reported to a waiting client. | **[ACTION NOW]** Recorded here for the requesting team; this stage does not change any source repository. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Is there a real user interface for the platform anywhere, or is this diagnostic page the only browser experience? | **[ACTION NOW]** Confirm with the requesting team; the empty `Pacco.Web` repository suggests one was planned. |
| Q2 | Should this service use the same reliable-delivery mechanism as the others? | **[handled later by the ADR authoring stage]** Record whether missing an operation update is acceptable. |
| Q3 | Who is the intended consumer of the gRPC interface? Only a sample console client uses it in this workspace. | **[handled later by the ADR authoring stage]** Record the purpose of the gRPC surface, or note that it is unused. |
| Q4 | Why can anyone read an operation's status, including the user it belongs to, without signing in? | **[ACTION NOW]** Confirm with the requesting team whether this is deliberate for the demo. |
