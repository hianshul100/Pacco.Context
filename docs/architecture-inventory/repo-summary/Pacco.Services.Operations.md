# Repository: `Pacco.Services.Operations`

`operations-service` (also known as: Operations Service, `Pacco.Services.Operations`, Docker image
`devmentors/pacco.services.operations`) observes every message on the platform and projects
operation status to clients over HTTP, SignalR and gRPC.

- **Repository:** `Pacco.Services.Operations`, path: `src/Pacco.Services.Operations.Api`
- **Second project:** `Pacco.Services.Operations.GrpcClient`, path:
  `src/Pacco.Services.Operations.GrpcClient` — a console demo client, not containerised
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5005`

---

## README vs repository

`README.md` is the platform boilerplate plus one service-specific instruction.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI.

**Claimed in README, contradicted by the tree — Stale doc.** The README instructs starting the
service from `/src/Pacco.Services.Operations`. That directory does not exist. The actual host
project is `/src/Pacco.Services.Operations.Api`, as confirmed by `Dockerfile`
(`dotnet publish src/Pacco.Services.Operations.Api`) and `scripts/start.sh`. **Code is
authoritative:** the correct path is `src/Pacco.Services.Operations.Api`.

**Present on disk, absent from README (disk-only):**

- `messages.json` — the platform-wide message catalogue naming 8 exchanges, 26 commands, 30 events
  and 31 rejected events owned by other repositories.
- `Infrastructure/Subscriptions.cs` — runtime type emission with `System.Reflection.Emit` to build
  subscription types from that catalogue.
- The **SignalR hub** at `/pacco` and the **gRPC service** — two additional protocols beyond HTTP,
  mentioned nowhere.
- `wwwroot/ui/` — the only frontend code anywhere in the workspace.
- `Operations.proto` and the `Pacco.Services.Operations.GrpcClient` console project.
- Redis-backed operation state with a 300-second expiry.
- The committed symmetric JWT signing key in `appsettings.json`.

**Unknown:** whether operation state is held in MongoDB or Redis. `appsettings.json` configures
`mongo.database: operations-service`, but no `AddMongoRepository<...>` call exists anywhere in the
repository, while Redis and `requests.expirySeconds: 300` are configured and used. **Needs
validation at runtime.**

---

## 1. Primary purpose

Give clients a way to find out what happened to a request. In the platform's asynchronous mode the
gateway returns no domain result for writes, so this service subscribes to every command, event and
rejected event on every exchange, records each as an operation, and pushes status to connected
clients in real time.

## 2. Main runtime / service type

ASP.NET Core 3.1 host serving **three protocols in one process**:

1. HTTP — `GET operations/{operationId}` via Convey dispatcher endpoints.
2. **SignalR** — `endpoints.MapHub<PaccoHub>("/pacco")`.
3. **gRPC** — `endpoints.MapGrpcService<GrpcServiceHost>()`, including a server-streaming method.

It is also a RabbitMQ consumer subscribed to the whole platform. It owns no domain and publishes no
domain message.

## 3. Key entrypoints

- `src/Pacco.Services.Operations.Api/Program.cs` — composition root; maps the dispatcher endpoint,
  the SignalR hub and the gRPC service.
- `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` — reads `messages.json` and
  builds a subscription for every declared message at startup.
- `src/Pacco.Services.Operations.Api/messages.json` — the data that drives the above; changing it
  changes what the service listens to, with no code change.
- `src/Pacco.Services.Operations.Api/Operations.proto` — the gRPC contract.
- `src/Pacco.Services.Operations.GrpcClient/Program.cs` — console client demonstrating the
  streaming subscription.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Operations.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Module | Role |
|---|---|
| `Infrastructure/Subscriptions.cs` | Uses `AssemblyBuilder.DefineDynamicAssembly` and `ModuleBuilder.DefineType` to emit a type per message name, attaches `MessageAttribute(exchange, null, null, true)`, then invokes `IBusSubscriber.Subscribe` reflectively for every command, event and rejected event in `messages.json` |
| `Handlers/GenericCommandHandler.cs`, `GenericEventHandler.cs`, `GenericRejectedEventHandler.cs` | Three handlers that between them cover all 87 catalogued messages — no per-message code exists |
| `Hubs/PaccoHub.cs` | SignalR hub; clients call `initializeAsync` with a JWT and then receive pushes |
| `Services/OperationsService.cs` | Records and retrieves operation status |
| `Operations.proto` | `package Services.Operations;` — `service GrpcOperationsService { rpc GetOperation (GetOperationRequest) returns (GetOperationResponse); rpc SubscribeOperations (Empty) returns (stream GetOperationResponse); }`; `GetOperationResponse` fields `id`, `userId`, `name`, `state`, `code`, `reason` |
| `messages.json` | The catalogue (see dimension 7) |

**Key packages:** `Convey` and the usual CQRS/RabbitMQ/Redis/Consul/Fabio/Jaeger/metrics set, plus
`Grpc.AspNetCore`, `Google.Protobuf`, `Grpc.Tools`, `Microsoft.AspNetCore.SignalR.Redis`.

**Project 2 — `Pacco.Services.Operations.GrpcClient`:** a `Grpc.Net.Client` console application
that calls `SubscribeOperations` and prints the stream. Not referenced by `Dockerfile` or any
deployment manifest.

## 5. External integrations

RabbitMQ (subscribes to 8 exchanges), Redis (operation state and the SignalR backplane via
`Microsoft.AspNetCore.SignalR.Redis`), MongoDB (configured, apparently unused — see dimension 6),
Consul, Fabio, Jaeger, Seq, Prometheus. It calls no other service over HTTP — `httpClient.services`
is empty.

## 6. Data stores / state

- **Configured:** MongoDB database `operations-service`, **and** Redis with instance prefix
  `operations:` plus `requests.expirySeconds: 300`.
- **Query mechanism:** **no `AddMongoRepository<...>` registration exists in this repository.**
  Every other persistent service in the workspace has one. Operation state therefore appears to be
  Redis-only, with a five-minute expiry. **Unknown — needs validation** by reading
  `Services/OperationsService.cs` end to end and confirming at runtime.
- **Migration tool:** **none.**
- **Collections for the primary domain:** none registered. If MongoDB is genuinely unused here,
  the `mongo` configuration section is inert.
- **Cross-domain coupling:** this service holds a shadow of *every* domain's activity — operation
  records keyed by request, carrying message names owned by eight other services. It reads no other
  service's database and writes to none, so there is no data-layer coupling; the coupling is
  entirely contractual, through `messages.json`.
- **Consequence of the expiry, if Redis is the store:** an operation older than 300 seconds is
  gone. A saga or workflow running longer than five minutes would lose its observable status before
  finishing. **Needs validation.**

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Own exchange:** `operations` (declared in `appsettings.json`).
- **Conventions:** `snakeCase`; queue template `operations-service/{{exchange}}.{{message}}`;
  headers `message_context` and `span_context`.

**Subscribes to everything.** `Infrastructure/Subscriptions.cs` iterates `messages.json` and
subscribes to all 87 messages across these exchanges: `availability`, `customers`, `deliveries`,
`identity`, `ordermaker`, `orders`, `parcels`, `vehicles`. The full catalogue — command, event and
rejected-event names verbatim — is reproduced in `repo-inventory.md` §3.2, since it is a
platform-level artefact rather than a fact about this service alone.

**Publishes:** no domain message. It emits SignalR client messages (`connected`, `disconnected`,
`operation_pending`, `operation_completed`) and gRPC stream items, not AMQP messages.

**Payload fields — the important limitation.** The types `Subscriptions.cs` emits have **no
fields**. The service binds to message *names* only and does not deserialise message bodies into
typed properties. What arrives on the wire for each of the 87 subscriptions is therefore
**unknown — requires runtime capture**. The observable projection is limited to what
`GetOperationResponse` carries: `id`, `userId`, `name`, `state`, `code`, `reason`.

## 8. APIs exposed / consumed

**Exposed:**

| Protocol | Surface | Detail |
|---|---|---|
| HTTP | `GET operations/{operationId}` | The only HTTP route; fronted by the gateway module `operations` |
| SignalR | hub `/pacco` | Client invokes `initializeAsync` with a JWT; server pushes `connected`, `disconnected`, `operation_pending`, `operation_completed` |
| gRPC | `Services.Operations.GrpcOperationsService` | `GetOperation(GetOperationRequest) → GetOperationResponse`; `SubscribeOperations(Empty) → stream GetOperationResponse` |

Swagger UI at route prefix `docs`.

**Consumed:** no service APIs. The bundled `Pacco.Services.Operations.GrpcClient` consumes this
service's own gRPC surface.

**Note:** the SignalR hub and the gRPC service have **no gateway route** — the gateway's
`operations` module maps `GET /{operationId}` only. `wwwroot/ui/js/app.js` connects directly to
`http://localhost:5005/pacco`, bypassing the gateway. **Needs validation** for any deployed
environment.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`;
  `dotnet publish src/Pacco.Services.Operations.Api`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Operations.Api.dll`.
  **Only the `.Api` project is published — `GrpcClient` is not containerised.**
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.operations`.
- Port `5005` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5005:80`), and the
  gateway `localUrl`.
