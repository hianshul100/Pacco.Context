# Repository: `Pacco.Services.OrderMaker`

`ordermaker-service` (also known as: OrderMaker Service, `Pacco.Services.OrderMaker`, Docker image
`devmentors/pacco.services.ordermaker`) orchestrates end-to-end order creation as a long-running
saga across Orders, Parcels, Vehicles and Availability.

- **Repository:** `Pacco.Services.OrderMaker`, path: `src/Pacco.Services.OrderMaker`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5015` (per its own `appsettings.json`)
- **Deployment status: not in any platform manifest — see dimension 9**

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no saga, entity, endpoint, event or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only) — everything that makes this repository
distinctive:**

- That it is a **saga orchestrator** built on `Chronicle_` 3.2.1 — the only saga implementation on
  the platform.
- `Sagas/AIOrderMakingSaga.cs`, `Sagas/AIMakingOrderData.cs`, `Handlers/AIOrderMakingHandler.cs`.
- That it is a **single-project** service with no clean-architecture split, unlike nine of its
  siblings.
- That it publishes commands onto **other services' exchanges**, driving them directly.
- The synchronous dependencies on `availability-service` and `vehicles-service`.
- That it has **no MongoDB** and **no Jaeger tracing**.
- That it exposes `POST /orders` — a second, competing order-creation entry point alongside
  `orders-service`.

**Stale doc:** none identified in this repository's own README. However, `Pacco/README.md`'s
"start all services" instruction does not start this service, because it is in none of the platform
manifests. **Documented claim not fully reflected in the tree** — recorded in `Pacco.md`.

**Unknown:** whether this service is deployed at all, and where Chronicle keeps saga state.

---

## 1. Primary purpose

Turn a single `MakeOrder` request into a completed order by choosing a vehicle, choosing a resource
reservation, creating the order, adding every parcel to it, assigning the vehicle, and waiting for
approval — coordinating four other services and compensating on failure. The naming ("AI order
making", and the comment `// typical AI in a startup` above a `FirstOrDefault()` call) indicates the
selection logic is deliberately trivial placeholder behaviour.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API, RabbitMQ publisher and subscriber, and **Chronicle saga host**, all in
one process and one project.

**Distinguishing detail:** `Program.cs` calls `.UseApp()` rather than the `UseInfrastructure()`
used by every clean-architecture sibling, and it has **no `.UseVault()`** — this is the only
broker-participating service that does not load secrets from Vault.

## 3. Key entrypoints

- `src/Pacco.Services.OrderMaker/Program.cs` — composition root and route table.
  `GET /` returns the literal string `Welcome to Pacco uber AI order maker Service!`;
  `POST orders` dispatches `MakeOrder`.
- `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` — the orchestration itself; the real
  entrypoint for the service's behaviour.
- `src/Pacco.Services.OrderMaker/Extensions.cs` — DI wiring, including
  `AddInMemoryCommandDispatcher()`, `AddInMemoryEventDispatcher()`, `AddRedis()` and
  `builder.Services.AddChronicle()`.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

**Single project:** `src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj`. There is no
`.Api`/`.Application`/`.Core`/`.Infrastructure` split.

| Module | Role |
|---|---|
| `Sagas/AIOrderMakingSaga.cs` | `Saga<AIMakingOrderData>` implementing `ISagaStartAction<MakeOrder>`, `ISagaAction<OrderCreated>`, `ISagaAction<ParcelAddedToOrder>`, `ISagaAction<VehicleAssignedToOrder>`, `ISagaAction<OrderApproved>`. Declares `const string SagaHeader = "Saga"`; `ResolveId` maps every message to `OrderId` |
| `Sagas/AIMakingOrderData.cs` | Saga state: `OrderId`, `CustomerId`, `VehicleId`, `ReservationDate`, `ReservationPriority`, `ParcelIds`, `AddedParcelIds`, `AllPackagesAddedToOrder` |
| `Handlers/AIOrderMakingHandler.cs` | Handles the inbound `MakeOrder` command |
| `Services/Clients/VehiclesServiceClient.cs` | `GET {vehicles}/vehicles` then `.Items.FirstOrDefault()` — carries the comment `// typical AI in a startup` |
| `Services/Clients/AvailabilityServiceClient.cs` | `GET {availability}/resources/{resourceId}` |
| `Services/ResourceReservationsService.cs` | `GetBestAsync(vehicleId)` — reservation selection |

**Key packages:** `Chronicle_` **3.2.1** (the saga library), `Convey`,
`Convey.CQRS.Commands/.Events/.Queries`, `Convey.MessageBrokers.RabbitMQ`,
`Convey.Persistence.Redis`, `Convey.Discovery.Consul`, `Convey.LoadBalancing.Fabio`, `Convey.HTTP`,
`Convey.Logging`, `Convey.Metrics.AppMetrics`, `Convey.Security`, `Convey.WebApi`, `.WebApi.CQRS`.

