# Repository summary — `hianshul100_Pacco.Services.Operations`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Operations` (also known as: Pacco.Services.Operations, the Operations service)
**Deployable:** `Pacco.Services.Operations.Api` (also known as: `operations-service` — its Consul service name, container name and compose service name — and `devmentors/pacco.services.operations`, its published image). Repository: `hianshul100_Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.Api`.
**Second project (not deployed):** `Pacco.Services.Operations.GrpcClient` — a console gRPC client. Repository: `hianshul100_Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.GrpcClient`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

It answers the question every asynchronous command raises: *did my request actually go through?* The gateway accepts a command, returns a correlation identifier and publishes to RabbitMQ. This service listens to every message on the platform, tracks each correlation identifier as an "operation" moving through pending → completed or rejected, and pushes that status to the originating user over a live connection. It is the platform's callback channel and its only cross-cutting message listener.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, exposing four surfaces at once: a small REST API, a **SignalR** hub, a **gRPC** service (unary plus server streaming), and a static browser page. It is the only service in the platform doing anything other than plain HTTP and RabbitMQ.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Operations.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Operations.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Operations.Api.dll` |
| Composition | `src/Pacco.Services.Operations.Api/Infrastructure/Extensions.cs` |
| Subscription builder | `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` |
| Message catalogue | `src/Pacco.Services.Operations.Api/messages.json` |
| SignalR hub | `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs` → mapped at `/pacco` |
| gRPC contract | `src/Pacco.Services.Operations.Api/Operations.proto` (duplicated at `src/Pacco.Services.Operations.GrpcClient/Operations.proto`) |
| Client console | `src/Pacco.Services.Operations.GrpcClient/Program.cs` |
| Browser page | `src/Pacco.Services.Operations.Api/wwwroot/ui/index.html` |
| Proto regeneration | `scripts/proto/{lin,mac,win}-compile.sh` |
| Request collection | `Pacco.Services.Operations.rest` |

HTTP routes, registered with `UseEndpoints` rather than the CQRS dispatcher:

| Method | Route | Behaviour |
|---|---|---|
| GET | `` (root) | writes the configured `app.name` |
| GET | `operations/{operationId}` | returns the operation, or 404 |

## 4. Important modules / packages

Two projects in `Pacco.Services.Operations.sln`:

- `src/Pacco.Services.Operations.Api/Pacco.Services.Operations.Api.csproj` — the service.
- `src/Pacco.Services.Operations.GrpcClient/Pacco.Services.Operations.GrpcClient.csproj` — a `netcoreapp3.1` console executable with a two-option menu: fetch one operation by identifier, or subscribe to the operations stream. It is a developer tool; nothing deploys it.

Notable code in the Api project:

- `Infrastructure/Subscriptions.cs` — the most unusual file in the platform. It reads `messages.json`, and for every message name **emits a .NET type at runtime** with `AssemblyBuilder`/`ModuleBuilder`, decorates it with `[Message(exchange, null, null, true)]`, then subscribes to it by reflection. No message class is declared in source.
- `Handlers/GenericCommandHandler.cs`, `GenericEventHandler.cs`, `GenericRejectedEventHandler.cs` — three handlers cover every message on the platform.
- `Handlers/Extensions.cs` — `GetSagaState()`, which reads the `Saga` header (`pending`/`completed`/`rejected`) written by the OrderMaker saga.
- `Services/OperationsService.cs` — the operation store, over `IDistributedCache`.
- `Services/HubService.cs`, `HubWrapper.cs`, `Hubs/PaccoHub.cs` — the SignalR side.
- `Infrastructure/GrpcServiceHost.cs` — the gRPC side.
- `Types/{Command,Event,RejectedEvent,IMessage,OperationState,SignalrOptions}.cs`, `DTO/OperationDto.cs`, `Queries/GetOperation.cs`, `Infrastructure/{CorrelationContext,ExceptionToResponseMapper,RequestsOptions}.cs`.