- Consul service name `operations-service`.
- **Scale-out consideration:** the SignalR Redis backplane is referenced, which suggests multiple
  instances are anticipated. Whether more than one instance is actually run is **Unknown**.
- **Runtime coupling to configuration:** subscriptions are built from `messages.json` at startup,
  so adding a message to the platform requires editing this file and redeploying this service.

## 10. Security / auth clues

- **JWT bearer** with `validIssuer: pacco`; `jwt.allowAnonymousEndpoints: ['/sign-in','/sign-up']`.
- **The symmetric JWT `issuerSigningKey` is committed in plaintext** in
  `src/Pacco.Services.Operations.Api/appsettings.json` — the same value that appears in the
  gateway's four `ntrada*.yml` files. Recorded as an observation; the value is not reproduced here.
- **SignalR authentication is application-level, not pipeline-level:** the browser passes a JWT as
  an argument to `initializeAsync` (`wwwroot/ui/js/app.js`, `Hubs/PaccoHub.cs`) rather than
  relying on the standard bearer flow. What the hub does with an invalid or absent token, and
  whether it scopes pushes to the caller's own operations, is **Unknown — needs validation**.
- **gRPC authentication:** no per-method authorisation is expressed in `Operations.proto`, and
  `SubscribeOperations` streams operations without any filter argument. Whether that stream is
  restricted per caller is **Unknown — needs validation**. `GetOperationResponse` includes
  `userId`, which implies the stream may carry other users' operations.