**Packages notably absent:** `Convey.Persistence.MongoDB`, `Convey.MessageBrokers.Outbox`,
`Convey.Tracing.Jaeger`, `Convey.Secrets.Vault`, `Convey.WebApi.Swagger`.

## 5. External integrations

| Integration | How |
|---|---|
| `availability-service` | HTTP `GET {availability}/resources/{resourceId}` |
| `vehicles-service` | HTTP `GET {vehicles}/vehicles` |
| RabbitMQ | Own exchange `ordermaker`; publishes commands onto `orders` and `availability` |
| Redis | `AddRedis()`, instance prefix `ordermaker:` |
| Consul | Registration on port 5015 |
| Seq, Prometheus | Logs and metrics |

**No Vault. No Jaeger. No MongoDB.** `httpClient.type` is `""` rather than `fabio`, so outbound
calls do **not** go through the load balancer as they do in every other service. **Needs
validation** — this may mean direct addressing or default HTTP client behaviour.

## 6. Data stores / state

- **MongoDB: none.** No `mongo` section in `appsettings.json`, no `Convey.Persistence.MongoDB`
  package, no repository registration. This service owns no domain data.
- **Redis:** registered via `AddRedis()` with instance prefix `ordermaker:`.
- **Saga state — the material question.** `AIMakingOrderData` is the state a running order-making
  saga carries. `Extensions.cs` calls `builder.Services.AddChronicle()` with **no persistence
  configuration and no `Chronicle.Persistence.*` package**. Chronicle's default backing store is
  in-memory, which would mean saga state is lost on restart and not shared between instances.
  **Unknown — needs validation.**
- **ORM / query mechanism:** none — there is no database access layer in this repository.
- **Migration tool:** none, and none needed given no owned store.
- **Cross-domain coupling — significant.** This service holds no data of its own but reads and
  writes across four domains at once. Its saga state carries `OrderId`, `CustomerId`, `VehicleId`,
  `ReservationDate`, `ParcelIds` — identifiers owned by `orders-service`, `customers-service`,
  `vehicles-service`, `availability-service` and `parcels-service` respectively, with no
  referential enforcement anywhere. If the saga is in-memory, that cross-domain state is also
  volatile.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Own exchange:** `ordermaker`, type `topic`.
- **Conventions:** `snakeCase`; queue template `ordermaker-service/{{exchange}}.{{message}}`;
  headers `message_context` and `span_context`.
- **No transactional outbox** — `Convey.MessageBrokers.Outbox` is not referenced. This is the only
  publishing service without one, so a crash between a saga step and its publish loses the command
  with no retry.

**Commands published onto other services' exchanges** (names verbatim): `create_order`,
`add_parcel_to_order`, `assign_vehicle_to_order`, `approve_order`, `cancel_order` (all on the
`orders` exchange) and `reserve_resource` (on the `availability` exchange). Published commands
carry the header `[SagaHeader] = SagaStates.Pending.ToString()`, i.e. header key `Saga`.

**Events published on its own exchange:**

| Event | Observable payload fields |
|---|---|
| `make_order_completed` | `OrderId`, `CustomerId` |
| `make_order_rejected` | `OrderId`, `Reason`, `Code` |

**External events consumed** (saga actions): `order_created`, `parcel_added_to_order`,
`vehicle_assigned_to_order`, `order_approved` (all from `orders`), and `resource_reserved` (from
`availability`).

**Correlation:** `ResolveId` maps every one of those messages to `OrderId`, so `OrderId` is the
saga correlation key throughout.

**Saga flow observable from `AIOrderMakingSaga.cs`:** `MakeOrder` starts the saga → the saga calls
`_vehiclesServiceClient.GetBestAsync()` and `_resourceReservationsService.GetBestAsync(vehicleId)`
→ publishes `CreateOrder` → on `OrderCreated`, publishes `AddParcelToOrder` per parcel → tracks
`AddedParcelIds` until `AllPackagesAddedToOrder` → publishes `AssignVehicleToOrder` → on
`VehicleAssignedToOrder`, proceeds toward approval → on `OrderApproved`, completes.

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.OrderMaker/Program.cs`, verbatim):

| Method | Route | Behaviour |
|---|---|---|
| `GET` | `/` | Returns the literal string `Welcome to Pacco uber AI order maker Service!` |
| `POST` | `orders` | Dispatches `MakeOrder`, starting the saga |

**No Swagger** — `Convey.WebApi.Swagger` is not referenced, unlike every other service.

**Consumed:** `GET {availability}/resources/{resourceId}` on `availability-service`;
`GET {vehicles}/vehicles` on `vehicles-service`.

**Upstream: none.** There is **no `ordermaker` module in any of the four gateway configurations**,
so `POST /orders` on this service is unreachable from the platform edge. Note also that the path
collides conceptually with `orders-service`'s own `POST /orders`, which the gateway *does* route —
two different services accept an order-creation request at the same path shape, and only one is
publicly reachable.

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.OrderMaker.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.ordermaker`.
- Consul registration on port `5015` — outside the platform's contiguous 5000–5009 block.
- **Absent from every platform deployment manifest.** It appears in **none** of:
  `Pacco/services.yml`, `Pacco/prod-services.yml`, `Pacco/compose/services.yml`,
  `Pacco/compose/services-local.yml`, or any `ntrada*.yml`. It has a build and an image but no
  declared place in a running platform. **Unverifiable from the repositories — needs validation.**