Packages: the usual Convey `0.4.*` set (Auth, Secrets.Vault, CQRS.Queries, Discovery.Consul, LoadBalancing.Fabio, HTTP, Logging, MessageBrokers(.CQRS/.RabbitMQ), Metrics.AppMetrics, Persistence.MongoDB, Persistence.Redis, Security, Tracing.Jaeger(.RabbitMQ), WebApi.CQRS, WebApi.Swagger) plus `Google.Protobuf 3.11.4`, `Grpc.AspNetCore 2.28.0`, `Grpc.Tools 2.28.1`, `Microsoft.AspNetCore.SignalR 1.1.0`, `Microsoft.AspNetCore.SignalR.Redis 1.1.5`.

## 5. External integrations

RabbitMQ (its own exchange `operations`, plus subscriptions on eight other exchanges), Redis (as both operation store and SignalR backplane), Consul (`operations-service`, port 5005, ping endpoint `ping`), Fabio, Vault, Jaeger (service name `operations`), Prometheus, Seq. MongoDB is configured and registered but never used — see section 6.

## 6. Data stores and state handling

**Store:** Redis, reached through `IDistributedCache`. Key pattern `requests:{correlationId}`, value a JSON `OperationDto { Id, UserId, Name, State, Code, Reason }`, written with a **sliding expiration of `requests.expirySeconds` = 300 seconds**. The Redis instance prefix is `operations:`, so keys land as `operations:requests:{id}`.

`TrySetAsync` refuses to overwrite an operation already in `Completed` or `Rejected` state, which is what stops a late message from resurrecting a finished operation.

**ORM:** none. **Migration tool:** none. **Tables/collections:** none — the operation is a JSON string under one key.

**Mongo is dead weight.** `AddMongo()` is called in `Infrastructure/Extensions.cs` and `appsettings.json` carries a full `mongo` block (`database: operations-service`) plus a Vault dynamic-credential lease for it, yet no document class, repository or Mongo call exists anywhere in the repository. The service declares a database dependency it does not use.

**Cross-domain coupling.** There is no foreign key and no replicated aggregate. The coupling here is of a different kind and is much stronger: `messages.json` is a **duplicate of eight other services' message catalogues**, listing 24 commands, 29 events and 27 rejected events across `availability-service`, `customers-service`, `deliveries-service`, `identity-service`, `ordermaker-service`, `orders-service`, `parcels-service` and `vehicles-service`. Every message rename anywhere on the platform must be mirrored here by hand, and nothing enforces that.

**Consequence of the 300-second expiry:** an operation that takes longer than five minutes without an update disappears. A client polling `GET /operations/{id}` afterwards receives a 404 and cannot tell "never existed" from "expired".

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Jaeger plugin (`AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`). **No outbox** — this service only consumes.

**Broker settings:** own exchange `operations`, `conventionsCasing: snakeCase`, queue name template `operations-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`.

**Published:** none. Its output goes over SignalR and gRPC, not the bus.

**Consumed:** everything listed in `messages.json`, subscribed dynamically. The catalogue on disk:

| Service section | Exchange | Commands | Events | Rejected events |
|---|---|---|---|---|
| `availability-service` | `availability` | 4 | 5 | 4 |
| `customers-service` | `customers` | 2 | 3 | 2 |
| `deliveries-service` | `deliveries` | 4 | 4 | 3 |
| `identity-service` | `identity` | 2 | 2 | 2 |
| `ordermaker-service` | `ordermaker` | 0 | 1 | 1 |
| `orders-service` | `orders` | 7 | 9 | 10 |
| `parcels-service` | `parcels` | 2 | 2 | 2 |
| `vehicles-service` | `vehicles` | 3 | 3 | 3 |
| **Total** | | **24** | **29** | **27** |

**Payload key fields:** the generated types carry **no properties at all** — `Subscriptions.BindMessages` creates an empty subclass of `Command`, `Event` or `RejectedEvent` per message name. This service therefore never reads a message body. Everything it needs comes from the message envelope: `CorrelationId` (the operation identifier), the `message_context` header (deserialised into `CorrelationContext` for `Name` and `User.Id`), and the `Saga` header for state.