- No Vault section and no `security.certificate` ACL in this repository.

## 11. Observability / logging / tracing

This service *is* the platform's request-level observability projection — its whole purpose is to
make asynchronous outcomes visible. For its own telemetry:

- **Tracing:** Jaeger, `serviceName: operations`, UDP `6831`, `const` sampler.
- **Logging:** console, file and Seq sinks; ELK present but disabled.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`; `/metrics` and
  `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `src/Pacco.Services.Operations.Api/messages.json` | **The platform's message contract catalogue** — the most decision-bearing file in the workspace. It is the only place where all 87 message names across eight services appear together, and it is maintained by hand with no generation or validation step |
| `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` | That subscriptions are generated at runtime by emitting types with `System.Reflection.Emit`, so no per-message C# exists — a deliberate trade of compile-time safety for zero-code extensibility |
| `src/Pacco.Services.Operations.Api/Handlers/Generic*Handler.cs` | That all messages are handled uniformly as opaque operations, not as typed domain messages |
| `src/Pacco.Services.Operations.Api/Operations.proto` | That gRPC, including server streaming, is a first-class client protocol on this platform |
| `src/Pacco.Services.Operations.Api/Program.cs` | That HTTP, SignalR and gRPC are co-hosted in one process |
| `src/Pacco.Services.Operations.Api/appsettings.json` | Redis-backed operation state with `requests.expirySeconds: 300` — an implicit statement about how long an operation is expected to take |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**. `messages.json` acts as a
runtime-configurable subscription list, which is configuration-driven behaviour but not a feature
flag system.