- **Scale-out consideration:** if Chronicle saga state is in-memory, running more than one instance
  would split saga state across processes and break correlation. **Needs validation.**

## 10. Security / auth clues

- `jwt` configuration is present in `appsettings.json` with `validIssuer: pacco`, and
  `Convey.Security` is referenced.
- **`Program.cs` calls `.UseApp()`, not `UseInfrastructure()`.** Whether the JWT middleware is
  actually in the pipeline is therefore **Unknown — needs validation**.
- **The saga acts without a caller identity.** It constructs an empty
  `CorrelationContext.UserContext` when publishing commands, so the commands it sends to
  `orders-service` and `availability-service` carry no authenticated user. Any authorisation those
  services base on the message context would see an anonymous actor. **Needs validation** of
  whether the receiving services check it.
- **No Vault** — the only broker-participating service without `Convey.Secrets.Vault`, so it has no
  path to dynamic secrets or PKI certificates.
- **No `security.certificate` ACL**, and its outbound HTTP calls attach no client certificate,
  unlike `availability-service`'s call to `customers-service`.

## 11. Observability / logging / tracing

- **Tracing: none.** `Convey.Tracing.Jaeger` is **not referenced**, and there is no `jaeger`
  section in `appsettings.json`. This is the only broker-participating service without distributed
  tracing — which is notable precisely because it is the service whose work spans the most other
  services. A saga failure cannot be followed end to end in Jaeger.
- **Logging:** `Convey.Logging` with console, file and Seq sinks; ELK present but disabled.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`.
- **Saga visibility:** published commands carry the `Saga` header with the saga state, which is the
  only in-band signal that a message belongs to a saga.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` | **The order-creation choreography in full** — which services are called, in what order, on which events, and with which correlation key. This is the most decision-bearing file in the repository |