**How state is decided**, from `GenericEventHandler.HandleAsync`: no correlation identifier means the message is dropped; the operation name is the correlation context's `Name`, falling back to the CLR type name; the state is `GetSagaState()` if the `Saga` header is present, and **`Completed` otherwise**. So an ordinary event without a saga header immediately completes the operation. `GenericRejectedEventHandler` and `GenericCommandHandler` follow the same shape for the rejected and pending cases.

**Client-facing message names** pushed over SignalR: `connected`, `disconnected`, `operation_pending`, `operation_completed`, `operation_rejected`. Payload fields: `id`, `name` on pending and completed; `id`, `name`, `code`, `reason` on rejected.

## 8. APIs exposed and consumed

**HTTP:** the two routes in section 3, plus Swagger at `docs`, `ping` for Consul, and the static page at `/ui/index.html`. Local base URL `http://localhost:5005`; port 80 in the container.

**Through the gateway:** `GET operations/{operationId}` is routed in all four Ntrada profiles (`ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml`) to `downstream: operations-service/operations/{operationId}`. The SignalR hub and the gRPC service are **not** routed through the gateway.

**SignalR:** hub at `/pacco`. The client calls `initializeAsync(token)`; the hub validates the JWT with `IJwtHandler.GetTokenPayload`, and on success adds the connection to group `users:{subject}` (`ToUserGroup`, identifier formatted `"N"`) and emits `connected`; on any failure it emits `disconnected`. Messages are then sent per-user with `Clients.Group(...)`.

**gRPC** (`Operations.proto`, package `Services.Operations`, service `GrpcOperationsService`):

| RPC | Request | Response |
|---|---|---|
| `GetOperation` | `GetOperationRequest { string id = 1 }` | `GetOperationResponse { id, userId, name, state, code, reason }` |
| `SubscribeOperations` | `Empty` | `stream GetOperationResponse` |

`SubscribeOperations` is fed by a `BlockingCollection<OperationDto>` filled from the `IOperationsService.OperationUpdated` event.

**Consumed:** none. `httpClient.services` is empty — this service calls nobody.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes only `src/Pacco.Services.Operations.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- `compose/services.yml`: host port `5005:80`, and `depends_on` all eight other services — the longest dependency list in the platform.
- `launchSettings.json` exposes two local addresses: `http://localhost:5005` and `https://localhost:50050`. The gRPC client defaults to `https://localhost:50050`.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.operations`.
- `scripts/proto/*-compile.sh` expect a vendored `tools/Grpc.Tools.1.22.0/` directory that is **not in the repository**, and pin protoc 1.22.0 while the csproj uses `Grpc.Tools 2.28.1`.

## 10. Security and auth clues

- `Convey.Auth` with `AddJwt()`; `jwt.certificate.location: certs/localhost.cer`; `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` — routes this service does not have, copied from the Identity service's configuration.
- **The JWT signing key is committed in plain text** in `appsettings.json` and again in `appsettings.docker.json`: `issuerSigningKey` set to the same 80-character literal that appears in the Identity service and the API gateway. Anyone with the repository can mint a token the whole platform accepts.
- Vault is configured (kv v2 at `operations-service/settings`, PKI role `operations-service`, common name `operations-service.pacco.io`, dynamic Mongo lease) and `Program.cs` calls `.UseVault()`, but `appsettings.docker.json` disables Vault entirely — the composed stack runs on the committed key.
- **`GET /operations/{operationId}` performs no authorisation.** It returns any operation to any caller, including `UserId`, operation name and rejection reason. Correlation identifiers are guessable only insofar as they are GUIDs, but they are also handed to the client by the gateway and travel in headers.
- The SignalR hub is the one place that does check identity properly, and it isolates users into per-user groups.
- `logger.excludeProperties` masks secret-like values; `httpClient.requestMasking` is enabled with `maskTemplate: "*****"`.

## 11. Observability, logging and tracing clues

Jaeger tracing enabled with service name `operations`, including the RabbitMQ plugin so consumed messages join the trace. Prometheus metrics via `Convey.Metrics.AppMetrics`. Structured logs to console, file and Seq; ELK present but disabled. `logger.excludePaths` and `jaeger.excludePaths` both drop `/`, `/ping` and `/metrics`.