## 13. Open questions / ambiguities

1. Whether operation state lives in MongoDB or Redis.
2. Whether a 300-second expiry is compatible with the order-creation saga's duration.
3. Whether the SignalR hub and the gRPC stream scope operations to the calling user.
4. How `messages.json` is kept in step with eight independently changing services.
5. Whether the SignalR hub and gRPC endpoint are meant to be reached directly rather than through
   the gateway.
6. Whether `Pacco.Services.Operations.GrpcClient` is a demo or a shipped artefact.

## 14. Frontend stack

**Frontend assets are present — the only ones anywhere in the workspace.**

- **Location:** `src/Pacco.Services.Operations.Api/wwwroot/ui/`.
- `wwwroot/ui/index.html` — page titled "Pacoo SignalR"; loads **Bootstrap 4.0.0 from the maxcdn
  CDN**; provides a JWT text input and a Connect button.
- `wwwroot/ui/js/signalr.js` — the SignalR JavaScript client, committed as a pre-built **webpack
  UMD bundle**.
- `wwwroot/ui/js/app.js` — the application script:
  `new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco')`, then
  `connection.invoke('initializeAsync', jwt)`, with handlers for `connected`, `disconnected`,
  `operation_pending` and `operation_completed`.

**Stack characterisation:** plain ES5/ES6 JavaScript with no framework (no React, Angular or Vue),
no `package.json`, no bundler configuration, no TypeScript, no test setup, and no build step — the
vendor bundle is committed rather than installed. Bootstrap arrives from a public CDN at runtime.
The hub URL is hard-coded to `http://localhost:5005`, so this page works only against a local
instance. It is best characterised as a developer diagnostic page, not a product frontend.

**Directories checked:** `src/Pacco.Services.Operations.Api/wwwroot/` (**found**),
`wwwroot/ui/`, `wwwroot/ui/js/`, `src/Pacco.Services.Operations.GrpcClient/`, and the repository
root. No `public/`, `static/`, `assets/`, `resources/js/`, or `web/` directory exists, and there
are no server-side view templates.

---

## Evidence

