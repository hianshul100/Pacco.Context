# Component internals — `deliveries-service`

| | |
| --- | --- |
| **Component** | `deliveries-service` |
| **Source repository** | `hianshul100_Pacco.Services.Deliveries` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository) |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 2 of 7 |
| **Status** | New artifact — no prior `component-internals/deliveries-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` and `repo-summary/Pacco.Services.Deliveries.md` remain valid and are **complemented**, not replaced: those catalogue the surface, this document models the internals. |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full
> (`src/Pacco.Services.Deliveries.{Core,Application,Infrastructure,Api}`). It contains **no test
> project of any kind** — `find . -name '*.csproj'` returns exactly those four production projects
> and there is no `tests/` directory, yet `scripts/test.sh` still runs `dotnet test` (§3.46).
> `Convey 0.4.*` — which supplies the CQRS dispatchers, the Mongo repository, the RabbitMQ client,
> the outbox and the WebApi endpoint mapping — is a NuGet reference with **no source in this
> workspace**. Mechanisms owned by Convey are marked `[convey]` and, where their exact semantics
> change a conclusion, flagged `Unverifiable — Missing Source Evidence`. The upstream half of every
> inbound contract is modelled in `component-internals/api-gateway.md`.

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

`deliveries-service` owns **one aggregate — `Delivery`** — and answers one question for the rest of
the platform: *where has this order got to, and did it arrive?* It is the platform's **physical
fulfilment ledger**: a delivery is opened against an order, accumulates an append-only set of
free-text registrations ("scanned at depot", "out for delivery"), and terminates in exactly one of
two absorbing states, `Completed` or `Failed`.

Three properties distinguish it from its siblings:

1. **It is a pure command sink with a keyhole read.** Four commands mutate; exactly **one** query
   reads, by id (`…Api/Program.cs:34`). There is **no list endpoint and no query-by-order**
   (§3.22) — the read model exists only to satisfy a single-delivery lookup.
2. **Its lifecycle is one-way and self-terminating.** `Complete()` and `Fail()` both refuse to run
   from `Completed` or `Failed` (`…Core/Entities/Delivery.cs:61-92`), so a delivery has exactly one
   terminal transition and the aggregate becomes permanently immutable afterwards (§3.6).
3. **It is the only Pacco service whose write path is driven entirely by a saga.** No in-platform
   component calls it over HTTP; every production write arrives as an AMQP command from
   `Pacco.Services.Operations`' saga, and every event it emits is consumed only by `orders-service`
   (§3.35, §6.5).

| Responsibility | Where it lives |
| --- | --- |
| Model a delivery as id + order id + status + notes + registration set | `src/Pacco.Services.Deliveries.Core/Entities/Delivery.cs:10-93` |
| Enforce that registrations may be added only while `InProgress` | `…Core/Entities/Delivery.cs:46-59` (`AddRegistration`) |
| Enforce the one-way status machine and emit a state-change event per transition | `…Core/Entities/Delivery.cs:38-44,61-92` |
| De-duplicate registrations by value | `…Core/ValueObjects/DeliveryRegistration.cs:16-37`; `…Core/Entities/Delivery.cs:53-56` |
| Derive the delivery's "last update" timestamp from its registrations | `…Core/Entities/Delivery.cs:22-25`; `…Application/DTO/DeliveryDto.cs:15-18` |
| Buffer domain events and expose them for publication | `…Core/Entities/AggregateRoot.cs:15-32` |
| Accept 4 commands over HTTP **and** the same 4 over AMQP | `…Api/Program.cs:35-39`; `…Infrastructure/Extensions.cs:87-90` |
| Answer exactly 1 query from a Mongo read model | `…Infrastructure/Mongo/Queries/Handlers/GetDeliveryHandler.cs:18-23` |
| Translate domain events into integration events and publish them | `…Infrastructure/Services/{EventMapper,MessageBroker}.cs` |
| Translate exceptions into HTTP responses **and** into rejected events | `…Infrastructure/Exceptions/ExceptionTo{Response,Message}Mapper.cs` |
| Persist deliveries in MongoDB (`deliveries` collection, `deliveries-service` DB) | `…Infrastructure/Mongo/Repositories/DeliveriesMongoRepository.cs`; `…Api/appsettings.json:95-99` |
| Publish reliably via a Mongo-backed transactional outbox | `…Infrastructure/Decorators/Outbox*Decorator.cs`; `…Api/appsettings.json:100-108` |
| Register with Consul, emit Jaeger spans, expose Prometheus metrics | `…Infrastructure/Extensions.cs:63-71`; `…Api/appsettings.json:7-22,68-94` |
| Fetch its own secrets and dynamic Mongo credentials from Vault | `…Api/Program.cs:41` (`UseVault`); `…Api/appsettings.json:164-193` |

### 1.2 What this component explicitly is **not**

- **Not an authenticator or an authorizer.** There is **no JWT validation, no
  `UseAuthentication`, no `UseAuthorization`, no `[Authorize]` and — unlike `customers-service` —
  no certificate authentication either.** `UseInfrastructure` (`…Infrastructure/Extensions.cs:78-93`)
  never calls `UseCertificateAuthentication()`, and `appsettings.json` has **no `security` section
  at all**. The only security registration is `AddSecurity()` (`:75`), which `[convey]` supplies
  hashing/encryption primitives that **no code in this repository consumes** (§3.37). The `jwt`
  block (`…Api/appsettings.json:77-85`) is inert configuration no code reads. Every HTTP request
  that reaches the process is executed.
- **Not the owner of the delivery decision.** It never decides *that* a delivery should start —
  the saga in `Pacco.Services.Operations` does, on `OrderApproved`. This service only records.
- **Not a courier/vehicle model.** Registrations are `(string Description, DateTime DateTime)` and
  nothing more (`…Core/ValueObjects/DeliveryRegistration.cs:7-8`). There is no carrier, no
  location, no ETA, no proof-of-delivery.
- **Not a projection host for other aggregates.** It subscribes to **zero external events**
  (`…Infrastructure/Extensions.cs:87-90` contains four `SubscribeCommand` calls and no
  `SubscribeEvent`). It holds no replica of orders, customers, parcels or vehicles — only a bare
  `OrderId` (§3.11).
- **Not a scheduler.** Nothing expires, times out or sweeps an in-progress delivery. A delivery
  that is never completed or failed stays `InProgress` forever (§3.6 failure modes).
- **Not a deleter.** `DeleteAsync` exists on both the interface and the Mongo repository
  (`…Core/Repositories/IDeliveriesRepository.cs:13`;
  `…Infrastructure/Mongo/Repositories/DeliveriesMongoRepository.cs:35-36`) and has **no caller
  anywhere in the repository** (§3.25).

### 1.3 The boundary is dual-transport, and the two halves are not equivalent

Every one of the four commands is reachable two ways, and the two paths differ in **who validates
the payload, what a failure looks like, and whether the caller learns about it**:

| | HTTP (`…Api/Program.cs:35-39`) | AMQP (`…Infrastructure/Extensions.cs:87-90`) |
| --- | --- | --- |
| Entry | `UseDispatcherEndpoints` → `ICommandDispatcher` `[convey]` | `SubscribeCommand<T>()` → `ICommandDispatcher` `[convey]` |
| Payload shape | JSON body + route-template binding | JSON message body only |
| On exception | `ExceptionToResponseMapper` → **HTTP 400** with `{code, reason}` | `ExceptionToMessageMapper` → **rejected event** on the `deliveries` exchange |
| Unmapped exception | still 400, code `error` (`…Exceptions/ExceptionToResponseMapper.cs:22-23`) | mapper returns `null` → **silent** `[convey]` |
| Caller in practice | gateway only, no in-platform caller (§6.5) | `Pacco.Services.Operations` saga |