This service is itself an observability tool — it is how a user sees what happened to their command — but it does not persist anything for longer than five minutes, so it is not an audit trail.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` records the decision to build subscriptions from configuration rather than code. `src/Pacco.Services.Operations.Api/messages.json` is the resulting platform-wide message catalogue and the closest thing Pacco has to a single message contract document. `src/Pacco.Services.Operations.Api/Handlers/GenericEventHandler.cs` defines what "pending", "completed" and "rejected" mean. `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` sets the retention rule. `src/Pacco.Services.Operations.Api/Infrastructure/Extensions.cs` records the technology choices, including the SignalR backplane switch.

**Feature flag system: none.** No flag library, no flag store, no flag keys. The switches available are `ASPNETCORE_ENVIRONMENT`, the Convey `enabled` booleans, and one genuine behaviour switch — `signalR.backplane`, which is `"redis"` in every configuration file and falls back to in-process SignalR for any other value.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**Frontend assets are present**, in `src/Pacco.Services.Operations.Api/wwwroot/ui/` — the only frontend in the whole platform.

- `index.html` — a plain HTML page, no build step, titled `Pacoo SignalR` (spelled that way on disk). Styling from **Bootstrap 4.0.0** loaded off `maxcdn.bootstrapcdn.com` with an SRI integrity hash.
- `js/app.js` — 45 lines of plain ES5/ES6, no framework. Builds a `signalR.HubConnectionBuilder` against a **hard-coded `http://localhost:5005/pacco`**, takes a JWT from a text input, calls `initializeAsync`, and appends each of the five hub messages to a list.
- `js/signalr.js` — the vendored, unminified SignalR JavaScript client (about 4,000 lines), committed rather than installed.

No `package.json`, no bundler, no transpiler, no test runner, no lock file, no CSS of its own. Static files are served by `UseStaticFiles()`; there is no default-document middleware, so the page must be requested as `/ui/index.html`. Because the hub URL is hard-coded to localhost, the page is unusable in the composed stack as shipped. This is a developer demonstration harness, not a product frontend.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5005`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.operations`; `Pacco.Services.Operations.rest` lists the HTTP requests (it lists exactly one, `GET /operations/{operationId}`).

**Present on disk but absent from the README:** essentially the entire service. The README is the shared nine-service template and never mentions SignalR, gRPC, the browser page, the dynamic subscription mechanism, `messages.json`, the five-minute retention, or what an "operation" is. A reader of the documentation alone would not learn that this service exists for a reason different from all the others.

**Conflicts to surface:**