| Fact | File |
|---|---|
| Three-protocol host, hub mapping, gRPC mapping, dispatcher endpoint | `src/Pacco.Services.Operations.Api/Program.cs` |
| Runtime subscription type emission | `src/Pacco.Services.Operations.Api/Infrastructure/Subscriptions.cs` |
| Platform message catalogue (8 exchanges, 87 messages) | `src/Pacco.Services.Operations.Api/messages.json` |
| Uniform message handling | `src/Pacco.Services.Operations.Api/Handlers/GenericCommandHandler.cs`, `GenericEventHandler.cs`, `GenericRejectedEventHandler.cs` |
| SignalR hub and client authentication call | `src/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs` |
| Operation recording and retrieval | `src/Pacco.Services.Operations.Api/Services/OperationsService.cs` |
| gRPC contract and streaming method | `src/Pacco.Services.Operations.Api/Operations.proto` |
| gRPC consumer demo | `src/Pacco.Services.Operations.GrpcClient/Program.cs`, `Pacco.Services.Operations.GrpcClient.csproj` |
| Frontend | `src/Pacco.Services.Operations.Api/wwwroot/ui/index.html`, `wwwroot/ui/js/app.js`, `wwwroot/ui/js/signalr.js` |
| Redis state, 300s expiry, committed signing key, exchange, JWT, logging, metrics, tracing | `src/Pacco.Services.Operations.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Absence of any Mongo repository registration | `src/Pacco.Services.Operations.Api/` (no `AddMongoRepository` call in the repository) |
| Package set including gRPC and SignalR Redis backplane | `src/Pacco.Services.Operations.Api/Pacco.Services.Operations.Api.csproj` |
| Project list | `Pacco.Services.Operations.sln` |
| Container build publishing only `.Api` | `Dockerfile` |
| CI and image publication | `.travis.yml`, `scripts/build.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Stale start path | `README.md` |
| Gateway route for operations only | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `messages.json` is a complete and accurate list of the platform's messages | Cross-checking it against every service's `Commands/` and `Events/` folders found no message present in a service but missing from the catalogue | Any message absent from the catalogue is invisible to this service, so operations driven by it would never report an outcome to the client | The catalogue is hand-maintained with no validation step — re-check it whenever any service adds a message |
| A2 | The emitted subscription types bind on message name alone and carry no payload | `Subscriptions.cs` defines types with no fields and attaches only `MessageAttribute` | Payload data might be available after all, in which case the operation projection could be richer than described | Capture a message on the broker and inspect what the handler receives at runtime |
| A3 | This service holds no domain authority and losing its data affects visibility only | It publishes no domain message, owns no aggregate, and every other service functions without it | If any service or client depends on operation records for correctness rather than visibility, the 300-second expiry becomes a correctness problem | Confirm no consumer treats operation status as authoritative |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** The symmetric JWT signing key is committed in plaintext in this service's `appsettings.json`, matching the one in the gateway configuration. Anyone with it can mint a token this service and the gateway both accept | Any conclusion that the platform's authentication is sound, and any later work on the token model | Whoever owns Pacco authentication | Confirm whether the key is live; if so, rotate it, move it into Vault, and purge the value from git history | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Where does operation state actually live — MongoDB or Redis? | MongoDB is configured but no repository is registered, while Redis is configured with a five-minute expiry. If Redis is the store, every operation older than five minutes disappears, so a client polling for a slow result gets nothing back and cannot tell success from failure | Read `Services/OperationsService.cs` end to end and confirm against a running instance | Platform architect |
| Q2 | **[ACTION NOW]** Does the SignalR hub or the gRPC stream restrict what each caller can see? | `SubscribeOperations` streams `GetOperationResponse` values that include `userId`, with no filter parameter, and hub authentication happens inside `initializeAsync` rather than in the request pipeline. If neither scopes results, one customer can watch another customer's activity | Read `Hubs/PaccoHub.cs` and the gRPC host implementation, then test with two different tokens | Whoever owns Pacco authentication |
| Q3 | **[ACTION NOW]** How is `messages.json` kept in step with the eight services that own those names? | It is a hand-maintained copy of contracts owned elsewhere, with no generation step and no validation. A service that renames a message silently stops being observable here, and nobody finds out until someone notices missing operations | Generate the catalogue from the services, or add a startup check that fails loudly on a mismatch | Platform architect |
| Q4 | **[ACTION NOW]** Are the SignalR hub and gRPC endpoint meant to be reached directly, bypassing the gateway? | The gateway routes only `GET /operations/{operationId}`, and the bundled UI connects straight to `localhost:5005`. In async mode this service is how clients learn outcomes, so its real access path matters for both routing and authentication | Confirm the intended client topology for real-time updates | Platform architect |
| Q5 | **[handled later by HLD]** Is a 300-second operation expiry long enough? | The order-creation saga spans several services and messages. If it takes longer than five minutes, its operation record expires before it completes and the client watching for a result sees nothing | Measure real saga durations and set the expiry accordingly, or persist operations durably | Platform architect |
| Q6 | **[handled later by HLD]** Is `Pacco.Services.Operations.GrpcClient` a demo or a supported artefact? | It is a project in the solution but is not containerised, not in CI's publish step, and not in any deployment manifest. Whether it needs maintaining is unclear | Mark it explicitly as a sample, or give it a build and release path | Domain owner for Operations |