The asymmetry is load-bearing: **adding a registration to a `Completed` delivery over AMQP fails
silently**, because `CannotAddDeliveryRegistrationException` is mapped only when the message is a
`StartDelivery` (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:15-19`). See §3.16.

### 1.4 Position in the platform

- **Upstream (writes):** `Pacco.Services.Operations` saga publishes `StartDelivery`,
  `CompleteDelivery`, `FailDelivery`; the gateway can publish all four
  (`api-gateway.md`, `ntrada.yml:189-204`).
- **Downstream (events):** `orders-service` is the **only** consumer — `DeliveryStarted` →
  `SetDelivering()`, `DeliveryCompleted` → `Complete()`, `DeliveryFailed` →
  `Cancel(@event.Reason)`. `RegistrationAddedToDelivery` has **no consumer in the workspace**
  (§3.14).
- **Nobody reads it synchronously.** No service in the workspace declares a `deliveries-service`
  entry under `httpClient.services`; this service's own `httpClient.services` is `{}`
  (`…Api/appsettings.json:26`).

---

## 2. Core concepts (exhaustive)

Every concept this component owns or materially operates, in the order §3 models them. "Owner"
distinguishes what this repository defines and can change unilaterally (**service**) from what
`Convey` or infrastructure imposes (**framework** / **platform**).

| # | Concept | Owner | Primary source |
| --- | --- | --- | --- |
| 1 | `Delivery` aggregate root | service | `Core/Entities/Delivery.cs` |
| 2 | `AggregateId` identity value object | service | `Core/Entities/AggregateId.cs` |
| 3 | `AggregateRoot` & buffered domain events | service | `Core/Entities/AggregateRoot.cs` |
| 4 | `HasChanged` change flag | service | `Core/Entities/AggregateRoot.cs:21` |
| 5 | `Version` / `IncrementVersion` (dead concurrency hook) | service | `Core/Entities/AggregateRoot.cs:11,20,30-31` |
| 6 | `DeliveryStatus` one-way state machine | service | `Core/Entities/DeliveryStatus.cs`; `Delivery.cs:61-92` |
| 7 | `DeliveryRegistration` value struct | service | `Core/ValueObjects/DeliveryRegistration.cs` |
| 8 | The registration **set** and value de-duplication | service | `Core/Entities/Delivery.cs:16-27,53-56` |
| 9 | `LastUpdate` derived timestamp | service | `Core/Entities/Delivery.cs:22-25`; `DTO/DeliveryDto.cs:15-18` |
| 10 | `Notes` — the failure-reason channel | service | `Core/Entities/Delivery.cs:14,90` |
| 11 | `OrderId` — the cross-service correlation key | service | `Core/Entities/Delivery.cs:12`; `Repositories/DeliveriesMongoRepository.cs:23-27` |
| 12 | Domain events (`DeliveryStateChanged`, `DeliveryRegistrationAdded`) | service | `Core/Events/*.cs` |
| 13 | `EventMapper` — domain→integration translation | service | `Infrastructure/Services/EventMapper.cs` |
| 14 | Integration events & the `[Contract]` marker | service | `Application/Events/*.cs` |
| 15 | Rejected events | service | `Application/Events/Rejected/*.cs` |
| 16 | `ExceptionToMessageMapper` — AMQP failure contract | service | `Infrastructure/Exceptions/ExceptionToMessageMapper.cs` |
| 17 | `ExceptionToResponseMapper` & error codes | service | `Infrastructure/Exceptions/ExceptionToResponseMapper.cs` |
| 18 | Exception hierarchy (`DomainException` / `AppException`) | service | `Core/Exceptions/DomainException.cs`; `Application/Exceptions/AppException.cs` |
| 19 | The four commands | service | `Application/Commands/*.cs` |
| 20 | `StartDelivery` client-side id materialisation | service | `Application/Commands/StartDelivery.cs:16` |
| 21 | Command-handler shape (get → mutate → persist → map → publish) | service | `Application/Commands/Handlers/*.cs` |
| 22 | `GetDelivery` query & `DeliveryDto` | service | `Application/Queries/GetDelivery.cs`; `DTO/DeliveryDto.cs` |
| 23 | Read-model projection `AsDto` | service | `Infrastructure/Mongo/Documents/Extensions.cs:28-40` |
| 24 | `IDeliveriesRepository` domain port | service | `Core/Repositories/IDeliveriesRepository.cs` |
| 25 | `DeliveriesMongoRepository` (incl. the uncalled `DeleteAsync`) | service | `Infrastructure/Mongo/Repositories/DeliveriesMongoRepository.cs` |
| 26 | Hand-written document mapping `AsDocument` / `AsEntity` | service | `Infrastructure/Mongo/Documents/Extensions.cs:10-26` |
| 27 | Mongo storage configuration & collection binding | service+framework | `Infrastructure/Extensions.cs:68,73`; `Api/appsettings.json:95-99` |
| 28 | Transactional outbox & inbox idempotence | service+framework | `Infrastructure/Decorators/*.cs`; `Api/appsettings.json:100-108` |
| 29 | `MessageBroker` publication path | service | `Infrastructure/Services/MessageBroker.cs` |
| 30 | Correlation context, `IAppContext`, `IIdentityContext` (registered, unconsumed) | service | `Infrastructure/Contexts/*.cs` |
| 31 | `Saga` header forwarding | service | `Infrastructure/Extensions.cs:100-114` |
| 32 | Span-context propagation & Jaeger | service+framework | `Infrastructure/Extensions.cs:65,71,116-129` |
| 33 | Handler log templates & property redaction | service+platform | `Infrastructure/Logging/MessageToLogTemplateMapper.cs`; `Api/appsettings.json:32-67` |
| 34 | RabbitMQ topology: exchange, queue template, subscriptions | service+platform | `Infrastructure/Extensions.cs:86-90`; `Api/appsettings.json:109-150` |
| 35 | The dual-transport write surface | service | `Api/Program.cs:32-39`; `Infrastructure/Extensions.cs:87-90` |
| 36 | HTTP dispatcher endpoints & route binding | service+framework | `Api/Program.cs:32-39` |
| 37 | Security posture (`AddSecurity`, no auth of any kind) | service | `Infrastructure/Extensions.cs:75`; absence of a `security` config section |
| 38 | Public-contracts endpoint | framework | `Infrastructure/Extensions.cs:84` |
| 39 | Consul registration & Fabio routing | service+platform | `Infrastructure/Extensions.cs:63-64`; `Api/appsettings.json:7-22` |
| 40 | Metrics | service+platform | `Infrastructure/Extensions.cs:70`; `Api/appsettings.json:86-94` |
| 41 | Vault: KV settings, service PKI, dynamic Mongo lease | service+platform | `Api/Program.cs:41`; `Api/appsettings.json:164-193` |
| 42 | Environment configuration layering | service | `Api/appsettings.{local,docker,development}.json` |
| 43 | Registered-but-unused dependencies (Redis, `IDateTimeProvider`) | service | `Infrastructure/Extensions.cs:52,69` |
| 44 | Swagger documentation | service+framework | `Infrastructure/Extensions.cs:74,81`; `Api/appsettings.json:155-163` |
| 45 | Container image & release path | service | `Dockerfile`; `scripts/dockerize.sh` |
| 46 | Test posture (there are no tests) | service | absence of any test `.csproj`; `scripts/test.sh` |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement**, **Extension procedure**, **Failure modes**.

### 3.1 `Delivery` aggregate root

**Definition.** The single aggregate of this service: a record that order `OrderId` is being
physically delivered, its current `DeliveryStatus`, a free-text `Notes` field used only for a
failure reason, and a set of `DeliveryRegistration` value objects
(`…Core/Entities/Delivery.cs:10-93`).

**Representation & storage.** In memory: a class deriving `AggregateRoot`, all state with
`protected set` so mutation is only possible through its four methods
(`Delivery.cs:12-14`). Registrations are held in a private `ISet<DeliveryRegistration>` backed by a
`HashSet` (`:27`), exposed as `IEnumerable<DeliveryRegistration>` whose setter re-wraps the incoming
sequence into a new `HashSet` (`:16-20`). At rest: one `DeliveryDocument` per delivery in the
`deliveries` collection (§3.26, §3.27).

**Lifecycle.**
- **Created** only by `Delivery.Create(id, orderId, status)` (`:38-44`), called from exactly one
  place — `StartDeliveryHandler.cs:34`. The public constructor (`:29-36`) emits **no event** and
  exists for rehydration from Mongo (`Mongo/Documents/Extensions.cs:24-26`).
- **Read** by `DeliveriesMongoRepository.GetAsync` / `GetForOrderAsync` (`:17-27`), both of which
  rehydrate via `AsEntity()`.
- **Mutated** by exactly three methods: `AddRegistration`, `Complete`, `Fail`.
- **Versioned** — no. `Version` is never incremented (§3.5); documents are overwritten wholesale.
- **Deleted** — never in practice (§3.25).

**Invariants & enforcement.**

| Invariant | Enforced at | Failure style |
| --- | --- | --- |
| Id is not `Guid.Empty` | `AggregateId(Guid)` ctor (`AggregateId.cs:15-23`) | **loud** — `InvalidAggregateIdException` |
| Registrations only while `InProgress` | `Delivery.cs:48-51` | **loud** — `CannotAddDeliveryRegistrationException` |
| No transition out of a terminal state | `Delivery.cs:63-71,79-87` | **loud** — `CannotChangeDeliveryStateException` |
| A duplicate registration is not stored | `Delivery.cs:53-56` | **silent** — method returns, no event, no error |
| One delivery per order | *not enforced* | **silent** — see §3.11 |

**Extension procedure.** To add a field: add the property with `protected set` to `Delivery.cs`;
add the matching property to `DeliveryDocument.cs`; extend **both** `AsDocument` and `AsEntity`
(`Mongo/Documents/Extensions.cs:10-26`) — *and* `AsDto` if it should be readable (§3.23). Skipping
`AsEntity` **silently** drops the value on every read; skipping `AsDocument` **silently** drops it
on every write. Nothing in the build or at runtime detects either omission.

**Failure modes.**
- A field added to the document but not to `AsEntity` reads back as `default` with no warning.
- Because `AsDocument` rewrites the whole document, a concurrent update loses one writer's
  registrations entirely (§3.5, §5.4).

### 3.2 `AggregateId` identity value object

**Definition.** A wrapper over `Guid` that refuses `Guid.Empty`
(`…Core/Entities/AggregateId.cs:6-50`).

**Representation & storage.** A class with a single `Guid Value` (`:8`), value equality (`:25-36`),
and **implicit two-way conversion** to/from `Guid` (`:43-47`). It is never persisted as itself:
`AsDocument` writes `delivery.Id` into a `Guid` field via the implicit operator
(`Mongo/Documents/Extensions.cs:13`).

**Lifecycle.** Created implicitly at the boundary — `Delivery.Create(command.DeliveryId, …)`
(`StartDeliveryHandler.cs:34`) passes a `Guid` and the implicit operator (`AggregateId.cs:46-47`)
constructs the `AggregateId`, running the empty check. Never mutated; `Value` is get-only.

**Invariants & enforcement.** `Guid.Empty` → `InvalidAggregateIdException` (`:17-20`), **loud** but
**unmapped**: `ExceptionToMessageMapper` has no arm for it (`:12-40`), so over AMQP it produces
**no rejected event**; over HTTP it falls to the `DomainException` arm and returns
`400 {"code":"invalid_aggregate_id"}` (`ExceptionToResponseMapper.cs:18-19`). In practice the
check is unreachable from `StartDelivery`, because that command already replaces an empty id
(§3.20).

**Extension procedure.** To add a rule (e.g. reject a non-v4 GUID), edit the `AggregateId(Guid)`
constructor. There is no other construction path — the parameterless constructor (`:10-13`)
generates a fresh GUID and is never called in this repository.

**Failure modes.** The implicit `Guid → AggregateId` conversion means the validation runs at
surprising places; a `Guid.Empty` arriving through a future code path throws inside domain code
rather than at the edge, and over AMQP that failure is invisible.

### 3.3 `AggregateRoot` & buffered domain events

**Definition.** The base class that lets a mutation record what happened without knowing how it
will be published (`…Core/Entities/AggregateRoot.cs:15-32`).

**Representation & storage.** A `List<IDomainEvent> _events` (`:17`) exposed as `Events` (`:18`),
plus `Id` (`:19`), `Version` (`:20`) and `HasChanged` (`:21`). `protected void AddEvent` (`:23-26`)
appends; `ClearEvents()` (`:28`) empties. `IDomainEvent` is a **bare marker interface** —
`Core/Events/IDomainEvent.cs` declares no members. Events are **never persisted**; the buffer lives
only for the duration of one handler call.

**Lifecycle.** Populated inside `Delivery.Create` (`:41`), `AddRegistration` (`:58`), `Complete`
(`:74`), `Fail` (`:91`). Drained by the handler: `_eventMapper.MapAll(delivery.Events)` then
`_messageBroker.PublishAsync(...)` (e.g. `CompleteDeliveryHandler.cs:34-35`).

**Invariants & enforcement.** **None.** In particular `ClearEvents()` is **never called by any
handler** — the buffer is discarded with the object. This is safe only because each handler loads a
fresh aggregate; reusing an aggregate across two operations would re-publish the first operation's
events. **Silent** if violated.

**Extension procedure.** To add a domain event, declare a class implementing `IDomainEvent` in
`Core/Events/`, `AddEvent` it from the mutating method, and add a `case` to
`EventMapper.Map` (§3.13). **Omitting the `EventMapper` case is silent**: `Map` falls through to
`return null` (`EventMapper.cs:35`) and `MessageBroker` skips nulls (`MessageBroker.cs:69-72`), so
the event vanishes with no log line at `information` level.

**Failure modes.**
- The publish is **after** the repository write in every handler, so a crash between them loses the
  events unless the outbox is enabled (§3.28) — and the outbox is `enabled: false` in `local`
  (`appsettings.local.json:35-37`).
- `MapAll` returns a lazily-evaluated `Select` (`EventMapper.cs:13-14`); handlers force it with
  `.ToArray()` before publishing, which is what makes the `null` filtering in `MessageBroker`
  reachable rather than deferred.

### 3.4 `HasChanged` change flag

**Definition.** `public bool HasChanged => _events.Any();`
(`…Core/Entities/AggregateRoot.cs:21`) — "did the last operation actually change anything?"

**Representation & storage.** Purely derived from the in-memory event buffer; not stored.

**Lifecycle.** Read in exactly **one** place: `AddDeliveryRegistrationHandler.cs:34`. The other
three handlers ignore it.

**Invariants & enforcement.** It is the **only** guard preventing a duplicate registration from
triggering a pointless Mongo write and a spurious `RegistrationAddedToDelivery` event. Because
`AddRegistration` returns silently on a duplicate without adding an event (`Delivery.cs:53-56`),
`HasChanged` is `false` and the handler skips persist-and-publish entirely (`:34-39`). This
correctness is **local to that one handler**, not enforced by the base class.

**Extension procedure.** Any new handler whose domain method can no-op **must** replicate the
`if (delivery.HasChanged)` guard. Nothing enforces this; forgetting it produces an idempotent-looking
write that still re-publishes nothing (because the buffer is empty) but does perform a redundant
Mongo update.

**Failure modes.** Because `StartDeliveryHandler` does **not** check `HasChanged`, it always writes
and publishes — which is correct there (`Create` always emits) but means the pattern is
inconsistent and easy to mis-copy.

### 3.5 `Version` / `IncrementVersion` — the dead concurrency hook

**Definition.** `IAggregateRoot` declares `int Version { get; }` and `void IncrementVersion()`
(`…Core/Entities/AggregateRoot.cs:11-12`); `AggregateRoot` implements the setter as
`protected set` (`:20`) and provides an **explicit interface implementation**
`void IAggregateRoot.IncrementVersion() => Version++;` (`:30-31`).

**Representation & storage.** `Version` is a plain `int` on the entity. It is **not** a member of
`DeliveryDocument` (`Mongo/Documents/DeliveryDocument.cs:10-14`), so it is never written to or read
from Mongo — it is always `0` on every rehydrated aggregate.

**Lifecycle.** Never incremented. `grep -rn "IncrementVersion" src` finds only the declaration and
the implementation; the explicit implementation means it is not even callable on a `Delivery`
reference without first casting to `IAggregateRoot`, and no such cast exists.

**Invariants & enforcement.** **There is no optimistic concurrency control in this service.**
`UpdateAsync` calls `_repository.UpdateAsync(delivery.AsDocument())` `[convey]`
(`DeliveriesMongoRepository.cs:32-33`) with no version predicate, so it is a last-writer-wins
replace of the entire document. Concurrent writers fail **silently**.

**Extension procedure.** To make this real: (1) add `Version` to `DeliveryDocument` and to both
`AsDocument`/`AsEntity`; (2) call `((IAggregateRoot)delivery).IncrementVersion()` in each handler;
(3) replace `UpdateAsync` with a filtered update that matches on `Id` **and** the loaded `Version`
— Convey's `IMongoRepository` surface would need a conditional-update primitive, which this
workspace cannot confirm exists. **`Unverifiable — Missing Source Evidence` (Convey 0.4 has no
source here).**

**Failure modes.** Two `AddDeliveryRegistration` commands processed concurrently each load the
delivery, each add their registration to their own copy, and each write the full document — one
registration is lost with no error, no log and no event. Realistically mitigated only by
`outbox.type: "sequential"` (`appsettings.json:102`) and single-consumer queues, neither of which
this repository can prove serialises handler execution. **`Unverifiable — Missing Source
Evidence`.**

### 3.6 `DeliveryStatus` — the one-way state machine

**Definition.** `enum DeliveryStatus { InProgress, Completed, Failed }`
(`…Core/Entities/DeliveryStatus.cs:3-8`). Note there is **no `Unknown` member** and `InProgress`
is the zero value — the opposite choice from `customers-service`, whose `State.Unknown = 0` gives it
a detectable "unset" (`[[customers-service]]` §3.4).

**Representation & storage.** Persisted as the **enum**, i.e. an `int` ordinal, in
`DeliveryDocument.Status` (`Mongo/Documents/DeliveryDocument.cs:12`). Serialised into
`DeliveryDto.Status` as the enum too (`DTO/DeliveryDto.cs:12`; `Extensions.cs:33`) — again unlike
`customers-service`, which lowercases its state to a string for the API.

**Lifecycle.**
- Set to `InProgress` at creation — hard-coded at the call site, not in the aggregate
  (`StartDeliveryHandler.cs:34`).
- → `Completed` only via `Delivery.Complete()` (`Delivery.cs:73`).
- → `Failed` only via `Delivery.Fail(reason)` (`Delivery.cs:89`).
- Both transitions `AddEvent(new DeliveryStateChanged(this))` (`:74`, `:91`) **after** assigning the
  new status, which is why `EventMapper` can switch on the *current* status to pick the integration
  event (§3.13).
- There is **no transition back to `InProgress`** and no reopen path.

**Invariants & enforcement.** `Complete()` and `Fail()` each check both terminal states explicitly
(`:63-71`, `:79-87`) and throw `CannotChangeDeliveryStateException(Id, current, next)` — **loud**,
mapped to HTTP 400 and (for all three state commands) to a rejected event
(`ExceptionToMessageMapper.cs:20-26`). Note the asymmetry: `Complete()` from `Completed` throws,
so **completing an already-completed delivery is an error, not an idempotent no-op**.

**Extension procedure.** To add a state (e.g. `Returned`): (1) append the member to
`DeliveryStatus` — **append only**, because existing documents store the ordinal (§5.3); (2) add a
mutating method with its own guards to `Delivery.cs`; (3) add a `case` to the inner switch of
`EventMapper.Map` (`EventMapper.cs:21-29`) — **omitting it is silent**, the outer `break` falls
through to `return null` and the state change is never announced; (4) add an integration event and
a rejected event; (5) add the exception arms to `ExceptionToMessageMapper`; (6) register the
command in `Program.cs` and `Infrastructure/Extensions.cs`; (7) add the message to
`Pacco.Services.Operations`' `messages.json` if the saga must see it.

**Failure modes.**
- **Ordinal drift.** Inserting a member anywhere but the end silently re-labels every stored
  delivery (§5.3).
- **No absorbing-state sweep.** A delivery whose saga dies stays `InProgress` forever; nothing
  detects it. There is no scheduler, no TTL and no `LastUpdate`-based query (§3.9).
- **`DeliveryStatus.InProgress == 0`** means a `DeliveryDocument` written without a `Status` (e.g.
  by a future migration or a manual insert) deserialises as a live in-progress delivery, not as an
  obviously-invalid one.

### 3.7 `DeliveryRegistration` value struct

**Definition.** An immutable `(string Description, DateTime DateTime)` pair recording one event in
the physical journey (`…Core/ValueObjects/DeliveryRegistration.cs:5-38`).

**Representation & storage.** A **`struct`**, not a class — the only value type in the domain model
— implementing `IEquatable<DeliveryRegistration>` with get-only properties (`:7-8`) set once in the
constructor (`:10-14`). Persisted as an element of `DeliveryDocument.Registrations`, each a
`DeliveryRegistrationDocument { string Description; DateTime DateTime; }`
(`Mongo/Documents/DeliveryRegistrationDocument.cs:5-9`) — a nested array inside the delivery
document, not a separate collection.

**Lifecycle.** Constructed at exactly two call sites, both from command fields:
`StartDeliveryHandler.cs:35` and `AddDeliveryRegistrationHandler.cs:33`. Never mutated (there is no
setter). Never individually deleted — it can only disappear when the whole document is overwritten.

**Invariants & enforcement.** **None on the value itself.** `Description` may be `null`, empty or
arbitrarily long; `DateTime` may be `default`, in the future, or in the past. There is no
validator, no `[Required]`, no length check anywhere — `grep -rn "Required\|MaxLength\|Validator" src`
returns nothing. Both fields are attacker-controlled through the gateway's `POST
/deliveries/{deliveryId}/registrations` route. **Silent** acceptance of anything.

Equality is by value on **both** fields (`:16-29`), with a hand-written
`GetHashCode` — `((Description?.GetHashCode() ?? 0) * 397) ^ DateTime.GetHashCode()` inside an
`unchecked` block (`:31-37`). `string.Equals(Description, other.Description)` is the **ordinal**
overload, so `"Depot"` and `"depot"` are different registrations.

**Extension procedure.** To add a field (e.g. `Location`): add it to the struct and constructor,
add it to `DeliveryRegistrationDocument`, extend the two projections in
`Mongo/Documents/Extensions.cs:17-21` (`AsDocument`) and `:26` (`AsEntity`) and the DTO projection
at `:35-39`, add it to the `AddDeliveryRegistration` / `StartDelivery` command constructors, and —
critically — **decide whether it participates in equality**, because that changes de-duplication
semantics (§3.8). Forgetting `Equals`/`GetHashCode` means two registrations differing only in the
new field collapse into one, **silently**.

**Failure modes.**
- Because it is a struct with a parameterless default, a `DeliveryRegistrationDocument` with a null
  `Description` rehydrates to a registration whose `Description` is `null`, which then flows into
  `RegistrationAddedToDelivery.Message` and out to the broker unchecked
  (`EventMapper.cs:32`).
- `DateTime` is stored with whatever `Kind` the JSON deserialiser produced; nothing normalises to
  UTC, and `IDateTimeProvider` — which exists precisely for this — is **never injected** (§3.43).
  Timestamps are therefore **caller-supplied and untrusted**.

### 3.8 The registration set and value de-duplication

**Definition.** A delivery holds a **set**, not a list, of registrations, so adding the same
`(Description, DateTime)` twice is a no-op (`…Core/Entities/Delivery.cs:16-27,53-56`).

**Representation & storage.** Private `ISet<DeliveryRegistration> _registrations = new
HashSet<DeliveryRegistration>()` (`:27`). The public property's setter re-wraps any assigned
sequence into a new `HashSet` (`:19`), which is what makes rehydration de-duplicating too: a
document that somehow contains two identical registration sub-documents collapses to one on read
(`Extensions.cs:24-26` → `Delivery` ctor `:35`). At rest it is a **JSON/BSON array**, which does
*not* enforce uniqueness — the set semantics exist only in memory.

**Lifecycle.** Seeded at construction from the ctor argument, defaulting to empty when `null`
(`:35`). Grown only by `AddRegistration` (`:53-58`). Never shrunk.

**Invariants & enforcement.**

| Invariant | Enforced | Style |
| --- | --- | --- |
| No two registrations with the same `(Description, DateTime)` | `HashSet.Add` returns `false` → early `return` (`:53-56`) | **silent** — caller gets HTTP 200/`Accepted` and no event |
| Registrations only while `InProgress` | `:48-51` | **loud** — throws before the set is touched |
| Set is non-empty for a started delivery | *by construction only* — `StartDeliveryHandler.cs:35` always adds one | not enforced |

**Extension procedure.** To allow duplicates, change `ISet`/`HashSet` to `IList`/`List` at `:16-27`
and drop the `Add` result check at `:53-56`. Note this also changes `AsEntity` behaviour (dupes
would survive a round-trip) and would make `HasChanged` (§3.4) always true for
`AddDeliveryRegistration`. To change what "duplicate" means, edit `Equals`/`GetHashCode` on the
struct (§3.7) — the aggregate itself has no comparer.

**Failure modes.**
- **The silent-duplicate path is externally observable as success.** A retrying saga or a
  double-clicking client that re-sends the identical `AddDeliveryRegistration` receives a
  non-error response but produces **no** `RegistrationAddedToDelivery` event; a downstream
  consumer waiting for that event waits forever. Note the retry is only "identical" if the caller
  reuses the same `DateTime` — since the timestamp is caller-supplied (§3.7), a retry that stamps a
  new `DateTime.UtcNow` **does** create a second registration.
- The outbox's inbox de-duplication (§3.28) operates on message id, not on registration value, so
  the two mechanisms overlap without covering the same case.

### 3.9 `LastUpdate` derived timestamp

**Definition.** "When did something last happen to this delivery?" — computed, never stored
(`…Core/Entities/Delivery.cs:22-25`; duplicated on the DTO at `…Application/DTO/DeliveryDto.cs:15-18`).

**Representation & storage.** `Registrations.OrderByDescending(r => r.DateTime).Select(r =>
r.DateTime).FirstOrDefault()` typed as `DateTime?`. It is **not** a member of `DeliveryDocument`,
so it is recomputed on every read and cannot be queried or indexed in Mongo.

**Lifecycle.** Read-only, evaluated on access. The DTO copy is evaluated at JSON-serialisation
time from the DTO's own `Registrations`.

**Invariants & enforcement.** None. Two subtleties worth knowing before relying on it:

1. **It reflects registrations only.** `Complete()` and `Fail()` add **no** registration
   (`Delivery.cs:61-92`), so a delivery that is completed hours after its last scan reports a
   `LastUpdate` from the scan, not from the completion.
2. **`FirstOrDefault()` on a `DateTime` sequence returns `default(DateTime)`, not `null`.** The
   property is declared `DateTime?`, but the LINQ chain projects to `DateTime` before
   `FirstOrDefault`, so an empty registration set yields `DateTime.MinValue` (`0001-01-01`) boxed
   into the nullable — **never `null`**. Any consumer testing `lastUpdate == null` to mean "no
   activity" is wrong; it must test `DateTime.MinValue`. This is **silent**.

**Extension procedure.** To make it authoritative, add a real `UpdatedAt` to `Delivery` and
`DeliveryDocument`, assign it in each mutating method, and map it in all three projections. To make
it queryable (e.g. "find stale deliveries"), it must become a stored document field — the derived
property cannot appear in a Mongo filter expression.

**Failure modes.** As above: `0001-01-01` masquerading as a real timestamp, and completion/failure
being invisible to it.

### 3.10 `Notes` — the failure-reason channel

**Definition.** A single nullable `string` on the aggregate (`…Core/Entities/Delivery.cs:14`) that
carries **only** the reason a delivery failed.

**Representation & storage.** `public string Notes { get; protected set; }`; persisted as
`DeliveryDocument.Notes` (`DeliveryDocument.cs:13`); surfaced on `DeliveryDto.Notes`
(`DTO/DeliveryDto.cs:13`; `Extensions.cs:34`).

**Lifecycle.** Assigned in exactly one place — `Fail(reason)` at `Delivery.cs:90` — from the
caller-supplied `FailDelivery.Reason` (`Commands/FailDelivery.cs:10`). Never set on creation,
never set by `Complete()`, never cleared. It therefore doubles as a **de facto marker that the
delivery failed at least once**: a delivery that fails, and is then re-started (which creates a
*new* document, §3.11), leaves the old document's `Notes` populated forever.

**Invariants & enforcement.** None — `reason` is not validated, not length-limited and may be
`null`. It is **verbatim attacker-controlled text** from the gateway route
`POST /deliveries/{deliveryId}/fail`, and it is **re-published to the broker** inside
`DeliveryFailed(…, e.Delivery.Notes)` (`EventMapper.cs:28`), from where `orders-service` writes it
into `Order.Cancel(@event.Reason)`. Untrusted text therefore crosses two service boundaries and is
persisted twice with no sanitisation at any hop.

**Extension procedure.** To repurpose `Notes` as a general annotation field you must decide what
happens to the failure-reason contract, because `EventMapper.cs:28` reads it unconditionally when
status is `Failed`; a `Notes` written by any other path would be published as a failure reason.
Safer: add a separate `FailureReason` property and leave `Notes` alone.

**Failure modes.**
- Null `Notes` produces `DeliveryFailed.Reason = null`, which `orders-service` stores as the
  cancellation reason.
- No length bound means a large payload propagates into an event, into the outbox collection, and
  into the log line `"Failed the delivery with id: {DeliveryId}, reason: {Reason}"`
  (`Logging/MessageToLogTemplateMapper.cs:28`) — note `Reason` is **not** in the redaction list
  (§3.33).

### 3.11 `OrderId` — the cross-service correlation key

**Definition.** The id of the order this delivery fulfils (`…Core/Entities/Delivery.cs:12`). It is
the service's only link to the rest of the platform and the key of its only non-id lookup.

**Representation & storage.** `Guid OrderId { get; protected set; }`; persisted as
`DeliveryDocument.OrderId` (`DeliveryDocument.cs:11`). **No index is declared anywhere** —
`grep -rn "Index\|EnsureIndex" src` returns nothing, and Convey's `AddMongoRepository` is invoked
with only a collection name (`Infrastructure/Extensions.cs:73`), so whether it creates any index
beyond `_id` is **`Unverifiable — Missing Source Evidence`**.

**Lifecycle.** Written once at creation from `StartDelivery.OrderId` (`StartDeliveryHandler.cs:34`);
never changed. Read by `GetForOrderAsync(d => d.OrderId == id)`
(`DeliveriesMongoRepository.cs:23-27`), used by exactly one caller — `StartDeliveryHandler.cs:28`,
the duplicate-start check.

**Invariants & enforcement.** **There is no uniqueness constraint on `OrderId`, and the code
deliberately creates duplicates.** `StartDeliveryHandler` (`:28-36`) reads:

```csharp
var delivery = await _repository.GetForOrderAsync(command.OrderId);
if (delivery is {} && delivery.Status != DeliveryStatus.Failed)
{
    throw new DeliveryAlreadyStartedException(command.OrderId);
}
delivery = Delivery.Create(command.DeliveryId, command.OrderId, DeliveryStatus.InProgress);
…
await _repository.AddAsync(delivery);
```

A *failed* delivery does **not** block a restart — but the restart calls `AddAsync`, not
`UpdateAsync`, so it inserts a **second document with the same `OrderId`** and a different `Id`.
`GetForOrderAsync` then resolves to *whichever document Mongo returns first* for that filter, which
is unspecified without a sort. After one failure and one restart, the duplicate-start guard becomes
**non-deterministic**: if the query returns the old failed document, a third `StartDelivery` is
allowed and inserts a third document; if it returns the new in-progress one, the third is rejected.
This is **silent** — no error, no log, no event.

**Extension procedure.** To make "one live delivery per order" real: either (a) make
`StartDeliveryHandler` call `UpdateAsync` on the failed aggregate after resetting it (which
requires a `Restart()` method on `Delivery`, since `Status` has no public setter and there is no
transition back to `InProgress` — §3.6), or (b) declare a unique partial index on `OrderId` in
Mongo and handle the duplicate-key error, which has no home in the current
`ExceptionToResponseMapper`/`ExceptionToMessageMapper` and would surface as the generic
`code: "error"` 400 (`ExceptionToResponseMapper.cs:22-23`). Option (b) also needs
`GetForOrderAsync` to order deterministically.

**Failure modes.**
- Duplicate documents per order, as above.
- `orders-service` keys off `DeliveryId`/`OrderId` in the integration events; two live deliveries
  for one order produce interleaved `DeliveryStarted`/`DeliveryFailed` for the same order, and
  `orders-service` has no way to tell which delivery a given event refers to beyond `DeliveryId`,
  which it does not store. **`Unverifiable — Missing Source Evidence`** as to the downstream effect.
- An unindexed `OrderId` means every `StartDelivery` performs a collection scan.

### 3.12 Domain events

**Definition.** Two in-process notifications emitted by the aggregate, both defined in
`…Core/Events/`:

| Event | Payload | Emitted by |
| --- | --- | --- |
| `DeliveryStateChanged` | `Delivery Delivery` (`DeliveryStateChanged.cs:7`) | `Create` (`Delivery.cs:41`), `Complete` (`:74`), `Fail` (`:91`) |
| `DeliveryRegistrationAdded` | `Delivery Delivery`, `DeliveryRegistration Registration` (`DeliveryRegistrationAdded.cs:8-9`) | `AddRegistration` (`Delivery.cs:58`) |

**Representation & storage.** Plain classes implementing the empty marker `IDomainEvent`. **Both
carry a reference to the live aggregate, not a snapshot** — so `EventMapper` reads
`e.Delivery.Status` *after* the mutation completed, which is precisely how one event class covers
three different integration events (§3.13). Never serialised, never stored.

**Lifecycle.** Constructed inside the mutating method, buffered by `AddEvent`, mapped and published
by the handler, then discarded with the aggregate.

**Invariants & enforcement.** Note the deliberate omission: **`DeliveryStateChanged` carries no
previous state**, unlike `customers-service`'s `CustomerStateChanged(customer, previousState)`
(`[[customers-service]]` §3.14). Consumers cannot distinguish "started" from "restarted" or learn
what the delivery was before. Adding a `previousStatus` field is a one-line change here but a
contract change downstream.

**Extension procedure.** See §3.3. The critical, silent-failure step is the `EventMapper` case.

**Failure modes.** Because the event holds a live reference, mutating the aggregate again before
publishing would retroactively change what the earlier buffered event maps to. No current handler
performs two mutations — but `StartDeliveryHandler` comes close: it calls `Create` **then**
`AddRegistration` (`:34-35`), buffering two events, and both are mapped after both mutations. That
happens to be correct only because the second mutation does not change `Status`.

### 3.13 `EventMapper` — domain→integration translation

**Definition.** The single place where an internal domain event becomes a public integration event
(`…Infrastructure/Services/EventMapper.cs:11-37`), behind the `IEventMapper` port declared in
`…Application/Services/IEventMapper.cs`.

**Representation & storage.** A stateless singleton (`Infrastructure/Extensions.cs:49`). `MapAll`
is `events.Select(Map)` — **lazy** (`:13-14`). `Map` is a nested switch:

```csharp
case DeliveryStateChanged e:
    switch (e.Delivery.Status)
    {
        case DeliveryStatus.InProgress: return new DeliveryStarted(e.Delivery.Id, e.Delivery.OrderId);
        case DeliveryStatus.Completed:  return new DeliveryCompleted(e.Delivery.Id, e.Delivery.OrderId);
        case DeliveryStatus.Failed:     return new DeliveryFailed(e.Delivery.Id, e.Delivery.OrderId, e.Delivery.Notes);
    }
    break;
case DeliveryRegistrationAdded e:
    return new RegistrationAddedToDelivery(e.Delivery.Id, e.Delivery.OrderId, e.Registration.Description);
…
return null;
```

(`EventMapper.cs:20-35`.) `e.Delivery.Id` is an `AggregateId`, converted to `Guid` by the implicit
operator (§3.2).

**Lifecycle.** Invoked once per handler, immediately after the repository write, e.g.
`FailDeliveryHandler.cs:34`.

**Invariants & enforcement.** The mapping is **status-driven, not transition-driven** — the mapper
inspects the delivery's *current* status rather than what changed. Consequences:

- `Create` and any future `InProgress` transition both yield `DeliveryStarted`. A restart after a
  failure (§3.11) therefore emits a **second** `DeliveryStarted` for the same `OrderId`.
- The `default` case is `return null`, reached whenever the inner switch has no arm for the status
  or the outer switch has no arm for the event type — **completely silent** (§3.3, §3.29).

**Extension procedure.** Adding a domain event **or** a status requires an arm here. There is no
registration table, no reflection scan, no attribute — a missing arm compiles, runs and drops the
event. The only way to detect it is to look for the absent message on the broker.

**Failure modes.**
- Silent event loss, as above. This is the single highest-risk extension hazard in the service.
- `DeliveryFailed` carries `e.Delivery.Notes`, which may be `null` (§3.10).
- `RegistrationAddedToDelivery` carries `e.Registration.Description` under the field name
  `Message` (`RegistrationAddedToDelivery.cs:11`) — a rename at the boundary that makes the event
  and the command look unrelated.

### 3.14 Integration events & the `[Contract]` marker

**Definition.** The four public events this service publishes, all in
`…Application/Events/`, all `[Contract]`-marked and implementing Convey's `IEvent`:

| Event | Fields | Broker routing key | Consumer in workspace |
| --- | --- | --- | --- |
| `DeliveryStarted` (`:6-17`) | `DeliveryId`, `OrderId` | `delivery_started` | `orders-service` → `SetDelivering()` |
| `DeliveryCompleted` (`:6-17`) | `DeliveryId`, `OrderId` | `delivery_completed` | `orders-service` → `Complete()` |
| `DeliveryFailed` (`:6-19`) | `DeliveryId`, `OrderId`, `Reason` | `delivery_failed` | `orders-service` → `Cancel(Reason)` |
| `RegistrationAddedToDelivery` (`:6-19`) | `DeliveryId`, `OrderId`, `Message` | `registration_added_to_delivery` | **none** |

**Representation & storage.** Immutable classes with get-only properties and a single constructor.
Routing keys are not written anywhere in this repository — there is **no `[Message("…")]`
attribute** on any event class here. The names above are Convey's snake-case convention applied to
the type name, driven by `rabbitMq.conventionsCasing: "snakeCase"`
(`Api/appsettings.json:113`) and confirmed by `Pacco.Services.Operations`' `messages.json`, which
lists exactly `delivery_completed`, `delivery_failed`, `delivery_started`,
`registration_added_to_delivery`. The exchange is the service's own, `deliveries`
(`appsettings.json:131-137`). The precise convention algorithm is `[convey]` and
**`Unverifiable — Missing Source Evidence`**.

**Lifecycle.** Constructed by `EventMapper`, handed to `MessageBroker.PublishAsync`, then either
written to the outbox collection or published directly (§3.28, §3.29). Never stored in the
service's own domain data.

**Invariants & enforcement.** `[Contract]` is what makes a type visible on the public-contracts
endpoint (`UsePublicContracts<ContractAttribute>()`, `Infrastructure/Extensions.cs:84`, §3.38).
Omitting it does **not** stop publication — it only hides the type from the contracts document.
**Silent** either way.

**Extension procedure.** Add the class under `Application/Events/`, mark `[Contract]`, implement
`IEvent`, add the `EventMapper` arm, and add the snake-case name to
`Pacco.Services.Operations/.../messages.json` if the saga must react. Adding a **field** to an
existing event is backward-compatible for consumers that ignore unknown properties; **removing or
renaming** one is not, and nothing in this repository detects it — there are no contract tests here
(§3.46), though the platform pattern `[[consumer-driven-contract-test-pair]]` describes where they
would live.

**Failure modes.**
- `RegistrationAddedToDelivery` is published on every accepted registration and consumed by
  nobody — pure broker traffic. Note the corollary: the *only* way an external observer learns
  about a registration is this unconsumed event or a direct `GET /deliveries/{id}`.
- `DeliveryStarted` can be emitted twice for one order (§3.13); `orders-service`'s
  `SetDelivering()` would then run twice.

### 3.15 Rejected events

**Definition.** Four `IRejectedEvent` types under `…Application/Events/Rejected/` that report an
AMQP command failure back to the bus, implementing the platform's
`[[rejected-event-failure-contract]]`.

| Type | Fields |
| --- | --- |
| `StartDeliveryRejected` (`:6-21`) | `DeliveryId`, `OrderId`, `Reason`, `Code` |
| `CompleteDeliveryRejected` | `DeliveryId`, `Reason`, `Code` |
| `FailDeliveryRejected` | `DeliveryId`, `Reason`, `Code` |
| `AddDeliveryRegistrationRejected` (`:6-19`) | `DeliveryId`, `Reason`, `Code` |

**Representation & storage.** Immutable `[Contract]` classes. `Reason` is the exception's
`Message`; `Code` is the exception's `Code` (§3.18). Published by Convey's RabbitMQ subscriber when
`IExceptionToMessageMapper.Map` returns non-null `[convey]`; **there is no code in this repository
that publishes them** — `grep -rn "Rejected" src` finds only the class definitions and the mapper.

**Lifecycle.** Created inside `ExceptionToMessageMapper.Map` (§3.16), handed back to Convey, and
published on the `deliveries` exchange. Never created on the HTTP path.

**Invariants & enforcement.** The manifest coupling is imperfect and verifiable:
`Pacco.Services.Operations/.../messages.json` lists `start_delivery_rejected`,
`complete_delivery_rejected` and `fail_delivery_rejected` but **not**
`add_delivery_registration_rejected`. So the one rejected event the mapper can produce for
registrations (§3.16, on `DeliveryNotFoundException`) is published to an exchange where no declared
subscriber exists. **Silent** on both sides.

**Extension procedure.** A new rejected event needs: the class (`[Contract]`, `IRejectedEvent`), a
mapper arm keyed on **both** exception type and command type, and — if a saga must react — an entry
in the operations `messages.json`. Missing the mapper arm is the silent case; missing the manifest
entry means the message is published and ignored.

**Failure modes.** The `Code` string is the only machine-readable discriminator, and it is
duplicated by hand in each exception class (§3.18); a typo produces a rejected event no consumer
matches.

### 3.16 `ExceptionToMessageMapper` — the AMQP failure contract

**Definition.** The switch that decides which rejected event, if any, an AMQP command failure
produces (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:12-40`). Registered via
`AddExceptionToMessageMapper<ExceptionToMessageMapper>()` (`Infrastructure/Extensions.cs:67`).

**Representation & storage.** A nested C# 8 switch expression: outer arm on exception type, inner
arm on the **message** (command) type, with `_ => null` at both levels. Full coverage matrix:

| Exception ↓ / Command → | `StartDelivery` | `AddDeliveryRegistration` | `CompleteDelivery` | `FailDelivery` |
| --- | --- | --- | --- | --- |
| `CannotAddDeliveryRegistrationException` (`:15-19`) | `StartDeliveryRejected` | **`null` — silent** | **`null`** | **`null`** |
| `CannotChangeDeliveryStateException` (`:20-26`) | `StartDeliveryRejected` | `null` | `CompleteDeliveryRejected` | `FailDeliveryRejected` |
| `DeliveryAlreadyStartedException` (`:27-31`) | `StartDeliveryRejected` | `null` | `null` | `null` |
| `DeliveryNotFoundException` (`:32-38`) | `null` | `AddDeliveryRegistrationRejected` | `CompleteDeliveryRejected` | `FailDeliveryRejected` |
| `InvalidAggregateIdException` | **unmapped — silent** | — | — | — |
| anything else (`:39`) | **unmapped — silent** | — | — | — |

**Lifecycle.** Called by Convey's subscriber when a handler throws `[convey]`. Not called on the
HTTP path.

**Invariants & enforcement.** The load-bearing gap is the first row.
`CannotAddDeliveryRegistrationException` is thrown by `Delivery.AddRegistration` when the delivery
is not `InProgress` (`Delivery.cs:48-51`). The realistic path to that exception is an
`AddDeliveryRegistration` command against a completed or failed delivery — and that combination
maps to `null`. **Adding a registration to a terminated delivery over AMQP therefore fails with no
rejected event, no HTTP status (there is none) and no `information`-level log line.** The saga
simply never hears back.

By contrast, the `StartDelivery` arm for that exception is nearly unreachable:
`StartDeliveryHandler` always calls `AddRegistration` on a freshly-created `InProgress` delivery
(`:34-35`), so the guard cannot trip.

**Extension procedure.** Every new (exception, command) pair needs an explicit arm. There is no
default fallback that produces a generic rejection, and no test asserts coverage. When adding a
command, walk the exception list in `Core/Exceptions/` and `Application/Exceptions/` and add an arm
per exception the new handler can throw.

**Failure modes.** All silent, all invisible from inside the service: an unmapped pair produces a
message that is (per Convey's subscriber contract) acknowledged or dead-lettered with no domain
signal. Whether Convey acks, nacks or requeues after a mapper returns `null` is
**`Unverifiable — Missing Source Evidence`** — it determines whether the failing command is
retried forever or dropped.

### 3.17 `ExceptionToResponseMapper` & error codes

**Definition.** The HTTP half of the failure contract
(`…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:11-46`), registered by
`AddErrorHandler<ExceptionToResponseMapper>()` (`Infrastructure/Extensions.cs:59`) and activated by
`UseErrorHandler()` (`:80`, first in the pipeline).

**Representation & storage.** Three arms, all producing the **same status code**:

```csharp
DomainException ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.BadRequest),
AppException    ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.BadRequest),
_               => new ExceptionResponse(new {code = "error", reason = "There was an error."}, HttpStatusCode.BadRequest)
```

(`:18-23`.) `GetCode` (`:26-45`) prefers the exception's own `Code`, falls back to
`Name.Underscore().Replace("_exception", "")` (`:39`), and memoises into a static
`ConcurrentDictionary<Type,string>` (`:13`).

**Lifecycle.** Invoked per failing HTTP request by Convey's error-handling middleware `[convey]`.

**Invariants & enforcement.**
- **Every error is 400.** A missing delivery returns `400 {"code":"delivery_not_found"}`, not 404.
  A completed-delivery conflict returns 400, not 409. Clients must parse `code`, not the status.
- **`reason` leaks the exception message verbatim**, including the interpolated delivery id and the
  caller-supplied failure reason. For this service the messages are benign, but the mechanism is
  unconditional — any future exception message is published to the caller as-is.
- The final arm converts an unexpected exception (`NullReferenceException`, a Mongo timeout) into
  `400 {"code":"error"}` — **a server fault reported as a client error**, and the original message
  is suppressed (good) while the status is misleading (bad).

**Extension procedure.** To differentiate status codes, add arms **before** the two base-type arms —
C# switch expressions match top-down, so a `DeliveryNotFoundException => …NotFound` arm must precede
`AppException`. Placing it after compiles fine and never matches: **silent**.

**Failure modes.** Static memoisation is keyed on `Type`, so a `Code` that varies per instance
(none currently does — all are initialised inline field values) would be cached from the first
instance seen.

### 3.18 Exception hierarchy

**Definition.** Two parallel abstract bases, one per layer:

- `Core/Exceptions/DomainException.cs:5-12` — `public virtual string Code`, `protected
  DomainException(string message)`. Subclasses: `CannotChangeDeliveryStateException`
  (`code: cannot_change_delivery_state`), `CannotAddDeliveryRegistrationException`
  (`cannot_add_delivery_registration`), `InvalidAggregateIdException` (`invalid_aggregate_id`).
- `Application/Exceptions/AppException.cs:5-12` — identical shape. Subclasses:
  `DeliveryNotFoundException` (`delivery_not_found`), `DeliveryAlreadyStartedException`
  (`delivery_already_started`).

**Representation & storage.** Each subclass sets `public override string Code { get; } = "…";` as
an inline field initialiser and formats its message in the constructor, e.g.
`$"Cannot change state for delivery with id: '{id}' from {currentStatus} to {nextStatus}'"`
(`CannotChangeDeliveryStateException.cs:11` — note the stray trailing apostrophe, present in the
published `reason` string).

**Lifecycle.** Thrown from the aggregate (domain) or the handler (application), caught by Convey
middleware/subscriber, translated by §3.16 or §3.17.

**Invariants & enforcement.** The split matters: `Core` cannot reference `Application`, so a domain
rule can never throw `DeliveryNotFoundException` — "not found" is inherently an application
concern. Both bases are treated identically by `ExceptionToResponseMapper`, so the distinction has
**no effect on the HTTP surface**; it is purely an architectural boundary marker
(`[[inward-dependency-service-skeleton]]`).

**Extension procedure.** New rule in the aggregate → subclass `DomainException` in `Core/Exceptions`;
new rule in a handler → subclass `AppException` in `Application/Exceptions`. Always set `Code`
explicitly: relying on the `GetCode` fallback (`ExceptionToResponseMapper.cs:39`) yields a code
derived from the class name, which then changes if the class is ever renamed — a **silent** contract
break.

**Failure modes.** `Code` is a free string with no registry and no uniqueness check; two exceptions
could share a code and no build step would notice.

### 3.19 The four commands

**Definition.** The complete write vocabulary, all in `…Application/Commands/`, all `[Contract]`,
all implementing Convey's `ICommand`, all immutable with get-only properties:

| Command | Fields | Source |
| --- | --- | --- |
| `StartDelivery` | `DeliveryId`, `OrderId`, `Description`, `DateTime` | `StartDelivery.cs:6-21` |
| `AddDeliveryRegistration` | `DeliveryId`, `Description`, `DateTime` | `AddDeliveryRegistration.cs:6-19` |
| `CompleteDelivery` | `DeliveryId` | `CompleteDelivery.cs:6-13` |
| `FailDelivery` | `DeliveryId`, `Reason` | `FailDelivery.cs:6-17` |

**Representation & storage.** Constructor-only initialisation with no parameterless constructor —
so deserialisation depends on the JSON serialiser matching constructor parameters to properties by
name (Convey/Newtonsoft behaviour, `[convey]`). Route-template values (`{deliveryId}`) are bound by
`UseDispatcherEndpoints` `[convey]`; how it reconciles a route value with an immutable constructor
parameter is **`Unverifiable — Missing Source Evidence`**.

**Lifecycle.** Constructed by deserialisation on either transport, dispatched to the matching
`ICommandHandler<T>` through the in-memory dispatcher (`Application/Extensions.cs:13`), wrapped by
the outbox decorator (§3.28), then discarded.

**Invariants & enforcement.** **There is no validation layer.** No FluentValidation, no data
annotations, no guard clauses in any command constructor except `StartDelivery`'s id substitution
(§3.20). `Description`, `Reason` and `DateTime` are accepted as-is. All validation that exists is
domain-level, inside `Delivery` (§3.1).

**Extension procedure.** To add a command: (1) class under `Application/Commands/` with
`[Contract]` and `ICommand`; (2) `internal sealed` handler under `Commands/Handlers/` —
`AddCommandHandlers()` (`Application/Extensions.cs:11`) discovers it by assembly scan `[convey]`,
so **no explicit registration is needed**; (3) an HTTP route in `Api/Program.cs` **and/or** (4) a
`SubscribeCommand<T>()` line in `Infrastructure/Extensions.cs:87-90` — these are two independent
registrations and **omitting either silently removes that transport**; (5) exception arms in
`ExceptionToMessageMapper` (§3.16); (6) a log template in `MessageToLogTemplateMapper` (§3.33);
(7) a route in the gateway's `ntrada.yml`; (8) the snake-case name in the operations `messages.json`.
Eight registration points, of which only (2) is automatic.

**Failure modes.** The asymmetry between (3) and (4) is the common bug: a command subscribed on
AMQP but not routed on HTTP is invisible to the gateway; routed but not subscribed, and the saga's
message is never consumed and accumulates in — or is never routed to — a queue.

### 3.20 `StartDelivery` client-side id materialisation

**Definition.** `StartDelivery` is the only command that **manufactures its own identity**:

```csharp
DeliveryId = deliveryId == Guid.Empty ? Guid.NewGuid() : deliveryId;
```

(`…Application/Commands/StartDelivery.cs:16`.)

**Representation & storage.** The resolved value becomes the aggregate's `Id` and the document's
`_id`. It is also read back **after dispatch** to build the `Location` header:
`afterDispatch: (cmd, ctx) => ctx.Response.Created($"deliveries/{cmd.DeliveryId}")`
(`Api/Program.cs:35-36`).

**Lifecycle.** Evaluated once, in the constructor, i.e. at deserialisation time — **before** any
handler runs and before any duplicate check.

**Invariants & enforcement.** This is what makes the HTTP `POST /deliveries` `201 Created`
meaningful: the id exists before the command is dispatched, so the response can name the resource
even though the dispatcher returns nothing. It also makes `AggregateId`'s empty-guid check
unreachable from this path (§3.2). Upstream, the gateway generates the id itself —
`resourceId: {property: deliveryId, generate: true}` in `ntrada.yml` — so in the normal edge flow
the substitution never fires.

**Extension procedure.** Do not copy this pattern to a command that mutates an existing aggregate:
substituting a fresh GUID for an empty one would turn "you forgot the id" into "not found" (or, on
`StartDelivery`, into "create a new one"). The other three commands correctly leave `DeliveryId`
alone, so an empty id there produces `DeliveryNotFoundException` — **loud**.

**Failure modes.** A client that sends `deliveryId: "00000000-0000-0000-0000-000000000000"`
believing it is addressing a specific delivery instead **creates a new one** at a random id, learns
about it only from the `Location` header (HTTP) or not at all (AMQP).

### 3.21 Command-handler shape

**Definition.** All four handlers (`…Application/Commands/Handlers/`) are `internal sealed`, take
exactly `(IDeliveriesRepository, IMessageBroker, IEventMapper)`, and follow one skeleton:

1. load the aggregate;
2. throw if absent;
3. mutate;
4. `UpdateAsync` (or `AddAsync`);
5. `_eventMapper.MapAll(delivery.Events)`;
6. `_messageBroker.PublishAsync(events.ToArray())`.

**Representation & storage.** Stateless; resolved per dispatch. Concrete variations:

| Handler | Load | Guard | Mutation | Persist | Publish guard |
| --- | --- | --- | --- | --- | --- |
| `StartDeliveryHandler` (`:26-39`) | `GetForOrderAsync` | `DeliveryAlreadyStartedException` unless the existing delivery is `Failed` | `Create` + `AddRegistration` | **`AddAsync`** | none |
| `AddDeliveryRegistrationHandler` (`:25-40`) | `GetAsync` | `DeliveryNotFoundException` | `AddRegistration` | `UpdateAsync` | **`if (delivery.HasChanged)`** |
| `CompleteDeliveryHandler` (`:24-36`) | `GetAsync` | `DeliveryNotFoundException` | `Complete()` | `UpdateAsync` | none |
| `FailDeliveryHandler` (`:24-36`) | `GetAsync` | `DeliveryNotFoundException` | `Fail(Reason)` | `UpdateAsync` | none |

**Lifecycle.** Discovered by `AddCommandHandlers()` (`Application/Extensions.cs:11`) and decorated
by `OutboxCommandHandlerDecorator<>` (`Infrastructure/Extensions.cs:55`).

**Invariants & enforcement.**
- **Persist-then-publish, never atomic.** The Mongo write and the event publish are separate
  awaits with no transaction; the outbox exists to close that gap (§3.28) and
  `disableTransactions: true` (`appsettings.json:107`) says even the outbox does not use Mongo
  transactions.
- **No handler injects `IAppContext`.** Not one of the four looks at who is calling (§3.30).
- **No handler calls `ClearEvents()`** (§3.3).
- **No handler is `async`-safe against reordering** — nothing serialises two commands for the same
  delivery (§3.5).

**Extension procedure.** Copy the skeleton exactly, including the `HasChanged` guard if the domain
method can no-op. Do **not** publish before persisting: the outbox decorator wraps the whole
`HandleAsync`, so a publish that precedes a failed write would still be committed to the outbox.

**Failure modes.** `StartDeliveryHandler`'s use of `AddAsync` rather than `UpdateAsync` on the
restart path is the source of the duplicate-document behaviour in §3.11.

### 3.22 `GetDelivery` query & `DeliveryDto`

**Definition.** The entire read surface: one query by id
(`…Application/Queries/GetDelivery.cs:7-10`), returning `DeliveryDto`
(`…Application/DTO/DeliveryDto.cs:8-21`).

**Representation & storage.** `GetDelivery` is a **mutable** class — `public Guid DeliveryId { get;
set; }` — unlike the commands, because route/query-string binding writes into it `[convey]`.
`DeliveryDto` carries `Id`, `OrderId`, `Status` (the enum), `Notes`, the derived `LastUpdate`
(§3.9) and `IEnumerable<DeliveryRegistrationDto>`; `DeliveryRegistrationDto` is
`{ string Description; DateTime DateTime; }` (`DTO/DeliveryRegistrationDto.cs:5-9`).

**Lifecycle.** Bound from `deliveries/{deliveryId}` (`Api/Program.cs:34`), dispatched through
`AddInMemoryQueryDispatcher()` (`Infrastructure/Extensions.cs:61`) to `GetDeliveryHandler`
(`Mongo/Queries/Handlers/GetDeliveryHandler.cs:18-23`), which reads the document directly through
`IMongoRepository<DeliveryDocument, Guid>` — **bypassing `IDeliveriesRepository` and the domain
entity entirely** — and returns `document?.AsDto()`.

**Invariants & enforcement.**
- **Read/write asymmetry is deliberate.** Writes go through the domain port; reads go straight to
  the document store. Changing `IDeliveriesRepository` therefore does not affect reads, and vice
  versa.
- **No authorization.** `GetDeliveryHandler` does not consult `IAppContext`; any caller reaching the
  process can read any delivery by id. The gateway route for `GET /deliveries/{deliveryId}` requires
  authentication but **no admin claim** (`ntrada.yml:189-204`), and there is no owner check
  anywhere — a delivery is not linked to a customer in this service's model.
- **A null result becomes `200` with an empty body**, not `404` — `HandleAsync` returns `null` and
  Convey's query endpoint writes what it is given `[convey]`; the exact status is
  **`Unverifiable — Missing Source Evidence`**.

**Extension procedure.** To add a query (e.g. `GetDeliveriesForOrder`): add the query class, add a
handler under `Infrastructure/Mongo/Queries/Handlers/` (auto-discovered by `AddQueryHandlers()`,
`Infrastructure/Extensions.cs:60`), and add a `.Get<TQuery, TResult>("route")` line in
`Api/Program.cs`. **Forgetting the route line is silent** — the handler exists and is registered but
unreachable.

**Failure modes.** There is **no list endpoint and no pagination**, so the only way to enumerate
deliveries is out-of-band Mongo access. Compare `customers-service`, which does expose an
unpaginated `FindAsync(_ => true)` list (`[[customers-service]]` §3.30) — this service has the
opposite problem: not a scalability risk, an observability gap.

### 3.23 Read-model projection `AsDto`

**Definition.** `DeliveryDocument → DeliveryDto`
(`…Infrastructure/Mongo/Documents/Extensions.cs:28-40`).

**Representation & storage.** A straight field-for-field copy of all five stored fields plus a
`Select` over registrations. **All stored data is exposed**; there is no field the service holds
back — contrast `customers-service`, whose `AsDto` exposes 3 of 8 fields and reserves the rest for
an admin-only `AsDetailsDto` (`[[customers-service]]` §3.28).

**Lifecycle.** Called only from `GetDeliveryHandler.cs:22`.

**Invariants & enforcement.** `Status` is copied as the **enum**, so its JSON representation
depends on the serializer's enum handling — an integer unless a string converter is configured, and
no converter is configured in this repository. Clients therefore see `"status": 0|1|2`, and the
meaning of those integers is the ordinal order in `DeliveryStatus.cs`. This makes §3.6's
append-only rule an **API** compatibility constraint, not merely a storage one.

**Extension procedure.** Adding a field to the DTO without adding it to `AsDto` yields `null`/
`default` in every response — **silent**. There is no test asserting projection completeness.

**Failure modes.** `document.Registrations.Select(...)` at `:35` has **no null guard**: a
`DeliveryDocument` whose `Registrations` is absent (BSON null) throws
`NullReferenceException` inside the query handler, which `ExceptionToResponseMapper`'s catch-all
turns into `400 {"code":"error"}` (§3.17). The same unguarded `Select` appears in `AsEntity`
(§3.26), where it breaks the write path too.

### 3.24 `IDeliveriesRepository` — the domain port

**Definition.** The Core-owned interface every write path uses
(`…Core/Repositories/IDeliveriesRepository.cs:7-14`):

```csharp
Task<Delivery> GetAsync(Guid id);
Task<Delivery> GetForOrderAsync(Guid id);
Task AddAsync(Delivery delivery);
Task UpdateAsync(Delivery delivery);
Task DeleteAsync(Delivery delivery);
```

**Representation & storage.** Declared in `Core`, which references **nothing** — the inversion that
keeps the domain free of MongoDB (`[[inward-dependency-service-skeleton]]`). Bound to
`DeliveriesMongoRepository` as `AddTransient` (`Infrastructure/Extensions.cs:51`).

**Lifecycle.** Injected into all four command handlers. Not used by the query handler (§3.22).

**Invariants & enforcement.** The port deals in **aggregates**, so every read materialises a full
`Delivery` and every write serialises one. There is no partial update, no projection, no
`$push`-style append for registrations — a single new registration rewrites the whole document
(§3.5).

**Extension procedure.** Adding a method requires editing the interface in `Core` and the
implementation in `Infrastructure`; the compiler enforces both. This is the **only** extension
point in the service that fails loudly at build time rather than silently at runtime.

**Failure modes.** `GetForOrderAsync(Guid id)` names its parameter `id`, not `orderId`, and takes
the same type as `GetAsync` — transposing the two at a call site compiles cleanly and returns the
wrong aggregate.

### 3.25 `DeliveriesMongoRepository` (and the uncalled `DeleteAsync`)

**Definition.** The adapter translating the port into Convey's generic repository
(`…Infrastructure/Mongo/Repositories/DeliveriesMongoRepository.cs:10-37`).

**Representation & storage.** `internal class` holding one
`IMongoRepository<DeliveryDocument, Guid>` `[convey]`. Each method is one line:

| Port method | Implementation | Line |
| --- | --- | --- |
| `GetAsync(id)` | `_repository.GetAsync(d => d.Id == id)` → `document?.AsEntity()` | `:17-21` |
| `GetForOrderAsync(id)` | `_repository.GetAsync(d => d.OrderId == id)` → `document?.AsEntity()` | `:23-27` |
| `AddAsync` | `_repository.AddAsync(delivery.AsDocument())` | `:29-30` |
| `UpdateAsync` | `_repository.UpdateAsync(delivery.AsDocument())` | `:32-33` |
| `DeleteAsync` | `_repository.DeleteAsync(delivery.Id)` | `:35-36` |

**Lifecycle.** Transient; a new instance per resolution.

**Invariants & enforcement.**
- **No filter carries a version or status predicate** — every update is an unconditional whole-
  document replace keyed on `_id` `[convey]` (§3.5).
- `GetForOrderAsync` uses `GetAsync` under the hood, i.e. a **single-document** read against a
  non-unique field. With more than one document per `OrderId` (§3.11) the result is whichever
  document the driver returns first; no `OrderBy`, no `Limit`, no error.
- **`DeleteAsync` has no caller.** `grep -rn "DeleteAsync" src` finds only the interface
  declaration and this implementation. It is reachable only by a future caller — meaning the
  service has a fully-wired hard-delete capability that nothing exercises and no test covers.

**Extension procedure.** To add a query shape, add the method to the port (§3.24) and implement it
here with a lambda over `DeliveryDocument`. **Do not** add domain logic here: the class exists
solely to convert between document and entity.

**Failure modes.** If `DeleteAsync` is ever wired up, it deletes by `_id` with no soft-delete, no
audit and no event — the delivery vanishes from the read model while `orders-service` still holds
whatever state the last `DeliveryStarted`/`Completed`/`Failed` left it in.

### 3.26 Hand-written document mapping

**Definition.** Two static extension methods that are the entire ORM
(`…Infrastructure/Mongo/Documents/Extensions.cs:10-26`).

**Representation & storage.**

```csharp
public static DeliveryDocument AsDocument(this Delivery delivery)
    => new DeliveryDocument { Id = delivery.Id, OrderId = …, Status = …, Notes = …,
         Registrations = delivery.Registrations.Select(r => new DeliveryRegistrationDocument { … }) };

public static Delivery AsEntity(this DeliveryDocument document)
    => new Delivery(document.Id, document.OrderId, document.Status,
         document.Registrations.Select(r => new DeliveryRegistration(r.Description, r.DateTime)));
```

`AsEntity` uses the **public constructor**, which emits no domain event (`Delivery.cs:29-36`) —
that is what makes rehydration silent.

**Lifecycle.** `AsDocument` on every write; `AsEntity` on every read of an aggregate.

**Invariants & enforcement.** None automated. The pairing is maintained by hand, and the
platform-wide consequence is described in `[[database-per-service-with-document-mapping]]`.

**Extension procedure.** Every field change must touch **three** projections here
(`AsDocument`, `AsEntity`, `AsDto`) plus the document class plus the entity. Nothing verifies the
set is complete.

**Failure modes.**
- **`AsEntity` dereferences `document.Registrations` with no null check** (`:26`). A delivery
  document written without a `Registrations` array — a legacy document, a manual insert, or a
  future partial-update path — throws `NullReferenceException` on **every** read, which becomes
  `400 {"code":"error"}` over HTTP and, over AMQP, an unmapped exception with **no rejected event**
  (§3.16). Every one of the four commands loads the aggregate first, so such a document is
  permanently unmutatable and undiagnosable from the API.
- `Registrations` is assigned a lazy `Select` in `AsDocument`; it is enumerated by the BSON
  serialiser during the write, after the method has returned.

### 3.27 Mongo storage configuration & collection binding

**Definition.** How the service is bound to a physical database and collection.

**Representation & storage.** Two registrations and one config block:

- `.AddMongo()` (`Infrastructure/Extensions.cs:68`) reads the `mongo` section `[convey]`.
- `.AddMongoRepository<DeliveryDocument, Guid>("deliveries")` (`:73`) binds the document type to
  the **`deliveries`** collection and registers `IMongoRepository<DeliveryDocument, Guid>`.
- `Api/appsettings.json:95-99`: `connectionString: "mongodb://localhost:27017"`,
  `database: "deliveries-service"`, `seed: false`.

Overrides: `appsettings.docker.json:64-68` switches the host to `mongodb://mongo:27017` (same
database name, same `seed: false`); `appsettings.local.json` does not touch `mongo` at all.
`Api/appsettings.development.json` is `{}`.

**Lifecycle.** `database-per-service` — nothing else in the platform names `deliveries-service` as
its Mongo database. The collection is created lazily by the driver on first write.

**Invariants & enforcement.** `seed: false` means Convey's seeder never runs, so **there is no
seed data class in this repository and none is expected** `[convey]`. There is **no index
declaration** and **no schema validator** — the collection accepts any shape (§3.11, §5.2).

When Vault's dynamic Mongo lease is enabled (`appsettings.json:182-192`), the connection string is
replaced at startup by
`mongodb://{{username}}:{{password}}@localhost:27017` with per-lease credentials (§3.41), so the
static `connectionString` above is the fallback used only when Vault is disabled — which is the
case in **both** `local` and `docker` (`appsettings.local.json:38-51`,
`appsettings.docker.json:87-101`).

**Extension procedure.** A second collection needs another `AddMongoRepository<TDoc, TId>("name")`
line plus a repository adapter. Renaming `"deliveries"` silently orphans all existing data — there
is no migration mechanism (§5).

**Failure modes.** The `docker` profile points at a bare `mongo` host with no credentials and no
replica set; `disableTransactions: true` in the outbox config (`appsettings.json:107`) is the
consequence — Mongo transactions require a replica set.

### 3.28 Transactional outbox & inbox idempotence

**Definition.** The mechanism that makes "persist then publish" survive a crash, implemented as two
handler decorators over a Mongo-backed store.

**Representation & storage.**
- `OutboxCommandHandlerDecorator<TCommand>` (`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:10-36`),
  `[Decorator]`-marked, registered via `TryDecorate(typeof(ICommandHandler<>), …)`
  (`Infrastructure/Extensions.cs:55`). It captures a message id — the inbound
  `IMessageProperties.MessageId` if present, otherwise a fresh `Guid.NewGuid().ToString("N")`
  (`:26-29`) — and wraps the inner handler:

  ```csharp
  public Task HandleAsync(TCommand command)
      => _enabled ? _outbox.HandleAsync(_messageId, () => _handler.HandleAsync(command))
                  : _handler.HandleAsync(command);
  ```

  (`:32-35`.)
- `OutboxEventHandlerDecorator<TEvent>` is registered identically (`Extensions.cs:56`) but has
  **nothing to decorate** — this service subscribes to zero external events (§1.2), so no
  `IEventHandler<>` exists in the assembly. It is dead registration.
- `.AddMessageOutbox(o => o.AddMongo())` (`Extensions.cs:66`) supplies `IMessageOutbox` `[convey]`.
- Configuration (`Api/appsettings.json:100-108`): `enabled: true`, `type: "sequential"`,
  `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: "inbox"`,
  `outboxCollection: "outbox"`, `disableTransactions: true`.

**Lifecycle.** On an inbound command: the decorator asks the outbox to run the handler under
`_messageId`; Convey's outbox records the id in `inbox` and buffers any `PublishAsync` into
`outbox` `[convey]`. A background processor then drains `outbox` every 2000 ms, and entries expire
after 3600 s. `MessageBroker` consults `_outbox.Enabled` (`MessageBroker.cs:76`) and routes each
event to `SendAsync` or straight to `IBusPublisher`.

**Invariants & enforcement.**
- **Idempotence is per message id, not per command content.** A saga retry that reuses the message
  id is suppressed; a retry with a new id runs the handler again — which for
  `AddDeliveryRegistration` is then caught by the value-based set semantics (§3.8) only if the
  timestamp is identical, and for `CompleteDelivery` raises
  `CannotChangeDeliveryStateException` (§3.6).
- **On HTTP the message id is always fresh** (there are no inbound message properties), so the
  inbox provides **no** de-duplication for HTTP callers.
- `disableTransactions: true` means the inbox write and the domain write are **not** atomic; the
  exact ordering guarantee is `[convey]` and **`Unverifiable — Missing Source Evidence`**.
- **The outbox is `enabled: false` in `local`** (`appsettings.local.json:35-37`), so local
  behaviour — direct publish, no idempotence — differs from `docker`/production, where the base
  `true` stands (`appsettings.docker.json` does not override `outbox`).

**Extension procedure.** Nothing to do per command: the decorator is generic and applies to every
`ICommandHandler<>` automatically. If a new handler must **not** be idempotent-wrapped, there is no
opt-out short of removing the `TryDecorate` line or not implementing `ICommandHandler<>`.

**Failure modes.**
- With `expiry: 3600`, an inbox entry older than an hour is presumably purged `[convey]`, after
  which the same message id would be processed again. **`Unverifiable — Missing Source Evidence`**.
- A publish that reaches the outbox but whose domain write failed is still delivered — the decorator
  wraps the handler, but `MessageBroker.PublishAsync` is called *inside* it, and without
  transactions there is no rollback of the outbox entry.

### 3.29 `MessageBroker` publication path

**Definition.** The service's own abstraction over publishing
(`…Infrastructure/Services/MessageBroker.cs:16-87`), behind the `IMessageBroker` port in
`…Application/Services/IMessageBroker.cs`. Registered `AddTransient`
(`Infrastructure/Extensions.cs:50`).

**Representation & storage.** Holds `IBusPublisher`, `IMessageOutbox`,
`ICorrelationContextAccessor`, `IHttpContextAccessor`, `IMessagePropertiesAccessor`, `ITracer` and
`ILogger` (`:19-25`). `PublishAsync(IEnumerable<IEvent>)` (`:47-86`):

1. `null` collection → return (`:49-52`).
2. Read `MessageId`, `CorrelationId` and the span context from the inbound message properties
   (`:54-57`).
3. If no span context on the message, fall back to `_tracer.ActiveSpan?.Context.ToString()`
   (`:58-61`).
4. `headers = messageProperties.GetHeadersToForward()` — the `Saga`-only filter (§3.31).
5. `correlationContext = _contextAccessor.CorrelationContext ??
   _httpContextAccessor.GetCorrelationContext()` (`:64-65`) — broker first, HTTP header second
   (§3.30).
6. Per event: **skip `null`** (`:69-72`); mint `messageId = Guid.NewGuid().ToString("N")` (`:74`);
   `_logger.LogTrace(…)` (`:75`); then either `_outbox.SendAsync(...)` (`:78-79`) or
   `_busPublisher.PublishAsync(...)` (`:83-84`).

**Lifecycle.** Called once per command handler, with the mapped event array.

**Invariants & enforcement.**
- **`if (@event is null) continue;` is the sink for every `EventMapper` miss** (§3.13). Combined
  with `LogTrace` — which is *below* the configured `information` level
  (`Api/appsettings.json:33`) and below `verbose`'s intent only in `local`
  (`appsettings.local.json:15`) — a dropped event produces **no output at all** in `docker` or
  production. This is the mechanism that makes §3.13's silent failure genuinely invisible.
- Each event gets its **own fresh `messageId`**, while `originatedMessageId` preserves the causing
  message — the chain the outbox and downstream tracing rely on.
- The outbox check is per-event and per-call, so toggling `outbox.enabled` takes effect without a
  restart only if `IMessageOutbox.Enabled` is re-read — which it is, on every event.

**Extension procedure.** Nothing to register per event. To add a forwarded header, edit
`GetHeadersToForward` (§3.31) — the header whitelist is the only extension point, and it is a
`const string`.

**Failure modes.** Publication failures propagate as exceptions out of `PublishAsync`, i.e. **after**
the Mongo write has already committed. Without the outbox the write survives and the event is lost;
with the outbox the event is retried but the caller still sees an error.

### 3.30 Correlation context, `IAppContext`, `IIdentityContext`

**Definition.** The transport-agnostic caller identity plumbing
(`[[transport-agnostic-caller-context]]`), fully built and **entirely unconsumed by domain code**.

**Representation & storage.**
- `Application/IAppContext.cs:3-7` — `string RequestId`, `IIdentityContext Identity`.
- `Application/IIdentityContext.cs:6-…` — `Guid Id`, `string Role`, `bool IsAuthenticated`,
  `bool IsAdmin`, `IDictionary<string,string> Claims`.
- `Infrastructure/Contexts/CorrelationContext.cs:6-24` — the wire shape:
  `CorrelationId`, `SpanContext`, `User { Id, IsAuthenticated, Role, Claims }`, `ResourceId`,
  `TraceId`, `ConnectionId`, `Name`, `CreatedAt`.
- `Infrastructure/Contexts/IdentityContext.cs:24-31` — `Id = Guid.TryParse(id, out var userId) ?
  userId : Guid.Empty`, `Role = role ?? ""`, **`IsAdmin = Role.Equals("admin",
  InvariantCultureIgnoreCase)`**.
- `Infrastructure/Contexts/AppContextFactory.cs:19-33` — prefers the broker's
  `ICorrelationContextAccessor.CorrelationContext` (round-tripped through JSON at `:23-27`), else
  the HTTP `Correlation-Context` header via
  `Extensions.GetCorrelationContext` (`Infrastructure/Extensions.cs:95-98`), else
  `AppContext.Empty`.
- Registered at `Infrastructure/Extensions.cs:53-54`: the factory, plus
  `AddTransient(ctx => ctx.GetRequiredService<IAppContextFactory>().Create())` so `IAppContext`
  itself is resolvable.

**Lifecycle.** Constructed per resolution of `IAppContext`. **`grep -rn "IAppContext" src` shows
the only references are the interface, the factory and the two DI lines** — no handler, no query,
no repository injects it. The identity is therefore parsed on demand and then never asked for.

**Invariants & enforcement.** The header is **trusted absolutely**: `GetCorrelationContext`
deserialises whatever JSON the `Correlation-Context` header contains with no signature, no
validation and no origin check (`Extensions.cs:95-98`). Because the service has no authentication
(§3.37), a caller that can reach the port can assert any identity — but since nothing reads the
identity, the only present-day impact is on logging and on the correlation context forwarded to
downstream events. **The exposure becomes real the moment any handler starts consulting
`IAppContext`.**

**Extension procedure.** To make an operation owner-scoped: inject `IAppContext` into the handler,
read `Identity.Id`/`IsAdmin`, and throw an `AppException`. Note the delivery aggregate has **no
customer reference**, so an ownership check would first require joining through `OrderId` to
`orders-service` — a synchronous dependency this service currently does not have (§1.4).

**Failure modes.** A malformed `Correlation-Context` header throws inside
`JsonConvert.DeserializeObject` `[convey]`/Newtonsoft, which surfaces as the generic
`400 {"code":"error"}` — but only if something resolves `IAppContext`, which nothing does.

### 3.31 `Saga` header forwarding

**Definition.** The one broker header this service propagates
(`…Infrastructure/Extensions.cs:100-114`).

**Representation & storage.**

```csharp
const string sagaHeader = "Saga";
if (messageProperties?.Headers is null || !messageProperties.Headers.TryGetValue(sagaHeader, out var saga))
    return null;
return saga is null ? null : new Dictionary<string, object> { [sagaHeader] = saga };
```

**Lifecycle.** Called once per `PublishAsync` (`MessageBroker.cs:63`); the result is passed to the
outbox or the publisher as the outgoing headers.

**Invariants & enforcement.** **It is a whitelist of exactly one header.** Every other inbound
header is dropped. This is what lets `Pacco.Services.Operations` correlate a `DeliveryCompleted`
back to the saga instance that issued `StartDelivery`. On the **HTTP** path there are no message
properties at all, so `GetHeadersToForward` returns `null` and a gateway-originated write emits
events **without** the `Saga` header — meaning a delivery started through the gateway cannot be
correlated to a saga.

**Extension procedure.** Add the header name to the method. There is no configuration key; the
name is a `const` in code, and it must match `Pacco.Services.Operations`' expectation exactly
(case-sensitive `TryGetValue` on the header dictionary).

**Failure modes.** Silent loss of correlation on the HTTP path, as above; and a header value of a
non-`byte[]` type is forwarded verbatim without inspection.

### 3.32 Span-context propagation & Jaeger

**Definition.** Distributed-trace continuity across the broker
(`[[correlation-and-span-propagation]]`).

**Representation & storage.**
- `GetSpanContext(this IMessageProperties, string header)`
  (`Infrastructure/Extensions.cs:116-129`): returns `""` for null properties; else
  `Headers.TryGetValue(header, out var span) && span is byte[] spanBytes` →
  `Encoding.UTF8.GetString(spanBytes)`; else `""`.
- The header name comes from `rabbitMq.spanContextHeader: "span_context"`
  (`Api/appsettings.json:149`), defaulting to the same literal if blank
  (`MessageBroker.cs:18,40-42`).
- `.AddJaeger()` (`Extensions.cs:71`) + `UseJaeger()` (`:82`) + the RabbitMQ plugin
  `AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())` (`:65`).
- Config (`Api/appsettings.json:68-76`): `serviceName: "deliveries"` — **note it is `deliveries`,
  not `deliveries-service`, unlike the Consul/Vault/Mongo names**; `sampler: "const"`;
  `excludePaths: ["/", "/ping", "/metrics"]`. Disabled in `local`
  (`appsettings.local.json:23-25`), enabled in `docker` against host `jaeger`
  (`appsettings.docker.json:46-54`).

**Lifecycle.** Inbound: the Jaeger RabbitMQ plugin reads the span header and activates a span
`[convey]`. Outbound: `MessageBroker` re-reads the header, or falls back to
`_tracer.ActiveSpan.Context.ToString()` (`:58-61`), and passes it to the outbox/publisher.

**Invariants & enforcement.** The span context survives only if the header is a `byte[]`. A
publisher that writes it as a `string` produces `""` and **silently starts a new trace** —
`GetSpanContext` has no logging and no error path.

**Extension procedure.** Changing `spanContextHeader` requires changing it in **every** service's
`appsettings.json` simultaneously; there is no negotiation and no default agreement beyond the
shared literal.

**Failure modes.** `sampler: "const"` with no `samplerParam` in config means the sampling rate is
Convey's default (`[convey]`, **`Unverifiable — Missing Source Evidence`**) — most likely
"sample everything", which is fine for a low-volume service and expensive otherwise.

### 3.33 Handler log templates & property redaction

**Definition.** Per-command log lines plus a platform-standard redaction list
(`[[structured-logging-with-property-redaction]]`).

**Representation & storage.**
- `MessageToLogTemplateMapper` (`…Infrastructure/Logging/MessageToLogTemplateMapper.cs:8-44`) is an
  `internal sealed` `IMessageToLogTemplateMapper` `[convey]` holding a
  `IReadOnlyDictionary<Type, HandlerLogTemplate>` **rebuilt on every property access** (`:10-11`
  — an expression-bodied property, so a new dictionary is allocated per `Map` call). Entries:

  | Command | `After` template |
  | --- | --- |
  | `AddDeliveryRegistration` | `"Added a registration for the delivery with id: {DeliveryId}."` |
  | `CompleteDelivery` | `"Completed the delivery with id: {DeliveryId}."` |
  | `FailDelivery` | `"Failed the delivery with id: {DeliveryId}, reason: {Reason}"` |
  | `StartDelivery` | `"Started the delivery with id: {DeliveryId}."` |

  Only `After` is set on every template — **no `Before` and no `OnError` template exists**, so a
  failing command produces no templated line.
- `Map<TMessage>` returns the template or `null` (`:39-43`); `null` means Convey logs nothing for
  that message `[convey]`.
- Activated by `.AddHandlersLogging()` (`Infrastructure/Extensions.cs:72`). **Note the mapper class
  is `internal sealed` and is never referenced by name in `Extensions.cs`** — it must be picked up
  by assembly scan `[convey]`; that is **`Unverifiable — Missing Source Evidence`**.
- Redaction: `logger.excludeProperties` (`Api/appsettings.json:35-48`) lists `api_key`,
  `access_key`, `ApiKey`, `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`, `Password`,
  `Email`, `Login`, `Secret`, `Token`. `logger.excludePaths` is `["/", "/ping", "/metrics"]`
  (`:34`). Sinks: console on, file on (`logs/logs.txt`, daily), Seq on
  (`http://localhost:5341`, `apiKey: "secret"`), ELK off (`:49-65`). `local` sets
  `level: "verbose"` and turns file and Seq **off** (`appsettings.local.json:14-22`); `docker`
  turns file **off** and points Seq at `http://seq:5341` (`appsettings.docker.json:27-45`).

**Lifecycle.** Evaluated per handled message, after the handler returns successfully.

**Invariants & enforcement.** `{Reason}` on `FailDelivery` is **not** in the redaction list, so the
caller-supplied failure text (§3.10) is written verbatim to console, file and Seq. Nothing bounds
its length.

**Extension procedure.** Add a `{ typeof(NewCommand), new HandlerLogTemplate { After = "…" } }`
entry. Omitting it is **silent** — the command simply produces no log line, which is the current
state of every **query** and every failure path.

**Failure modes.** No `OnError` template means the only record of a failed command is whatever
Convey's error middleware emits; combined with §3.16's silent rejections, an AMQP command that
throws an unmapped exception may leave **no trace in any sink**.

### 3.34 RabbitMQ topology: exchange, queue template, subscriptions

**Definition.** The service's messaging surface, following
`[[service-owned-topic-exchange-messaging]]`.

**Representation & storage.** `Api/appsettings.json:109-150`:

| Key | Value | Meaning |
| --- | --- | --- |
| `connectionName` | `deliveries-service` | connection label in the broker UI |
| `retries` / `retryInterval` | `3` / `2` | connection retry policy |
| `conventionsCasing` | `snakeCase` | drives every routing key (§3.14) |
| `exchange` | `{declare: true, durable: true, autoDelete: false, type: "topic", name: "deliveries"}` | **the service owns and declares its own exchange** |
| `queue.template` | `deliveries-service/{{exchange}}.{{message}}` | one queue per (publisher-exchange, message) pair, namespaced to this consumer |
| `context` | `{enabled: true, header: "message_context"}` | correlation context travels in this header |
| `spanContextHeader` | `span_context` | §3.32 |
| `logger.enabled` | `true` | Convey's own broker logging |

Registration: `.AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`
(`Infrastructure/Extensions.cs:65`); activation `UseRabbitMq()` (`:86`) followed by **four**
subscriptions and **zero** event subscriptions (`:87-90`):

```csharp
.SubscribeCommand<StartDelivery>()
.SubscribeCommand<CompleteDelivery>()
.SubscribeCommand<FailDelivery>()
.SubscribeCommand<AddDeliveryRegistration>()
```

`hostnames` is `["localhost"]` in the base file (`:121-123`) and `["rabbitmq"]` under docker
(`appsettings.docker.json:69-73`). Credentials are the RabbitMQ defaults `guest`/`guest`
(`:117-118`) — development-only values checked into the repository; production is expected to
override them from Vault KV (§3.41).

**Lifecycle.** Exchange and queues are declared at startup (`declare: true`) and are `durable`,
so they survive a broker restart. Subscriptions are wired inside `UseInfrastructure`, i.e. before
the HTTP pipeline's `UseDispatcherEndpoints` in `Program.cs:31-32`.

**Invariants & enforcement.** The queue template namespaces by consumer, so two services
subscribing to the same message each get their own queue and both receive it — a fan-out, not a
work queue. Within this service there is one queue per subscribed command, and Convey's consumer
concurrency is **`Unverifiable — Missing Source Evidence`**; that gap is what leaves §3.5's
lost-update window open.

**Extension procedure.** Adding a subscription is one line here **plus** the handler
(auto-discovered). Removing a `SubscribeCommand` line leaves the durable queue in place, silently
accumulating messages.

**Failure modes.** Because the exchange is declared by this service, a consumer that starts before
`deliveries-service` has ever run may fail to bind — the exchange does not exist yet.

### 3.35 The dual-transport write surface

**Definition.** The composite of §3.19, §3.34 and §3.36: every command is reachable over HTTP and
over AMQP, with different failure semantics (`[[dual-mode-edge-write]]`).

**Representation & storage.** Two independent registration lists that happen to contain the same
four commands:

| Command | HTTP route (`Api/Program.cs`) | AMQP (`Infrastructure/Extensions.cs`) |
| --- | --- | --- |
| `StartDelivery` | `POST deliveries` (`:35`) | `:87` |
| `FailDelivery` | `POST deliveries/{deliveryId}/fail` (`:37`) | `:89` |
| `CompleteDelivery` | `POST deliveries/{deliveryId}/complete` (`:38`) | `:88` |
| `AddDeliveryRegistration` | `POST deliveries/{deliveryId}/registrations` (`:39`) | `:90` |

**Lifecycle.** Both paths converge on the same `ICommandDispatcher` and therefore the same
decorated handler — the domain logic is identical. Everything that differs is at the edges.

**Invariants & enforcement.** The differences that matter operationally:

| Aspect | HTTP | AMQP |
| --- | --- | --- |
| Idempotence | none (fresh message id per request, §3.28) | inbox de-duplication by message id |
| Failure signal | 400 + `{code, reason}`, always | rejected event, **or nothing** (§3.16) |
| Correlation | `Correlation-Context` header, `Saga` header **absent** (§3.31) | full message context + `Saga` |
| Response body | `201 Created` + `Location` for `StartDelivery`; empty for the rest | none |
| Who calls it | the gateway only (§6.5) | the operations saga |

**Extension procedure.** Register both, or knowingly register one. There is no single place that
declares "this command is HTTP-only" — the absence is expressed by omission in two different files.

**Failure modes.** The two lists are maintained by hand and can drift; nothing compares them.

### 3.36 HTTP dispatcher endpoints & route binding

**Definition.** The five HTTP routes, declared inline in `Program.cs:32-39` via
`UseDispatcherEndpoints` `[convey]`:

| Route | Handler | Kind |
| --- | --- | --- |
| `GET ""` | writes `AppOptions.Name` (`:33`) | liveness/name probe |
| `GET deliveries/{deliveryId}` | `GetDelivery` → `DeliveryDto` (`:34`) | read |
| `POST deliveries` | `StartDelivery`, `afterDispatch` → `Created($"deliveries/{cmd.DeliveryId}")` (`:35-36`) | write |
| `POST deliveries/{deliveryId}/fail` | `FailDelivery` (`:37`) | write |
| `POST deliveries/{deliveryId}/complete` | `CompleteDelivery` (`:38`) | write |
| `POST deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` (`:39`) | write |

**Representation & storage.** There are **no controllers and no attribute routing** in this
repository — `grep -rn "Controller" src` returns nothing. Routes exist only as these six fluent
calls. `AppOptions` is resolved from DI inside the `GET ""` lambda (`:33`).

**Lifecycle.** Built once at startup, after `UseInfrastructure()` (`:31`).

**Invariants & enforcement.**
- Only `StartDelivery` customises its response. The other three writes rely on the dispatcher's
  default success status `[convey]` — **`Unverifiable — Missing Source Evidence`** (Convey
  conventionally returns `202 Accepted` for a dispatched command, but that cannot be confirmed
  here).
- Route-template values bind into command properties by name; `{deliveryId}` → `DeliveryId`.
  Whether a body value beats a route value on conflict is `[convey]` and
  **`Unverifiable — Missing Source Evidence`** — worth knowing before trusting the route id as
  authoritative.
- **No route declares any authorization.** Convey's endpoint builder supports an auth argument
  `[convey]`; none is passed. All access control lives in the gateway (§6.5, §3.37).

**Extension procedure.** Add a `.Post<TCommand>("route")` or `.Get<TQuery, TResult>("route")` line.
The command/query type is the only registration — the handler is found by scan. A route added here
is **not** automatically added to the gateway, so it stays unreachable from outside the cluster.

**Failure modes.** `GET ""` returns the app name to any caller with no authentication; it is also
the path excluded from logging and metrics (`excludePaths`), so probes are invisible.

### 3.37 Security posture

**Definition.** What protects this service. The answer is: **the gateway, and nothing else in this
process**.

**Representation & storage.** Positive evidence of what *is* registered:
`.AddSecurity()` (`Infrastructure/Extensions.cs:75`) — Convey's hashing/encryption/signing
primitives `[convey]`. Negative evidence, all verified by grep over `src/`:

| Mechanism | Present? |
| --- | --- |
| `AddJwt` / `UseAuthentication` / `UseAuthorization` / `[Authorize]` | **no** |
| `AddCertificateAuthentication` / `UseCertificateAuthentication` | **no** (present in `customers-service`; absent here) |
| A `security` section in any `appsettings*.json` | **no** |
| Any consumer of `IHasher`/`IEncryptor`/`ISigner` from `AddSecurity()` | **no** |
| Any use of `IAppContext` in a handler | **no** (§3.30) |
| Any per-route auth argument | **no** (§3.36) |

The `jwt` block (`Api/appsettings.json:77-85`) is **inert** — nothing reads it, and
`appsettings.local.json:26-30` blanks its certificate path.

**Lifecycle.** Not applicable — there is no authentication lifecycle.

**Invariants & enforcement.** The only enforced access control for `deliveries-service` lives in
the gateway (`ntrada.yml:189-204`): `GET /{deliveryId}`, `POST /`, `POST /{deliveryId}/fail`,
`POST /{deliveryId}/complete` and `POST /{deliveryId}/registrations` all require authentication,
and — notably — **none of them requires an `admin` claim**, unlike four of the six
`customers-service` routes. Any authenticated Pacco user can complete or fail **any** delivery by
id. Whether that is intended is an open question (§8.3, Q-3).

Anything that can reach port 5003 (or port 80 in the container) directly bypasses the gateway
entirely. In `docker` the service is published on the compose network and, per
`compose/services.yml`, mapped to a host port — so on a developer machine the service is reachable
without the gateway.

**Extension procedure.** To require identity inside the service: inject `IAppContext` into the
handler and check `Identity.IsAuthenticated`. To make that trustworthy you must first stop trusting
the `Correlation-Context` header (§3.30) — i.e. add real authentication — otherwise the check is
cosmetic. To copy `customers-service`'s model, add `AddCertificateAuthentication()` +
`UseCertificateAuthentication()` and a `security.certificate` config block with an ACL.

**Failure modes.** A misconfigured network exposing 5003 gives an unauthenticated caller full write
access to every delivery, including `POST /{id}/fail` with an arbitrary `reason` that propagates
into `orders-service` as an order cancellation (§3.10).

### 3.38 Public-contracts endpoint

**Definition.** `UsePublicContracts<ContractAttribute>()`
(`…Infrastructure/Extensions.cs:84`) — Convey publishes a machine-readable document describing every
`[Contract]`-marked type `[convey]`.

**Representation & storage.** The marked types are the 4 commands, 4 integration events and 4
rejected events (12 in total). The URL and payload format are Convey's;
**`Unverifiable — Missing Source Evidence`** — no path is configured here and no route appears in
`Program.cs`.

**Lifecycle.** Computed at startup from assembly metadata; served for the process lifetime.

**Invariants & enforcement.** Marking is opt-in and has no effect on behaviour (§3.14). The
endpoint is **unauthenticated**, like every other route (§3.37), so the service's full message
vocabulary is readable by anyone who can reach it.

**Extension procedure.** Add `[Contract]`. Nothing else.

**Failure modes.** Forgetting the attribute silently omits a type from the platform's contract
surface, which is the artifact other teams read to integrate.

### 3.39 Consul registration & Fabio routing

**Definition.** How the service becomes addressable
(`[[registry-mediated-discovery-and-routing]]`).

**Representation & storage.** `.AddConsul()` and `.AddFabio()`
(`Infrastructure/Extensions.cs:63-64`). Config `Api/appsettings.json:7-22`:
`consul.url http://localhost:8500`, `service: "deliveries-service"`,
`address: "docker.for.win.localhost"`, `port: "5003"`, `pingEnabled: true`,
`pingEndpoint: "ping"`, `pingInterval: 3`, `removeAfterInterval: 3`; `fabio.url
http://localhost:9999`, `service: "deliveries-service"`. Docker override
(`appsettings.docker.json:6-21`): `consul.url http://consul:8500`, `address:
"deliveries-service"`, **`port: "80"`**, `fabio.url http://fabio:9999`. Both disabled in `local`
(`appsettings.local.json:2-8`).

Also `httpClient.type: "fabio"` with `services: {}` (`appsettings.json:23-31`) — the service is
configured to *call* others through Fabio but names **no** downstream service, confirming §1.4's
"no synchronous dependencies".

**Lifecycle.** Registered at startup; deregistered after 3 failed pings
(`removeAfterInterval: 3`).

**Invariants & enforcement.** The `address` value is what other services will dial. The base
value `docker.for.win.localhost` is a **Windows-Docker-specific** hostname that resolves nowhere
else — a Linux or macOS developer running with the default profile registers an unreachable
address. The `docker` profile corrects it.

**Extension procedure.** To call another service, add an entry under `httpClient.services` and
inject Convey's `IHttpClient`; the Fabio type resolves the logical name. This service has no such
call today.

**Failure modes.** `pingEndpoint: "ping"` must exist — it is served by `UseConvey()`
(`Extensions.cs:83`) `[convey]`; nothing in this repository declares it, so a change to the Convey
pipeline could silently break health checking. `appsettings.local.json:4` sets `pingEndpoint: ""`
alongside `enabled: false`.

### 3.40 Metrics

**Definition.** `.AddMetrics()` (`Infrastructure/Extensions.cs:70`) + `UseMetrics()` (`:85`) —
AppMetrics with a Prometheus scrape endpoint `[convey]`.

**Representation & storage.** `Api/appsettings.json:86-94`: `enabled: true`,
`influxEnabled: false`, `prometheusEnabled: true`, `database: "pacco"`, `env: "local"`,
`interval: 5`. Docker sets `env: "docker"` (`appsettings.docker.json:55-63`); `local` disables
metrics entirely (`appsettings.local.json:31-34`). The scrape path `/metrics` is excluded from
logging and tracing (`logger.excludePaths`, `jaeger.excludePaths`). The platform's
`compose/prometheus/prometheus.yml` is the scrape configuration.

**Lifecycle.** Counters are collected in-process and exposed on demand.

**Invariants & enforcement.** **No custom metric is defined anywhere in this repository** — there is
no delivery-started counter, no completion-rate gauge, no in-progress-age histogram. Everything
observable is Convey's generic HTTP/handler instrumentation. Combined with §3.6's missing sweep,
there is no signal that would reveal a delivery stuck `InProgress`.

**Extension procedure.** Inject AppMetrics' `IMetrics` and record explicitly; there is no existing
example in this repository to copy.

**Failure modes.** `influxUrl` points at `http://localhost:8086` with `influxEnabled: false` — inert
configuration that will mislead anyone who flips the flag without also fixing the host.

### 3.41 Vault: KV settings, service PKI, dynamic Mongo lease

**Definition.** How the service obtains secrets and credentials
(`[[vault-issued-dynamic-credentials-and-service-pki]]`).

**Representation & storage.** `.UseVault()` on the host builder (`Api/Program.cs:41`) — a
**configuration-source** hook, so it runs before `ConfigureServices` values are bound `[convey]`.
Config `Api/appsettings.json:164-193`:

| Block | Value | Effect |
| --- | --- | --- |
| root | `enabled: true`, `url http://localhost:8200`, `authType: "token"`, `token: "secret"` | dev token, checked in |
| `kv` | `enabled: true`, `engineVersion: 2`, `mountPoint: "kv"`, `path: "deliveries-service/settings"` | overlays remote settings onto local config |
| `pki` | `enabled: true`, `roleName: "deliveries-service"`, `commonName: "deliveries-service.pacco.io"` | issues the service's own X.509 identity |
| `lease.mongo` | `type: "database"`, `roleName: "deliveries-service"`, `enabled: true`, `autoRenewal: true`, `templates.connectionString: "mongodb://{{username}}:{{password}}@localhost:27017"` | short-lived Mongo credentials, auto-renewed |

`local` disables all four (`appsettings.local.json:38-51`); `docker` disables all four while
pointing the URL at `http://vault:8200` (`appsettings.docker.json:87-101`).

**Lifecycle.** At startup Vault is authenticated, KV values are merged into `IConfiguration`, a PKI
certificate is issued, and a Mongo lease is taken and thereafter renewed in the background
`[convey]`.

**Invariants & enforcement.** The PKI certificate is issued but, in this service, **used for
nothing** — there is no certificate authentication (§3.37) and no outbound mTLS client. The
`customers-service` counterpart at least consumes an ACL. Here the PKI block is provisioning
without a consumer.

**Because Vault is disabled in *both* runnable profiles, every mechanism above is inert in every
environment this repository can actually start.** The static `mongodb://localhost:27017` /
`mongodb://mongo:27017` connection strings and the literal `"secret"` tokens are what run.

**Extension procedure.** To add a secret, write it to `kv/deliveries-service/settings` under the
same key path as the `appsettings.json` key you want to override; no code change is needed. To
enable Vault in `docker`, flip the four `enabled` flags and provide a real `token` through the
environment.

**Failure modes.** `token: "secret"` and `password: "secret"` are placeholders committed to the
repository; they are harmless only because Vault is off. Enabling Vault without replacing them
fails authentication at startup — **loud**, the host will not start.

### 3.42 Environment configuration layering

**Definition.** Four files with `ASPNETCORE_ENVIRONMENT`-based overlay
(`[[composable-per-concern-environment-stacks]]`).

**Representation & storage.**

| File | Lines | Role |
| --- | --- | --- |
| `Api/appsettings.json` | 194 | the complete baseline: every key the service reads |
| `Api/appsettings.local.json` | 52 | developer laptop: disables consul, fabio, jaeger, metrics, **outbox**, vault; `logger.level: "verbose"`; file+Seq sinks off; blanks `jwt.certificate.location` and `httpClient.type` |
| `Api/appsettings.docker.json` | 102 | compose: consul/fabio/jaeger/metrics **on** against container hostnames, `consul.port: "80"`, mongo `mongodb://mongo:27017`, rabbit host `rabbitmq`, redis host `redis`, Seq `http://seq:5341`, **vault off** |
| `Api/appsettings.development.json` | — | **`{}`** — an empty object; the `Development` environment inherits the baseline unchanged, i.e. localhost everything with Vault **enabled** |

`Dockerfile:10` sets `ENV ASPNETCORE_ENVIRONMENT docker`; `scripts/start.sh` sets the local
profile.

**Lifecycle.** Resolved once at host construction; Vault KV is layered on top (§3.41) when enabled.

**Invariants & enforcement.** Two behavioural differences worth internalising before debugging:

1. **`local` has no outbox** (`appsettings.local.json:35-37`), so events publish directly and
   inbox idempotence does not exist. A bug that only manifests with the outbox on will never
   reproduce locally.
2. **`development` is empty**, so running with `ASPNETCORE_ENVIRONMENT=Development` attempts to
   contact Vault at `http://localhost:8200` with token `"secret"` and will fail at startup unless a
   dev Vault is running — the least-usable of the three profiles.

**Extension procedure.** Add the key to `appsettings.json` first (it is the schema of record), then
override per environment. A key present only in an environment file is invisible to anyone reading
the baseline.

**Failure modes.** `docker` does not override `mongo.seed`, `outbox.*` or `security` (which does not
exist), so those inherit the baseline — outbox **on** in docker, off in local.

### 3.43 Registered-but-unused dependencies

**Definition.** Two services wired into DI that no code resolves.

**Representation & storage.**
- **Redis.** `.AddRedis()` (`Infrastructure/Extensions.cs:69`) with
  `redis.connectionString: "localhost"` and `instance: "deliveries:"`
  (`Api/appsettings.json:151-154`; docker → host `redis`, same prefix). `grep -rn
  "IDistributedCache\|ICacheService\|IConnectionMultiplexer" src` returns **nothing** — the
  key-prefix reservation described by `[[prefix-partitioned-shared-cache]]` exists, but this
  service stores nothing in it.
- **`IDateTimeProvider`.** Declared in `Application/Services/IDateTimeProvider.cs:5-8`
  (`DateTime Now`), implemented as `DateTime.UtcNow`
  (`Infrastructure/Services/DateTimeProvider.cs:8`), registered `AddSingleton`
  (`Infrastructure/Extensions.cs:52`) — and **injected nowhere**. `grep -rn "IDateTimeProvider" src`
  finds only those three lines.

**Lifecycle.** Constructed lazily by the container; since nothing resolves them, never constructed
at all (Redis's connection is `[convey]` and may connect eagerly —
**`Unverifiable — Missing Source Evidence`**).

**Invariants & enforcement.** The `IDateTimeProvider` gap is the load-bearing one: **every
timestamp in this service is caller-supplied** (`StartDelivery.DateTime`,
`AddDeliveryRegistration.DateTime`, §3.7). The abstraction that would let the service stamp its own
time — and let a test control it — exists and is ignored.

**Extension procedure.** Both are one constructor parameter away from being used. Injecting
`IDateTimeProvider` into a handler and stamping `_dateTimeProvider.Now` instead of trusting the
command would be a **contract change**, since the command's `DateTime` field would become
advisory or removable.

**Failure modes.** Redis being configured but unused means a `redis` outage cannot affect this
service — unless `AddRedis()` connects eagerly at startup, in which case it can prevent boot for no
functional benefit.

### 3.44 Swagger documentation

**Definition.** `.AddWebApiSwaggerDocs()` (`Infrastructure/Extensions.cs:74`) + `UseSwaggerDocs()`
(`:81`).

**Representation & storage.** `Api/appsettings.json:155-163`: `enabled: true`,
`reDocEnabled: false`, `name: "v1"`, `title: "API"`, `version: "v1"`, `routePrefix: "docs"`,
`includeSecurity: true`. `docker` repeats the same block verbatim
(`appsettings.docker.json:78-86`); `local` does not override it, so Swagger is on in every profile.

**Lifecycle.** Generated at startup from the dispatcher endpoint registrations `[convey]`.

**Invariants & enforcement.** `UseSwaggerDocs()` is **second** in the pipeline
(`Extensions.cs:80-81`), before `UseJaeger`/`UseConvey`, and there is no authentication (§3.37) —
so `/docs` is publicly readable by anything that can reach the service. `includeSecurity: true`
adds a security scheme to the document even though the service enforces none.

**Extension procedure.** Documentation follows the routes automatically; there are no XML doc
comments in this repository to enrich it.

**Failure modes.** The generated document reflects the **HTTP** surface only — the four AMQP
subscriptions (§3.34) are invisible in it, so Swagger under-describes the service's actual write
surface by half.

### 3.45 Container image & release path

**Definition.** How the service is built and shipped
(`[[independent-per-repository-release]]`).

**Representation & storage.**
- `Dockerfile` — two stages: `mcr.microsoft.com/dotnet/core/sdk:3.1` running
  `dotnet publish src/Pacco.Services.Deliveries.Api -c release -o out` (`:1-4`), then
  `mcr.microsoft.com/dotnet/core/aspnet:3.1` with `ASPNETCORE_URLS http://*:80`,
  `ASPNETCORE_ENVIRONMENT docker` and
  `ENTRYPOINT dotnet Pacco.Services.Deliveries.Api.dll` (`:6-11`).
- `scripts/dockerize.sh:16-21` — tags `$DOCKER_USERNAME/pacco.services.deliveries` with both a
  moving tag and a version tag, then pushes both.
- `scripts/build.sh`, `scripts/start.sh`, `scripts/test.sh` — the standard trio.
- Platform side: `compose/services.yml` runs image `devmentors/pacco.services.deliveries` and maps
  host port **5003**.

**Lifecycle.** Build → publish → image → registry → compose/orchestrator. There is no CI
configuration in this repository.

**Invariants & enforcement.** The image bakes `ASPNETCORE_ENVIRONMENT=docker`, so a container run
anywhere uses `appsettings.docker.json` — including its **container hostnames** (`mongo`,
`rabbitmq`, `consul`, `fabio`, `seq`, `redis`). Running the image outside that network fails to
resolve them. Overriding the environment variable is the only escape, and there is no
`appsettings.production.json` to switch to.

**Extension procedure.** Nothing service-specific; the Dockerfile hard-codes the project path and
DLL name, so renaming the Api project requires editing `Dockerfile:4` and `:11`.

**Failure modes.** `.dockerignore` is not present in the file listing, so `COPY . .` (`:3`) copies
the whole working tree including any local `logs/` and `bin/`/`obj/` output into the build context.

### 3.46 Test posture — there are no tests

**Definition.** The verification gap.

**Representation & storage.** `find . -name "*.csproj"` returns exactly four paths, all under
`src/`: `Core`, `Api`, `Infrastructure`, `Application`. There is **no `tests/` directory, no
`*.Tests` project, no xUnit/NUnit/MSTest reference, and no `.sln`-level test target**. Yet
`scripts/test.sh` contains:

```bash
#!/bin/bash
dotnet test
```

which, with no test project in the tree, is a no-op that **exits successfully** — a green signal
that verifies nothing. Any CI step calling it passes unconditionally.

**Lifecycle.** N/A.

**Invariants & enforcement.** Every silent failure mode catalogued in this document — the missing
`EventMapper` arm (§3.13), the missing `ExceptionToMessageMapper` pair (§3.16), the incomplete
projection (§3.26), the duplicate `OrderId` (§3.11), the `LastUpdate` `MinValue` (§3.9), the
unguarded `Registrations` (§3.23, §3.26) — is exactly the class of defect a test would catch and
the compiler will not. The platform patterns `[[layered-service-test-suite]]` and
`[[consumer-driven-contract-test-pair]]` describe the shape such tests take elsewhere in Pacco;
neither is realised here.

**Extension procedure.** A `tests/Pacco.Services.Deliveries.Tests` project referencing `Core` and
`Application` would cover the aggregate's guards without any infrastructure. Priority order, by
risk: (1) `Delivery` state-machine guards, (2) registration de-duplication and `HasChanged`,
(3) `EventMapper` exhaustiveness over every `(event, status)` pair, (4) `ExceptionToMessageMapper`
coverage over every `(exception, command)` pair, (5) round-trip `AsDocument`/`AsEntity`/`AsDto`.

**Failure modes.** A refactor of `Delivery` cannot be validated by anything but manual inspection.

---

## 4. Primary control flows

### 4.1 Start a delivery (AMQP — the production path)

1. The operations saga publishes `start_delivery` on the `deliveries` exchange with a `Saga`
   header and a message context.
2. Convey's subscriber (`Infrastructure/Extensions.cs:87`) deserialises `StartDelivery`.
   Its constructor substitutes a fresh GUID if `DeliveryId` is empty
   (`StartDelivery.cs:16`, §3.20).
3. `OutboxCommandHandlerDecorator` captures the inbound `MessageId` and calls
   `_outbox.HandleAsync(messageId, …)` (`:32-35`) — a replay of the same message id is suppressed.
4. `StartDeliveryHandler.HandleAsync` (`:26-39`):
   - `GetForOrderAsync(command.OrderId)` → `DeliveriesMongoRepository.cs:23-27` → Mongo
     `deliveries` collection → `AsEntity()` (**NRE here if `Registrations` is missing**, §3.26).
   - If a delivery exists and is not `Failed` → `throw DeliveryAlreadyStartedException` → step 7b.
   - `Delivery.Create(DeliveryId, OrderId, InProgress)` → buffers `DeliveryStateChanged`
     (`Delivery.cs:41`).
   - `AddRegistration(new DeliveryRegistration(Description, DateTime))` → status is `InProgress`
     so the guard passes; the set is empty so `Add` succeeds → buffers
     `DeliveryRegistrationAdded` (`Delivery.cs:58`).
   - `AddAsync` → `AsDocument()` → **insert** (a second document if this is a restart, §3.11).
   - `MapAll` → `[DeliveryStarted, RegistrationAddedToDelivery]` (`EventMapper.cs:24,32`).
   - `PublishAsync` → both events to the outbox with the forwarded `Saga` header
     (`MessageBroker.cs:76-80`).
5. The outbox processor drains to the `deliveries` exchange within ~2 s
   (`intervalMilliseconds: 2000`).
6. `orders-service` consumes `delivery_started` → `Order.SetDelivering()`. Nothing consumes
   `registration_added_to_delivery`.
7. Failure branches:
   - **7a** `InvalidAggregateIdException` — unreachable here (§3.20).
   - **7b** `DeliveryAlreadyStartedException` → `ExceptionToMessageMapper.cs:27-31` →
     `StartDeliveryRejected(DeliveryId, OrderId, message, "delivery_already_started")` published.
   - **7c** Any other exception (Mongo down, NRE from §3.26) → mapper returns `null` → **no
     rejected event, no log at `information`** → the saga stalls.

### 4.2 Start a delivery (HTTP)

Identical from step 4 onward, but: the message id is fresh so there is **no** inbox
de-duplication (§3.28); `GetHeadersToForward` returns `null` so the published events carry **no
`Saga` header** (§3.31); on success the `afterDispatch` lambda writes
`201 Created` with `Location: deliveries/{DeliveryId}` (`Program.cs:35-36`); and on any exception
`ExceptionToResponseMapper` returns **400** with `{code, reason}` — `delivery_already_started` for
7b, `error` for 7c (§3.17).

### 4.3 Add a registration

1. `AddDeliveryRegistration{DeliveryId, Description, DateTime}` arrives on either transport.
2. `AddDeliveryRegistrationHandler.cs:27` → `GetAsync(DeliveryId)`; `null` →
   `DeliveryNotFoundException` → HTTP 400 `delivery_not_found`, or
   `AddDeliveryRegistrationRejected` over AMQP (`ExceptionToMessageMapper.cs:34`) — **an event no
   declared subscriber listens for** (§3.15).
3. `delivery.AddRegistration(...)` (`Delivery.cs:46-59`):
   - status ≠ `InProgress` → `CannotAddDeliveryRegistrationException` → HTTP 400
     `cannot_add_delivery_registration`; **over AMQP, `null` → silence** (§3.16).
   - duplicate `(Description, DateTime)` → **silent return, no event** (§3.8).
   - otherwise → buffers `DeliveryRegistrationAdded`.
4. `if (delivery.HasChanged)` (`:34`) — false on the duplicate path, so no write and no publish.
5. Otherwise `UpdateAsync` (whole-document replace, §3.5) → `MapAll` →
   `RegistrationAddedToDelivery(Id, OrderId, Registration.Description)` → publish.
6. Log line `"Added a registration for the delivery with id: {DeliveryId}."` (§3.33) — emitted only
   on the non-duplicate path? **No**: `AddHandlersLogging` decorates the handler, which returns
   normally in both cases, so the "Added" line is written **even when nothing was added**.

### 4.4 Complete a delivery

1. `CompleteDelivery{DeliveryId}` → `CompleteDeliveryHandler.cs:26` → `GetAsync`; `null` →
   `DeliveryNotFoundException` → 400 / `CompleteDeliveryRejected`.
2. `delivery.Complete()` (`Delivery.cs:61-75`): from `Failed` **or** from `Completed` →
   `CannotChangeDeliveryStateException` → 400 `cannot_change_delivery_state` /
   `CompleteDeliveryRejected` (`ExceptionToMessageMapper.cs:23`). **Completing twice is an error,
   not a no-op.**
3. Otherwise `Status = Completed`, buffers `DeliveryStateChanged`.
4. `UpdateAsync` → `MapAll` → `DeliveryCompleted(Id, OrderId)` (`EventMapper.cs:26`) → publish.
5. `orders-service` consumes `delivery_completed` → `Order.Complete()`.

### 4.5 Fail a delivery

As §4.4 with `Fail(command.Reason)` (`Delivery.cs:77-92`), which additionally sets
`Notes = reason` (`:90`). `EventMapper.cs:28` emits `DeliveryFailed(Id, OrderId, Notes)`;
`orders-service` consumes it as `Order.Cancel(@event.Reason)`. The caller-supplied `reason` is
logged unredacted (§3.33) and persisted in `Notes` (§3.10).

### 4.6 Read a delivery

`GET deliveries/{deliveryId}` → route binds `GetDelivery.DeliveryId` → in-memory query dispatcher
→ `GetDeliveryHandler.cs:20` → `IMongoRepository.GetAsync(d => d.Id == query.DeliveryId)` **direct,
not through `IDeliveriesRepository`** → `document?.AsDto()`. No authorization, no ownership check,
no 404 for a miss (§3.22). `Status` serialises as an integer ordinal (§3.23); `LastUpdate` is
`0001-01-01` when there are no registrations (§3.9).

### 4.7 Startup

`Program.Main` (`Api/Program.cs:22-43`):
`AddConvey()` → `AddWebApi()` → `AddApplication()` (command/event handlers + in-memory dispatchers)
→ `AddInfrastructure()` (`Extensions.cs:47-76`: seven concrete registrations, two decorators, then
the Convey builder chain ending in `AddSecurity()`) → `Build()`;
then `UseInfrastructure()` (`:78-93`: `UseErrorHandler` → `UseSwaggerDocs` → `UseJaeger` →
`UseConvey` → `UsePublicContracts` → `UseMetrics` → `UseRabbitMq` → four `SubscribeCommand`s) →
`UseDispatcherEndpoints` (six routes) → `UseLogging()` → `UseVault()` → `RunAsync()`.

Order facts worth knowing: `UseErrorHandler` is **first**, so it wraps everything downstream;
there is **no** `UseCertificateAuthentication` between `UseMetrics` and `UseRabbitMq`, where
`customers-service` has one; and `UseVault()` is applied to the **host**, not the app pipeline, so
it participates in configuration rather than request handling.

---

## 5. Persistence & schema evolution

### 5.1 What is stored, and where

| Store | Database | Collection | Written by | Read by |
| --- | --- | --- | --- | --- |
| MongoDB | `deliveries-service` | `deliveries` | `DeliveriesMongoRepository` (§3.25) | `DeliveriesMongoRepository`, `GetDeliveryHandler` (§3.22) |
| MongoDB | `deliveries-service` | `outbox` | Convey outbox `[convey]` | Convey outbox processor |
| MongoDB | `deliveries-service` | `inbox` | Convey outbox `[convey]` | Convey outbox processor |
| Redis | — | prefix `deliveries:` | **nothing** (§3.43) | **nothing** |

The `outbox`/`inbox` collection names come from `Api/appsettings.json:105-106`; their document
shape is Convey's and **`Unverifiable — Missing Source Evidence`**.

### 5.2 The `deliveries` document shape

Derived from `DeliveryDocument.cs:8-15` and `DeliveryRegistrationDocument.cs:5-9`:

```jsonc
{
  "_id":    "<Guid>",                 // IIdentifiable<Guid>.Id
  "OrderId":"<Guid>",                 // unindexed, non-unique (§3.11)
  "Status": 0,                        // DeliveryStatus ordinal: 0=InProgress, 1=Completed, 2=Failed
  "Notes":  "<string|null>",          // failure reason only (§3.10)
  "Registrations": [                  // may be absent → NRE on read (§3.26)
    { "Description": "<string>", "DateTime": "<ISODate>" }
  ]
}
```

Property naming (PascalCase vs camelCase in BSON) is decided by the Mongo driver conventions
Convey installs; **`Unverifiable — Missing Source Evidence`**. Note what is **not** stored:
`Version` (§3.5), `LastUpdate` (§3.9), any created-at timestamp, and any audit of who acted.

### 5.3 Enum ordinals are the schema's most fragile part

`Status` is persisted as `DeliveryStatus`'s **integer ordinal** (§3.6), and the same ordinal is
what `AsDto` returns to API clients (§3.23). Consequences, in order of severity:

1. **Inserting a member anywhere but the end silently re-labels every existing document.** Adding
   `Unknown` at position 0 — the shape `customers-service` uses — would turn every stored
   `InProgress` (0) into `Unknown`, every `Completed` (1) into `InProgress`, and so on. There is no
   migration to fix it and no validation that would surface it.
2. **It is simultaneously an API break**, because clients see the integer.
3. Removing a member leaves orphaned ordinals that deserialise to an undefined enum value; C#
   permits this without error, and the `EventMapper` inner switch would then fall through to
   `return null` — a delivery whose state changes produce no events at all.

**Rule: `DeliveryStatus` is append-only.** If a reordering is ever required it needs a written
migration (§5.5) plus a coordinated client change.

### 5.4 Write semantics

Every write is a **whole-document replace** built by `AsDocument()` from a fully-rehydrated
aggregate (§3.26). There is no `$set`, no `$push`, and no conditional filter — no `Version`
(§3.5), no expected-status predicate. Therefore:

- Two concurrent `AddDeliveryRegistration` commands for the same delivery lose one registration,
  silently.
- A concurrent `CompleteDelivery` and `AddDeliveryRegistration` can produce a completed delivery
  that is missing its final registration, or an in-progress delivery that has silently discarded a
  completion — depending on write order.
- `StartDelivery` uses `AddAsync` (insert), so a restart after failure adds a document rather than
  replacing one (§3.11).

The only serialisation mechanisms present are `outbox.type: "sequential"`
(`appsettings.json:102`) and whatever consumer concurrency Convey's RabbitMQ subscriber uses;
neither is verifiable from this workspace.

### 5.5 There is no migration mechanism

Confirmed by absence: no `migrations/` directory, no `Migrate`/`Upgrade` class, no schema-version
field on `DeliveryDocument`, and `mongo.seed: false` in every profile
(`appsettings.json:98`, `appsettings.docker.json:67`) so Convey's seeder never runs. This matches
the platform-wide finding — **no Pacco service has a migration framework.**

Evolution is therefore **implicitly backward-compatible or not at all**:

| Change | Effect on existing documents | Detected by |
| --- | --- | --- |
| Add a nullable field | reads back as `default`; safe | nothing |
| Add a non-nullable value-type field | reads back as `0`/`false`/`MinValue`, indistinguishable from a real value | nothing |
| Rename a field | old data becomes invisible; new field is `default` | nothing |
| Remove a field | data remains in Mongo, orphaned | nothing |
| Reorder `DeliveryStatus` | **every document is silently re-labelled** (§5.3) | nothing |
| Rename the collection | all data orphaned | nothing |

Any change beyond "add a nullable field" needs a hand-written one-off script run against the
`deliveries` collection out-of-band, plus a decision about what to do with in-flight outbox
entries.

### 5.6 Data lifecycle and retention

- **Nothing is ever deleted.** `DeleteAsync` has no caller (§3.25); there is no TTL index, no
  archival job and no purge.
- **Nothing is ever cleaned up.** A delivery stuck `InProgress` because its saga died persists
  indefinitely and is invisible to any query the service exposes (there is no list endpoint,
  §3.22, and `LastUpdate` is not stored, §3.9).
- `outbox` entries expire after `expiry: 3600` seconds `[convey]`; `inbox` entries presumably share
  that expiry, which bounds the idempotence window (§3.28).

---

## 6. Surface → internals map

### 6.1 HTTP endpoints

| Method & path | Internal mechanism driven | Read/mutate | Failure surface |
| --- | --- | --- | --- |
| `GET /` | resolves `AppOptions` from DI and writes `Name` (`Program.cs:33`) | read-only | none |
| `GET /deliveries/{deliveryId}` | `GetDelivery` → `GetDeliveryHandler` → `IMongoRepository.GetAsync` → `AsDto` (§3.22, §3.23) | read-only | NRE→`400 error` if `Registrations` absent (§3.26); miss → empty body, not 404 |
| `POST /deliveries` | `StartDelivery` → id substitution (§3.20) → outbox decorator → `StartDeliveryHandler` → `GetForOrderAsync` guard → `Delivery.Create` + `AddRegistration` → **insert** → `DeliveryStarted` + `RegistrationAddedToDelivery` | mutating (insert) | `400 delivery_already_started`; `201 Created` + `Location` on success |
| `POST /deliveries/{deliveryId}/fail` | `FailDelivery` → `FailDeliveryHandler` → `Delivery.Fail(reason)` → `Notes` set → `DeliveryFailed` | mutating (replace) | `400 delivery_not_found`, `400 cannot_change_delivery_state` |
| `POST /deliveries/{deliveryId}/complete` | `CompleteDelivery` → `Delivery.Complete()` → `DeliveryCompleted` | mutating (replace) | as above |
| `POST /deliveries/{deliveryId}/registrations` | `AddDeliveryRegistration` → `Delivery.AddRegistration` → `HasChanged` guard → `RegistrationAddedToDelivery` | mutating (replace) **or no-op** | `400 delivery_not_found`, `400 cannot_add_delivery_registration`, **or silent success on a duplicate** |
| `GET /ping` | Convey health endpoint `[convey]`; consumed by Consul (§3.39) | read-only | — |
| `GET /metrics` | AppMetrics/Prometheus (§3.40) | read-only | — |
| `GET /docs` | Swagger UI (§3.44) | read-only | — |
| public-contracts endpoint | `UsePublicContracts<ContractAttribute>` (§3.38) — path unknown | read-only | — |

**Absent surfaces, by design or omission:** no list endpoint, no query by `OrderId`, no delete, no
update of a registration, no reopen/restart endpoint, no admin surface, no health detail beyond
`/ping`.

### 6.2 AMQP command subscriptions

| Routing key | Command type | Internal mechanism | Rejection |
| --- | --- | --- | --- |
| `start_delivery` | `StartDelivery` | §6.1 row 3, plus inbox de-duplication and `Saga` forwarding | `StartDeliveryRejected` for `DeliveryAlreadyStartedException`, `CannotChangeDeliveryStateException`, `CannotAddDeliveryRegistrationException` |
| `complete_delivery` | `CompleteDelivery` | §6.1 row 5 | `CompleteDeliveryRejected` for `DeliveryNotFoundException`, `CannotChangeDeliveryStateException` |
| `fail_delivery` | `FailDelivery` | §6.1 row 4 | `FailDeliveryRejected` for the same two |
| `add_delivery_registration` | `AddDeliveryRegistration` | §6.1 row 6 | `AddDeliveryRegistrationRejected` **only** for `DeliveryNotFoundException`; `CannotAddDeliveryRegistrationException` → **silence** (§3.16) |

### 6.3 Published events

| Routing key | Emitted when | Carries | Consumed by |
| --- | --- | --- | --- |
| `delivery_started` | `Delivery.Create` (any `InProgress` state change) | `DeliveryId`, `OrderId` | `orders-service` → `SetDelivering()` |
| `delivery_completed` | `Delivery.Complete()` | `DeliveryId`, `OrderId` | `orders-service` → `Complete()` |
| `delivery_failed` | `Delivery.Fail(reason)` | `DeliveryId`, `OrderId`, `Reason` (= `Notes`) | `orders-service` → `Cancel(Reason)` |
| `registration_added_to_delivery` | `AddRegistration` (non-duplicate) | `DeliveryId`, `OrderId`, `Message` (= `Description`) | **nobody** |
| `start_delivery_rejected` | AMQP `StartDelivery` failure | `DeliveryId`, `OrderId`, `Reason`, `Code` | operations saga (declared in `messages.json`) |
| `complete_delivery_rejected` | AMQP `CompleteDelivery` failure | `DeliveryId`, `Reason`, `Code` | operations saga |
| `fail_delivery_rejected` | AMQP `FailDelivery` failure | `DeliveryId`, `Reason`, `Code` | operations saga |
| `add_delivery_registration_rejected` | AMQP `AddDeliveryRegistration` not-found | `DeliveryId`, `Reason`, `Code` | **not declared in `messages.json`** (§3.15) |

### 6.4 Consumed events

**None.** `Infrastructure/Extensions.cs:87-90` contains four `SubscribeCommand` calls and zero
`SubscribeEvent` calls. This service holds no replica of any other aggregate — a deliberate
contrast with `customers-service`, which consumes `SignedUp` and `OrderCompleted`
(`[[customers-service]]` §3.21–§3.22), and with the platform's
`[[event-carried-reference-replica]]` pattern, which this service does not participate in.

### 6.5 Gateway routes (the only external surface)

From `Pacco.Services.Api`'s `ntrada.yml:189-204+`, `deliveries-service` module:

| Gateway route | Downstream / message | Claims required |
| --- | --- | --- |
| `GET /deliveries/{deliveryId}` | `downstream:` the service's `GET deliveries/{deliveryId}` | authentication only |
| `POST /deliveries` | `use: rabbitmq`, `resourceId: {property: deliveryId, generate: true}` | authentication only |
| `POST /deliveries/{deliveryId}/fail` | `use: rabbitmq` | authentication only |
| `POST /deliveries/{deliveryId}/complete` | `use: rabbitmq` | authentication only |
| `POST /deliveries/{deliveryId}/registrations` | `use: rabbitmq` | authentication only |

**No deliveries route requires an `admin` claim** — unlike four of six `customers-service` routes.
The gateway generates the delivery id (`generate: true`), which is why the `Guid.Empty` substitution
in `StartDelivery` (§3.20) rarely fires in practice. Writes go through the broker, so the gateway
returns immediately (`[[acknowledge-then-notify-completion]]`) and the caller never sees a domain
error — the rejected event is the only failure signal, and it goes to the saga, not the user.

### 6.6 Mechanisms with no surface at all

| Mechanism | Why it has no surface |
| --- | --- |
| `IDeliveriesRepository.DeleteAsync` | no caller (§3.25) |
| `IAggregateRoot.IncrementVersion` | no caller (§3.5) |
| `IDateTimeProvider` | registered, never injected (§3.43) |
| Redis | registered, never used (§3.43) |
| `AddSecurity()` primitives | registered, never consumed (§3.37) |
| `OutboxEventHandlerDecorator<>` | no `IEventHandler<>` exists to decorate (§3.28) |
| Vault PKI certificate | issued, never presented (§3.41) |
| `jwt` config block | inert (§3.37) |
| `httpClient` + Fabio | configured, `services: {}` — no outbound call (§3.39) |
| `RegistrationAddedToDelivery` | published, never consumed (§3.14) |

---

## 7. Change/extension guide

### 7.1 The registration map — what is automatic and what is not

| Thing you add | Auto-discovered? | Manual registrations required |
| --- | --- | --- |
| Command handler | **yes** — `AddCommandHandlers()` scans (`Application/Extensions.cs:11`) | — |
| Query handler | **yes** — `AddQueryHandlers()` (`Infrastructure/Extensions.cs:60`) | — |
| Command (HTTP reachable) | no | `.Post<T>("route")` in `Api/Program.cs` |
| Command (AMQP reachable) | no | `.SubscribeCommand<T>()` in `Infrastructure/Extensions.cs` |
| Query (reachable) | no | `.Get<TQuery,TResult>("route")` in `Api/Program.cs` |
| Domain event → integration event | no | a `case` in `EventMapper.Map` |
| Exception → HTTP status | partially — base-type arms catch everything as 400 | an arm **before** `DomainException`/`AppException` for any other status |
| Exception → rejected event | no | an `(exception, command)` arm in `ExceptionToMessageMapper` |
| Public contract visibility | no | `[Contract]` on the type |
| Log line for a command | no | an entry in `MessageToLogTemplateMapper` |
| Repository method | **compiler-enforced** | interface + implementation |
| Mongo field | no | `DeliveryDocument` + `AsDocument` + `AsEntity` (+ `AsDto`) |
| External reachability | no | a route in the gateway's `ntrada.yml` |
| Saga participation | no | the snake-case name in operations' `messages.json` |

**Only two rows fail loudly** (handler discovery is a no-op if you get it wrong, and the repository
row is a compile error). Everything else in this table fails **silently at runtime**.

### 7.2 Adding a new command end-to-end

1. `Application/Commands/YourCommand.cs` — `[Contract]`, `ICommand`, immutable, constructor-only.
2. `Application/Commands/Handlers/YourCommandHandler.cs` — `internal sealed`, inject
   `(IDeliveriesRepository, IMessageBroker, IEventMapper)`, follow the §3.21 skeleton, include the
   `HasChanged` guard if the domain method can no-op.
3. `Core/Entities/Delivery.cs` — the mutating method, with explicit guards that throw a
   `DomainException` subclass, and `AddEvent(...)` at the end.
4. `Core/Events/YourDomainEvent.cs` if the mutation is new in kind.
5. `Infrastructure/Services/EventMapper.cs` — the `case`. **Do not skip.**
6. `Application/Events/YourIntegrationEvent.cs` — `[Contract]`, `IEvent`.
7. `Application/Events/Rejected/YourCommandRejected.cs` — `[Contract]`, `IRejectedEvent`.
8. `Infrastructure/Exceptions/ExceptionToMessageMapper.cs` — one arm per exception the handler can
   throw, keyed on the new command type.
9. `Infrastructure/Logging/MessageToLogTemplateMapper.cs` — an `After` template.
10. `Api/Program.cs` — the route, if HTTP-reachable.
11. `Infrastructure/Extensions.cs` — `.SubscribeCommand<YourCommand>()`, if AMQP-reachable.
12. `Pacco.Services.Api/ntrada.yml` — the gateway route (separate repository).
13. `Pacco.Services.Operations/.../messages.json` — the message names (separate repository).

### 7.3 Adding a field to `Delivery`

`Core/Entities/Delivery.cs` (property with `protected set` + assignment in the mutating method) →
`Infrastructure/Mongo/Documents/DeliveryDocument.cs` → `AsDocument` → `AsEntity` → `AsDto` →
`Application/DTO/DeliveryDto.cs`. Five files, none of them checked. Verify by writing a delivery
and reading it back; there is no test to lean on (§3.46).

### 7.4 Adding a `DeliveryStatus` member

**Append only** (§5.3). Then: a mutating method with guards, an `EventMapper` inner-switch case, an
integration event, a rejected event, mapper arms, a command, both transport registrations, the
gateway route and the operations manifest. Also re-check `Complete()`/`Fail()` — they enumerate
terminal states explicitly rather than using a helper, so a new terminal state must be added to
**both** methods' guards (`Delivery.cs:63-71`, `:79-87`).

### 7.5 Fixing the duplicate-`OrderId` behaviour

The smallest correct change (§3.11): give `Delivery` a `Restart()` method that transitions
`Failed → InProgress` with its own guard and emits `DeliveryStateChanged`; have
`StartDeliveryHandler` call `UpdateAsync` on the existing failed aggregate instead of
`AddAsync`-ing a new one. Consequences to accept: `DeliveryId` in the command would be ignored on
the restart path (the existing id wins), so the HTTP `Location` header would be wrong; and
`orders-service` would see a second `DeliveryStarted` for a `DeliveryId` it already knows.
Alternatively, add a unique index on `OrderId` and handle the duplicate-key error — but that
requires an out-of-band index creation (§5.5) and a new exception arm in both mappers.

### 7.6 Adding optimistic concurrency

See §3.5. Requires a `Version` document field, `IncrementVersion` calls in every handler, and a
conditional update primitive whose availability in Convey's `IMongoRepository` this workspace
cannot confirm. If it is unavailable, the fallback is to bypass `IMongoRepository` and inject
`IMongoDatabase` directly in `DeliveriesMongoRepository`.

### 7.7 Adding authorization

See §3.37. Two independent decisions: (a) do you trust the gateway's `Correlation-Context` header,
or add real authentication in-process; (b) what does "owner" mean for a delivery, given the
aggregate has no customer reference. Until (b) is answered, the only enforceable rule is
role-based — e.g. require `IsAdmin` for `fail`/`complete` — which can be done entirely in the
gateway's `ntrada.yml` with a `claims:` block, with **no change to this service**. That is the
cheapest first step.

### 7.8 Making silent failures loud

Ranked by effort-to-value:

1. Change `MessageBroker.cs:75`'s `LogTrace` to `LogWarning` **in the null-event branch**
   (`:69-72`) — a one-line change that converts §3.13's invisible event loss into an alert.
2. Add a `_ =>` arm to `ExceptionToMessageMapper` that publishes a generic rejection, or at minimum
   logs — currently `null` at both switch levels is indistinguishable from success.
3. Add null guards to `AsEntity`/`AsDto`'s `document.Registrations.Select(...)` (§3.26) —
   `?? Enumerable.Empty<…>()`.
4. Change `LastUpdate` to `Registrations.Any() ? … : (DateTime?)null` (§3.9).
5. Add an `OnError` template to `MessageToLogTemplateMapper` (§3.33).

None of these changes any contract.

### 7.9 What to leave alone

- **`DeliveryStatus` ordering** (§5.3).
- **The `[Contract]` set** — the 12 marked types are the platform's published vocabulary.
- **`rabbitMq.exchange.name`, `queue.template`, `conventionsCasing`, `context.header`,
  `spanContextHeader`** — all cross-service agreements with no negotiation mechanism.
- **`Delivery.AddRegistration`'s silent duplicate return** — changing it to throw would turn every
  saga retry into a rejected event.
- **`mongo.database`, the `deliveries` collection name** — no migration path (§5.5).

### 7.10 The maintenance contract

**Any later phase that changes this component's internals must update this document in the same
change.** Concretely: a new command, a new status, a new field on `Delivery` or
`DeliveryDocument`, a new subscription, a change to either exception mapper, or a change to the
`EventMapper` switch all require edits to §2, the relevant §3 subsection, §6 and — if storage is
involved — §5.

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

| # | Assumption | Basis | If wrong |
| --- | --- | --- | --- |
| A-1 | Convey's snake-case convention maps `DeliveryStarted` → `delivery_started` etc. | `conventionsCasing: "snakeCase"` (`appsettings.json:113`) **corroborated** by the exact names in `Pacco.Services.Operations`' `messages.json` | routing keys in §3.14/§6.3 are wrong and no message reaches a consumer |
| A-2 | `SubscribeCommand<T>()` acks or dead-letters a message whose exception maps to `null`, rather than requeueing forever | Convey convention; **no source** | §3.16's silent failures become infinite redelivery loops instead of drops |
| A-3 | `UseDispatcherEndpoints` binds route-template values into command constructor parameters by name | the routes are written as `deliveries/{deliveryId}` and commands have matching `DeliveryId`; **no source** | HTTP writes never address the intended delivery |
| A-4 | `AddHandlersLogging()` discovers the `internal sealed MessageToLogTemplateMapper` by assembly scan | it is never referenced by name in `Extensions.cs` | no command produces a log line at all |
| A-5 | `outbox.expiry: 3600` bounds the inbox de-duplication window | the key is shared by both collections in one config block | replay protection lasts longer or shorter than assumed |
| A-6 | The container is deployed with `ASPNETCORE_ENVIRONMENT=docker` as baked in | `Dockerfile:10` | the service starts with the baseline profile and tries `localhost` for Mongo, Rabbit, Consul and Vault |
| A-7 | Nothing outside the workspace consumes `registration_added_to_delivery` | grep across all 14 clones | removing it would break an unknown consumer |

### 8.2 Blockers

| # | Blocker | Effect on this document |
| --- | --- | --- |
| B-1 | **Convey 0.4.\* has no source in the workspace.** | Every `[convey]` mechanism — dispatchers, `IMongoRepository` semantics, outbox internals, subscriber ack policy, endpoint binding, contract-endpoint path, default success status codes — is described by its call site and configuration only. §3.5, §3.16, §3.28, §3.36 and §3.38 each carry an explicit `Unverifiable` marker. |
| B-2 | **No test project.** | No behaviour in this document is confirmed by executing it; every claim is static reading. |
| B-3 | **No runtime access.** | Cannot inspect the live `deliveries` collection to see whether duplicate-`OrderId` documents actually exist (§3.11), nor whether any document lacks `Registrations` (§3.26). |
| B-4 | **`Pacco.Services.Api` and `Pacco.Services.Operations` are separate repositories.** | Gateway claims (§6.5) and saga message declarations (§3.15) are read from those clones and are accurate as of the same base ref, but this document does not own them. |

### 8.3 Open questions

| # | Question | Why it matters | Where to answer it |
| --- | --- | --- | --- |
| Q-1 | Is the failed-restart path (`StartDelivery` on a `Failed` delivery inserting a **second** document) intentional? | It makes `GetForOrderAsync` non-deterministic and the duplicate-start guard unreliable (§3.11). | product owner + `StartDeliveryHandler.cs:28-36` |
| Q-2 | Should `CompleteDelivery` on an already-`Completed` delivery be an error or an idempotent no-op? | It is currently an error (§3.6), so a saga retry after a lost ack produces a rejected event rather than success. | operations saga's retry policy |
| Q-3 | Should `POST /deliveries/{id}/fail|complete` require an `admin` claim? | Today any authenticated user can cancel any order by failing its delivery (§3.37, §6.5). | `ntrada.yml` — a gateway-only fix |
| Q-4 | Is `registration_added_to_delivery` intended to have a consumer? | It is published on every registration and consumed by nobody (§3.14). Either a consumer is missing or the event is dead weight. | platform roadmap |
| Q-5 | Why is `add_delivery_registration_rejected` absent from the operations `messages.json`? | The one rejected event the mapper can produce for registrations is undeliverable to the saga (§3.15). | operations repository |
| Q-6 | Does `AddHandlersLogging` emit the "Added a registration" line even when the duplicate guard suppressed the write? | The handler returns normally in both cases, so the log almost certainly lies about the duplicate path (§4.3 step 6) — but the decorator's exact trigger is `[convey]`. | Convey source, or an experiment |
| Q-7 | Should timestamps be service-stamped rather than caller-supplied? | `IDateTimeProvider` exists for exactly this and is unused (§3.43); today a client can backdate a registration arbitrarily. | domain owner |
| Q-8 | Is a delivery ever expected to reach a terminal state without a saga? | Nothing sweeps stale `InProgress` deliveries and nothing can even find them (§3.6, §5.6). | operations + SRE |
| Q-9 | Is the missing `security` section deliberate, or was certificate authentication meant to be copied from `customers-service`? | It is the single largest asymmetry between the two services in this batch (§3.37). | ADR / platform security owner |

### 8.4 Related patterns

| Pattern | How this component realises it |
| --- | --- |
| `[[inward-dependency-service-skeleton]]` | Four projects, `Core` references nothing; §3.18, §3.24 |
| `[[dispatcher-bound-cqrs-endpoints]]` | `UseDispatcherEndpoints` with 6 routes, no controllers; §3.36 |
| `[[dual-mode-edge-write]]` | 4 commands on both HTTP and AMQP with divergent failure semantics; §1.3, §3.35 |
| `[[aggregate-buffered-domain-events]]` | `AggregateRoot._events`, drained by the handler; §3.3 |
| `[[database-per-service-with-document-mapping]]` | `deliveries-service` DB, hand-written `AsDocument`/`AsEntity`/`AsDto`; §3.26, §5 |
| `[[transactional-outbox-handler-decorator]]` | `OutboxCommandHandlerDecorator` + Mongo `inbox`/`outbox`; §3.28 |
| `[[service-owned-topic-exchange-messaging]]` | self-declared durable topic exchange `deliveries`, per-consumer queue template; §3.34 |
| `[[rejected-event-failure-contract]]` | 4 `IRejectedEvent` types via `ExceptionToMessageMapper`, with two coverage gaps; §3.15, §3.16 |
| `[[declarative-message-manifest-subscription]]` | operations' `messages.json` names this service's commands and events; §3.15 |
| `[[declarative-configuration-driven-api-gateway]]` | all external access via `ntrada.yml`; §6.5 |
| `[[acknowledge-then-notify-completion]]` | gateway writes go `use: rabbitmq`; the caller never sees a domain error; §6.5 |
| `[[saga-process-manager]]` | every production write originates from the operations saga; §1.1, §3.31 |
| `[[edge-enforced-authentication-with-identity-binding]]` | enforced **only** at the gateway; this service authenticates nothing; §3.37 |
| `[[transport-agnostic-caller-context]]` | `IAppContext`/`IIdentityContext` fully built and entirely unconsumed; §3.30 |
| `[[correlation-and-span-propagation]]` | `Correlation-Context` header, `message_context`, `span_context`, `Saga`; §3.31, §3.32 |
| `[[structured-logging-with-property-redaction]]` | 12-entry `excludeProperties`, per-command templates; §3.33 |
| `[[registry-mediated-discovery-and-routing]]` | Consul + Fabio, `deliveries-service` on port 5003/80; §3.39 |
| `[[vault-issued-dynamic-credentials-and-service-pki]]` | KV + PKI + dynamic Mongo lease, all disabled in every runnable profile; §3.41 |
| `[[composable-per-concern-environment-stacks]]` | `local` / `docker` / empty `development` overlays; §3.42 |
| `[[independent-per-repository-release]]` | own Dockerfile and `dockerize.sh`, own image tag; §3.45 |
| `[[framework-supplied-platform-conventions]]` | every `[convey]` mechanism in this document; B-1 |
| `[[narrow-synchronous-point-read]]` | **not realised** — one point read exists but no other service calls it; §1.4, §6.5 |
| `[[event-carried-reference-replica]]` | **not realised** — zero event subscriptions, no replica; §6.4 |
| `[[prefix-partitioned-shared-cache]]` | **not realised** — `deliveries:` prefix reserved, never written; §3.43 |
| `[[layered-service-test-suite]]` | **not realised** — no test project; §3.46 |
| `[[consumer-driven-contract-test-pair]]` | **not realised** — no contract tests; §3.14, §3.46 |
| `[[real-time-push-with-shared-backplane]]` | **not realised** — no SignalR; delivery progress is not pushed to clients |