- **Stale doc.** The README says to run `dotnet run` in `/src/Pacco.Services.Operations`. That directory does not exist; the project is `src/Pacco.Services.Operations.Api`, as `scripts/start.sh` correctly shows. The same stale path appears in the other services' READMEs.
- **Configuration conflict.** `appsettings.json` configures MongoDB and a Vault dynamic Mongo credential lease, and `Extensions.cs` calls `AddMongo()`, but no code in the repository touches Mongo. The declared dependency is not real.
- **Configuration conflict.** `jwt.allowAnonymousEndpoints` lists `/sign-in` and `/sign-up`. Neither route exists here; both belong to the Identity service.
- **Duplicate registration.** `Extensions.cs` calls `.AddRedis()` twice in the same chain. Harmless in effect, but it signals the composition was assembled without review.
- **Reachability conflict.** The gateway routes only `GET operations/{operationId}`. The SignalR hub at `/pacco` — the mechanism the whole service exists to provide — has no gateway route, so a browser must reach `operations-service` directly.
- **gRPC reachability conflict.** `launchSettings.json` binds `https://localhost:50050` for local development, but the `Dockerfile` sets `ASPNETCORE_URLS http://*:80` only and `compose/services.yml` maps just `5005:80`. In the composed stack there is no HTTPS endpoint and no published gRPC port, while the client hard-codes `https://…:50050`. gRPC works on a developer machine and not in Docker.
- **Tooling conflict.** `scripts/proto/*-compile.sh` reference `tools/Grpc.Tools.1.22.0/`, which is not in the repository, and pin a protoc version eight minor releases behind the `Grpc.Tools 2.28.1` package the projects actually build with.
- **Catalogue conflict.** `messages.json` has an `ordermaker-service` section with one event and one rejected event but **no commands**, so `make_order` is not tracked. It also carries entries that no service publishes and omits classes that exist on disk in other repositories — the mismatches are recorded in the Availability, Deliveries, Orders and Identity summaries.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, but this repository contains **no test project**. `Pacco.Services.Operations.sln` lists only the Api and GrpcClient projects.
- **Duplicated contract.** `Operations.proto` exists twice, byte-for-byte, once per project, kept in step only by the compile scripts.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next. The shared plaintext JWT key (B1) and the unauthorised operations endpoint (B2) are the items that matter most here.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | `messages.json` is meant to be the authoritative platform message catalogue. | It is the only file listing every message across all eight services, and this service builds live subscriptions from it. |
| A2 | The five-minute expiry is a deliberate choice for a status channel, not an oversight. | `requests.expirySeconds: 300` appears identically in every environment file, and the value is read through a dedicated options class. |
| A3 | `wwwroot/ui/` is a demonstration harness rather than a shipped frontend. | The hub URL is hard-coded to localhost, there is no build tooling, and the page title is misspelled. |
| A4 | The GrpcClient project is a developer tool. | Nothing builds, publishes or composes it, and it is an interactive console menu. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** The JWT signing key is committed in plain text in `appsettings.json` and `appsettings.docker.json`, and is the same key used by the Identity service and the gateway. | Anyone with repository access can forge a token the entire platform accepts. | Security-architecture owner, with the Identity, gateway and Operations service maintainers together — the key must be rotated in all three places at once. |
| B2 | **[ACTION NOW]** `GET /operations/{operationId}` requires no authentication and returns another user's operation, including `UserId` and rejection reason. | Anyone who learns or guesses a correlation identifier reads another user's activity. | Operations service maintainer, with the security-architecture owner. |
| B3 | **[ACTION NOW]** MongoDB is configured, registered and leased from Vault but never used. | The service and its Vault policy claim a database dependency that does not exist, which misleads anyone reasoning about failure modes or access. | Operations service maintainer. |
| B4 | **[ACTION NOW]** No test project exists, while CI runs `scripts/test.sh`. | The dynamic subscription code — reflection-generated types bound by name — is the most fragile code in the platform and nothing verifies it. | Operations service maintainer. |
| B5 | **[ACTION NOW]** In the composed stack there is no HTTPS binding and no published gRPC port, so `Operations.proto` cannot be consumed as deployed. | A published interface is unreachable outside a developer machine. | Operations service maintainer, with the deployment owner. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** How should `messages.json` be kept in step with the services it mirrors? | It is maintained by hand and already disagrees with several services' code; a stale entry silently stops an operation from ever completing. | Operations service maintainer, with each service maintainer. |
| Q2 | **[ACTION NOW]** Should the SignalR hub be reachable through the API gateway? | Today a browser must connect directly to `operations-service`, which the gateway's CORS and auth settings do not cover. | Gateway maintainer with the Operations service maintainer. |
| Q3 | **[ACTION NOW]** Is five minutes long enough for an operation to survive? | An order-making saga that stalls loses its status record, and the client can no longer distinguish "failed" from "never happened". | Operations service maintainer with the OrderMaker service maintainer. |
| Q4 | **[handled later by the domain-model stage]** Should an event without a `Saga` header really mean the operation is complete? | That default decides the outcome for most messages on the platform, and a multi-step flow can be reported complete after its first step. | Domain-model stage. |
| Q5 | **[handled later by the integration-architecture stage]** Should this service persist operations rather than cache them? | If operation history is ever needed for support or audit, Redis with a sliding expiry cannot provide it. | Integration-architecture stage. |
| Q6 | **[handled later by the deployment-architecture stage]** Should `wwwroot/ui/` ship in the production image? | It is served publicly, hard-codes a localhost address, and takes a raw JWT in a text box. | Deployment-architecture stage. |
