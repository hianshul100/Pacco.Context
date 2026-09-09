# Component internals — `vehicles-service`

| | |
| --- | --- |
| **Component** | `vehicles-service` |
| **Source repository** | `hianshul100_Pacco.Services.Vehicles` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository; four projects under `src/`) |
| **Base ref** | `feature/12998/aidlc` (`HEAD` = `af43bcf`, *Updated outbox - disable local TX*) |
| **Batch** | 6 of 7 |
| **Status** | New artifact — no prior `component-internals/vehicles-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` §2.3 and `repo-summary/Pacco.Services.Vehicles.md` remain valid and are **complemented**, not replaced. Where this model corrects or extends a baseline it says so and names the section (§8.4). |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full — **69 tracked
> files** across **four projects** (`Core`, `Application`, `Infrastructure`, `Api`), declared in
> `Pacco.Services.Vehicles.sln`. There is **no test project of any kind** and no `tests/` directory,
> yet `scripts/test.sh` runs `dotnet test` in CI (§3.46). `Convey 0.4.*` — which supplies the CQRS
> dispatchers, the Mongo repository, the RabbitMQ transport, the outbox, the handler-logging
> decorators, Consul/Fabio, and the Vault bootstrap — is referenced from NuGet with **no source in
> this workspace**; mechanisms it owns are tagged `[convey]` and, where their exact semantics would
> change a conclusion, marked `Unverifiable — Missing Source Evidence`. The API gateway's half of the
> vehicle routes is modelled in `component-internals/api-gateway.md`; the downstream consumers of
> this service's events are modelled in `component-internals/availability-service.md` and
> `component-internals/orders-service.md`.

---

## Contents