| `src/Pacco.Services.OrderMaker/Sagas/AIMakingOrderData.cs` | What state a running saga carries across service boundaries |
| `src/Pacco.Services.OrderMaker/Extensions.cs` | The choice of `Chronicle_` 3.2.1 with no persistence configuration; in-memory command and event dispatchers; Redis; and the absence of Vault, Jaeger, Mongo and the outbox |
| `src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj` | The single-project layout — a deliberate departure from the clean-architecture split used by nine sibling services, consistent with the platform README's "or another style that is the best fit" |
| `src/Pacco.Services.OrderMaker/Program.cs` | That this service exposes its own `POST /orders` entry point, parallel to `orders-service` |
| `src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs` | That vehicle selection is a placeholder (`FirstOrDefault()`), explicitly flagged in a comment |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. Whether this service is deployed at all.
2. Where Chronicle stores saga state, and what happens to an in-flight order on restart.
3. How callers reach `POST /orders` here, given no gateway route.
4. Why this service alone lacks Jaeger tracing, Vault, and a transactional outbox.
5. Whether the receiving services accept commands that carry no user context.
6. Whether `httpClient.type: ""` means outbound calls bypass Fabio.
7. Whether the relationship between this service's `POST /orders` and `orders-service`'s
   `POST /orders` is intentional.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.OrderMaker/` and all
subdirectories (`Sagas/`, `Handlers/`, `Services/`, `Services/Clients/`, `certs/`, `Properties/`),
and the repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor). Unlike its sibling services this repository does not even
serve a Swagger UI, since `Convey.WebApi.Swagger` is not referenced — its only browser-visible
output is the plain-text welcome string returned by `GET /`.

---

## Evidence

| Fact | File |
|---|---|
| Route table, welcome string, `UseApp()`, absence of `UseVault()` | `src/Pacco.Services.OrderMaker/Program.cs` |
| Saga definition, actions, correlation, published commands, `Saga` header | `src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs` |
| Saga state fields | `src/Pacco.Services.OrderMaker/Sagas/AIMakingOrderData.cs` |
| Inbound command handling | `src/Pacco.Services.OrderMaker/Handlers/AIOrderMakingHandler.cs` |
| DI wiring, `AddChronicle()` without persistence, in-memory dispatchers, Redis | `src/Pacco.Services.OrderMaker/Extensions.cs` |
| Vehicle selection placeholder | `src/Pacco.Services.OrderMaker/Services/Clients/VehiclesServiceClient.cs` |
| Availability lookup | `src/Pacco.Services.OrderMaker/Services/Clients/AvailabilityServiceClient.cs` |
| Reservation selection | `src/Pacco.Services.OrderMaker/Services/ResourceReservationsService.cs` |
| Package set: Chronicle present; Mongo, outbox, Jaeger, Vault, Swagger absent | `src/Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.csproj` |
| Port 5015, exchange, `httpClient.type: ""`, service map, absence of `mongo` and `jaeger` sections | `src/Pacco.Services.OrderMaker/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| Absence from platform manifests | `../hianshul100_Pacco/services.yml`, `../hianshul100_Pacco/prod-services.yml`, `../hianshul100_Pacco/compose/services.yml`, `../hianshul100_Pacco/compose/services-local.yml` |
| Absence of a gateway module | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada.docker.yml`, `ntrada-async.yml`, `ntrada-async.docker.yml` |
| Message catalogue entry for the `ordermaker` exchange | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |
| Commands this service drives | `../hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Application/Commands/`, `../hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Application/Commands/` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Chronicle 3.2.1 defaults to in-memory saga state when no persistence is configured | `AddChronicle()` is called with no options and no persistence package is referenced; in-memory is the usual default for this library | If Chronicle actually persists somewhere by default, the durability concern raised throughout this document is unfounded | Read the Chronicle 3.2.1 source, or restart the service mid-saga and see whether it resumes |
| A2 | The saga is the only way this service does anything | `POST /orders` dispatches `MakeOrder`, which starts the saga, and there is no other behaviour in the repository | If another path exists, the service's role is broader than described | Read `Handlers/AIOrderMakingHandler.cs` alongside the saga |
| A3 | The commands this service publishes are accepted by the receiving services without a user context | The saga builds an empty `CorrelationContext.UserContext`, and the receiving services subscribe to those commands | If any receiving service rejects or mis-attributes anonymous commands, the saga fails partway and leaves an order in an incomplete state | Publish one of these commands against a running platform and observe the receiving service |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** This service has a Dockerfile, a CI pipeline and a port, but appears in no deployment manifest and behind no gateway route. Nobody on this side can tell whether the order-creation saga is live code or dead code | Any description of how orders are actually created on this platform, and whether the coordination logic here needs governing at all | Platform owner / operations | Someone must state whether `ordermaker-service` runs in any environment and, if it does, how callers reach it | TBD |
| B2 | **[ACTION NOW]** If this service is live, its saga state has no confirmed durable store, no transactional outbox, and no distributed tracing — so a restart mid-order can lose an order that is already half-created in four other services, with no trace to follow and no automatic retry | Treating order creation as reliable, and any later work that depends on the saga completing | Platform architect | Confirm the Chronicle persistence backend; if it is in-memory, decide whether to add persistence or accept the loss window explicitly | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** How is a caller meant to reach `POST /orders` on this service? | No gateway configuration mentions it, so from outside the platform it does not exist. Meanwhile `orders-service` exposes a route at the same path shape that the gateway *does* publish — so it is unclear which one is the real way to place an order | Either a gateway module is missing, or this service is intended for internal or demonstration use only | Platform architect |
| Q2 | **[ACTION NOW]** Should this service have Jaeger tracing? | It is the one service whose work spans four others, and it is the only broker-participating service with no tracing at all. When a saga stalls halfway, there is no trace connecting the steps and no way to see where it stopped | Add `Convey.Tracing.Jaeger` and the RabbitMQ plugin as every sibling service has | Platform architect |
| Q3 | **[ACTION NOW]** Should this service have a transactional outbox? | Every other publishing service has one. Here, a crash between a saga step and its publish drops the command silently — the order stays half-built and nothing retries | Add `Convey.MessageBrokers.Outbox` as the siblings do, or state why the saga's own retry behaviour is sufficient | Platform architect |
| Q4 | **[handled later by HLD]** Is the vehicle and reservation selection meant to stay a placeholder? | `GetBestAsync()` takes the first item from the list and the code says so in a comment. If real selection logic is expected, an entire piece of intended behaviour is missing rather than merely simple | Confirm whether "best vehicle" is meant to mean anything, and if so define the rule | Domain owner for Orders |
| Q5 | **[ACTION NOW]** Why does this service alone skip Vault, and does `httpClient.type: ""` bypass Fabio? | Every sibling loads secrets from Vault and routes outbound calls through the load balancer. This one does neither, so it may be addressing services directly and holding configuration differently from the rest of the platform | Align with the sibling configuration or record why this service differs | Platform architect |