1. [Purpose & boundary](#1-purpose--boundary)
2. [Core concepts (exhaustive)](#2-core-concepts-exhaustive)
3. [Per concept](#3-per-concept)
4. [Primary control flows](#4-primary-control-flows)
5. [Persistence & schema evolution](#5-persistence--schema-evolution)
6. [Surface → internals map](#6-surface--internals-map)
7. [Change/extension guide](#7-changeextension-guide)
8. [Assumptions, Blockers & Open Questions](#8-assumptions-blockers--open-questions)

---

## 1. Purpose & boundary

### 1.1 What this component is responsible for

`vehicles-service` is the **catalogue of vehicles** the platform can dispatch: their brand, model,
description, payload and loading capacities, price per service, and the set of cargo *variants* each
one is certified to carry. It owns one Mongo database, one RabbitMQ exchange, one CRUD surface, and
one small piece of genuine domain logic — the bitmask of variants (§3.3, §3.4).

It is, structurally, the **most conventional** Pacco service: it instantiates nearly every platform
pattern in its plainest form. That makes it the best reference implementation in the inventory, and
it also makes the handful of places where it deviates worth reading carefully — each deviation is a
live defect or trap (§3.4, §3.13, §3.18, §3.45).

| Responsibility | Where it lives |
| --- | --- |
| Hold vehicle identity, specification and price | `…Core/Entities/Vehicle.cs:6-75` |
| Model which cargo variants a vehicle may carry | `…Core/Entities/Variants.cs:5-13` |
| Enforce that capacities are positive, description non-empty, price positive | `…Core/Entities/Vehicle.cs:24-25,37-55` |
| Accept `AddVehicle`, `UpdateVehicle`, `DeleteVehicle` over **both** HTTP and AMQP | `…Api/Program.cs:37-40`; `…Infrastructure/Extensions.cs:86-88` |
| Persist vehicles as Mongo documents in database `vehicles-service`, collection `vehicles` | `…Infrastructure/Extensions.cs:72`; `appsettings.json:95-99` |
| Serve point reads and paged searches from the document store directly | `…Infrastructure/Mongo/Queries/Handlers/*.cs` |
| Publish `VehicleAdded` / `VehicleUpdated` / `VehicleDeleted` on exchange `vehicles` | `…Infrastructure/Services/MessageBroker.cs:47-86`; `appsettings.json:131-137` |
| Publish `*Rejected` events when an AMQP-delivered command fails | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:12-40` |
| Deduplicate inbound messages and stage outbound ones through a Mongo outbox | `…Infrastructure/Decorators/*.cs`; `appsettings.json:100-108` |
| Register with Consul, address peers via Fabio, trace to Jaeger, expose Prometheus metrics | `…Infrastructure/Extensions.cs:61-62,68-69` |
| Fetch settings, a PKI certificate and **dynamic Mongo credentials** from Vault | `…Api/Program.cs:43`; `appsettings.json:164-193` |

### 1.2 What this component explicitly is **not**

- **Not an aggregate root with buffered domain events.** `Vehicle` has no `AggregateRoot` base, no
  `Events` collection, no `Version` and no `ClearEvents` (`…Core/Entities/Vehicle.cs:6-16`). Compare
  `customers-service`, where `AggregateRoot` supplies exactly that
  (`component-internals/customers-service.md` §3.2). Here, **integration events are published by the
  command handler**, not raised by the entity ([[aggregate-buffered-domain-events]] is *not*
  instantiated — §3.14).
- **Not the owner of vehicle availability or booking.** A vehicle's *bookability* lives in
  `availability-service` as a `Resource`; this service only tells that service when a vehicle
  disappears (§3.47). It holds no calendar, no reservation and no state machine.
- **Not the owner of pricing.** `PricePerService` is a per-vehicle number stored here and read by
  `orders-service` (`hianshul100_Pacco.Services.Orders/…/Clients/VehiclesServiceClient.cs:20-21`);
  the discount ladder that turns an order price into a final price lives in `pricing-service`
  (`component-internals/pricing-service.md` §3.6). The two never meet inside this repository.
- **Not an authenticator or an authorizer.** There is **no `AddCertificateAuthentication()`**, no
  `security` section in any profile, no `AddJwt`, and no `[Authorize]`. The `jwt` block
  (`appsettings.json:77-85`) and the committed `certs/localhost.cer` are inert (§3.41). All five
  vehicle routes are `auth: true` **at the gateway** (`ntrada.yml:418-451`) and unauthenticated at
  the service. Anything that can reach port 5009 can delete any vehicle.
- **Not identity-aware.** `IAppContext` and `IIdentityContext` are fully defined and fully wired
  (§3.38) and **injected by nothing** (§3.40). `IsAdmin` is computed and never read.
- **Not a caller of any other service.** `httpClient.services` is an **empty object** in all three
  populated profiles (`appsettings.json:26`, `appsettings.local.json:12`,
  `appsettings.docker.json:25`), and no client class exists. `AddHttpClient()`
  (`…Infrastructure/Extensions.cs:60`) is registered for nothing.
- **Not a subscriber to any external event.** There is no `Events/External` folder and no
  `SubscribeEvent<T>()` call. It subscribes only to its **own three commands** (§3.32). Traffic flows
  out of this service, never in — except as commands.
- **Not transactional across the write and the publish.** `outbox.disableTransactions: true`
  (`appsettings.json:107`) — set by the `HEAD` commit itself (`af43bcf`, "Updated outbox - disable
  local TX") — removes the Mongo transaction that would otherwise make the state change and the
  outbox insert atomic (§3.30).
- **Not tested.** No test project (§3.46).

### 1.3 The dual-transport command surface

This is the property that shapes the whole failure model. Each of the three commands has **two entry
points with different semantics**, and the difference is not cosmetic
([[dual-mode-edge-write]]):

| | HTTP path | AMQP path |
| --- | --- | --- |
| Entry | `Post/Put/Delete<T>` in `…Api/Program.cs:37-40` | `SubscribeCommand<T>()` in `…Infrastructure/Extensions.cs:86-88` |
| Producer | the gateway in `downstream` mode (`ntrada.yml:432-451`) or any in-cluster caller | the gateway in `rabbitmq` mode (`ntrada-async.yml:500-529`) |
| Message id | **absent** → the outbox decorator generates a fresh `Guid` every time (`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:27-29`) → **inbox deduplication is inert** (§3.30) | supplied by the broker → deduplication works |
| Caller context | `Correlation-Context` header, if present (`…Infrastructure/Extensions.cs:94`) | the `message_context` header (`appsettings.json:145-148`) |
| Failure surfaced as | HTTP **400** with `{code, reason}` (§3.26) | a `*Rejected` event on exchange `vehicles` — **or silently dropped** if unmapped (§3.28) |
| Caller learns outcome | synchronously | only by subscribing to the rejected event |
| Success signalled by | `201 Created` (POST) or `200` | `VehicleAdded`/`Updated`/`Deleted` |

The consequence to hold on to: **the same command, sent two ways, has different idempotency and
different error reporting.** Nothing in the code marks which path a handler is executing on.

### 1.4 Position in the platform

| Direction | Counterpart | Mechanism | Evidence |
| --- | --- | --- | --- |
| Inbound (sync) | `api-gateway` | 5 routes, all `use: downstream`, all `auth: true` | `ntrada.yml:415-456`; identical in `ntrada.docker.yml` |
| Inbound (async) | `api-gateway` | POST/PUT/DELETE become `use: rabbitmq` on exchange `vehicles`, routing keys `add_vehicle` / `update_vehicle` / `delete_vehicle` | `ntrada-async.yml:500-529` |
| Inbound (sync) | `orders-service` | `GET /vehicles/{id}` for `PricePerService` | `hianshul100_Pacco.Services.Orders/…/Clients/VehiclesServiceClient.cs:20-21` |
| Outbound (async) | `availability-service` | `VehicleDeleted` on exchange `vehicles` → `DeleteResource` | `hianshul100_Pacco.Services.Availability/…/Events/External/VehicleDeleted.cs:7`; `…/Handlers/VehicleDeletedHandler.cs:17`; subscribed at `…Availability.Infrastructure/Extensions.cs:112` |
| Outbound (async) | **nobody** for `VehicleAdded` / `VehicleUpdated` | published, never consumed — see §3.27 and **Q-3** |
| Outbound (sync) | **nobody** | `httpClient.services` is `{}` |

---

## 2. Core concepts (exhaustive)

Every distinct internal mechanism this service implements or deliberately omits. **Owner** names the
file that *defines* the concept.

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | `Vehicle` — the entity, and what it is not | `…Core/Entities/Vehicle.cs` | §3.1 |
| 2 | Four-project layering and the dependency-free `Core` | the four `*.csproj` files | §3.2 |
| 3 | `Variants` — the `[Flags]` bitmask | `…Core/Entities/Variants.cs` | §3.3 |
| 4 | The Standard floor — every vehicle is silently `Standard` | `…Core/Entities/Vehicle.cs:27` | §3.4 |
| 5 | The params-constructor chain and the re-floor on rehydration | `…Core/Entities/Vehicle.cs:30-35`; `…Infrastructure/Mongo/Documents/Extensions.cs:9-18` | §3.5 |
| 6 | `ChangeDescription` / `ChangePricePerService` — validating mutators | `…Core/Entities/Vehicle.cs:37-55` | §3.6 |
| 7 | `ChangeVariants` / `AddVariants` / `RemoveVariants` — three write modes, one unused | `…Core/Entities/Vehicle.cs:57-74` | §3.7 |
| 8 | Capacity validation — the ternary-throw idiom | `…Core/Entities/Vehicle.cs:24-25` | §3.8 |
| 9 | `DomainException` and the three domain error codes | `…Core/Exceptions/*.cs` | §3.9 |
| 10 | `IVehiclesRepository` — the Core-owned port | `…Core/Repositories/IVehiclesRepository.cs` | §3.10 |
| 11 | `AddVehicle` — id generation inside the command | `…Application/Commands/AddVehicle.cs:22` | §3.11 |
| 12 | `UpdateVehicle` — the immutability boundary | `…Application/Commands/UpdateVehicle.cs` | §3.12 |
| 13 | `DeleteVehicle` — the constructor-parameter-name anomaly | `…Application/Commands/DeleteVehicle.cs:11-12` | §3.13 |
| 14 | Command handlers — write-then-publish, and the missing atomicity | `…Application/Commands/Handlers/*.cs` | §3.14 |
| 15 | `AppException` / `VehicleNotFoundException` | `…Application/Exceptions/*.cs` | §3.15 |
| 16 | Queries — `GetVehicle`, `SearchVehicles`, `PagedQueryBase` | `…Application/Queries/*.cs` | §3.16 |
| 17 | `SearchVehiclesHandler` — the two-branch filter | `…Infrastructure/Mongo/Queries/Handlers/SearchVehiclesHandler.cs:18-34` | §3.17 |
| 18 | The exact-match variants filter — a silently empty result set | same, `:27-29` | §3.18 |
| 19 | `VehicleDto` and the `Variants` string split | `…Application/DTO/VehicleDto.cs`; `…Infrastructure/Mongo/Documents/Extensions.cs:49` | §3.19 |
| 20 | `VehicleDocument` — the persisted shape | `…Infrastructure/Mongo/Documents/VehicleDocument.cs` | §3.20 |
| 21 | `AsEntity` / `AsDocument` / `AsDto` — the three mappings | `…Infrastructure/Mongo/Documents/Extensions.cs` | §3.21 |
| 22 | Enum-as-ordinal persistence — no `[BsonRepresentation]` | `…Infrastructure/Mongo/Documents/VehicleDocument.cs:16` | §3.22 |
| 23 | `VehiclesMongoRepository` — the port adapter | `…Infrastructure/Mongo/Repositories/VehiclesMongoRepository.cs` | §3.23 |
| 24 | Mongo database and the `"vehicles"` collection literal | `…Infrastructure/Extensions.cs:72`; `appsettings.json:95-99` | §3.24 |
| 25 | Absence of optimistic concurrency — last-writer-wins | `…Core/Entities/Vehicle.cs`; `VehicleDocument.cs` | §3.25 |
| 26 | `ExceptionToResponseMapper` — everything is HTTP 400 | `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` | §3.26 |
| 27 | Integration events — three published, one consumed | `…Application/Events/*.cs` | §3.27 |
| 28 | Rejected events and `ExceptionToMessageMapper` — the null drop | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` | §3.28 |
| 29 | `MessageBroker` — the publish pipeline | `…Infrastructure/Services/MessageBroker.cs` | §3.29 |
| 30 | Outbox, inbox, and `disableTransactions: true` | `…Infrastructure/Decorators/*.cs`; `appsettings.json:100-108` | §3.30 |
| 31 | RabbitMQ wiring — exchange, queue template, conventions, Jaeger plugin | `…Infrastructure/Extensions.cs:63`; `appsettings.json:109-150` | §3.31 |
| 32 | `SubscribeCommand<T>()` — the AMQP command surface | `…Infrastructure/Extensions.cs:86-88` | §3.32 |
| 33 | Dispatcher-bound HTTP endpoints and `afterDispatch` | `…Api/Program.cs:33-41` | §3.33 |
| 34 | `[Contract]` and `UsePublicContracts` | `…Application/ContractAttribute.cs`; `…Infrastructure/Extensions.cs:83` | §3.34 |
| 35 | `MessageToLogTemplateMapper` and handler logging | `…Infrastructure/Logging/*.cs` | §3.35 |
| 36 | Redis — configured with no consumer | `…Infrastructure/Extensions.cs:67`; `appsettings.json:151-154` | §3.36 |
| 37 | Log redaction, path exclusion and sinks | `appsettings.json:32-67` | §3.37 |
| 38 | `Correlation-Context` ingestion, `IAppContext`, `IIdentityContext` | `…Infrastructure/Contexts/*.cs`; `…Infrastructure/Extensions.cs:93-96` | §3.38 |
| 39 | `Saga` header forwarding and `span_context` propagation | `…Infrastructure/Extensions.cs:98-127` | §3.39 |
| 40 | The caller context is built on every request and consumed by nothing | `…Infrastructure/Extensions.cs:51-52` | §3.40 |
| 41 | Inert JWT configuration and the absent certificate authentication | `appsettings.json:77-85`; `…Infrastructure/Extensions.cs:74` | §3.41 |
| 42 | Consul registration and Fabio addressing | `…Infrastructure/Extensions.cs:61-62`; `appsettings.json:7-22` | §3.42 |
| 43 | Vault — KV, PKI and the dynamic Mongo credential lease | `…Api/Program.cs:43`; `appsettings.json:164-193` | §3.43 |
| 44 | Environment layering — four profiles, and what `docker` inherits | `…Api/appsettings*.json` | §3.44 |
| 45 | Metrics, tracing, and the duplicated `AddMongo()` | `…Infrastructure/Extensions.cs:66,70` | §3.45 |
| 46 | Absence of a test suite, and the green-but-empty test step | `scripts/test.sh`; `.travis.yml` | §3.46 |
| 47 | Deployment identity and downstream consumers | `Dockerfile`; `hianshul100_Pacco/*.yml`; `ntrada*.yml` | §3.47 |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement**, **Extension procedure**, **Failure modes**.

### 3.1 `Vehicle` — the entity, and what it is not

**Definition.** The single domain type. Eight properties, all `public … { get; protected set; }`
(`…Core/Entities/Vehicle.cs:8-15`): `Id` (`Guid`), `Brand`, `Model`, `Description` (`string`),
`PayloadCapacity`, `LoadingCapacity` (`double`), `PricePerService` (`decimal`), `Variants`
(`Variants`).

**Representation & storage.** In memory it is a plain class with no base type. On disk it becomes a
`VehicleDocument` (§3.20). It is never serialised directly: every crossing of a boundary goes through
an explicit mapping (§3.21).

**Lifecycle.** Constructed by `AddVehicleHandler` (`…Application/Commands/Handlers/AddVehicleHandler.cs:26`)
or rehydrated by `AsEntity` (`…Infrastructure/Mongo/Documents/Extensions.cs:9-18`); mutated only by
`UpdateVehicleHandler` (`:29-31`); discarded by `DeleteVehicleHandler`. There is no in-memory cache,
so an entity lives exactly as long as one request.

**Invariants & enforcement.** `protected set` means nothing outside the class (and nothing at all,
since there is no subclass) can assign a property. Every write goes through a method that validates
first (§3.6, §3.8). This is real encapsulation and it holds — with one hole: **`Brand` and `Model`
have no validating mutator and no constructor check.** Both are assigned directly at
`…Core/Entities/Vehicle.cs:21-22` with no null or empty test, so `AddVehicle` with
`"brand": ""` succeeds and persists an empty brand. Fails **silently**.

**Extension procedure.** To add a property: add it to the entity with `protected set`; assign it in
the 7-arg constructor (`:17-28`) and thread it through the 8-arg overload (`:30-35`); add a
`ChangeX` mutator if it is meant to be updatable; add the field to `VehicleDocument`, `AsEntity`,
`AsDocument` and `AsDto` (§3.21) — **all four**, or the value round-trips as `default`; add it to
`AddVehicle`/`UpdateVehicle` and to the handler call sites. Nine files minimum. Nothing in the
codebase detects a mapping you forgot (§3.21, failure modes).

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| Empty or null `Brand`/`Model` persisted | no validation on those two fields (`:21-22`) |
| A new field silently reads back as `null`/`0` | one of the four mapping sites not updated (§3.21) |
| Two concurrent updates, one lost | no version/concurrency token (§3.25) |

### 3.2 Four-project layering and the dependency-free `Core`

**Definition.** The repository realises [[inward-dependency-service-skeleton]] in its purest form in
this inventory. Four projects, each referencing exactly one inward neighbour:

| Project | ProjectReference | Package references |
| --- | --- | --- |
| `Pacco.Services.Vehicles.Core` | *(none)* | **zero** — `Core/Pacco.Services.Vehicles.Core.csproj` is 7 lines, a `<PropertyGroup>` and nothing else |
| `…Application` | → `Core` | 6 (`Convey`, `Convey.CQRS.Commands/Events/Queries`, `Convey.MessageBrokers`, `Convey.WebApi`) |
| `…Infrastructure` | → `Application` | 21, incl. `Convey.Persistence.MongoDB`, `Convey.Persistence.Redis`, `Convey.MessageBrokers.RabbitMQ`, `Convey.MessageBrokers.Outbox.Mongo`, `Convey.Logging.CQRS`, `Convey.Security`, `Convey.Tracing.Jaeger.RabbitMQ` |
| `…Api` | → `Infrastructure` | 3 |

**Representation & storage.** The layering is a compile-time fact enforced by the `.csproj` graph;
there is no architecture test asserting it (there is no test project at all — §3.46).

**Lifecycle.** Fixed at build time. `Api/Program.cs` composes the whole graph at startup by calling
`AddInfrastructure()`/`UseInfrastructure()` (`…Api/Program.cs:24,28`), which internally call
`AddApplication()` (`…Application/Extensions.cs:9-14`).

**Invariants & enforcement.** The invariant "`Core` depends on nothing" is enforced by the compiler:
a `Core` file cannot reference `Convey`, MongoDB or ASP.NET types because those assemblies are not on
its reference path. **This is why `Core` is the one place in the service where a change cannot be
accidentally coupled to infrastructure**, and it is worth preserving. Contrast `pricing-service`,
which collapses the same four names into folders inside one project and consequently has its HTTP
client type visible from its domain service (`component-internals/pricing-service.md` §3.18).

**Extension procedure.** New domain rules go in `Core` and must compile with zero packages. If a rule
needs I/O, express the I/O as an interface in `Core/Repositories` or a new `Core/Services` folder and
implement it in `Infrastructure` — that is exactly what `IVehiclesRepository` does (§3.10). Never add
a `PackageReference` to `Core.csproj`: doing so removes the only structural guarantee this service
has.

**Failure modes.** A maintainer who adds a package to `Core` gets a green build and destroys the
invariant with no warning. Fails **silently**, permanently, and is invisible in code review unless the
`.csproj` diff is read.

### 3.3 `Variants` — the `[Flags]` bitmask

**Definition.** `…Core/Entities/Variants.cs:5-13` declares a `[Flags]` enum with five members:
`Standard = 1 << 0`, `Chemistry = 1 << 1`, `Explosives = 1 << 2`, `Food = 1 << 3`, `Organ = 1 << 4` —
values 1, 2, 4, 8, 16. A vehicle's certification is the OR of the variants it can carry. This is the
service's only genuine piece of domain modelling.

**Representation & storage.** In memory: a single `Variants`-typed field on the entity. On the wire
inbound: a **single integer** — `AddVehicle.Variants` and `UpdateVehicle.Variants` are each one
`Variants Variants { get; }` (`…Application/Commands/AddVehicle.cs:16`,
`…Application/Commands/UpdateVehicle.cs:13`), and the sample requests send `"variants": 1` and
`"variants": 2` (`Pacco.Services.Vehicles.rest:14,26`). On disk: an **ordinal int32** (§3.22). On the
wire outbound: an `IEnumerable<string>` produced by splitting `ToString()` (§3.19). Four different
representations of one value, converted at four different places.

**Lifecycle.** Client int → command → `new Vehicle(…, variants)` → `AddVariants` OR-ing (§3.4) →
document int → search filter compares ints (§3.18) → `AsDto` splits the string (§3.19).

**Invariants & enforcement.** **There is no validation of the integer at any point.** `Variants` is
an enum-typed property bound by the framework from a JSON number; a request with `"variants": 999`
binds to the enum without error (C# enums are not range-checked on cast), is stored as `999`, and
`ToString()` on it produces `"999"` — which `AsDto` then returns as the single-element array
`["999"]`. Fails **silently**, all the way to the client. `0` is likewise accepted and then
immediately overwritten to `1` by the Standard floor (§3.4).

**Extension procedure.** To add a sixth variant, append `NewName = 1 << 5` to the enum. **Do not
renumber or reorder the existing members** — the ordinal is the persisted value (§3.22), so
renumbering silently reinterprets every stored document. Adding at the end is safe. Also update the
gateway's documentation if any exists and tell every consumer that `AsDto` may now emit a new string.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A stored vehicle reports a variant nobody defined | out-of-range int accepted unvalidated |
| Every stored vehicle's variants shift meaning after a release | enum members renumbered; persisted as ordinal (§3.22) |
| Search by a variant returns nothing | the Standard floor plus exact-match filter (§3.4 + §3.18) |

### 3.4 The Standard floor — every vehicle is silently `Standard`

**Definition.** The most consequential line in the repository. The 7-argument constructor ends with
`AddVariants(Variants.Standard);` (`…Core/Entities/Vehicle.cs:27`), **after** the caller-supplied
variants have already been assigned by the 8-argument overload's chain (§3.5). Because `AddVariants`
ORs (`Variants |= …`, `:60-66`), bit 0 is set on **every vehicle that has ever been constructed**.

**Representation & storage.** The effect is persisted, not derived: `AsDocument` writes
`vehicle.Variants` (`…Infrastructure/Mongo/Documents/Extensions.cs:23-34`) after the constructor has
already OR-ed the bit in.

**Lifecycle.** Applied at construction and at every rehydration (§3.5), so a document that somehow
lacked bit 0 acquires it the first time it is read and re-saved.

**Invariants & enforcement.** The de-facto invariant is *`Variants & Standard != 0` always*. It is
enforced, but nowhere stated — no comment, no named method, no test. And it is enforced in a way that
**silently contradicts the caller**: a client posting `"variants": 2` (Chemistry) gets a vehicle
stored with `3` (Standard | Chemistry).

**The trap.** Combine that with the exact-match search filter (§3.18):

1. `POST /vehicles` with `"variants": 2` → stored `Variants = 3`.
2. `GET /vehicles?variants=2` → filter is `v.Variants == 2`
   (`…Infrastructure/Mongo/Queries/Handlers/SearchVehiclesHandler.cs:28-29`) → **no match**.
3. The client gets an empty page, HTTP 200, no error.

**A vehicle can never be found by the variant it was created with**, unless the client asks for
`Standard | thatVariant`. This is a live, reproducible defect and it is invisible from any surface
catalogue: both endpoints work, both return 200, and the data is present. It is only visible by
reading the constructor and the filter together.

**Extension procedure.** Three options, in increasing order of correctness: (a) change the filter to
a bitmask containment test (`(v.Variants & query.Variants) == query.Variants`) — a query-side fix,
no migration needed, and it makes the floor harmless; (b) delete `:27` and let the caller decide —
requires a data migration to add bit 0 to the vehicles that relied on it, or accept that existing
data already has it; (c) keep both and document the floor. **(a) is the recommended fix**: it is one
line, it changes no stored data, and it is the semantics a `[Flags]` enum implies.

**Failure modes.** Searching by a single non-Standard variant returns an empty result set with a 200
status. Fails **silently**. A client cannot distinguish "no such vehicle" from "your query can never
match".

### 3.5 The params-constructor chain and the re-floor on rehydration

**Definition.** `Vehicle` has two constructors. The 8-argument one
(`…Core/Entities/Vehicle.cs:30-35`) takes `params Variants[] variants`, chains to the 7-argument one
passing `Variants.Standard` as the seed, then calls `AddVariants(variants)` (`:34`), which folds the
array with `|=`.

**Representation & storage.** No storage of its own. It exists so callers can pass a variadic list.

**Lifecycle.** Called from exactly one place: `AsEntity`
(`…Infrastructure/Mongo/Documents/Extensions.cs:9-18`) passes `document.Variants` — a **single**
`Variants` value — into the `params` slot, which C# implicitly wraps as a one-element array.
`AddVehicleHandler` uses the 7-argument form instead (`AddVehicleHandler.cs:26`).

**Invariants & enforcement.** The chain guarantees the Standard bit is set twice over: once by the
seed, once by `:27`. On rehydration this means **a stored `Variants` value is re-floored on every
read**. If a migration ever cleared bit 0 in the database, the next read-modify-write would put it
back. The floor is therefore not merely a creation-time default; it is a repair loop.

**Extension procedure.** If §3.4(b) is chosen (removing the floor), **both** the `:27` call and the
`Variants.Standard` seed at `:32` must go, or the floor survives via rehydration and the bug appears
to be unfixed.

**Failure modes.** A data migration that strips bit 0 appears to succeed and then silently reverts,
document by document, as vehicles are updated. Fails **silently** and non-atomically — the collection
ends up in a mixed state.

### 3.6 `ChangeDescription` / `ChangePricePerService` — validating mutators

**Definition.** Two guarded write methods on the entity.

| Method | Guard | Throws | Code |
| --- | --- | --- | --- |
| `ChangeDescription(string description)` (`…Core/Entities/Vehicle.cs:37-45`) | `string.IsNullOrEmpty(description)` | `InvalidVehicleDescriptionException` | `invalid_vehicle_description` |
| `ChangePricePerService(decimal pricePerService)` (`:47-55`) | `pricePerService <= 0` | `InvalidVehiclePricePerServiceException` | `invalid_vehicle_price_per_service` |

**Representation & storage.** They assign the `protected set` properties directly.

**Lifecycle.** Called from the 7-argument constructor (`:23,26`) — so creation and update run the
*same* validation — and from `UpdateVehicleHandler` (`…Application/Commands/Handlers/UpdateVehicleHandler.cs:29-30`).

**Invariants & enforcement.** "Description is non-empty" and "price is strictly positive" hold for
every `Vehicle` instance that exists, because there is no path to the fields that bypasses these
methods. Both fail **loudly**: a `DomainException` (§3.9) surfaces as HTTP 400 with the exception's
`Code` over HTTP (§3.26), or as a `*Rejected` event over AMQP (§3.28). This is the correct pattern
and the one to copy.

One gap worth naming: the description guard is `IsNullOrEmpty`, **not** `IsNullOrWhiteSpace`
(`…Core/Entities/Vehicle.cs:39`). A description of `" "` — a single space — passes validation and is
persisted. `Brand` and `Model` have no guard at all (§3.1), so the practical rule is that three of
the four string-ish fields are effectively unvalidated against blank input.

**Extension procedure.** For a new validated field, follow this shape exactly: private guard, throw a
`DomainException` subclass with a snake_case `Code`, assign, and call the method from both the
constructor and the update handler. Do **not** validate in the command or the handler — that is the
mistake `pricing-service` makes with its unvalidated DTO (`component-internals/pricing-service.md`
§3.11).

**Failure modes.** None internal. Externally: because `ExceptionToResponseMapper` maps every domain
exception to **400** (§3.26), a caller cannot distinguish "price invalid" from "vehicle not found"
by status code — only by reading the `code` field in the body.

### 3.7 `ChangeVariants` / `AddVariants` / `RemoveVariants` — three write modes, one unused

**Definition.** Three different ways to write the bitmask:

| Method | Semantics | Called from |
| --- | --- | --- |
| `ChangeVariants(Variants variants)` (`…Core/Entities/Vehicle.cs:57-58`) | **overwrite** — expression-bodied, `Variants = variants` | `UpdateVehicleHandler.cs:31` |
| `AddVariants(params Variants[] variants)` (`:60-66`) | **OR in** — `foreach … Variants |= variant` | the constructors (`:27,34`) |
| `RemoveVariants(params Variants[] variants)` (`:68-74`) | **AND-NOT out** — `Variants &= ~variant` | **nowhere** |

**Representation & storage.** All three mutate the same field.

**Lifecycle.** `AddVariants` runs at construction and rehydration; `ChangeVariants` runs on update;
`RemoveVariants` never runs.

**Invariants & enforcement.** **None of the three validates anything** — no check that the value is a
defined enum member, no check that the result is non-zero. Critically, `ChangeVariants` **overwrites
rather than ORs**, so `PUT /vehicles/{id}` with `"variants": 2` stores exactly `2` — with the
Standard bit **cleared**. This is the one place a vehicle can end up without bit 0, contradicting
§3.4's invariant, and it means **create and update have different variant semantics for the same
field**: create ORs Standard in, update does not. A vehicle created with `"variants": 2` (stored `3`)
and then updated with `"variants": 2` (stored `2`) changes its search behaviour without the client
doing anything different.

`RemoveVariants` is dead code: verified by absence — no call site exists in the repository.

**Extension procedure.** If a "remove a variant" operation is wanted, the entity method already
exists; add a command, a handler, a route and an event. If instead update should behave like create,
change `UpdateVehicleHandler.cs:31` to `AddVariants` — but note that then a variant can never be
un-set, which is presumably why `ChangeVariants` was written.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A vehicle stops matching `?variants=3` after an unrelated update | `ChangeVariants` overwrote the Standard bit |
| `AsDto` returns `["0"]` for a vehicle | updated with `"variants": 0`; overwrite leaves the field zero (§3.19) |
| Dead method drifts out of sync with the others | no caller, no test |

### 3.8 Capacity validation — the ternary-throw idiom

**Definition.** The two capacities are validated inline in the constructor using a conditional
expression whose false branch throws (`…Core/Entities/Vehicle.cs:24-25`): the assignment is
`PayloadCapacity = payloadCapacity > 0 ? payloadCapacity : throw new InvalidVehicleCapacity(payloadCapacity)`,
and the same shape for `LoadingCapacity` at `:25`. Note the exception carries **only the capacity**,
not the vehicle id (`…Core/Exceptions/InvalidVehicleCapacity.cs:7-8`), so its message reads
`"Vehicle capacity is invalid: 0."` with no way to tell which vehicle or which of the two capacities
failed.

**Representation & storage.** `double` on the entity and on the document.

**Lifecycle.** Runs on every construction — including **every rehydration from Mongo** via `AsEntity`
(§3.5). That is the interesting part: the guard is not a create-time-only check, it is a read-time
check too.

**Invariants & enforcement.** "Both capacities are strictly positive" holds for every live `Vehicle`.
Fails **loudly** with `InvalidVehicleCapacity` (code `invalid_vehicle_capacity`,
`…Core/Exceptions/InvalidVehicleCapacity.cs:3-11`).

The consequence worth spelling out: **if a document with a non-positive capacity ever reaches the
database — by manual edit, by migration, or by a future code path that bypasses the constructor —
every subsequent read of that document throws.** `GetVehicle` for it returns 400 forever;
`SearchVehicles` does *not*, because the search path never constructs an entity (it maps
`VehicleDocument` straight to `VehicleDto` — §3.17), so the poisoned row appears in listings but
cannot be fetched or updated. That asymmetry is a genuinely confusing failure and is worth
remembering.

**Extension procedure.** New numeric constraints should use the same idiom for consistency. If the
read-time strictness is unwanted, move the guard out of the constructor into an explicit
`Validate()` called only from the command handlers — but that weakens the entity's guarantee, so
prefer keeping it.

**Failure modes.** A single bad document makes one vehicle permanently unreadable and unfixable
through the API (the update path must read before it writes —
`…Application/Commands/Handlers/UpdateVehicleHandler.cs:23`). Repair requires a direct database edit.

### 3.9 `DomainException` and the domain error codes

**Definition.** `…Core/Exceptions/DomainException.cs:5-12` is an `abstract class DomainException :
Exception` carrying `public virtual string Code { get; }` (`:7`) and a protected constructor taking
the message. Three concrete subclasses live beside it, each overriding `Code` with a snake_case
literal:

| Exception | `Code` | Message shape | Evidence |
| --- | --- | --- | --- |
| `InvalidVehicleCapacity` | `invalid_vehicle_capacity` | `Vehicle capacity is invalid: {capacity}.` | `InvalidVehicleCapacity.cs:5,8` |
| `InvalidVehicleDescriptionException` | `invalid_vehicle_description` | `Vehicle description is invalid: {description}.` | `InvalidVehicleDescriptionException.cs:5,8` |
| `InvalidVehiclePricePerServiceException` | `invalid_vehicle_price_per_service` | `Vehicle price per service is invalid: {pricePerService}.` | `InvalidVehiclePricePerServiceException.cs:5,8` |

`…Application/Exceptions/AppException.cs:5-12` is the identical shape one layer out, with
`VehicleNotFoundException` → `vehicle_not_found` (`VehicleNotFoundException.cs:7`) as its only
subclass. Two parallel abstract bases with the same `virtual Code` member and no shared ancestor —
the response mapper has to switch on both (§3.26).

**Representation & storage.** Not stored. The `Code` string is the stable, machine-readable contract;
the `Message` is human-readable and interpolates the offending values. Note that
`InvalidVehicleCapacity` is the one exception in the repository whose name does **not** end in
`Exception` — a naming inconsistency that happens to be harmless (§3.26 explains why).

**Lifecycle.** Thrown in `Core` or `Application`, propagated up through the handler, caught by
Convey's error-handling middleware and translated by `ExceptionToResponseMapper` (§3.26) or, on the
AMQP path, by `ExceptionToMessageMapper` (§3.28).

**Invariants & enforcement.** `Code` is **`virtual`, not `abstract`** — the base supplies an
auto-property with no assignment, so it returns `null`. A new subclass that forgets to override it
therefore **compiles**. The compiler does not enforce the error taxonomy; `ExceptionToResponseMapper.GetCode`
covers for it by falling back to `exception.GetType().Name.Underscore().Replace("_exception", "")`
when `Code` is null or whitespace (`…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:36-39`).
So an un-overridden `Code` degrades to a derived code rather than to `null` — recovery, not
enforcement, and it means the HTTP contract of a new exception is its **class name** until someone
notices.

**Extension procedure.** Subclass `DomainException` in `Core/Exceptions`, supply a snake_case `Code`,
throw it from the entity. Then — and this is the step that gets missed — decide whether the AMQP path
needs it: `ExceptionToMessageMapper` (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:12-40`)
handles exceptions by **type**, so a new exception type is **not** mapped and its AMQP failures are
silently dropped (§3.28). Adding the type to that switch is mandatory, not optional.

**Failure modes.** A new domain exception works correctly over HTTP and vanishes over AMQP. Fails
**silently** on exactly one of the two transports — the hardest kind of bug to notice, because the
HTTP tests pass.

### 3.10 `IVehiclesRepository` — the Core-owned port

**Definition.** `…Core/Repositories/IVehiclesRepository.cs:7-13` — four members:
`Task<Vehicle> GetAsync(Guid id)`, `Task AddAsync(Vehicle vehicle)`, `Task UpdateAsync(Vehicle vehicle)`,
`Task DeleteAsync(Guid id)`.

**Representation & storage.** An interface in `Core`, implemented in `Infrastructure` by
`VehiclesMongoRepository` (§3.23) and registered as a transient at
`…Infrastructure/Extensions.cs:49`.

**Lifecycle.** Resolved per handler invocation.

**Invariants & enforcement.** The port is **write-model only**: it has no browse or list method. That
is deliberate and it is the [[database-per-service-with-document-mapping]] split — reads bypass the
domain entirely and query documents (§3.17). The invariant "no query ever constructs a `Vehicle`" is
maintained by the fact that the query handlers depend on `IMongoRepository<VehicleDocument, Guid>`
directly, not on this interface (`…Infrastructure/Mongo/Queries/Handlers/GetVehicleHandler.cs:18-22`).

Note that `DeleteAsync` takes a `Guid`, not a `Vehicle` — so deletion does not require the entity to
be constructible. But `DeleteVehicleHandler` reads the vehicle first anyway
(`…Application/Commands/Handlers/DeleteVehicleHandler.cs:23-27`) in order to raise
`VehicleNotFoundException`, which re-introduces the §3.8 read-time throw on the delete path.

**Extension procedure.** New write operations belong here and only here. Do **not** add browse
methods to this interface: that would put `PagedResult` (a Convey type) into `Core` and break the
zero-dependency invariant (§3.2). Reads go through `Infrastructure/Mongo/Queries`.

**Failure modes.** None internal. The interface is a straight pass-through with no behaviour of its
own.

### 3.11 `AddVehicle` — id generation inside the command

**Definition.** `…Application/Commands/AddVehicle.cs` — `[Contract]` (`:7`),
`ICommand`, eight **get-only** properties (`:10-17`): `VehicleId`, `Brand`, `Model`, `Description`,
`PayloadCapacity`, `LoadingCapacity`, `PricePerService`, `Variants`. Its constructor contains the one
piece of logic any command in this service has:
`VehicleId = vehicleId == Guid.Empty ? Guid.NewGuid() : vehicleId;` (`:22`).

**Representation & storage.** Not stored. Bound from a JSON body by the WebApi dispatcher on the HTTP
path, or deserialised from an AMQP payload on the async path (§3.32).

**Lifecycle.** Constructed by the framework per request; consumed by `AddVehicleHandler`; discarded.

**Invariants & enforcement.** "Every accepted `AddVehicle` has a non-empty id" is enforced here, and
it is what makes the create path **caller-supplied-id-capable**: a caller may pass its own `Guid` for
idempotency, or omit it and let the service mint one. The gateway uses the second form and relies on
the mint — `ntrada.yml` declares `resourceId: {property: vehicleId, generate: true}` on the POST
route (`:437-439`), which generates the id **at the gateway** so it can return a `Location` header
before the service has replied. `[ntrada]` **`Unverifiable — Missing Source Evidence`** — whether
Ntrada's `generate: true` injects the generated value into the forwarded body (making `:22`'s
fallback dead on that path) or only uses it for the response header cannot be settled without
Ntrada's source, which is not in this workspace.

The id-generation branch is also the reason `AddVehicle` is **not idempotent over AMQP by default**:
if the caller omits `vehicleId`, a redelivered message mints a *different* id and creates a *second*
vehicle. The inbox would normally prevent redelivery, but see §3.30 for why it does not on the HTTP
path.

**Extension procedure.** Add the property to the command (get-only), to the constructor parameter
list **and body**, and to the `new Vehicle(…)` call in the handler. Get-only properties are
significant: the binder must use the constructor, which is what makes §3.13's parameter-name
mismatch fatal.

**Failure modes.** Omitting `vehicleId` on a retried message creates duplicates. Fails **silently** —
both vehicles are valid.

### 3.12 `UpdateVehicle` — the immutability boundary

**Definition.** `…Application/Commands/UpdateVehicle.cs` — `[Contract]`, `ICommand`, and only **four**
get-only properties (`:10-13`): `VehicleId`, `Description`, `PricePerService`, `Variants`.

**Representation & storage.** Not stored.

**Lifecycle.** Bound → `UpdateVehicleHandler` → three entity mutators → `UpdateAsync` →
`VehicleUpdated`.

**Invariants & enforcement.** The set of updatable fields is defined by *omission*: `Brand`, `Model`,
`PayloadCapacity` and `LoadingCapacity` are **immutable after creation** because no command carries
them and no route accepts them. This is a real design decision — a vehicle's physical identity and
capacity are fixed; its description, price and certifications are not — but it is expressed nowhere
except as four absent properties. There is no comment and no test.

The enforcement is total on the write path: `UpdateVehicleHandler`
(`…Application/Commands/Handlers/UpdateVehicleHandler.cs:29-31`) calls exactly `ChangeDescription`,
`ChangePricePerService` and `ChangeVariants`, and the entity exposes no other mutator (§3.7), so even
a buggy handler could not change the brand.

**Extension procedure.** To make a field updatable, add it to `UpdateVehicle`, add a `ChangeX`
validating mutator to `Vehicle` (§3.6), call it from the handler, and confirm the gateway's PUT route
forwards the body unchanged (it does — `ntrada.yml:441-445` has no `bind` on the body). To make a
field *immutable*, the reverse. Note that a field added to `UpdateVehicle` but not to `AddVehicle`
would be unsettable at creation and therefore always `default` until first update.

**Failure modes.** A client that PUTs a full vehicle representation including `brand` gets a **200
and a silently ignored brand** — the property does not exist on the command, so the binder drops it.
Fails **silently**, and looks like a lost update.

### 3.13 `DeleteVehicle` — the constructor-parameter-name anomaly

**Definition.** `…Application/Commands/DeleteVehicle.cs` — `[Contract]`, `ICommand`, one get-only
property `Guid VehicleId { get; }`, and an expression-bodied constructor:
`public DeleteVehicle(Guid id) => VehicleId = id;` (`:11-12`).

The parameter is named **`id`**; the property is named **`VehicleId`**. Every other command in this
repository names the constructor parameter to match its property in camelCase
(`AddVehicle.cs:19-30`, `UpdateVehicle.cs`). This is the only one that does not.

**Representation & storage.** Not stored.

**Lifecycle.** HTTP: `Delete<DeleteVehicle>("vehicles/{vehicleId}")` (`…Api/Program.cs:40`) — the
route template segment is `{vehicleId}`, and Convey's WebApi binder maps route values onto the
command. AMQP: `SubscribeCommand<DeleteVehicle>()` (`…Infrastructure/Extensions.cs:88`) deserialises a
JSON payload; the gateway's async route publishes `{"vehicleId": "…"}` after
`bind: - vehicleId:{vehicleId}` (`ntrada-async.yml:521-529`).

**Invariants & enforcement.** **This is the concept `pricing-service.md` §3.14 forward-references, and
it is the sharpest binding trap in the inventory.** Because all properties are get-only, the command
can only be materialised through the constructor, so **the deserialiser must match the JSON/route key
to the constructor parameter name, not to the property name** — that is how `System.Text.Json` and
`Newtonsoft.Json` both behave for parameterised construction. The incoming key is `vehicleId`; the
parameter is `id`.

Whether that binds depends entirely on the binder Convey installs and its name-matching policy:

| Binder behaviour | Result |
| --- | --- |
| Matches constructor parameters to JSON properties by name | `id` ≠ `vehicleId` → parameter defaults to `Guid.Empty` → `VehicleId = Guid.Empty` |
| Falls back to setting properties directly (reflection over private setters) | binds correctly |
| Matches by position for a single-parameter constructor | binds correctly |

`[convey]` **`Unverifiable — Missing Source Evidence`** — Convey 0.4 is NuGet-only in this workspace,
so which of the three applies cannot be settled from source here. The reason it matters: if the first
row holds, **`DELETE /vehicles/{id}` and the async `delete_vehicle` command both resolve to
`Guid.Empty`**, `DeleteVehicleHandler` reads nothing, throws `VehicleNotFoundException` for
`00000000-…`, and every delete returns 400 `vehicle_not_found` regardless of the id supplied.

That failure would be **loud and total** — impossible to miss in manual testing — which is indirect
evidence that the binder does *not* behave that way in practice, since the sample request file ships
a DELETE (`Pacco.Services.Vehicles.rest:31-36`). But indirect evidence is not source, so it is
recorded as **Q-1** rather than resolved. What is certain either way: the mismatch is latent, it is
unique to this command, and it is one binder upgrade away from breaking.

**Extension procedure.** Rename the parameter to `vehicleId`. It is a one-word change in one file
with no callers outside the binder (verified: no `new DeleteVehicle(` appears anywhere in the
repository), it removes the ambiguity permanently, and it makes the command consistent with the other
two. **Do this before adding any further get-only command.**

**Failure modes.** Delete silently targets the empty `Guid`, or delete works today and stops working
after a dependency bump, with no code change to blame.

### 3.14 Command handlers — write-then-publish, and the missing atomicity

**Definition.** Three handlers in `…Application/Commands/Handlers/`, each an
`ICommandHandler<T>` taking `IVehiclesRepository` and `IMessageBroker` and following the same
three-move shape.

| Handler | Moves | Evidence |
| --- | --- | --- |
| `AddVehicleHandler` | `new Vehicle(…)` → `AddAsync` → `PublishAsync(new VehicleAdded(command.VehicleId))` | `AddVehicleHandler.cs:21-35` |
| `UpdateVehicleHandler` | `GetAsync` → null ⇒ `throw new VehicleNotFoundException` → three mutators → `UpdateAsync` → `PublishAsync(new VehicleUpdated(...))` | `UpdateVehicleHandler.cs:21-35` |
| `DeleteVehicleHandler` | `GetAsync` → null ⇒ `throw new VehicleNotFoundException` → `DeleteAsync` → `PublishAsync(new VehicleDeleted(...))` | `DeleteVehicleHandler.cs:21-32` |

**Representation & storage.** Stateless; registered by `AddCommandHandlers()`
(`…Application/Extensions.cs:10`) and decorated by `OutboxCommandHandlerDecorator<>`
(`…Infrastructure/Extensions.cs:53`) — see §3.30.

**Lifecycle.** One instance per dispatch, from either transport.

**Invariants & enforcement.** Two invariants are intended here and only one holds.

1. *"A state change is always announced."* Enforced by convention only — every handler ends with a
   publish, but nothing checks it. A fourth handler that forgets to publish compiles and passes
   review easily. This is the load-bearing difference from [[aggregate-buffered-domain-events]]:
   where `customers-service` has the entity record the event so the handler cannot forget
   (`component-internals/customers-service.md` §3.2), here the handler *is* the only record.
2. *"The write and the announcement are atomic."* **Does not hold.** The repository write and the
   outbox insert are separate operations, and `outbox.disableTransactions: true`
   (`Api/appsettings.json:107`) removes the Mongo transaction that would have joined them (§3.30). A
   crash between `AddAsync` and `PublishAsync` leaves a vehicle in the database that no consumer ever
   hears about; a crash the other way is impossible only because the publish comes second.

Note also the **read-before-write on update and delete**: both call `GetAsync` first, which constructs
a `Vehicle` and therefore re-runs every constructor guard (§3.8). A vehicle with a corrupt stored
capacity cannot be updated *or* deleted through the API.

**Extension procedure.** Copy the shape exactly: repository call, then `PublishAsync`. If a new
command can fail in a new way, add the exception to `ExceptionToMessageMapper` (§3.28) in the same
change — the AMQP path is silent otherwise.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| Vehicle exists, `availability-service` never learned of its deletion | crash between `DeleteAsync` and the outbox write; no transaction (§3.30) |
| A new command's effects are invisible to consumers | handler omitted its publish; nothing enforces it |
| Update/delete returns 400 for an existing vehicle | constructor guard fired on rehydration (§3.8) |

### 3.15 `AppException` / `VehicleNotFoundException`

**Definition.** `…Application/Exceptions/AppException.cs:5-12` mirrors `DomainException` one layer
out — `abstract`, `public virtual string Code { get; }`, protected message constructor — and
`VehicleNotFoundException` (`:5-13`) is its only subclass, with `Code = "vehicle_not_found"` (`:7`), a
`Guid Id` property (`:8`) and the message `Vehicle not found: {id}.` (`:10`).

**Representation & storage.** Not stored.

**Lifecycle.** Thrown by `UpdateVehicleHandler` and `DeleteVehicleHandler` when `GetAsync` returns
null; **never** thrown by the query handlers (§3.16).

**Invariants & enforcement.** The split between `DomainException` (invariant violated) and
`AppException` (application-level precondition failed) is maintained correctly here, and mirrors the
same split in `customers-service` and `orders-service`. But the distinction is **erased at the
boundary**: `ExceptionToResponseMapper` maps both to HTTP 400 (§3.26), so a caller cannot tell "you
asked for something that does not exist" from "your data is invalid" by status code. `vehicle_not_found`
in particular ought to be a 404 and is a 400.

**Extension procedure.** New non-domain failures subclass `AppException` in `Application/Exceptions`
with a snake_case `Code`. Remember the `ExceptionToMessageMapper` step (§3.28).

**Failure modes.** A caller retrying on 4xx cannot distinguish retryable from permanent failures. The
only signal is the `code` string in the body.

### 3.16 Queries — `GetVehicle`, `SearchVehicles`, `PagedQueryBase`

**Definition.** Two queries in `…Application/Queries/`:

| Query | Shape | Evidence |
| --- | --- | --- |
| `GetVehicle` | `IQuery<VehicleDto>`, one settable `Guid VehicleId` | `GetVehicle.cs:7-10` |
| `SearchVehicles` | `PagedQueryBase, IQuery<PagedResult<VehicleDto>>`, with `double PayloadCapacity`, `double LoadingCapacity`, `Variants Variants` | `SearchVehicles.cs:7-12` |

`PagedQueryBase` is Convey's (`[convey]`) and supplies `Page`, `Results`, `OrderBy`, `SortOrder`.

**Representation & storage.** Not stored. Unlike the commands, query properties are **settable**, so
they bind from the query string without constructor involvement — which is why §3.13's trap does not
apply to reads.

**Lifecycle.** Bound from the query string by `Get<TQuery, TResult>` (`…Api/Program.cs:35-36`) →
dispatched by the in-memory query dispatcher → handled in `Infrastructure/Mongo/Queries/Handlers`.

**Invariants & enforcement.** **No query is validated.** A negative `PayloadCapacity`, a `Variants`
value of 999, a `Results` of 100000 — all accepted. The paging bounds are Convey's business
(`[convey]`, exact clamping behaviour **`Unverifiable — Missing Source Evidence`**). The absence of a
`Brand`/`Model` filter is also worth noting: **there is no way to search vehicles by brand or model**,
only by the two capacities and the variant bitmask, despite both fields being stored and returned.

**Extension procedure.** To add a filter: add a settable property to `SearchVehicles`, extend the
predicate in `SearchVehiclesHandler` (§3.17), and — critically — extend the **guard condition** at
`:21` too, or the new filter is ignored whenever the old three are unset. Add a Mongo index for the
new field; there is none for the existing ones either (§5.1).

**Failure modes.** An unvalidated query returns an empty page rather than an error. Fails
**silently**, every time (§3.18).

### 3.17 `SearchVehiclesHandler` — the two-branch filter

**Definition.** `…Infrastructure/Mongo/Queries/Handlers/SearchVehiclesHandler.cs:18-34`. The whole
body is a branch on whether *any* filter was supplied:

```csharp
PagedResult<VehicleDocument> pagedResult;
if (query.PayloadCapacity <= 0 && query.LoadingCapacity <= 0 && query.Variants <= 0)
{
    pagedResult = await _repository.BrowseAsync(_ => true, query);
}
else
{
    pagedResult = await _repository.BrowseAsync(v => v.PayloadCapacity >= query.PayloadCapacity
        && v.LoadingCapacity >= query.LoadingCapacity && v.Variants == query.Variants, query);
}

return pagedResult?.Map(d => d.AsDto());
```

**Representation & storage.** The handler depends on `IMongoRepository<VehicleDocument, Guid>`
directly — **not** on `IVehiclesRepository` (§3.10). Reads never touch the domain, so `AsDto` maps
document → DTO with no `Vehicle` in between (§3.21) and none of the constructor guards run.

**Lifecycle.** One dispatch per `GET /vehicles`.

**Invariants & enforcement.** The `else` branch is an **AND of three conditions**, and that is the
whole problem (§3.18): it is not "filter by whichever fields were supplied", it is "filter by all
three, using whatever defaults the unsupplied ones have". The two capacities degrade gracefully
because `>= 0` is always true; **`Variants` does not**, because `== 0` is true only for a vehicle with
no variants at all — and no such vehicle can exist (§3.4).

The `pagedResult?.Map(...)` null-conditional (`:33`) means a null result yields null rather than
throwing; whether `BrowseAsync` can return null is `[convey]`-internal and
**`Unverifiable — Missing Source Evidence`**.

**Extension procedure.** See §3.18 for the recommended fix. Any new filter must be added to **both**
branches' conditions.

**Failure modes.** Covered in §3.18.

### 3.18 The exact-match variants filter — a silently empty result set

**Definition.** The single clause `v.Variants == query.Variants`
(`…Infrastructure/Mongo/Queries/Handlers/SearchVehiclesHandler.cs:29`), read together with the
guard at `:21` and the Standard floor at `…Core/Entities/Vehicle.cs:27`.

**Representation & storage.** The comparison is an int-to-int equality in a Mongo predicate, because
`Variants` persists as an ordinal (§3.22).

**Lifecycle.** Every `GET /vehicles` with at least one non-default filter.

**Invariants & enforcement.** Two independent silent failures come out of this one line.

**Failure 1 — searching by capacity alone returns nothing.** `GET /vehicles?payloadCapacity=100`
supplies a capacity but no variants, so `query.Variants` is `0`, the guard at `:21` is false (payload
> 0), the `else` branch runs, and the predicate demands `v.Variants == 0`. No stored vehicle has
`Variants == 0` (§3.4). **Result: an empty page, HTTP 200.** The only searches that can return
anything are the fully-unfiltered one and those that name an exact variant bitmask.

**Failure 2 — searching by the variant you created with returns nothing.** As traced in §3.4: create
with `variants=2`, stored as `3`, search `variants=2` matches nothing.

The sample request file demonstrates Failure 2 without acknowledging it:
`Pacco.Services.Vehicles.rest:1-6` sends
`GET /vehicles?payloadCapacity=0&loadingCapacity=0&variants=1` — capacities deliberately zero and
`variants=1`, i.e. exactly the one query that works, because `1` is the Standard floor that every
vehicle carries. It is the only variant value for which exact match is reliable.

**Extension procedure — the recommended fix.** Change `:29` to bitmask containment and make the guard
per-field:

```csharp
pagedResult = await _repository.BrowseAsync(v =>
    (query.PayloadCapacity <= 0 || v.PayloadCapacity >= query.PayloadCapacity)
    && (query.LoadingCapacity <= 0 || v.LoadingCapacity >= query.LoadingCapacity)
    && (query.Variants <= 0 || (v.Variants & query.Variants) == query.Variants), query);
```

This collapses both branches into one, fixes both failures, requires no data migration, and makes the
Standard floor harmless. **`Unverifiable — Missing Source Evidence`** — whether the MongoDB LINQ
provider translates the bitwise `&` into a server-side `$bitsAllSet` query or forces client-side
evaluation is a driver question that cannot be settled from this workspace; if it does not translate,
the filter must be expressed with the driver's `Builders<T>.Filter.BitsAllSet` instead. Verify against
the actual driver version before shipping.

**Failure modes.** Both failures return **HTTP 200 with an empty `items` array**. There is no log
line, no metric, no exception. From the outside the service looks healthy and the catalogue looks
empty. This is the single highest-value thing in this document.

### 3.19 `VehicleDto` and the `Variants` string split

**Definition.** `…Application/DTO/VehicleDto.cs:6-16` — eight settable properties mirroring the
entity, except that `Variants` is an **`IEnumerable<string>`** (`:15`), produced by
`Variants = document.Variants.ToString().Split(',')`
(`…Infrastructure/Mongo/Documents/Extensions.cs:49`).

**Representation & storage.** Not stored; it is the outward shape of a vehicle on both read routes.

**Lifecycle.** Built by `AsDto` from a `VehicleDocument`, serialised to JSON by the dispatcher.

**Invariants & enforcement.** None — and the `ToString().Split(',')` produces three distinct
outputs that a consumer must handle, none of which is documented:

| Stored `Variants` | `ToString()` | `Split(',')` result |
| --- | --- | --- |
| `3` (Standard \| Chemistry) | `"Standard, Chemistry"` | `["Standard", " Chemistry"]` — **leading space on every element after the first** |
| `1` (Standard) | `"Standard"` | `["Standard"]` |
| `0` | `"0"` | `["0"]` — a number where a name is expected |
| `999` (undefined bits) | `"999"` | `["999"]` |

`Enum.ToString()` on a `[Flags]` enum joins with `", "` — a comma **and a space** — and `Split(',')`
splits on the comma only, leaving the space. So `["Standard", " Chemistry"]` is what every consumer
receives, and a naive `variants.Contains("Chemistry")` on the client **fails**. The `"0"` case is
reachable via `PUT` with `"variants": 0` (§3.7).

The asymmetry is the other half: the API **accepts an integer** and **returns an array of strings**.
`POST {"variants": 2}` then `GET` yields `["Standard", " Chemistry"]`. A client cannot round-trip its
own representation.

**Extension procedure.** Fix with `.Split(", ", StringSplitOptions.RemoveEmptyEntries)` or, better,
`document.Variants.ToString().Split(',').Select(v => v.Trim())`. **This is a breaking change to the
read contract** — any consumer that has compensated for the leading space will break — so it needs
coordination. The only known external consumer of these routes is `orders-service`, which reads
`PricePerService` and `Id` only (§3.47), so the practical blast radius is the gateway's clients.

**Failure modes.** Client-side variant matching fails on every element after the first. Fails
**silently** in the consumer, not here.

### 3.20 `VehicleDocument` — the persisted shape

**Definition.** `…Infrastructure/Mongo/Documents/VehicleDocument.cs:7-17` — `internal class
VehicleDocument : IIdentifiable<Guid>` with eight settable properties matching the entity field for
field, including `public Variants Variants { get; set; }` (`:16`) — the **domain enum type reused
directly in the persistence model**.

**Representation & storage.** One BSON document per vehicle in collection `vehicles` of database
`vehicles-service` (§3.24). `IIdentifiable<Guid>` (`[convey]`) is what lets `IMongoRepository` treat
`Id` as `_id`.

**Lifecycle.** Created by `AsDocument`, read by the repository and both query handlers, mapped out by
`AsEntity` or `AsDto`.

**Invariants & enforcement.** **There is no schema.** No `[BsonElement]`, no `[BsonId]`, no
`[BsonIgnoreExtraElements]`, no `[BsonRepresentation]` (§3.22), no `Version`. Field names are the C#
property names, so **renaming a property renames the stored field** and orphans every existing
document's data for that field (§5.2).

`internal` visibility is doing real work: the document type cannot leak out of the Infrastructure
assembly, so nothing in `Application` or `Core` can accidentally depend on the storage shape. That is
the [[database-per-service-with-document-mapping]] boundary, correctly drawn.

The reuse of the `Variants` enum in the document is the one place the boundary is soft: `Core` and
`Infrastructure` now share a type whose *ordinal values* are the persistence format (§3.22). Changing
the enum is a schema change disguised as a domain change.

**Extension procedure.** Add the property to the document **and** to all three mappings (§3.21).
Existing documents will read the new field as `default` — there is no migration mechanism (§5.4).
Never rename an existing property without a data migration.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A field reads as `null`/`0` for old documents | new property, no backfill; no schema versioning |
| A field's data disappears after a refactor | property renamed; stored field name is the property name |
| Every document reinterpreted after an enum edit | ordinal persistence (§3.22) |

### 3.21 `AsEntity` / `AsDocument` / `AsDto` — the three mappings

**Definition.** `…Infrastructure/Mongo/Documents/Extensions.cs` — an `internal static` class holding
five extension methods, three synchronous and two `Task`-lifting wrappers:

| Method | Direction | Lines | Note |
| --- | --- | --- | --- |
| `AsEntity(this VehicleDocument)` | storage → domain | `:9-18` | `document is null ? null :` — **null-safe** |
| `AsEntityAsync(this Task<VehicleDocument>)` | " | `:20-21` | awaits then delegates; null-safe by delegation |
| `AsDocument(this Vehicle)` | domain → storage | `:23-34` | **not** null-guarded |
| `AsDocumentAsync(this Task<Vehicle>)` | " | `:36-37` | **not** null-guarded |
| `AsDto(this VehicleDocument)` | storage → wire | `:39-50` | **not** null-guarded; transforms `Variants` (§3.19) |

**Representation & storage.** Pure functions, no state. This trio is the concrete instance of
[[database-per-service-with-document-mapping]] in this service.

**Lifecycle.** `AsEntity` on every write-path read (`VehiclesMongoRepository.cs`); `AsDocument` on
every write; `AsDto` on both read paths.

**Invariants & enforcement.** The contract is *"all eight fields are carried across all three
mappings"* and it is enforced by **nothing whatsoever** — no reflection check, no test, no analyzer.
Every mapping is a hand-written object initialiser. Adding a ninth property to the entity and to the
document, then forgetting `AsDto`, produces a compiling, running service whose API silently omits the
field. This is the highest-frequency maintenance hazard in the service (§7.1).

The three are **asymmetric in null handling**, and the asymmetry is compensated for at the call sites
rather than in the mappings. `AsEntity` guards internally (`:10` — `document is null ? null :`), so a
missing document yields `null` and the caller decides — which is exactly how `UpdateVehicleHandler`
and `DeleteVehicleHandler` produce `VehicleNotFoundException`. `AsDto` and `AsDocument` do **not**
guard; instead both read handlers apply the null-conditional operator themselves:
`return document?.AsDto()` (`…Infrastructure/Mongo/Queries/Handlers/GetVehicleHandler.cs:21`) and
`return pagedResult?.Map(d => d.AsDto())` (`SearchVehiclesHandler.cs:33`). So a missing vehicle
returns `null` from `GET /vehicles/{id}` rather than throwing — but the protection lives in two call
sites, not in the mapping, and a third caller that forgets the `?.` gets a `NullReferenceException`
mapped to an opaque 400 (§3.26).

`GetVehicleHandler` also queries by predicate rather than by key —
`_repository.GetAsync(v => v.Id == query.VehicleId)` (`:20`) — where `_id` is already the indexed
primary key. Functionally identical, but it takes the driver through the LINQ path for a lookup that
could use the id overload directly.

`AsEntity` is also where the variant re-floor happens: it passes the single `document.Variants` into
the **`params`** constructor (`:18` → `Vehicle.cs:30-35`), which ORs `Standard` back in (§3.5).

**Extension procedure.** Change all three together, in one commit. A useful discipline given the lack
of enforcement: keep the property order identical in `Vehicle`, `VehicleDocument`, `VehicleDto` and
all three initialisers, so a missing line is visible in a side-by-side diff. That discipline is
currently followed — all five lists are in the same order.

**Failure modes.** A field silently absent from the API, or silently `default` in storage. No error,
no log line.

### 3.22 Enum-as-ordinal persistence — no `[BsonRepresentation]`

**Definition.** `VehicleDocument.Variants` is typed `Variants` with **no BSON attribute**
(`…Infrastructure/Mongo/Documents/VehicleDocument.cs:16`). The MongoDB C# driver's default
serialisation for an enum is its **underlying integer**, so the stored field is an `int32`.

**Representation & storage.** BSON `int32`. A vehicle certified Standard + Chemistry stores `3`.

**Lifecycle.** Written on every `AddDocument`/`Update`, read on every query — including the search
predicate, where `v.Variants == query.Variants` becomes an integer comparison the server can execute
(§3.18).

**Invariants & enforcement.** **The enum's ordinal values are a persisted schema and nothing says so.**
`…Core/Entities/Variants.cs:5-13` looks like an ordinary domain enum, sits in the dependency-free
`Core` project, and carries no comment warning that its numbers are on disk. Reordering the members,
inserting a member in the middle, or switching to sequential numbering silently reinterprets **every
existing document**: a vehicle stored as `4` (Explosives today) would read as whatever bit 2 means
after the edit.

`[convey]`/driver behaviour: the choice of integer over string is the driver's default, not an
explicit decision in this repository. It is worth stating explicitly because the alternative —
`[BsonRepresentation(BsonType.String)]` — would make the stored data self-describing and immune to
renumbering, at the cost of breaking the integer comparison in the search predicate.

**Extension procedure.** Append new members with the next power of two; never renumber. If string
representation is wanted, it is a **data migration plus a query rewrite**, not an attribute change:
every document must be converted, and `SearchVehiclesHandler`'s predicate must switch from integer
comparison to something the driver can translate over strings.

**Failure modes.** A domain-looking refactor of an enum in the dependency-free project corrupts the
meaning of the entire collection. Fails **silently** — every read succeeds, every value is wrong.

### 3.23 `VehiclesMongoRepository` — the port adapter

**Definition.** `…Infrastructure/Mongo/Repositories/VehiclesMongoRepository.cs:17-29` — implements
`IVehiclesRepository` (§3.10) over `IMongoRepository<VehicleDocument, Guid>`, delegating each of the
four methods and applying the mapping extensions (§3.21) on the way through:
`GetAsync` → `_repository.GetAsync(id).AsEntityAsync()`, `AddAsync` → `AddAsync(vehicle.AsDocument())`,
`UpdateAsync` → `UpdateAsync(vehicle.AsDocument())`, `DeleteAsync` → `DeleteAsync(id)`.

**Representation & storage.** Registered as a transient (`…Infrastructure/Extensions.cs:49`); the
underlying `IMongoRepository` comes from `AddMongoRepository<VehicleDocument, Guid>("vehicles")`
(`:72`) `[convey]`.

**Lifecycle.** Per handler invocation.

**Invariants & enforcement.** The adapter is the **only** place `AsDocument`/`AsEntityAsync` are
called on the write path, so the mapping cannot be bypassed by a write. Reads bypass it entirely by
design (§3.10).

`UpdateAsync` is a **full-document replace**, not a field-level patch — `vehicle.AsDocument()`
materialises all eight fields and hands them to the driver. Combined with the absence of a version
token (§3.25), this makes concurrent updates last-writer-wins across *every* field, not just the
contended one: two simultaneous PUTs, one changing the price and one changing the variants, end with
one of the two changes gone.

`[convey]` **`Unverifiable — Missing Source Evidence`** — whether `IMongoRepository.UpdateAsync`
upserts or no-ops when the id is absent cannot be settled from source here. It matters only in the
narrow race where a vehicle is deleted between `UpdateVehicleHandler`'s `GetAsync` and its
`UpdateAsync`; on an upsert the vehicle would be resurrected.

**Extension procedure.** New write operations get a method here and on the `Core` interface. If a
partial update is ever needed for performance, it must not bypass the entity — build the entity,
mutate it, and replace, or the domain guards stop running.

**Failure modes.** Lost updates under concurrency (§3.25); possible resurrection on the delete/update
race.

### 3.24 Mongo database and the `"vehicles"` collection literal

**Definition.** The database name is configuration — `mongo.database: vehicles-service`
(`Api/appsettings.json:95-99`, alongside `connectionString` and `seed: false`) — while the collection
name is a **string literal in code**: `AddMongoRepository<VehicleDocument, Guid>("vehicles")`
(`…Infrastructure/Extensions.cs:72`).

**Representation & storage.** One database per service ([[database-per-service-with-document-mapping]]),
one collection inside it.

**Lifecycle.** Resolved at startup. `seed: false` means Convey's seeder does not run `[convey]`, so
the collection is created lazily by the first write and the database contains nothing on a fresh
deployment.

**Invariants & enforcement.** The split is worth noticing: **the database is environment-configurable
and the collection is not.** A deployment can point at a different database (and `appsettings.docker.json:64-68`
does exactly that, changing only the host) but cannot rename the collection without a rebuild.
Nothing enforces that the two stay consistent with the service's identity — and they already do not:
`mongo.database` is `vehicles-service` while `jaeger.serviceName` is `vehicles`
(`appsettings.json:70`) and `app.service` is `vehicles-service` (`:4`). Three names, two spellings
(§3.45).

**Extension procedure.** A second collection needs a second `AddMongoRepository<TDoc, TId>(name)` call
and a second document type. Prefer keeping the collection literal next to the others in
`Extensions.cs` rather than scattering it.

**Failure modes.** Pointing two environments at the same `mongo.connectionString` silently shares
data, because the database name is identical across profiles — only the host differs
(`appsettings.json:97` vs `appsettings.docker.json:66`). Verified by comparison; there is no
environment discriminator in the database name.

### 3.25 Absence of optimistic concurrency — last-writer-wins

**Definition.** Verified by absence. `Vehicle` has no `Version` (`…Core/Entities/Vehicle.cs:8-15`),
`VehicleDocument` has no `Version` and no ETag (`VehicleDocument.cs:9-16`), no command carries an
expected version, and `UpdateAsync` is an unconditional replace (§3.23).

**Representation & storage.** Nothing stored.

**Lifecycle.** N/A.

**Invariants & enforcement.** There is no lost-update protection at all. The service is small enough
that this may be an acceptable trade — vehicles are edited rarely and by administrators — but it is a
decision recorded nowhere. Contrast `customers-service`, which does carry a `Version` on its
aggregate (`component-internals/customers-service.md` §3.2), so the platform is inconsistent about it
rather than uniformly relaxed.

The exposure is widened by the read-modify-write in `UpdateVehicleHandler` and by full-document
replacement (§3.23): the window between `GetAsync` and `UpdateAsync` spans a network round trip, and
a loss inside it discards *all* of the competing update's fields.

**Extension procedure.** Add `Version` to entity and document, increment it in the mutators, and give
`IVehiclesRepository.UpdateAsync` an expected-version parameter that the adapter turns into a
filtered replace. Then add a `ConcurrencyException : AppException` and map it in **both**
`ExceptionToResponseMapper` (automatic — it is an `AppException`) and `ExceptionToMessageMapper`
(**manual** — §3.28). Existing documents lack the field and will read as `0`, which is a workable
starting version.

**Failure modes.** Silent lost updates under concurrent edits. No error, no log, no metric.

### 3.26 `ExceptionToResponseMapper` — everything is HTTP 400

**Definition.** `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:11-46` — an
`internal sealed class` implementing Convey's `IExceptionToResponseMapper`, registered at
`…Infrastructure/Extensions.cs:57`. `Map` is a three-arm switch expression (`:15-24`):

| Arm | Body | Status |
| --- | --- | --- |
| `DomainException ex` | `{code = GetCode(ex), reason = ex.Message}` | `BadRequest` |
| `AppException ex` | `{code = GetCode(ex), reason = ex.Message}` | `BadRequest` |
| `_` | `{code = "error", reason = "There was an error."}` | `BadRequest` |

`GetCode` (`:26-45`) memoises per exception type in a `static ConcurrentDictionary<Type, string>`
(`:13`) and derives the code as: the exception's `Code` if non-blank, otherwise
`exception.GetType().Name.Underscore().Replace("_exception", string.Empty)` (`:36-39`).

**Representation & storage.** The `Codes` dictionary is process-wide and unbounded — bounded in
practice by the number of exception types, so it cannot grow without bound.

**Lifecycle.** Invoked by Convey's error-handling middleware, installed by `UseErrorHandler()`
(`…Infrastructure/Extensions.cs:79`) `[convey]`.

**Invariants & enforcement.** **Every failure is HTTP 400.** Not found is 400. A null-reference bug is
400. A Mongo connection failure is 400. There is no 404, no 409, no 500 — the mapper cannot produce
one, because all three arms hard-code `HttpStatusCode.BadRequest`. Callers, load balancers, retry
policies and alerting all lose the ability to distinguish client error from server error. This is
platform-wide, identical in `pricing-service` (`component-internals/pricing-service.md` §3.13) and in
every other service in the inventory.

The `_` arm is the security-relevant one and it is **right**: an unexpected exception yields the
opaque `{code: "error", reason: "There was an error."}` with no message and no stack trace, so
internal detail does not leak. Only the two known-exception hierarchies get their `Message` echoed to
the caller — and those messages interpolate user-supplied values (e.g. the description, §3.9), which
is a reflected-input echo but not a disclosure of internal state.

The `Underscore()` fallback is why `InvalidVehicleCapacity`'s non-standard name (§3.9) is harmless:
its `Code` is overridden, so the fallback never runs for it. Had it not been, the derived code would
have been `invalid_vehicle_capacity` anyway — the `Replace("_exception", "")` is a no-op on a name
that does not end in `Exception`.

**Extension procedure.** To introduce real status codes, change the two typed arms to switch on
`ex.Code` or introduce a `NotFoundException` marker interface. **This is a breaking change for every
caller that treats 400 as "my fault"** — the gateway forwards status codes unchanged
(`ntrada.yml`), so it reaches the browser. Coordinate across services rather than changing one.

**Failure modes.** A caller cannot distinguish a bad request from a missing resource from a database
outage. Retry logic based on status codes is impossible. Fails **loudly but uninformatively**.

### 3.27 Integration events — three published, one consumed

**Definition.** `…Application/Events/` holds three `IEvent` types, each `[Contract]` (§3.34), each
with a single get-only `Guid VehicleId` and an expression-bodied constructor
`(Guid id) => VehicleId = id`:

| Event | Evidence | Consumers found in this workspace |
| --- | --- | --- |
| `VehicleAdded` | `VehicleAdded.cs:6-13` | **none** |
| `VehicleUpdated` | `VehicleUpdated.cs:6-13` | **none** |
| `VehicleDeleted` | `VehicleDeleted.cs:6-13` | `availability-service` |

**Representation & storage.** Published to exchange `vehicles` (§3.31); staged in the outbox
collection first (§3.30).

**Lifecycle.** Constructed at the end of a command handler (§3.14), passed to
`IMessageBroker.PublishAsync` (`…Application/Services/IMessageBroker.cs:7-11`).

**Invariants & enforcement.** The events carry **only the id** — no brand, no price, no variants.
That is a deliberate [[narrow-synchronous-point-read]]-style contract: consumers that need detail must
call back. But **no consumer does**: `availability-service` handles `VehicleDeleted` by dispatching
`DeleteResource(@event.VehicleId)`
(`hianshul100_Pacco.Services.Availability/…/Events/External/Handlers/VehicleDeletedHandler.cs:17`),
which needs nothing more; and `orders-service` reads vehicle price synchronously over HTTP rather
than by subscribing (§3.47). So the id-only contract has never been exercised as a "call me back"
pattern.

Note the **same parameter-name mismatch as §3.13** in all three events: the parameter is `id`, the
property `VehicleId`. For events it is harmless here, because consumers declare their **own**
matching class rather than sharing this assembly — `availability-service` defines
`VehicleDeleted(Guid vehicleId)` locally with `[Message("vehicles")]`
(`…Availability/…/Events/External/VehicleDeleted.cs:7,12-13`). The contract crosses the wire as JSON
`{"vehicleId": "…"}`, and each side binds it with its own constructor. The mismatch only bites where
*this* assembly deserialises, which is the command path (§3.13).

**`VehicleAdded` and `VehicleUpdated` are published to nobody** — Q-3. Neither appears in any
`SubscribeEvent<>` call or any `Events/External` folder in any repository in this workspace. They are
either forward-looking contracts or dead weight; there is no comment either way.

**Extension procedure.** A new event needs: the class in `Application/Events` with `[Contract]`, the
`PublishAsync` call in a handler, and — for anyone to receive it — a matching class plus
`SubscribeEvent<T>()` in the consuming service, bound to `[Message("vehicles")]`. Nothing here
generates or validates the consumer side; a typo in the consumer's exchange name produces a binding
that never matches and a handler that never runs (the failure recorded platform-wide for
`parcel_deleted` in `component-internals/index.md` §4).

**Failure modes.** Publishing to an exchange with no binding succeeds silently — RabbitMQ discards
unroutable messages by default with no error to the publisher. Two of this service's three events are
in that state today and nothing reports it.

### 3.28 Rejected events and `ExceptionToMessageMapper` — the null drop

**Definition.** `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:12-40` implements Convey's
`IExceptionToMessageMapper` (registered at `…Infrastructure/Extensions.cs:65`). It is a nested switch
— outer on exception type, inner on the in-flight command — producing one of three `IRejectedEvent`
types, each with `VehicleId`, `Reason` (the exception message) and `Code` (the exception code)
(`…Application/Events/Rejected/UpdateVehicleRejected.cs:9-18`, and the `Add`/`Delete` equivalents).

The complete mapping, read directly off the switch:

| Exception | `AddVehicle` | `UpdateVehicle` | `DeleteVehicle` | other |
| --- | --- | --- | --- | --- |
| `InvalidVehicleCapacity` (`:15-20`) | `AddVehicleRejected` | `UpdateVehicleRejected` | **`null`** | `null` |
| `InvalidVehicleDescriptionException` (`:21-26`) | `AddVehicleRejected` | `UpdateVehicleRejected` | **`null`** | `null` |
| `InvalidVehiclePricePerServiceException` (`:27-32`) | `AddVehicleRejected` | `UpdateVehicleRejected` | **`null`** | `null` |
| `VehicleNotFoundException` (`:33-38`) | **`null`** | `UpdateVehicleRejected` | `DeleteVehicleRejected` | `null` |
| **anything else** (`:39`) | `null` | `null` | `null` | `null` |

**Representation & storage.** The rejected event is published on the same exchange as the success
events (§3.31), so a caller subscribes to one exchange for both outcomes
([[rejected-event-failure-contract]]).

**Lifecycle.** Invoked by Convey's RabbitMQ subscriber when a `SubscribeCommand<T>()` handler throws
`[convey]`. **It runs only on the AMQP path** — HTTP failures go to `ExceptionToResponseMapper`
(§3.26) instead.

**Invariants & enforcement.** Returning `null` means **no message is published**. `[convey]`
**`Unverifiable — Missing Source Evidence`** — whether Convey then acks, nacks or dead-letters the
original message cannot be settled without its source; what is certain from this file is that no
rejection reaches the caller.

The reachable silent drops are not hypothetical:

- **`DeleteVehicle` + any `DomainException`.** `DeleteVehicleHandler` reads the vehicle first
  (`DeleteVehicleHandler.cs:23`), which rehydrates through the constructor and can throw
  `InvalidVehicleCapacity` (§3.8). The first three rows have no `DeleteVehicle` arm → `null` → the
  async caller waits forever for an outcome that never comes.
- **`AddVehicle` + `VehicleNotFoundException`.** Not reachable today, but the arm is `null`.
- **Any infrastructure exception** — a Mongo timeout, a serialisation failure, a
  `NullReferenceException` — hits `:39` and is dropped for **all three commands**. Over HTTP the same
  exception at least produces a 400 (§3.26). Over AMQP it produces nothing at all.

So the async surface reports **domain validation failures on two of three commands** and is silent
about everything else. That is the failure contract, and it is much narrower than it looks.

**Extension procedure.** Every new exception type **and** every new command needs an arm here. The
switch is exhaustive-looking but has no compiler support — `_ => null` swallows the gap. A practical
mitigation: change the final `_ => null` to log at warning level before returning null, so drops are
at least observable. That is a one-line change with no contract impact and is the recommended first
fix.

**Failure modes.** An async command fails and the caller is never told. Fails **silently** by
construction — this is the platform's most consequential silent-failure surface and it is identical
in every service in the inventory.

### 3.29 `MessageBroker` — the publish pipeline

**Definition.** `…Infrastructure/Services/MessageBroker.cs:16-87` — `internal sealed`, implements the
`Application`-owned `IMessageBroker` (`…Application/Services/IMessageBroker.cs:7-11`), registered as a
transient at `…Infrastructure/Extensions.cs:50`. Seven constructor dependencies (`:28-31`):
`IBusPublisher`, `IMessageOutbox`, `ICorrelationContextAccessor`, `IHttpContextAccessor`,
`IMessagePropertiesAccessor`, `RabbitMqOptions`, `ITracer`, plus a logger.

**Representation & storage.** One resolved field of interest: `_spanContextHeader`, set in the
constructor to `options.SpanContextHeader` or, if blank, the constant `"span_context"` (`:18,40-42`).
The configured value is `span_context` (`Api/appsettings.json:149`), so the fallback is never used —
but it means removing the config key changes nothing, which is worth knowing.

**Lifecycle.** `PublishAsync(params IEvent[])` (`:45`) delegates to the `IEnumerable` overload
(`:47-86`), which:

1. Returns immediately if `events` is null (`:49-52`) — a null publish is a **no-op, not an error**.
2. Reads the ambient message properties (`:54-56`): `originatedMessageId` and `correlationId` from the
   inbound message, both null on the HTTP path.
3. Resolves the span context (`:57-61`): the inbound header if present, else the active Jaeger span's
   `Context.ToString()`, else `string.Empty` (§3.39).
4. Collects headers to forward (`:63`) — see §3.39.
5. Resolves the correlation context (`:64-65`): the AMQP one if present, **else** the one built from
   the HTTP `Correlation-Context` header (§3.38).
6. For each event (`:67-85`): skips nulls (`:69-72`); mints a **fresh** `messageId` as
   `Guid.NewGuid().ToString("N")` (`:74`); logs at **Trace** (`:75`); then either stages it in the
   outbox (`:76-81`) or publishes it directly (`:83-84`), depending on `_outbox.Enabled`.

**Invariants & enforcement.** Three things are enforced here and each has a consequence.

- **Message ids are always fresh.** `:74` generates a new `Guid` per event with no deterministic
  input. So **an event's id is not derivable from the command that caused it**, and re-publishing the
  same logical event after a retry produces a different id. Any consumer-side deduplication keyed on
  message id therefore cannot deduplicate this service's events across retries. `originatedMessageId`
  (`:55`) is the only causal link, and it is null whenever the trigger arrived over HTTP.
- **Null events are skipped, not rejected.** `:69-72`. A handler that publishes `null` succeeds.
- **The outbox is chosen at publish time, per event.** `:76` reads `_outbox.Enabled` on every call, so
  toggling `outbox.enabled` in configuration changes behaviour without code change (§3.30).

The `LogTrace` at `:75` is effectively invisible: `logger.level` is `information`
(`Api/appsettings.json:33`), so **the one log line that records a publish never fires in any
environment**. There is no information-level record that an event was published anywhere in this
service.

**Extension procedure.** To make event ids deterministic (worth doing for consumer-side idempotency),
derive `messageId` from the command's id plus the event type rather than `Guid.NewGuid()`. To make
publishes observable, raise `:75` to `LogInformation` or add a metric — currently neither exists.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| No evidence a publish happened | the only log line is at Trace, below the configured level |
| Consumer processes the same event twice after a retry | fresh message id per publish (`:74`) |
| A handler's `PublishAsync(null)` silently does nothing | `:49-52` |

### 3.30 Outbox, inbox, and `disableTransactions: true`

**Definition.** Two `[Decorator]` classes implement [[transactional-outbox-handler-decorator]]:
`OutboxCommandHandlerDecorator<TCommand>` and `OutboxEventHandlerDecorator<TEvent>`
(`…Infrastructure/Decorators/*.cs:10`), applied by `TryDecorate(typeof(ICommandHandler<>), …)` and
`TryDecorate(typeof(IEventHandler<>), …)` (`…Infrastructure/Extensions.cs:53-54`) `[convey]`. The
event decorator is registered but never exercised: this service handles no events (§3.32). Each wraps the inner handler in
`_inbox.HandleAsync(messageId, () => _handler.HandleAsync(message))` when the outbox is enabled, and
calls the inner handler directly when it is not (`:32-35`). The `messageId` comes from the inbound
message properties, or — **when there is none** — from `Guid.NewGuid().ToString("N")` (`:27-29`).

**Representation & storage.** Two Mongo collections in the same database, named by configuration:
`inbox` and `outbox` (`Api/appsettings.json:105-106`). Wired by
`AddMessageOutbox(o => o.AddMongo())` (`…Infrastructure/Extensions.cs:64`).

The full outbox configuration, `Api/appsettings.json:100-108`:

| Key | Value | Effect |
| --- | --- | --- |
| `enabled` | `true` | inbox dedup + outbox staging are active |
| `type` | `sequential` | the dispatcher's processing mode `[convey]` |
| `expiry` | `3600` | processed-message retention, seconds `[convey]` |
| `intervalMilliseconds` | `2000` | outbox drain poll interval — the floor on publish latency |
| `inboxCollection` | `inbox` | dedup collection |
| `outboxCollection` | `outbox` | staging collection |
| `disableTransactions` | **`true`** | **no Mongo transaction around the state change and the outbox insert** |

**Lifecycle.** Inbound command → decorator → inbox check → inner handler → repository write →
`MessageBroker.PublishAsync` → outbox insert → background dispatcher drains every 2 s → RabbitMQ.

**Invariants & enforcement.** The pattern's whole purpose is *"the state change and the announcement
succeed or fail together"*, and **this service has switched that off**. `disableTransactions: true`
was set by the `HEAD` commit itself (`af43bcf`, *Updated outbox - disable local TX*) — almost
certainly because a single-node MongoDB cannot start a transaction, which is what `docker-compose`
provides. The consequence is that the outbox degrades from a correctness mechanism to a **retry
buffer with a 2-second delay**: it still guarantees that a staged message is eventually delivered,
but no longer that a state change is always staged.

`appsettings.docker.json` has **no `outbox` section at all**, so the Docker profile inherits the base
values including `disableTransactions: true` — the transaction is off in the containerised
environment too, not only locally. `appsettings.local.json:35-37` goes further and sets
`outbox.enabled: false`, so **local development exercises the direct-publish path (`MessageBroker.cs:83-84`)
and never touches the outbox or the inbox at all**. Anything that only works because of the outbox
will not be discovered locally.

The second, subtler defect is in the decorator's id fallback. On the **HTTP** path there is no inbound
message and therefore no message id, so `:27-29` mints a fresh `Guid` for every request. That id is
then handed to the inbox as the deduplication key — and a key that is unique per request can never
collide. **Inbox deduplication is structurally inert for HTTP-delivered commands**, always, in every
environment. It works only for AMQP-delivered ones. Nothing marks this; the decorator looks
symmetric.

**Extension procedure.** To restore atomicity, run MongoDB as a replica set (even single-node) and set
`disableTransactions: false` — a configuration and infrastructure change, no code. To make the HTTP
path deduplicable, accept a client-supplied idempotency key (e.g. an `Idempotency-Key` header) and use
it instead of the generated `Guid` at `:27-29`. Do **not** simply re-enable transactions without
changing the Mongo topology; the write will fail at runtime.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| Vehicle written, event never published | crash between write and outbox insert; no transaction |
| A retried HTTP POST creates a second vehicle | inbox key is a per-request `Guid` (`:27-29`) |
| Events appear ~2 s after the API returns | `intervalMilliseconds: 2000` |
| Outbox bugs never reproduce locally | `outbox.enabled: false` in the `local` profile |

### 3.31 RabbitMQ wiring — exchange, queue template, conventions, Jaeger plugin

**Definition.** One call — `AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`
(`…Infrastructure/Extensions.cs:63`) — plus `UseRabbitMq()` in the pipeline
(`…Infrastructure/Extensions.cs:85`), configured by `rabbitMq` at `Api/appsettings.json:109-150`. This
is the concept `pricing-service.md` §1 and §3.23 forward-reference as the thing that service does
**not** have — its `Convey.Tracing.Jaeger.RabbitMQ` package reference is dangling precisely because it
never calls this.

**Representation & storage.** Configuration only; the topology is declared by Convey at startup
`[convey]`.

The load-bearing keys:

| Key | Value | Effect |
| --- | --- | --- |
| `connectionName` | `vehicles-service` | identifies this service's connection in the broker's UI |
| `retries` | `3` | connection retry attempts |
| `conventionsCasing` | `snakeCase` (`:113`) | **`AddVehicle` → routing key `add_vehicle`; `VehicleDeleted` → `vehicle_deleted`** |
| `exchange.name` | `vehicles` (`:131-137`) | this service's own exchange |
| `exchange.type` | `topic` | routing-key matching, wildcard-capable |
| `exchange.durable` | `true` | survives broker restart |
| `exchange.declare` | `true` | the service creates it if absent |
| `queue.template` | `vehicles-service/{{exchange}}.{{message}}` (`:138-144`) | one queue **per message type**, namespaced by service |
| `context.enabled` / `context.header` | `true` / `message_context` (`:145-148`) | the correlation context travels in this header |
| `spanContextHeader` | `span_context` (`:149`) | matches `MessageBroker`'s constant (§3.29) |
| `hostnames` | `[localhost]` base, `[rabbitmq]` in docker (`appsettings.docker.json:69-73`) | the **only** key the Docker profile overrides |

**Lifecycle.** Exchange and queues declared at startup; the Jaeger plugin injects and extracts span
context on every publish and consume `[convey]`.

**Invariants & enforcement.** Three consequences worth holding.

- **The naming convention is the contract.** `conventionsCasing: snakeCase` is what makes the
  gateway's async routing keys (`add_vehicle`, `update_vehicle`, `delete_vehicle` —
  `ntrada-async.yml:500-529`) line up with the C# class names. **Renaming a command class silently
  changes its routing key**, and the gateway's YAML does not move with it. There is no shared
  constant and no test; the coupling is a string-transformation convention agreed in two repositories.
  This is [[service-owned-topic-exchange-messaging]] plus
  [[framework-supplied-platform-conventions]], and it is the platform's most fragile seam.
- **One queue per message type** (`{{exchange}}.{{message}}`) means each of the three commands has its
  own queue, so a poison message on `delete_vehicle` cannot block `add_vehicle`. Good isolation,
  obtained for free from the template.
- **The Docker profile overrides only `hostnames`**, inheriting exchange, queue template, conventions
  and headers from the base file. So the topology is genuinely identical across environments — the
  one thing in this service's configuration that is (§3.44).

**Extension procedure.** A new message type gets its queue automatically from the template. To
subscribe to another service's exchange, declare the external event class with
`[Message("<their-exchange>")]` and call `SubscribeEvent<T>()`; the exchange name is a **string in an
attribute** with no validation, which is exactly how the platform's `parcel_deleted` mis-binding
happened (`component-internals/index.md` §4). Check the publisher's `exchange.name` in *its*
`appsettings.json` before writing the attribute.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| The gateway's async writes stop arriving | a command class was renamed; routing key changed with it |
| A subscription never fires | `[Message("…")]` names an exchange nobody publishes to |
| Service will not start | `hostnames` unreachable; `retries: 3` then failure |

### 3.32 `SubscribeCommand<T>()` — the AMQP command surface

**Definition.** Three lines in the pipeline (`…Infrastructure/Extensions.cs:86-88`):
`SubscribeCommand<AddVehicle>()`, `SubscribeCommand<UpdateVehicle>()`,
`SubscribeCommand<DeleteVehicle>()`. There are **no** `SubscribeEvent<T>()` calls — this service
consumes no external events.

**Representation & storage.** Each subscription binds a queue named
`vehicles-service/vehicles.add_vehicle` (and equivalents) to exchange `vehicles` with the matching
routing key (§3.31).

**Lifecycle.** Declared at startup inside `UseInfrastructure`; the subscriber deserialises the payload
into the command type and dispatches it through the same in-memory command dispatcher the HTTP path
uses (`…Application/Extensions.cs:11`), through the same outbox decorator (§3.30), into the same
handler (§3.14).

**Invariants & enforcement.** **Commands are accepted on an exchange, not a private queue.** Exchange
`vehicles` carries this service's *outbound* events **and** its *inbound* commands. Anything with
publish rights to that exchange can create, update or delete any vehicle, and there is no
authentication on the AMQP path at all — no certificate check, no token, nothing (§3.41). The
gateway's async mode is the intended publisher (`ntrada-async.yml:500-529`), and the gateway does
enforce `auth: true` before publishing — but that enforcement lives entirely at the edge. **This is
[[dual-mode-edge-write]] with the authorization on only one of the two modes' front doors.**

Mixing commands and events on one exchange also means a consumer subscribing to `vehicles` with a
wildcard binding would receive this service's own commands. No consumer does today.

**Extension procedure.** A new command needs the class, the handler, the `SubscribeCommand<T>()` line,
an `ExceptionToMessageMapper` arm (§3.28), and — if the gateway should expose it — a route in all four
`ntrada*.yml` files. Adding the subscribe line without the mapper arm is the common mistake and it
fails silently.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A command sent over AMQP is never processed | routing key mismatch (§3.31); RabbitMQ discards it, publisher sees success |
| A command fails and nobody is told | unmapped exception (§3.28) |
| Unauthenticated writes | no auth on the AMQP path; edge-only enforcement |

### 3.33 Dispatcher-bound HTTP endpoints and `afterDispatch`

**Definition.** `…Api/Program.cs:33-41` — `UseDispatcherEndpoints` binds six routes to CQRS types with
no controller and no action method anywhere in the repository
([[dispatcher-bound-cqrs-endpoints]]):

| Route | Binding | Line |
| --- | --- | --- |
| `GET /` | inline lambda returning `AppOptions.Name` | `:34` |
| `GET /vehicles/{vehicleId}` | `Get<GetVehicle, VehicleDto>` | `:35` |
| `GET /vehicles` | `Get<SearchVehicles, PagedResult<VehicleDto>>` | `:36` |
| `POST /vehicles` | `Post<AddVehicle>` with `afterDispatch: (cmd, ctx) => ctx.Response.Created($"vehicles/{cmd.VehicleId}")` | `:37-38` |
| `PUT /vehicles/{vehicleId}` | `Put<UpdateVehicle>` | `:39` |
| `DELETE /vehicles/{vehicleId}` | `Delete<DeleteVehicle>` | `:40` |

**Representation & storage.** The routing table exists only as this expression tree, evaluated once at
startup.

**Lifecycle.** Request → route match → bind the query/command → dispatch through the in-memory
dispatcher → serialise the result (queries) or run `afterDispatch` (the POST).

**Invariants & enforcement.** Four things follow, and each matters.

- **The route table is the API contract and it lives in `Program.cs`.** There is no controller to
  search for, no attribute routing, and no OpenAPI-from-code. Anyone looking for "where is
  `DELETE /vehicles` handled" must know to read `Program.cs` — the handler is found by *type*, not by
  name.
- **`afterDispatch` runs after the command is dispatched, and it is the only route that customises the
  response.** The other two writes return whatever Convey's default is `[convey]` — almost certainly
  `200`/`202`, **`Unverifiable — Missing Source Evidence`**. So POST returns `201 Created` with a
  `Location` of `vehicles/{id}` and PUT/DELETE do not signal anything beyond success.
- **`afterDispatch` reads `cmd.VehicleId` after dispatch**, which is why `AddVehicle`'s constructor
  mints the id (§3.11) rather than the handler doing it: the response needs the id, and the command is
  the only object both the dispatcher and the callback can see.
- **Binding is by type, not by name.** `Get<GetVehicle, VehicleDto>` requires exactly one
  `IQueryHandler<GetVehicle, VehicleDto>` in the container. A missing handler is a **DI resolution
  failure at request time**, not at startup — the service boots healthy and 500s (well, 400s — §3.26)
  on first use.

**Extension procedure.** A new route is one line here plus a command/query type plus a handler. If it
is a write that should also be reachable asynchronously, add the `SubscribeCommand<T>()` line (§3.32).
If the gateway should expose it, add it to **all four** `ntrada*.yml` files — sync and async, local and
docker — or it will work in one deployment mode and 404 in another.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A route 404s despite the handler existing | the line was not added to `Program.cs` |
| A route works locally but not in Docker | `ntrada.yml` updated, `ntrada.docker.yml` not |
| First request to a new route fails | handler not registered; failure is at request time |

### 3.34 `[Contract]` and `UsePublicContracts`

**Definition.** `…Application/ContractAttribute.cs:5-7` is an empty marker attribute. It is applied to
all three commands, all three events and all three rejected events — nine types. `UseInfrastructure`
calls `UsePublicContracts<ContractAttribute>()` (`…Infrastructure/Extensions.cs:83`) `[convey]`, which
exposes the marked types' JSON schemas on a well-known endpoint.

**Representation & storage.** No storage. The endpoint path and payload shape are Convey's
`[convey]` — **`Unverifiable — Missing Source Evidence`** as to the exact route and format.

**Lifecycle.** Reflected over at startup.

**Invariants & enforcement.** The intent is a machine-readable, self-describing message contract: a
consumer can discover the shape of `VehicleAdded` without reading this repository. The intent is
undermined by two things.

- **Nothing verifies that `[Contract]` is applied to everything published.** A new event without the
  attribute is simply absent from the manifest, silently. Conversely, `[Contract]` on a type that is
  never published advertises a contract that does not exist.
- **No consumer in this workspace reads the manifest.** Every consuming service hand-writes its own
  matching class (§3.27). The manifest is published and unused — the same shape of finding as
  `VehicleAdded` having no subscriber (§3.27), and the reason `pricing-service.md` §3.28 notes the
  absence of this call as a non-loss.

Note also that the manifest is **unauthenticated**, like everything else on this service (§3.41). It
discloses the service's full message vocabulary to anything that can reach port 5009. That is a low
severity finding — the same information is in the gateway's YAML — but it is a disclosure that the
service does not need to make.

**Extension procedure.** Mark every new command, event and rejected event with `[Contract]` at the
moment it is created. Treat it as part of the class template, not as an optional annotation.

**Failure modes.** A published contract missing from the manifest, or a manifest entry for a message
nobody sends. Both fail **silently** and neither is detectable from inside this service.

### 3.35 `MessageToLogTemplateMapper` and handler logging

**Definition.** `…Infrastructure/Logging/MessageToLogTemplateMapper.cs:8-41` maps three command types
to `HandlerLogTemplate`s, each setting **only** `After`:

| Command | `After` template |
| --- | --- |
| `AddVehicle` (`:14-19`) | `Added a vehicle with id: {VehicleId}.` |
| `DeleteVehicle` (`:21-26`) | `Deleted a vehicle with id: {VehicleId}.` |
| `UpdateVehicle` (`:28-33`) | `Updated a vehicle with id: {VehicleId}.` |

`Map<TMessage>` (`:36-40`) looks the type up and returns `null` when absent. Registration is in
`…Infrastructure/Logging/Extensions.cs:10-19`: the mapper as a **singleton instance** (`:14`), then
`AddCommandHandlersLogging(assembly)` and `AddEventHandlersLogging(assembly)` over the `Application`
assembly (`:12,17-18`), called from the builder chain at `…Infrastructure/Extensions.cs:71`.

**Representation & storage.** `MessageTemplates` is an **expression-bodied property** (`:10-11`,
`=>` not `=`), so a **new dictionary is allocated on every `Map` call** — three `HandlerLogTemplate`
objects per logged command. Harmless at this service's volume, and a one-character fix (`=>` → `=`)
if it ever matters. Worth knowing because the mapper is registered as a singleton, which suggests the
author intended it to be allocated once.

**Lifecycle.** Convey's logging decorator calls `Map` around each command handler `[convey]`.

**Invariants & enforcement.** `{VehicleId}` is a **Serilog message-template property**, so the id is
captured as structured data, not just interpolated text — that is what makes the logs queryable in
Seq (§3.37).

Two gaps. First, only `After` is set: there is no `Before` and no `OnError` template, so **a failed
command logs nothing from this mechanism**. The failure surfaces as an exception the framework logs
generically, or — on the AMQP path with an unmapped exception (§3.28) — as nothing at all. Second,
`AddEventHandlersLogging` is called but this service has no event handlers, so it is inert (§3.30).

The `null` return for unmapped types (`:39`) means a fourth command added without a template logs
nothing at all — no error, no default template.

**Extension procedure.** Add an entry for every new command. Consider adding `OnError` templates to
all three existing entries: it is the cheapest available improvement to this service's observability,
and it directly mitigates the §3.28 silent-drop problem by at least producing a log line.

**Failure modes.** A new command's successful executions are invisible in the logs. A failed command
of any kind is invisible in the structured logs. Both fail **silently**.

### 3.36 Redis — configured with no consumer

**Definition.** `AddRedis()` is called in the builder chain (`…Infrastructure/Extensions.cs:67`) and
configured at `Api/appsettings.json:151-154` with a `connectionString` and
`instance: "vehicles:"`. **No type in this repository injects `IDistributedCache`,
`IConnectionMultiplexer`, or any Redis abstraction** — verified by absence across all 48 source files.

**Representation & storage.** A registered but unused connection.

**Lifecycle.** The multiplexer is created at startup and holds a connection to Redis for the life of
the process `[convey]`.

**Invariants & enforcement.** The `instance: "vehicles:"` prefix is the platform's
[[prefix-partitioned-shared-cache]] convention — one Redis instance shared by all services, each
namespacing its keys with `<service>:` so they cannot collide. `pricing-service.md` §3.17
forward-references this as the convention it does not participate in. Here the convention is
**declared and never exercised**: the prefix guards keys that are never written.

The practical consequence is a **startup dependency on Redis for no functional benefit**. If Redis is
unreachable, the behaviour depends on whether Convey's registration connects eagerly or lazily
`[convey]` — **`Unverifiable — Missing Source Evidence`**. If eagerly, an unavailable cache that
nothing uses can prevent the service from starting.

**Extension procedure.** If caching is wanted, the wiring is already there — inject the cache
abstraction into `GetVehicleHandler` and invalidate on the three write commands. Note that
invalidation must happen on **update and delete**, and that the AMQP path and the HTTP path both need
it, so the natural place is the command handler, not the endpoint. If caching is **not** wanted,
delete `:67`, the `redis` config block and the `Convey.Persistence.Redis` package reference — it is
three lines and removes a startup dependency.

**Failure modes.** Redis outage may block startup for no reason. Otherwise none — the code path does
not exist.

### 3.37 Log redaction, path exclusion and sinks

**Definition.** `Api/appsettings.json:32-67` configures Serilog through Convey's logging layer,
installed by `UseLogging()` (`…Api/Program.cs:42`). This is
[[structured-logging-with-property-redaction]].

| Key | Value | Effect |
| --- | --- | --- |
| `level` | `information` (`:33`) | **the `LogTrace` publish line never fires** (§3.29) |
| `excludePaths` | `["/", "/ping", "/metrics"]` (`:34`) | health and scrape traffic does not fill the logs |
| `excludeProperties` | 12 entries (`:35-47`) | named properties are removed from every log event |
| `console.enabled` | `true` | stdout, which is what Docker and PM2 capture |
| `file` | `logs/logs.txt`, daily rolling | on-disk logs inside the container — **not** volume-mounted in `compose/services.yml`, so they are lost on container removal |
| `seq` | url + `apiKey: "secret"` (`:64`) | centralised structured log sink |

**Representation & storage.** The 12 `excludeProperties` are the redaction list. They are property
**names**, matched against Serilog's structured properties, and they cover the credential-ish and
PII-ish names used across the platform (the same list appears in every Pacco service, including
`pricing-service` — `component-internals/pricing-service.md` §3.27).

**Lifecycle.** Applied at startup; the filter runs per log event.

**Invariants & enforcement.** The redaction is **name-based, not value-based**. It removes a property
called (for example) `password`; it does **not** redact a secret that appears inside a *message string*
or under a different property name. This service's own log templates only ever emit `{VehicleId}`
(§3.35), so there is nothing sensitive to leak here — but the guarantee is weaker than it looks and
should not be relied on if a future template interpolates user input.

`logger.seq.apiKey` is the literal string `"secret"` (`Api/appsettings.json:64`) — the unchanged
framework default, present identically in `appsettings.local.json` and `appsettings.docker.json`. It
is quoted here rather than merely cited because *the fact that it is an unmodified placeholder is the
finding*: it is not a real credential, and no real credential is committed to this repository. The
same is true of `vault.token: "secret"` (§3.43). Recorded as **B-2**.

**Extension procedure.** Add a property name to `excludeProperties` in **every** profile that defines
the block — `appsettings.json` and `appsettings.local.json` both do, and they must not drift. Prefer
not logging a sensitive value at all over relying on the exclusion list.

**Failure modes.** A sensitive value interpolated into a message string rather than passed as a named
property bypasses redaction entirely. Fails **silently**, into Seq.

### 3.38 `Correlation-Context` ingestion, `IAppContext`, `IIdentityContext`

**Definition.** The full [[transport-agnostic-caller-context]] implementation — five types, one
extension method, two DI registrations. `pricing-service.md` §3.19 forward-references this as the
fully wired case it lacks.

| Piece | What it does | Evidence |
| --- | --- | --- |
| `IAppContext` | `RequestId` + `Identity` | `…Application/IAppContext.cs:3-7` |
| `IIdentityContext` | `Id`, `Role`, `IsAuthenticated`, `IsAdmin`, `Claims` | `…Application/IIdentityContext.cs:6-13` |
| `CorrelationContext` | the deserialisation target, with a nested `UserContext` | `…Infrastructure/Contexts/CorrelationContext.cs:6-24` |
| `AppContext` | `internal`, three constructors, `Empty` static | `…Infrastructure/Contexts/AppContext.cs:6-27` |
| `IdentityContext` | `internal`, three constructors, `Empty` static | `…Infrastructure/Contexts/IdentityContext.cs:7-34` |
| `GetCorrelationContext` | reads and deserialises the HTTP header | `…Infrastructure/Extensions.cs:93-96` |
| `AppContextFactory` | chooses the AMQP source or the HTTP source | `…Infrastructure/Contexts/AppContextFactory.cs:19-33` |

**Representation & storage.** On the wire it is **JSON in a header**: `Correlation-Context` over HTTP
(`…Infrastructure/Extensions.cs:94`) and `message_context` over AMQP
(`Api/appsettings.json:145-148`). In process it is an `IAppContext` registered as a transient
factory delegate: `AddTransient(ctx => ctx.GetRequiredService<IAppContextFactory>().Create())`
(`…Infrastructure/Extensions.cs:52`).

**Lifecycle.** `AppContextFactory.Create()` runs **per resolution**, and it is transport-aware
(`:21-32`): if Convey's `ICorrelationContextAccessor` has an AMQP context, it round-trips it through
JSON (`:23,27`) — serialise then immediately deserialise, an odd but harmless deep copy; otherwise it
falls back to the HTTP header (`:30`); if neither yields anything it returns `AppContext.Empty`
(`:26,32`), which mints a fresh `RequestId` (`AppContext.cs:11`) and an empty identity.

**Invariants & enforcement.** Three details are load-bearing.

- **`IsAdmin` is derived, not transmitted.** `IdentityContext.cs:29` computes
  `Role.Equals("admin", StringComparison.InvariantCultureIgnoreCase)`. The role string arrives from
  the gateway inside the header; the boolean is this service's interpretation of it. **The string
  `"admin"` is the authorization vocabulary and it is a literal in one file**, matched
  case-insensitively.
- **A malformed id degrades silently.** `Id = Guid.TryParse(id, out var userId) ? userId : Guid.Empty`
  (`:26`). An unparseable user id becomes the empty `Guid` rather than an error, so a caller with a
  corrupt identity looks like an anonymous caller with a valid-looking id.
- **The header is attacker-controlled.** `GetCorrelationContext` deserialises whatever JSON is in the
  `Correlation-Context` header with no signature, no validation and no authentication
  (`…Infrastructure/Extensions.cs:94-95`). Anything that can reach port 5009 can assert
  `{"user": {"role": "admin", "isAuthenticated": true}}` and this service will construct an
  `IdentityContext` with `IsAdmin == true`. That is safe **today only because nothing reads it**
  (§3.40) — it is a trust boundary that exists in code, is not enforced, and would become a
  privilege-escalation path the moment the first authorization check is written against
  `IAppContext`. Recorded as **B-1**; see §7.5 for the prerequisite to doing that safely.

Note also that `JsonConvert.DeserializeObject` on a malformed header throws
(`…Infrastructure/Extensions.cs:95`), and since the factory is invoked per resolution, that would
surface as a 400 with `{code: "error"}` (§3.26) rather than as a meaningful message.

**Extension procedure.** See §7.5. The short version: do not read `IAppContext` for an authorization
decision until the header is either authenticated at the service (certificate or JWT) or the service
is unreachable except through the gateway.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A caller appears anonymous despite sending an id | unparseable `Guid` → `Guid.Empty` (`:26`) |
| Malformed header → opaque 400 | deserialisation throws; caught by the generic arm (§3.26) |
| Spoofed admin identity | header is unauthenticated (B-1) |

### 3.39 `Saga` header forwarding and `span_context` propagation

**Definition.** Two `internal static` helpers in `…Infrastructure/Extensions.cs`:

- `GetHeadersToForward(this IMessageProperties)` (`:98-112`) — looks for **one** header, the literal
  `"Saga"` (`:100`), and returns a one-entry dictionary containing it, or `null` if it is absent or
  its value is null (`:101-111`).
- `GetSpanContext(this IMessageProperties, string header)` (`:114-127`) — reads the named header,
  requires the value to be `byte[]` (`:121`), decodes it as UTF-8 (`:123`), and returns
  `string.Empty` otherwise (`:118,126`).

**Representation & storage.** Both operate on AMQP message headers. Their results feed
`MessageBroker.PublishAsync` (`…Infrastructure/Services/MessageBroker.cs:57,63`).

**Lifecycle.** Per publish, on the AMQP path only — on the HTTP path `messageProperties` is null, so
`GetHeadersToForward` returns `null` (`:101`) and `GetSpanContext` returns `string.Empty` (`:117-118`),
after which `MessageBroker` substitutes the active Jaeger span
(`…Infrastructure/Services/MessageBroker.cs:58-61`).

**Invariants & enforcement.** The forwarding list is **exactly one header, hard-coded**. This is the
[[saga-process-manager]] hook: `ordermaker-saga-service` stamps a `Saga` header on the commands it
issues, and every participating service must echo it back on the events it publishes, or the saga
cannot correlate the reply to its instance. **This service echoes it correctly** — but the mechanism
is a string literal duplicated in every service, with nothing checking that the spelling matches
(`"Saga"`, capital S). A service that spells it differently breaks saga correlation with no error on
either side.

`GetSpanContext`'s `byte[]` type check (`:121`) is a silent-drop guard: RabbitMQ delivers header
values as `byte[]` in the .NET client, so a header written as a `string` by a non-.NET publisher would
fail the check and yield `string.Empty` — the trace would break at this hop with no diagnostic.
[[correlation-and-span-propagation]].

**Extension procedure.** To forward another header, extend `GetHeadersToForward` — note it currently
returns `null` (not an empty dictionary) when the saga header is absent, so any addition must
restructure the early return at `:101-104` or the new header is dropped whenever `Saga` is missing.
That is the specific trap: the method's shape assumes exactly one header.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| A saga never completes | `Saga` header not echoed — would require this method to change; correct today |
| Traces break at this service | span header not `byte[]`, or absent, with no active span |
| A newly forwarded header vanishes intermittently | `:101` early-returns `null` when `Saga` is absent |

### 3.40 The caller context is built on every request and consumed by nothing

**Definition.** `AddTransient(ctx => ctx.GetRequiredService<IAppContextFactory>().Create())`
(`…Infrastructure/Extensions.cs:52`) registers `IAppContext` for injection. **No constructor in this
repository takes an `IAppContext` or an `IIdentityContext`** — verified by absence across all 48
source files. No handler, no repository, no query, no mapper.

**Representation & storage.** N/A.

**Lifecycle.** Because it is a **transient factory delegate**, `Create()` runs only when something
resolves `IAppContext`. Nothing does. So — and this is the correction to the intuitive reading —
the context is **not** actually built per request; the machinery is registered and never invoked.
The header parsing at `…Infrastructure/Extensions.cs:93-96` runs only via
`MessageBroker.PublishAsync`'s fallback (`…Infrastructure/Services/MessageBroker.cs:65`), which uses
the raw `CorrelationContext` and never reaches `AppContextFactory`.

**Invariants & enforcement.** Five types, one factory, one extension method and two DI registrations
exist to answer a question nobody asks. `pricing-service.md` §3.19 pairs this with its own opposite
case: pricing has no context machinery at all, vehicles has all of it and no consumers. Both services
end up equally identity-blind; they differ only in how much unused code they carry.

The consequence for a maintainer is precise and worth stating plainly: **adding
`IAppContext` to a handler's constructor is a one-line change that immediately activates an
unauthenticated, attacker-controlled input path** (§3.38, B-1). The code looks finished and safe. It
is finished and unsafe-when-used.

**Extension procedure.** Either (a) delete `Contexts/`, `IAppContextFactory`, `IAppContext`,
`IIdentityContext`, `GetCorrelationContext` and the two registrations — about 130 lines, no behaviour
change; or (b) keep them and add the authentication that makes them trustworthy (§7.5). Do not leave
it in the current state while writing new code that might reach for it. Recommendation: **(b)**, and
only if authorization is actually wanted here; otherwise (a).

**Failure modes.** None today. The failure mode is a future one, and it is a security failure.

### 3.41 Inert JWT configuration and the absent certificate authentication

**Definition.** Three related absences, each verified.

1. **`AddSecurity()`** is called (`…Infrastructure/Extensions.cs:74`), which registers Convey's
   security primitives — hashing, encryption and the certificate *services* `[convey]`.
2. **`AddCertificateAuthentication()` is not called**, and there is **no `security` section** in any
   of the four `appsettings*.json` files. Compare `customers-service`, which calls it and configures
   a full ACL (`hianshul100_Pacco.Services.Customers/…Infrastructure/Extensions.cs:79-80,91` and
   `…Customers.Api/appsettings.json:163-182`, where `availability-service` is granted
   `customers:read`).
3. **`jwt` is configured (`Api/appsettings.json:77-85`) and `AddJwt()` is never called.** The block is
   dead configuration.

**Representation & storage.** `Api/Pacco.Services.Vehicles.Api.csproj:20` includes `certs\**` as
content, so certificate files ship in the image — but nothing loads them for authentication.

**Lifecycle.** N/A — the code path does not exist.

**Invariants & enforcement.** **Every endpoint and every AMQP command on this service is
unauthenticated.** The only enforcement is the gateway's `auth: true` on all five routes
(`ntrada.yml:415-456`), which protects the path *through* the gateway and nothing else. Anything with
network access to port 5009 — any other service, any pod in the cluster, anything that can reach the
Docker network — can create, update or delete any vehicle, and can publish to exchange `vehicles`
(§3.32).

The asymmetry with `customers-service` is the finding: the platform demonstrably *has* a
service-to-service authentication mechanism ([[vault-issued-dynamic-credentials-and-service-pki]] plus
the certificate ACL) and this service does not use it, while a service with arguably lower-value data
does. There is no comment explaining the difference. Recorded as **B-1** and **Q-2**.

Note that `availability-service` demonstrates the *client* half of that mechanism, attaching a Vault
PKI certificate to outbound calls
(`hianshul100_Pacco.Services.Availability/…/Clients/CustomersServiceClient.cs:16-34`) — so both halves
exist in the platform and neither is wired here.

**Extension procedure.** To add certificate authentication: add `AddCertificateAuthentication()` to
the builder chain, add a `security.certificate` section modelled on
`hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-182` with
an ACL naming the services allowed to call this one (`orders-service` for reads, the gateway for
writes), and confirm that the Vault PKI role issues certificates the ACL will accept
(`Api/appsettings.json:177-181`). **`Unverifiable — Missing Source Evidence`** — whether Convey's
certificate authentication supports per-route permissions cannot be settled here; `customers-service`
declares `permissions: ["customers:read"]` in its ACL but passes no permission argument at any route
(`…Customers.Api/Program.cs:33-41`), so the permission list may be advisory. Verify before relying on
per-route granularity.

**Failure modes.** Unauthenticated writes from anywhere on the network, over either transport. Fails
**silently** — there is no rejection to observe, because there is no check.

### 3.42 Consul registration and Fabio addressing

**Definition.** `AddConsul()` and `AddFabio()` (`…Infrastructure/Extensions.cs:61-62`), configured at
`Api/appsettings.json:7-22`. [[registry-mediated-discovery-and-routing]].

| Key | Value | Effect |
| --- | --- | --- |
| `consul.enabled` | `true` | the service registers itself at startup |
| `consul.url` | `http://localhost:8500` (`docker`: `http://consul:8500`) | the agent |
| `consul.service` | `vehicles-service` | the **registered name** other services address |
| `consul.address` | `docker.for.win.localhost` (`:11`) | **the address advertised to the registry** |
| `consul.port` | `5009` (`:12`) | the advertised port |
| `consul.pingEnabled` / `pingEndpoint` / `pingInterval` / `removeAfterInterval` | `true` / `ping` / `5` / `10` | health checking; the instance is deregistered 10 s after failures start |
| `fabio.enabled` / `url` / `service` | `true` / `http://localhost:9999` / `vehicles-service` | outbound calls resolve logical names through Fabio |

**Representation & storage.** The registration lives in Consul, not here.

**Lifecycle.** Registered at startup, health-checked every 5 s, deregistered 10 s after failure.

**Invariants & enforcement.** **`consul.address: docker.for.win.localhost` is a Windows-Docker-specific
hostname** and it is in the **base** `appsettings.json`, which is the profile the production PM2
manifest runs (`hianshul100_Pacco/prod-services.yml` sets `ASPNETCORE_URLS` but **not**
`ASPNETCORE_ENVIRONMENT` — §3.44). On any non-Windows host that name does not resolve, so the service
registers an unreachable address and every Fabio-routed call to `vehicles-service` fails. The Docker
profile overrides it (`appsettings.docker.json:7-17`) and the local profile disables Consul entirely
(`appsettings.local.json`), so the defect is invisible in both developer environments. This is
**B-3**, and it is identical to the finding recorded for `pricing-service`
(`component-internals/pricing-service.md` §8.2 B-2).

`fabio` is enabled but `httpClient.services` is `{}` (§1.2) — this service calls nobody, so Fabio's
registration is as unused as Redis's (§3.36).

**Extension procedure.** To call another service, add its logical name to `httpClient.services` in
every profile and inject `IHttpClient`; Fabio resolves the name. Do not hard-code host:port —
`appsettings.local.json` shows the escape hatch (`httpClient.type: ""` plus direct URLs) for local
work.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| Registered but unreachable in production | `docker.for.win.localhost` in the base profile (B-3) |
| Instance disappears from the registry under load | ping timeout; `removeAfterInterval: 10` |

### 3.43 Vault — KV, PKI and the dynamic Mongo credential lease

**Definition.** `UseVault()` (`…Api/Program.cs:43`) installs Convey's Vault configuration provider
before the host builds, so Vault-sourced values participate in the configuration hierarchy `[convey]`.
Configured at `Api/appsettings.json:164-193`. [[vault-issued-dynamic-credentials-and-service-pki]].

| Key | Value | Effect |
| --- | --- | --- |
| `enabled` | `true` (`false` in `local` and `docker`) | the provider runs only in the base profile |
| `url` | `http://localhost:8200` | the Vault address |
| `authType` / `token` | `token` / `"secret"` (`:168`) | the unchanged development root token — **B-2** |
| `kv.enabled` / `path` | `true` / `vehicles-service/settings` (`:175`) | per-service settings overlay |
| `pki.enabled` / `roleName` / `commonName` | `true` / `vehicles-service` / `vehicles-service.pacco.io` (`:177-181`) | issues a client certificate for outbound mTLS — **which this service never uses** (§3.41) |
| `lease.mongo.*` | `enabled: true`, `role: vehicles-service`, `autoRenewal: true`, `templates.connectionString: mongodb://{{username}}:{{password}}@localhost:27017` (`:182-192`) | **dynamic, rotating Mongo credentials** |

**Representation & storage.** The lease is the interesting part: Vault issues a short-lived Mongo
username/password, Convey substitutes them into the connection-string template, and `autoRenewal: true`
renews the lease before expiry `[convey]`.

**Lifecycle.** Read once at startup for KV and PKI; the Mongo lease is renewed continuously for the
life of the process.

**Invariants & enforcement.** Three consequences.

- **The Mongo credentials in `appsettings.json` are placeholders**, not real credentials — the
  template at `:189` is `mongodb://{{username}}:{{password}}@localhost:27017`, and the real values
  never touch the repository. This is the correct pattern and it is the reason no database credential
  is committed anywhere in this repository. `vault.token: "secret"` (`:168`) is the one credential-ish
  literal, and it is the unmodified development default rather than a real token — quoted here for
  exactly that reason (§3.37).
- **`autoRenewal: true` makes Vault a hard runtime dependency, not just a startup one.** If Vault
  becomes unreachable mid-life and the lease expires, Mongo starts rejecting the service's
  connections. The failure appears as database errors, not as a Vault error — a long way from the
  cause.
- **The template hard-codes `localhost:27017`** (`:189`). Combined with `vault.enabled: false` in the
  Docker profile (`appsettings.docker.json:87-101`), the lease path is exercised only in the base
  profile — which is production (§3.44) — where `localhost` is presumably correct only if Mongo runs
  on the same host. Nothing in this workspace confirms that topology; **`Unverifiable — Missing Source
  Evidence`**, recorded as **Q-4**.

**Extension procedure.** New secrets go in the KV path `vehicles-service/settings` and are read as
ordinary configuration keys — no code change. To add a second dynamic credential (e.g. RabbitMQ), add
a `lease` entry with its own template. Never move a secret into `appsettings.json`; the KV overlay
exists precisely so that is unnecessary.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| Mongo auth failures hours after a healthy start | Vault unreachable; lease expired; `autoRenewal` could not run |
| Service will not start | Vault down and `enabled: true` |
| Config differs between environments unexpectedly | the KV overlay applies only where `vault.enabled` is true |

### 3.44 Environment layering — four profiles, and what `docker` inherits

**Definition.** Four files in `…Api/`, layered by ASP.NET Core's configuration system in the order
base → environment ([[composable-per-concern-environment-stacks]]):

| Profile | Lines | Character |
| --- | --- | --- |
| `appsettings.json` | 194 | the full configuration; **everything enabled** |
| `appsettings.local.json` | 52 | everything off — Consul, Fabio, Vault, metrics, **and the outbox** (`:35-37`) |
| `appsettings.docker.json` | 102 | container hostnames; Vault off (`:87-101`) |
| `appsettings.development.json` | `{}` | **empty** |

`launchSettings.json:14,22` sets `ASPNETCORE_ENVIRONMENT=local` for both launch profiles, and both
bind port `5009` (`:6,20`).

**Representation & storage.** JSON overlays; only the keys present in an overlay are replaced.

**Lifecycle.** Resolved once at startup.

**Invariants & enforcement.** The layering itself is sound. Three specific consequences are not.

- **The base profile is the production profile.** `hianshul100_Pacco/prod-services.yml:56-58` runs
  `vehicles` with `ASPNETCORE_URLS` set and **no `ASPNETCORE_ENVIRONMENT`**, so no overlay applies and
  the service runs `appsettings.json` verbatim — including `consul.address: docker.for.win.localhost`
  (B-3, §3.42) and `vault.token: "secret"` (B-2, §3.43). The development PM2 manifest
  (`hianshul100_Pacco/services.yml:38-41`) sets neither variable, so it *also* runs the base profile,
  on `dotnet run`.
- **`appsettings.development.json` is empty**, so `ASPNETCORE_ENVIRONMENT=Development` — the .NET
  default when nothing is set in many hosting setups — resolves to the base profile too. Three
  distinct ways to end up on the production configuration by accident.
- **Overlays are partial and the gaps matter.** `appsettings.docker.json` has **no `outbox` section**,
  so it inherits `disableTransactions: true` (§3.30); it overrides **only** `rabbitMq.hostnames`
  (`:69-73`), so exchange, queue template and conventions are shared with the base file (§3.31) —
  which is good; and it disables Vault, so the dynamic Mongo lease never runs in containers (§3.43).

**Extension procedure.** A new configuration key must be added to `appsettings.json` **and** to every
overlay that would otherwise inherit an unusable value — in practice `local` (for anything requiring
infrastructure) and `docker` (for anything with a hostname). Fill in `appsettings.development.json` or
delete it; an empty environment file is a trap.

**Failure modes.** The production deployment runs a configuration nobody tests. Recorded as **B-3**.

### 3.45 Metrics, tracing, and the duplicated `AddMongo()`

**Definition.** Three observability registrations plus one duplication, all in the builder chain
(`…Infrastructure/Extensions.cs:56-74`):

| Call | Line | Configured at |
| --- | --- | --- |
| `AddMongo()` | **`:66`** | `Api/appsettings.json:95-99` |
| `AddRedis()` | `:67` | `:151-154` (§3.36) |
| `AddMetrics()` | `:68` | `:86-94` |
| `AddJaeger()` | `:69` | `:68-76` |
| `AddMongo()` **again** | **`:70`** | — |

**Representation & storage.** `AddMongo()` appears **twice, four lines apart**, with `AddRedis`,
`AddMetrics` and `AddJaeger` between them. `[convey]` **`Unverifiable — Missing Source Evidence`** —
whether Convey's `AddMongo` is idempotent (guarded by a `TryAdd`) or registers a second set of
services cannot be settled without its source. The observable behaviour is that the service starts and
works, so it is at worst harmless duplication; at best it is a copy-paste artifact. It is recorded
because a maintainer deleting "the redundant one" needs to know that **neither line is provably the
redundant one** without checking the package.

`UseInfrastructure` mirrors the chain: `UseErrorHandler` → `UseSwaggerDocs` → `UseJaeger` →
`UseConvey` → `UsePublicContracts` → `UseMetrics` → `UseRabbitMq` → three `SubscribeCommand`s
(`:79-88`). Order matters here: `UseErrorHandler` is first so it wraps everything after it.

The observability configuration:

| Key | Value | Effect |
| --- | --- | --- |
| `metrics.enabled` / `prometheusEnabled` / `prometheusFormatter` | `true` / `true` / `prometheus` (`:86-94`) | `/metrics` scraped by `hianshul100_Pacco/compose/prometheus/prometheus.yml:50-52` |
| `jaeger.enabled` | `true` | tracing on |
| `jaeger.serviceName` | **`vehicles`** (`:70`) | **not** `vehicles-service` |
| `jaeger.sampler` | `const` with `maxTracesPerSecond: 5` | every request traced |

**Invariants & enforcement.** The **name mismatch is the finding**: this service is `vehicles-service`
to Consul (`:9`), `vehicles-service` to Prometheus, `vehicles-service` to RabbitMQ's `connectionName`,
`vehicles-service` as a Mongo database — and **`vehicles`** to Jaeger. Correlating a Jaeger trace with
a Prometheus metric or a Consul health check requires knowing that the two names refer to the same
process. Nothing records it. The identical mismatch exists in `pricing-service`
(`component-internals/pricing-service.md` §3.29), so it is a platform-wide convention drift, not a
local typo — which is exactly why it is unlikely to be fixed and important to document.

`sampler: const` means **100 % of requests are traced**. Fine at this service's volume; a cost and
storage consideration at scale, and a change that must be made per service.

**Extension procedure.** Aligning `jaeger.serviceName` to `vehicles-service` is a one-key change with
a real cost: **it orphans every existing trace** stored under `vehicles`, and Jaeger's UI groups by
service name. Do it across all services at once or not at all.

**Failure modes.** Cross-tool correlation requires out-of-band knowledge. Fails **silently** — every
tool works, they just disagree about the name.

### 3.46 Absence of a test suite, and the green-but-empty test step

**Definition.** Verified by absence: `Pacco.Services.Vehicles.sln` declares four projects, none of
them a test project; there is no `tests/` directory; no `*.Tests.csproj` exists anywhere in the
repository; and no test framework package is referenced by any of the four `.csproj` files.

Meanwhile `scripts/test.sh` runs `dotnet test`, and `.travis.yml` invokes the script set in CI.

**Representation & storage.** N/A.

**Lifecycle.** `dotnet test` against a solution with no test projects **exits 0**. CI is green and has
always been green, and it has never executed an assertion about this service.

**Invariants & enforcement.** Nothing in this document's §3 is protected by a test. Specifically
untested: the variant floor (§3.4), the search filter (§3.18), all four mapping functions (§3.21),
every `ExceptionToMessageMapper` arm (§3.28), the `DeleteVehicle` binding (§3.13), and every domain
guard (§3.6, §3.8).

The platform has both a testing pattern and a working example —
[[layered-service-test-suite]] as implemented in `orders-service` and `parcels-service` — so the
absence here is a gap, not a platform-wide policy. `component-internals/index.md` §2 excludes test
projects from the modelling target for a different reason (they compile into their owning host), which
should not be read as saying they do not matter.

**Extension procedure.** The highest-value first tests, in order, are all pure and need no
infrastructure:

1. `Vehicle` construction with `variants: Chemistry` → assert `Variants == (Standard | Chemistry)`.
   This pins §3.4 so a fix is a deliberate change.
2. `AsEntity`/`AsDocument`/`AsDto` round-trip over all eight fields — catches the §3.21 mapping
   hazard, which is the most likely future regression.
3. `AsDto` on `Variants = 3` → assert the exact array, including the leading space (§3.19).
4. `ExceptionToMessageMapper.Map` for every (exception, command) pair → assert non-null where a
   rejection is expected. This is the test that would have caught §3.28's `DeleteVehicle` gap.

Add a `tests/Pacco.Services.Vehicles.Tests` project referencing `Core` and `Infrastructure`, add it to
the `.sln`, and `scripts/test.sh` starts doing something.

**Failure modes.** Every failure in this document ships undetected. CI reports success.

### 3.47 Deployment identity and downstream consumers

**Definition.** How this service is built, named, deployed and consumed.

| Concern | Fact | Evidence |
| --- | --- | --- |
| Build | multi-stage; `dotnet publish -c release` | `Dockerfile:4,9,10` |
| Image | `pacco.services.vehicles` | `scripts/dockerize.sh` |
| Compose service | `vehicles-service`, image `devmentors/pacco.services.vehicles`, ports `5009:80` | `hianshul100_Pacco/compose/services.yml` |
| PM2 (dev) | `vehicles`, `dotnet run`, no env vars | `hianshul100_Pacco/services.yml:38-41` |
| PM2 (prod) | `vehicles`, `ASPNETCORE_URLS: http://*:5009`, **no `ASPNETCORE_ENVIRONMENT`** | `hianshul100_Pacco/prod-services.yml:56-58` |
| Prometheus | job scraping this service | `hianshul100_Pacco/compose/prometheus/prometheus.yml:50-52` |
| CI | Travis, running the four `scripts/*.sh` | `.travis.yml` |

**Consumers, and what they actually take:**

| Consumer | Takes | Evidence |
| --- | --- | --- |
| `api-gateway` (sync) | all five routes, `downstream`, `auth: true`; GET-list flattens to `response.data.items`; POST uses `resourceId: {property: vehicleId, generate: true}` | `ntrada.yml:415-456` |
| `api-gateway` (async) | GETs stay `downstream`; POST/PUT/DELETE become `use: rabbitmq` on exchange `vehicles` with `bind: - vehicleId:{vehicleId}` on PUT/DELETE | `ntrada-async.yml:483-534` |
| `orders-service` | `GET /vehicles/{id}`, deserialised into a **2-property** `VehicleDto` (`Id`, `PricePerService`) | `…Orders/…/Clients/VehiclesServiceClient.cs:14-21`; `…Orders.Application/DTO/VehicleDto.cs:5-9` |
| `availability-service` | `VehicleDeleted` → `DeleteResource` | `…Availability/…/Events/External/VehicleDeleted.cs:7`; `…/Handlers/VehicleDeletedHandler.cs:17`; `…Availability.Infrastructure/Extensions.cs:112` |

**Invariants & enforcement.** Three points a maintainer needs.

- **`orders-service` consumes 2 of the 8 fields.** Its local `VehicleDto` has only `Id` and
  `PricePerService`, so **the `Variants` string-array quirk (§3.19) does not affect it** and the
  read contract can be changed for the other six fields without breaking it. The two fields it does
  read are effectively frozen: renaming `PricePerService` breaks order pricing silently, because JSON
  deserialisation of a missing property yields `0` rather than an error — and a price of `0` flows
  into `pricing-service`'s discount calculation
  (`component-internals/pricing-service.md` §3.6) without any validation.
- **The sync and async gateway modes expose different write semantics** for the same three routes
  (§1.3), and both must be updated together when a route changes.
- **The image, the compose service, the PM2 process and the Consul registration use three different
  names** (`pacco.services.vehicles`, `vehicles-service`, `vehicles`). Add `jaeger.serviceName`
  (§3.45) and the count is unchanged but the mapping is one more thing to know.

**Extension procedure.** Adding a field to `VehicleDto` is backward-compatible for `orders-service`
(it ignores unknown properties). **Removing or renaming `Id` or `PricePerService` is not**, and the
break is silent. Coordinate through `component-internals/orders-service.md` §6 before touching either.

**Failure modes.**

| Symptom | Cause |
| --- | --- |
| Orders priced at 0 | `PricePerService` renamed; missing property deserialises to `0` |
| A route works in sync mode and not async | only one `ntrada*.yml` pair updated |
| Deleted vehicles remain bookable | `VehicleDeleted` not delivered (§3.30 non-atomicity, or an unroutable publish) |

---

## 4. Primary control flows

### 4.1 `POST /vehicles` over HTTP — creation end to end

The most instructive trace, because it passes through the variant floor, the outbox and the
`afterDispatch` callback.

| # | Step | Evidence |
| --- | --- | --- |
| 1 | Client calls the gateway; `auth: true` is enforced there — a missing or invalid token never reaches this service | `ntrada.yml:432-436` |
| 2 | Ntrada generates a `vehicleId` (`resourceId: {property: vehicleId, generate: true}`) and forwards the request to `localhost:5009` | `ntrada.yml:437-439` |
| 3 | Kestrel receives `POST /vehicles`; `UseErrorHandler` wraps everything downstream | `…Infrastructure/Extensions.cs:79` |
| 4 | The dispatcher endpoint matches and binds the JSON body to `AddVehicle` through its constructor | `…Api/Program.cs:37` |
| 5 | `AddVehicle`'s constructor mints an id if `vehicleId` is `Guid.Empty` | `…Application/Commands/AddVehicle.cs:22` |
| 6 | The command is dispatched to `ICommandHandler<AddVehicle>`, which is **decorated** | `…Infrastructure/Extensions.cs:53` |
| 7 | `OutboxCommandHandlerDecorator` finds no inbound message id and generates a fresh `Guid` — so the inbox key is unique and dedup cannot fire | `…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:27-29` |
| 8 | `_inbox.HandleAsync(messageId, …)` wraps the inner handler (outbox enabled in base/docker; **bypassed entirely** in `local`) | same, `:32-35`; `appsettings.local.json:35-37` |
| 9 | `AddVehicleHandler` constructs `new Vehicle(…)` — the 7-arg constructor | `AddVehicleHandler.cs:26` |
| 10 | `ChangeDescription` validates; the two capacity ternaries validate; `ChangePricePerService` validates | `…Core/Entities/Vehicle.cs:23-26` |
| 11 | **`AddVariants(Variants.Standard)` ORs bit 0 in** — the caller's `variants: 2` becomes `3` | `…Core/Entities/Vehicle.cs:27` |
| 12 | `_repository.AddAsync(vehicle)` → `AsDocument()` → `IMongoRepository.AddAsync` → collection `vehicles` | `VehiclesMongoRepository.cs`; `…Infrastructure/Extensions.cs:72` |
| 13 | `_messageBroker.PublishAsync(new VehicleAdded(command.VehicleId))` | `AddVehicleHandler.cs:33` |
| 14 | `MessageBroker` resolves span context (active Jaeger span, since there is no inbound message), correlation context (from the `Correlation-Context` header, if any), and headers to forward (`null` — no `Saga` header on HTTP) | `MessageBroker.cs:57-65` |
| 15 | A fresh `messageId` is minted; `LogTrace` fires **below the configured level and is discarded**; `_outbox.Enabled` is true so the event is staged, not published | `MessageBroker.cs:74-81`; `appsettings.json:33` |
| 16 | The handler returns. **No transaction spans steps 12 and 15** | `appsettings.json:107` |
| 17 | `MessageToLogTemplateMapper`'s `After` template logs `Added a vehicle with id: {VehicleId}.` with `VehicleId` as a structured property | `MessageToLogTemplateMapper.cs:17` |
| 18 | `afterDispatch` writes `201 Created` with `Location: vehicles/{cmd.VehicleId}` | `…Api/Program.cs:38` |
| 19 | Up to 2 s later the outbox dispatcher drains the message to exchange `vehicles`, routing key `vehicle_added` | `appsettings.json:104,113,131-137` |
| 20 | **Nothing is subscribed to `vehicle_added`.** RabbitMQ discards it | §3.27 |

**What the client observes:** `201` with a `Location`. **What is true afterwards:** a vehicle stored
with `Variants = 3` that `GET /vehicles?variants=2` will never return (§3.18), and an event that
reached no consumer.

### 4.2 `GET /vehicles?payloadCapacity=…` — the search path

| # | Step | Evidence |
| --- | --- | --- |
| 1 | Gateway forwards with `passQueryString` semantics; the query string arrives intact | `ntrada.yml:13`, route at `:418-424` |
| 2 | `Get<SearchVehicles, PagedResult<VehicleDto>>` binds the query string to `SearchVehicles`'s **settable** properties — no constructor involvement, so §3.13's trap does not apply | `…Api/Program.cs:36`; `SearchVehicles.cs:7-12` |
| 3 | Dispatched to `SearchVehiclesHandler`, which depends on `IMongoRepository<VehicleDocument, Guid>` — **not** on the domain repository | `SearchVehiclesHandler.cs:18` |
| 4 | The guard at `:21` asks whether *all three* filters are unset | `SearchVehiclesHandler.cs:21` |
| 5a | **All unset** → `BrowseAsync(_ => true, query)` → every vehicle, paged | `:23` |
| 5b | **Any set** → `BrowseAsync(v => v.PayloadCapacity >= … && v.LoadingCapacity >= … && v.Variants == query.Variants, query)` | `:27-29` |
| 6 | In branch 5b with `variants` unsupplied, the predicate demands `Variants == 0` — **which no vehicle satisfies** (§3.4) | §3.18 |
| 7 | `pagedResult?.Map(d => d.AsDto())` — document → DTO directly; **no `Vehicle` is constructed**, so no domain guard runs | `:33`; §3.21 |
| 8 | `AsDto` splits `Variants.ToString()` on `','`, leaving a leading space on every element after the first | `Mongo/Documents/Extensions.cs:49` |
| 9 | Serialised as `PagedResult<VehicleDto>`; the gateway flattens to `response.data.items` for the client | `ntrada.yml` GET-list `onSuccess.data` |

**The important asymmetry:** because step 7 bypasses the entity, a document with a non-positive
capacity appears in search results but throws on `GET /vehicles/{id}` and on any update or delete
(§3.8). Search and point-read disagree about which vehicles exist.

### 4.3 `delete_vehicle` over AMQP — the async write path

| # | Step | Evidence |
| --- | --- | --- |
| 1 | Gateway in async mode publishes to exchange `vehicles`, routing key `delete_vehicle`, body `{"vehicleId": "…"}` from `bind: - vehicleId:{vehicleId}` | `ntrada-async.yml:521-529` |
| 2 | The queue `vehicles-service/vehicles.delete_vehicle` receives it | `appsettings.json:138-144` |
| 3 | Convey deserialises the payload into `DeleteVehicle` — **via its single-parameter constructor whose parameter is named `id`, not `vehicleId`** (§3.13) | `DeleteVehicle.cs:11-12` |
| 4 | The message carries a broker-assigned message id, so the outbox decorator uses it — **inbox deduplication actually works on this path**, unlike HTTP | `OutboxCommandHandlerDecorator.cs:27-29` |
| 5 | `DeleteVehicleHandler` calls `GetAsync`, which **rehydrates the entity through the constructor** and re-runs every domain guard | `DeleteVehicleHandler.cs:23`; §3.8 |
| 6a | Vehicle absent → `VehicleNotFoundException` → `ExceptionToMessageMapper` → `DeleteVehicleRejected` published | `ExceptionToMessageMapper.cs:33-38` |
| 6b | Vehicle present but stored with a bad capacity → `InvalidVehicleCapacity` → the mapper's first arm has **no `DeleteVehicle` case** → `null` → **nothing is published; the caller never learns** | `ExceptionToMessageMapper.cs:15-20` |
| 7 | Success → `DeleteAsync` → `PublishAsync(new VehicleDeleted(id))` → outbox → exchange `vehicles` | `DeleteVehicleHandler.cs:29-31` |
| 8 | `availability-service` receives it on its own queue and dispatches `DeleteResource(@event.VehicleId)` | `…Availability/…/Handlers/VehicleDeletedHandler.cs:17` |

Step 6b is the concrete instance of §3.28's silent drop, and step 3 is the concrete instance of
§3.13's binding ambiguity. Both are on the same eight-step path.

### 4.4 Startup

| # | Step | Evidence |
| --- | --- | --- |
| 1 | `WebHost.CreateDefaultBuilder(args)` | `…Api/Program.cs:24` |
| 2 | `AddConvey().AddWebApi().AddApplication().AddInfrastructure().Build()` | `:26-30` |
| 3 | `AddApplication` registers command/event handlers and the in-memory command and event dispatchers | `…Application/Extensions.cs:9-14` |
| 4 | `AddInfrastructure` registers four transients, then applies the two `TryDecorate` calls | `…Infrastructure/Extensions.cs:49-54` |
| 5 | The builder chain runs 18 registrations, including `AddMongo()` **twice** (`:66`, `:70`) | `:56-74` |
| 6 | `AddMongoRepository<VehicleDocument, Guid>("vehicles")` binds the collection | `:72` |
| 7 | `Configure` runs `UseInfrastructure`: error handler, Swagger, Jaeger, Convey, public contracts, metrics, RabbitMQ | `:79-85` |
| 8 | Exchange `vehicles` is declared (`declare: true`, `durable: true`) and three command queues are bound | `appsettings.json:131-144`; `…Infrastructure/Extensions.cs:86-88` |
| 9 | `UseDispatcherEndpoints` registers the six routes | `…Api/Program.cs:33-41` |
| 10 | `UseLogging()` configures Serilog with the exclusions and sinks | `:42`; `appsettings.json:32-67` |
| 11 | `UseVault()` fetches KV settings, the PKI certificate, and **starts the auto-renewing Mongo credential lease** | `:43`; `appsettings.json:164-193` |
| 12 | Consul registration announces `vehicles-service` at `consul.address:5009` | `appsettings.json:7-17` |
| 13 | `RunAsync()` | `…Api/Program.cs:45` |

Startup **does not** verify: that Mongo is reachable, that any handler resolves, that the exchange
name matches what the gateway publishes to, or that `httpClient.services` contains anything. The first
three surface at first use; the fourth never surfaces because nothing is called.

### 4.5 Failure matrix

| Trigger | Path | Where it is caught | What the caller sees | Loud or silent |
| --- | --- | --- | --- | --- |
| Blank description | HTTP POST/PUT | `Vehicle.ChangeDescription` (`:39-42`) | 400 `invalid_vehicle_description` | **Loud** |
| Description of `" "` | HTTP POST/PUT | **not caught** — `IsNullOrEmpty` (§3.6) | 200/201, blank-looking vehicle | **Silent** |
| Blank `Brand`/`Model` | HTTP POST | **not caught** (§3.1) | 201 | **Silent** |
| Capacity ≤ 0 | HTTP POST | `Vehicle` ctor (`:24-25`) | 400 `invalid_vehicle_capacity` | **Loud** |
| Price ≤ 0 | HTTP POST/PUT | `Vehicle.ChangePricePerService` (`:49-52`) | 400 `invalid_vehicle_price_per_service` | **Loud** |
| `variants: 999` | HTTP POST | **not caught** (§3.3) | 201; later `AsDto` returns `["999"]` | **Silent** |
| Unknown vehicle | HTTP PUT/DELETE | `VehicleNotFoundException` | 400 `vehicle_not_found` (**not 404**) | **Loud but miscoded** |
| Unknown vehicle | AMQP update/delete | `ExceptionToMessageMapper:33-38` | `*Rejected` event | **Loud (async)** |
| Bad stored capacity | AMQP delete | mapper's first arm has no `DeleteVehicle` case | **nothing** | **Silent** |
| Mongo timeout | HTTP any | `_` arm (`ExceptionToResponseMapper.cs:22-23`) | 400 `{code: "error"}` | **Loud but uninformative** |
| Mongo timeout | AMQP any | `_ => null` (`ExceptionToMessageMapper.cs:39`) | **nothing** | **Silent** |
| Search by capacity only | HTTP GET | not an error | 200, empty `items` | **Silent** (§3.18) |
| Search by a created variant | HTTP GET | not an error | 200, empty `items` | **Silent** (§3.4) |
| Crash between write and outbox | either | nowhere | success, no event | **Silent** (§3.30) |
| Retried HTTP POST | HTTP | inbox key is per-request | two vehicles | **Silent** (§3.30) |
| Routing-key mismatch after a rename | AMQP | nowhere | publisher succeeds, nothing happens | **Silent** (§3.31) |
| Malformed `Correlation-Context` | HTTP | `JsonConvert` throws → `_` arm | 400 `{code: "error"}` | **Loud but uninformative** |

Read as a whole: **the HTTP surface fails loudly and imprecisely; the AMQP surface fails silently
outside a narrow band of domain validation errors; and the read surface never fails at all, it just
returns nothing.**

---

## 5. Persistence & schema evolution

### 5.1 What is stored

Three collections in one Mongo database, `vehicles-service` (`Api/appsettings.json:95-99`):

| Collection | Owner | Shape | Named by |
| --- | --- | --- | --- |
| `vehicles` | this service | `VehicleDocument` — 8 fields, `_id` = `Guid` | a **code literal** (`…Infrastructure/Extensions.cs:72`) |
| `outbox` | Convey | staged outbound messages | `outbox.outboxCollection` (`:106`) |
| `inbox` | Convey | processed message ids, `expiry: 3600` | `outbox.inboxCollection` (`:105`) |

`seed: false` (`:98`) means no seeding runs; collections are created lazily on first write.

**No index is declared anywhere in this repository.** Searching by `PayloadCapacity`,
`LoadingCapacity` or `Variants` (§3.17) is a full collection scan on every request. At catalogue
scale this is invisible; it is worth knowing before the collection grows. `_id` is indexed by MongoDB
automatically, which is why `GET /vehicles/{id}` is fine.

### 5.2 The four contracts that evolve independently

| Contract | Shape | Changing it breaks | Coordination needed |
| --- | --- | --- | --- |
| **Stored document** | `VehicleDocument` (§3.20) | nothing immediately; old documents read as `default` | data migration for renames |
| **`Variants` ordinals** | the enum's numeric values (§3.22) | **every stored document** if renumbered | never renumber; append only |
| **Read DTO** | `VehicleDto`, incl. the string-array variants (§3.19) | gateway clients; `orders-service` only for `Id`/`PricePerService` | §3.47 |
| **Messages** | three events + three rejected events, `[Contract]`-marked (§3.34) | `availability-service` for `VehicleDeleted`; nobody for the rest | §3.27 |

The **routing-key convention** is a fifth, implicit contract: `conventionsCasing: snakeCase` turns
class names into routing keys, so **renaming a command class is a wire-protocol change** that the
gateway's YAML does not follow (§3.31).

### 5.3 Configuration as schema

Several configuration keys are load-bearing enough that changing them is a schema change:

| Key | Why it is schema | Where |
| --- | --- | --- |
| `rabbitMq.conventionsCasing` | determines every routing key | `appsettings.json:113` |
| `rabbitMq.exchange.name` | the binding every consumer declares | `:132` |
| `rabbitMq.queue.template` | queue identity; changing it orphans in-flight messages | `:139` |
| `mongo.database` | which data the service sees | `:97` |
| `outbox.inboxCollection` / `outboxCollection` | changing them abandons pending messages | `:105-106` |
| `outbox.disableTransactions` | the atomicity guarantee (§3.30) | `:107` |

### 5.4 What a migration would actually cost

There is **no migration mechanism** — no versioned documents, no `seed`, no startup migration hook, no
migration project. Any schema change is a manual, out-of-band operation against Mongo. Concretely:

- **Adding a nullable field:** free. Old documents read as `default`; update all three mappings
  (§3.21).
- **Adding a required field:** requires a backfill script, because the entity's constructor guards run
  on **rehydration** (§3.8) — a document missing a guarded field makes that vehicle permanently
  unreadable.
- **Renaming a field:** a full-collection `$rename`, run while the service is stopped (there is no
  dual-read support).
- **Changing `Variants` to string representation:** backfill every document **and** rewrite the search
  predicate (§3.22, §3.18). The two must ship together.
- **Fixing the variant floor by data (§3.4 option b):** a backfill that is **self-undoing** — rehydration
  re-ORs the bit (§3.5) — so the code change must land first.

The practical rule: **code changes that alter how a document is read are riskier here than changes
that alter how it is written**, because the read path runs the domain constructor on data that was
written by an older version of that constructor.

---

## 6. Surface → internals map

### 6.1 HTTP routes

| Method & path | Binds to | Handler | Reads/writes | Notes |
| --- | --- | --- | --- | --- |
| `GET /` | inline lambda | — | — | returns `AppOptions.Name`; excluded from logs (`appsettings.json:34`) |
| `GET /vehicles/{vehicleId}` | `GetVehicle` | `GetVehicleHandler` (`Mongo/Queries/Handlers/GetVehicleHandler.cs:18-22`) | reads `vehicles` | `AsDto`; §3.19 quirks apply |
| `GET /vehicles` | `SearchVehicles` | `SearchVehiclesHandler` (`:18-34`) | reads `vehicles` | §3.18 — usually returns nothing when filtered |
| `POST /vehicles` | `AddVehicle` | `AddVehicleHandler` | writes `vehicles`, `outbox` | `201` + `Location` via `afterDispatch`; §3.4 floor applies |
| `PUT /vehicles/{vehicleId}` | `UpdateVehicle` | `UpdateVehicleHandler` | writes `vehicles`, `outbox` | `ChangeVariants` **overwrites** (§3.7) |
| `DELETE /vehicles/{vehicleId}` | `DeleteVehicle` | `DeleteVehicleHandler` | writes `vehicles`, `outbox` | §3.13 binding ambiguity |
| `/metrics` | AppMetrics | — | — | Prometheus scrape target |
| `/ping` | `[convey]` | — | — | Consul health check |
| Swagger UI/JSON | `AddWebApiSwaggerDocs()` / `UseSwaggerDocs()` | — | — | `appsettings.json:155-163` |
| Public contracts endpoint | `UsePublicContracts<ContractAttribute>()` | — | — | path is `[convey]`-internal (§3.34) |

**Every one of these is unauthenticated at the service** (§3.41).

### 6.2 Messages

| Direction | Message | Routing key | Exchange | Internals |
| --- | --- | --- | --- | --- |
| In | `AddVehicle` | `add_vehicle` | `vehicles` | `SubscribeCommand<AddVehicle>()` (`…Infrastructure/Extensions.cs:86`) |
| In | `UpdateVehicle` | `update_vehicle` | `vehicles` | `:87` |
| In | `DeleteVehicle` | `delete_vehicle` | `vehicles` | `:88` |
| Out | `VehicleAdded` | `vehicle_added` | `vehicles` | `AddVehicleHandler.cs:33` — **no consumer** |
| Out | `VehicleUpdated` | `vehicle_updated` | `vehicles` | `UpdateVehicleHandler.cs:33` — **no consumer** |
| Out | `VehicleDeleted` | `vehicle_deleted` | `vehicles` | `DeleteVehicleHandler.cs:31` → `availability-service` |
| Out | `AddVehicleRejected` | `add_vehicle_rejected` | `vehicles` | `ExceptionToMessageMapper.cs:17,23,29` |
| Out | `UpdateVehicleRejected` | `update_vehicle_rejected` | `vehicles` | `:18,24,30,35` |
| Out | `DeleteVehicleRejected` | `delete_vehicle_rejected` | `vehicles` | `:36` — **only** for `VehicleNotFoundException` |

Routing keys are derived from class names by `conventionsCasing: snakeCase` (§3.31), not declared
anywhere; the values above follow that transformation.

### 6.3 Outbound calls

**None.** `httpClient.services` is `{}` in all three populated profiles, and no client class exists
(§1.2). `AddHttpClient()` and `AddFabio()` are registered for nothing.

### 6.4 Inbound callers

| Caller | Surface used | Evidence |
| --- | --- | --- |
| `api-gateway` (sync) | all 5 vehicle routes | `ntrada.yml:415-456` |
| `api-gateway` (async) | 2 GETs downstream; 3 writes via RabbitMQ | `ntrada-async.yml:483-534` |
| `orders-service` | `GET /vehicles/{id}` | `…Orders/…/Clients/VehiclesServiceClient.cs:20-21` |
| `availability-service` | `vehicle_deleted` only | `…Availability.Infrastructure/Extensions.cs:112` |
| Prometheus | `/metrics` | `hianshul100_Pacco/compose/prometheus/prometheus.yml:50-52` |
| Consul | `/ping` | `appsettings.json:13-16` |

---

## 7. Change/extension guide

### 7.1 Adding a field to `Vehicle`

Nine edits, in this order. Skipping any of the middle four fails **silently**.

1. `…Core/Entities/Vehicle.cs` — property with `protected set`; assign in the 7-arg constructor; add a
   `ChangeX` validating mutator if updatable (§3.6).
2. `…Core/Exceptions/` — a `DomainException` subclass if the field is validated (§3.9).
3. `…Application/Commands/AddVehicle.cs` — property **and** constructor parameter **and** assignment.
4. `…Application/Commands/UpdateVehicle.cs` — only if updatable (§3.12).
5. `…Application/Commands/Handlers/` — pass it to `new Vehicle(…)` / call the mutator.
6. `…Infrastructure/Mongo/Documents/VehicleDocument.cs` — settable property.
7. `…Infrastructure/Mongo/Documents/Extensions.cs` — **`AsEntity`, `AsDocument` and `AsDto`, all
   three** (§3.21).
8. `…Application/DTO/VehicleDto.cs` — settable property.
9. `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` — an arm for the new exception, for
   **each** command that can throw it (§3.28).

Then: existing documents read the new field as `default` (§5.4). If it is guarded, backfill first.

### 7.2 Adding a variant

One edit — append `NewName = 1 << 5` to `…Core/Entities/Variants.cs`. **Never renumber** (§3.22).
Then decide whether the search filter should be fixed first (§7.3), because until it is, the new
variant is unsearchable like the others.

### 7.3 Fixing the variant search (recommended, highest value)

Change `…Infrastructure/Mongo/Queries/Handlers/SearchVehiclesHandler.cs:20-31` to the single-predicate
form in §3.18. No data migration, no contract change, no consumer coordination. Verify that the Mongo
LINQ provider translates the bitwise `&`; if it does not, use `Builders<VehicleDocument>.Filter.BitsAllSet`
instead. Add the §3.46 test 1 and a search test in the same change.

### 7.4 Adding a command

1. Command class in `…Application/Commands/` — `[Contract]`, get-only properties, **constructor
   parameters named to match the properties** (§3.13).
2. Handler in `…Application/Commands/Handlers/` — repository call, then `PublishAsync` (§3.14).
3. Event and rejected-event classes, both `[Contract]`.
4. `SubscribeCommand<T>()` in `…Infrastructure/Extensions.cs` if it should be reachable over AMQP.
5. Route in `…Api/Program.cs` if it should be reachable over HTTP.
6. **`ExceptionToMessageMapper` arms for every exception it can throw** — the step that gets missed.
7. `MessageToLogTemplateMapper` entry (§3.35).
8. All four `ntrada*.yml` files if the gateway should expose it.

### 7.5 Adding authorization (do this before reading `IAppContext`)

The context machinery is complete and the input is untrusted (§3.38, §3.40). The order matters:

1. **First**, make the caller trustworthy: add `AddCertificateAuthentication()` and a
   `security.certificate` section with an ACL, modelled on
   `hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Api/appsettings.json:163-182`
   (§3.41). Confirm the Vault PKI role (`appsettings.json:177-181`) issues certificates the ACL
   accepts.
2. **Then** inject `IAppContext` into the handler and branch on `Identity.IsAdmin` or a claim.
3. Note that the **AMQP path has no equivalent** — `SubscribeCommand` accepts anything published to
   the exchange (§3.32). Either restrict publish rights at the broker or accept that authorization
   applies to the HTTP surface only, and say so explicitly.

Doing step 2 without step 1 creates a privilege-escalation path through an unauthenticated header.

### 7.6 Restoring outbox atomicity

Run MongoDB as a replica set (single-node is sufficient for transactions) and set
`outbox.disableTransactions: false` in `appsettings.json:107`. No code change. Test in the Docker
profile, which currently inherits `true` (§3.44). Do not change the flag without changing the Mongo
topology — the write will fail at runtime.

### 7.7 Adding a consumer for `VehicleAdded` / `VehicleUpdated`

In the consuming service: declare a class with the same property names and `[Message("vehicles")]`,
add `SubscribeEvent<T>()`, and write the handler. **Check the exchange name against this service's
`rabbitMq.exchange.name`** (`appsettings.json:132`) — a mismatch produces a binding that never fires,
which is precisely the platform's existing `parcel_deleted` defect
(`component-internals/index.md` §4).

### 7.8 Changing the read DTO

`orders-service` reads only `Id` and `PricePerService` (§3.47). Adding fields is safe; renaming those
two is a **silent** break (a missing property deserialises to `0`, and a price of `0` flows into
`pricing-service`'s discount calculation unvalidated —
`component-internals/pricing-service.md` §3.8). Fixing the `Variants` leading space (§3.19) affects
gateway clients only.

### 7.9 The maintenance contract

**Any change to this component's internals must update this document in the same change.** Concretely:

| If you change… | Update here |
| --- | --- |
| `…Core/Entities/Vehicle.cs` | §3.1, §3.4–§3.8, §7.1 |
| `…Core/Entities/Variants.cs` | §3.3, §3.22, §5.2, §7.2 |
| any command or its constructor | §3.11–§3.13, §6.2, §7.4 |
| a command handler | §3.14, §4.1, §4.3 |
| `SearchVehiclesHandler` | §3.17, §3.18, §4.2, §7.3 |
| `Mongo/Documents/Extensions.cs` | §3.19, §3.21, §5.2 |
| `VehicleDocument` | §3.20, §5.1, §5.4 |
| `ExceptionToMessageMapper` | §3.28, §4.5, §6.2 |
| `ExceptionToResponseMapper` | §3.26, §4.5 |
| `…Infrastructure/Extensions.cs` | §3.31–§3.32, §3.42, §3.45, §4.4 |
| `…Api/Program.cs` | §3.33, §4.4, §6.1 |
| any `appsettings*.json` | §3.30, §3.37, §3.43–§3.45, §5.3 |
| adding a test project | §3.46, and this row |
| the gateway's vehicle routes | §1.3, §1.4, §3.47, §6.4 |

Also update `component-internals/index.md` §1 if the concept count, ABQ count or open-question count
changes.

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

| # | Assumption | Basis | If wrong |
| --- | --- | --- | --- |
| **A-1** | `Convey 0.4.*` behaves as its call sites imply — `AddMongoRepository` binds the named collection, `SubscribeCommand` binds queue-to-routing-key by convention, `UseDispatcherEndpoints` binds route-to-type | NuGet-only; no source in this workspace | most of §3.29–§3.34 would need re-deriving against the package |
| **A-2** | The MongoDB driver serialises an unattributed enum as its underlying integer | driver default; no attribute present (`VehicleDocument.cs:16`) | §3.22 and the §3.18 predicate analysis change |
| **A-3** | `base ref feature/12998/aidlc` at `af43bcf` is the intended state, including `disableTransactions: true` | the flag was set by that commit, titled *Updated outbox - disable local TX* | §3.30's reading of intent changes; the mechanics do not |
| **A-4** | Ntrada forwards the request body unchanged on PUT and the query string unchanged on GET | `ntrada.yml:13` `passQueryString`, no `bind` on the sync PUT | §4.2 step 1 and §7.8 change |
| **A-5** | The four repositories inspected for consumers (`APIGateway`, `Orders`, `Availability`, `Pacco`) are the complete set of consumers | full workspace enumeration | an unknown consumer may depend on fields §7.8 declares safe to change |
| **A-6** | `RemoveVariants` has no caller | grep across all 48 source files | §3.7's "dead code" claim is wrong |
| **A-7** | `IAppContext` is injected nowhere | grep across all 48 source files | §3.40 is wrong and B-1 becomes live rather than latent |

### 8.2 Blockers

| # | Blocker | Impact | What would resolve it |
| --- | --- | --- | --- |
| **B-1** | Every HTTP route and every AMQP command is unauthenticated at the service (§3.41), while `customers-service` protects a lower-value surface with a certificate ACL. The `Correlation-Context` header is attacker-controlled and would yield `IsAdmin` if read (§3.38) | anything with network access to port 5009 or publish rights to exchange `vehicles` can create, update or delete any vehicle | a decision: either the service is only ever reachable through the gateway (document and enforce it at the network layer), or add certificate authentication (§7.5) |
| **B-2** | `vault.token: "secret"` (`appsettings.json:168`) and `logger.seq.apiKey: "secret"` (`:64`) are unchanged framework defaults, present in the **base** profile — the one production runs (§3.44) | if these are the real deployed values, Vault is protected by a known token. **No real secret is committed** — that is the point of recording it | confirm the deployment pipeline overrides them (nothing in this workspace does) |
| **B-3** | `hianshul100_Pacco/prod-services.yml:56-58` sets no `ASPNETCORE_ENVIRONMENT`, so production runs `appsettings.json` verbatim, including `consul.address: docker.for.win.localhost` (§3.42) | production instances register an address peers cannot resolve; the service's own health check still passes | confirm a `prod` profile is injected elsewhere in the pipeline, or add `appsettings.prod.json` and set the variable |
| **B-4** | No test project exists, while `scripts/test.sh` runs `dotnet test` and exits 0 (§3.46) | every finding in §3 ships undetected; CI is green and vacuous | add `tests/Pacco.Services.Vehicles.Tests` to the `.sln` (§3.46 lists the first four tests) |
| **B-5** | The variant floor plus the exact-match filter make variant search structurally incapable of matching (§3.4, §3.18), and capacity-only search returns nothing | the catalogue's primary query is broken in production and reports success | apply §7.3 |

### 8.3 Open questions

| # | Question | Why it matters | Where to look |
| --- | --- | --- | --- |
| **Q-1** | Does Convey's binder match `DeleteVehicle`'s constructor parameter `id` to the incoming key `vehicleId` (§3.13)? | if not, every delete silently targets `Guid.Empty` | Convey 0.4 `WebApi` and `MessageBrokers.CQRS` sources |
| **Q-2** | Why does `customers-service` have a certificate ACL and `vehicles-service` none (§3.41)? | determines whether B-1 is an oversight or a deliberate trust boundary | project history outside this workspace |
| **Q-3** | Are `VehicleAdded` and `VehicleUpdated` forward-looking contracts or dead weight (§3.27)? | determines whether to keep publishing them | product backlog; no code answers it |
| **Q-4** | Is the Vault Mongo lease template's `localhost:27017` (`appsettings.json:189`) correct for the production topology (§3.43)? | if Mongo is not co-located, dynamic credentials resolve to an unreachable host | deployment topology, not in this workspace |
| **Q-5** | Is Convey's `AddMongo()` idempotent (§3.45)? | determines whether the duplicate at `:66`/`:70` is harmless or wasteful, and which line is safe to delete | Convey `Persistence.MongoDB` source |
| **Q-6** | What status codes do `Put<T>` and `Delete<T>` return without an `afterDispatch` (§3.33)? | the write contract for two of the three commands is undocumented | Convey `WebApi.CQRS` source |
| **Q-7** | Does Ntrada's `resourceId: {generate: true}` inject the generated id into the forwarded body (§3.11)? | determines whether `AddVehicle`'s id-minting branch is ever reached in production | Ntrada source |
| **Q-8** | Does `IMongoRepository.UpdateAsync` upsert when the id is absent (§3.23)? | determines whether a delete/update race can resurrect a vehicle | Convey `Persistence.MongoDB` source |
| **Q-9** | Does Convey ack, nack or dead-letter a message when `ExceptionToMessageMapper` returns `null` (§3.28)? | determines whether silently-dropped commands are also infinitely redelivered | Convey `MessageBrokers.RabbitMQ` source |

### 8.4 Cross-references, related patterns, baseline reconciliation

**Cross-references within this artifact set:**

| Document | Relationship |
| --- | --- |
| `component-internals/pricing-service.md` | the batch-6 sibling; the two are structural opposites — pricing is a single project with no persistence and no messaging, vehicles is four projects with both. Pricing forward-references this document at its §1, §3.13, §3.14, §3.17, §3.18, §3.19, §3.23, §3.27, §3.28 and §3.29 |
| `component-internals/api-gateway.md` | owns the gateway half of every route in §6.1 and the sync/async mode split in §1.3 |
| `component-internals/availability-service.md` | the only consumer of any event this service publishes (§3.27) |
| `component-internals/orders-service.md` | the only synchronous consumer of this service's read surface (§3.47) |
| `component-internals/customers-service.md` | §3.25 there is the certificate-ACL implementation this service lacks (§3.41, B-1); its §3.2 is the aggregate-with-buffered-events design this service does not use (§3.14) |
| `component-internals/index.md` | §3 fixes the conventions this document follows; §4 records the platform-wide `parcel_deleted` mis-binding referenced in §7.7 |

**Related patterns** (all verified against `../patterns/`):
[[inward-dependency-service-skeleton]] (§3.2) ·
[[database-per-service-with-document-mapping]] (§3.10, §3.20, §3.21, §5.1) ·
[[dispatcher-bound-cqrs-endpoints]] (§3.33) ·
[[dual-mode-edge-write]] (§1.3, §3.32) ·
[[transactional-outbox-handler-decorator]] (§3.30) ·
[[service-owned-topic-exchange-messaging]] (§3.31) ·
[[rejected-event-failure-contract]] (§3.28) ·
[[declarative-message-manifest-subscription]] (§3.27, §7.7) ·
[[framework-supplied-platform-conventions]] (§3.31) ·
[[registry-mediated-discovery-and-routing]] (§3.42) ·
[[vault-issued-dynamic-credentials-and-service-pki]] (§3.43) ·
[[composable-per-concern-environment-stacks]] (§3.44) ·
[[structured-logging-with-property-redaction]] (§3.37) ·
[[correlation-and-span-propagation]] (§3.39) ·
[[transport-agnostic-caller-context]] (§3.38) ·
[[prefix-partitioned-shared-cache]] (§3.36) ·
[[saga-process-manager]] (§3.39) ·
[[narrow-synchronous-point-read]] (§3.27, §3.47) ·
[[independent-per-repository-release]] (§3.47) ·
[[layered-service-test-suite]] (§3.46, by absence) ·
[[declarative-configuration-driven-api-gateway]] (§1.4, §6.4) ·
[[edge-enforced-authentication-with-identity-binding]] (§3.41, by absence at the service) ·
[[aggregate-buffered-domain-events]] (§3.14, **not** instantiated).

**Baseline reconciliation.** `baselines/service-summaries.md` §2.3 and
`repo-summary/Pacco.Services.Vehicles.md` remain accurate as surface catalogues; this document
complements them. Two additions rather than corrections: the baselines record that the service
publishes three events, but not that **two of them have no consumer** (§3.27); and they record the
five routes, but not that **the search route cannot match a filtered query** (§3.18). No baseline
statement was found to be wrong.

**Explicitly unverifiable in this workspace** (each marked at its point of use):
Convey's binder name-matching policy (Q-1, §3.13) · Convey's `AddMongo` idempotency (Q-5, §3.45) ·
default status codes for `Put`/`Delete` (Q-6, §3.33) · Convey's ack/nack behaviour on a null rejection
mapping (Q-9, §3.28) · `IMongoRepository.UpdateAsync` upsert semantics (Q-8, §3.23) · Convey's
`PagedQueryBase` clamping (§3.16) · whether the Mongo LINQ provider translates a bitwise `&` (§3.18,
§7.3) · Ntrada's `resourceId: generate` body injection (Q-7, §3.11) · the public-contracts endpoint
path and payload (§3.34) · whether `AddRedis()` connects eagerly (§3.36) · whether Convey's
certificate authentication supports per-route permissions (§3.41) · the production Mongo topology
behind the Vault lease template (Q-4, §3.43).

---

*Component-internals model for `vehicles-service`, batch 6 of 7. Registered in
`component-internals/index.md` §1. Any change to this component's internals must update this document
in the same change (§7.9).*
