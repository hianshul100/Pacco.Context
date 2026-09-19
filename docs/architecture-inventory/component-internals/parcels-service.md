# Component internals — `parcels-service`

| | |
| --- | --- |
| **Component** | `parcels-service` |
| **Source repository** | `hianshul100_Pacco.Services.Parcels` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository) |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 5 of 7 |
| **Status** | New artifact — no prior `component-internals/parcels-service.md` existed in this repository at the time of writing. `baselines/service-summaries.md` §2.3 and `repo-summary/Pacco.Services.Parcels.md` remain valid and are **complemented**, not replaced. |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full
> (`src/Pacco.Services.Parcels.{Core,Application,Infrastructure,Api}`) plus one test project,
> `tests/Pacco.Services.Parcels.PactProviderTests`, which is **not referenced by
> `Pacco.Services.Parcels.sln`** and therefore never runs (§3.38). `Convey 0.4.*` — which supplies the
> CQRS dispatchers, the Mongo repository, the RabbitMQ client, the outbox and the WebApi endpoint
> mapping — is a NuGet reference with **no source in this workspace**; mechanisms it owns are marked
> `[convey]`. The producer of every event this service consumes is modelled in
> `component-internals/orders-service.md` and `component-internals/customers-service.md`; the upstream
> half of its HTTP surface is in `component-internals/api-gateway.md`.

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

`parcels-service` owns the **catalogue of physical items a customer wants shipped**, and one piece of
mutable relationship state per item: which order, if any, that parcel currently belongs to. It is the
smallest write model on the platform that still has real domain content, and its shape is
instructive precisely because of what it omits — there is no aggregate root, no domain event buffer
and no event mapper here, which is a deliberate divergence from every sibling service (§3.2).

Three properties characterise it:

1. **It is a pure follower on the order relationship.** `OrderId` is never set by a command. It is set
   and cleared *only* by consuming four events published by `orders-service` (§3.23). The two commands
   this service accepts — add and delete — cannot attach or detach a parcel from an order.
2. **It has no synchronous dependencies at all.** The `httpClient.services` map in
   `…Api/appsettings.json` is **empty**, and no client class exists under `…Infrastructure/Services/`
   beyond the message broker (§3.34). Every interaction with the rest of the platform is
   message-based.
3. **It holds the platform's only geometric calculation.** `ParcelsService.CalculateVolume`
   (`…Core/Services/ParcelsService.cs:20-27`) converts a size band into a cubic-metre figure, and is
   the sole reason `Core` has a `Services` folder at all.

| Responsibility | Where it lives |
| --- | --- |
| Model a parcel as id + customer + variant + size + name + description + creation time + optional order | `src/Pacco.Services.Parcels.Core/Entities/Parcel.cs:8-16` |
| Validate name and description at construction | `…Core/Entities/Parcel.cs:25-28` |
| Attach and detach a parcel from an order | `…Core/Entities/Parcel.cs:33-41` |
| Compute the total volume of a set of parcels | `…Core/Services/ParcelsService.cs:20-27` |
| Accept 2 commands over HTTP **and** over AMQP | `…Api/Program.cs:38-45`; `…Infrastructure/Extensions.cs:89-90` |
| Answer 3 queries from a Mongo read model | `…Infrastructure/Mongo/Queries/Handlers/*.cs` |
| React to 5 external events from two other services | `…Application/Events/External/*.cs` |
| Publish 2 integration events and 2 rejected events | `…Application/Events/**` |
| Translate exceptions into HTTP responses **and** into rejected events | `…Infrastructure/Exceptions/ExceptionTo{Response,Message}Mapper.cs` |
| Persist parcels in MongoDB (`parcels` collection, `parcels-service` DB) and a customer-id replica in `customers` | `…Infrastructure/Mongo/Repositories/*.cs`; `…Infrastructure/Extensions.cs:74-75` |
| Publish via a Mongo-backed outbox, de-duplicate inbound messages via an inbox | `…Infrastructure/Decorators/Outbox*Decorator.cs`; the `outbox` section of `…Api/appsettings.json` |
| Register with Consul, emit Jaeger spans, expose Prometheus metrics, serve Swagger | `…Infrastructure/Extensions.cs:64-76` |
| Fetch settings, a PKI certificate and dynamic Mongo credentials from Vault | `…Api/Program.cs:47`; the `vault` section of `…Api/appsettings.json` |
| Serve as the **provider** half of a Pactify contract with `orders-service` | `tests/Pacco.Services.Parcels.PactProviderTests/PACT/ParcelsApiPactProviderTests.cs` |

### 1.2 What this component explicitly is **not**

- **Not an aggregate-based domain.** There is no `AggregateRoot.cs` and no `AggregateId.cs` in
  `…Core/Entities/` — unlike `orders-service`, `deliveries-service` and `availability-service`, which
  all have both. `Parcel` is a plain class with private setters (`…Core/Entities/Parcel.cs:6-16`), and
  **no domain event is raised anywhere in this service**. Integration events are constructed by hand
  in the handlers (§3.19).
- **Not the owner of the parcel↔order relationship.** It stores `OrderId` but cannot change it by
  command; `orders-service` is the authority and this service reacts (§3.23).
- **Not the owner of customers.** `Core/Entities/Customer.cs` is an id and nothing else; the local
  `customers` collection is an existence set fed once by `customer_created` (§3.6).
- **Not an authenticator.** No JWT validation, no `[Authorize]`, no `UseAuthentication`. The `jwt`
  configuration block and the committed `certs/localhost.cer` are read by nothing in `src/` (§3.35).
  Identity arrives as a JSON header from the gateway and is believed (§3.27).
- **Not a consistent authorizer.** `DeleteParcel` checks ownership; **`AddParcel` does not check
  anything** — it takes `CustomerId` from the request payload (§3.17). And the check that does exist
  is gated on `identity.IsAuthenticated &&`, so unauthenticated callers pass (§3.17).
- **Not a caller of anything.** No outbound HTTP (§3.34). This makes it the least coupled service in
  the workspace and the easiest to reason about operationally.
- **Not tested.** The one test project is excluded from the solution file (§3.38), and its Mongo
  fixture's initialisation is commented out.

### 1.3 The dual-transport boundary

Both commands have two entry points with different failure semantics:

| | HTTP path | AMQP path |
| --- | --- | --- |
| Entry | `UseDispatcherEndpoints` (`…Api/Program.cs:38-45`) | `SubscribeCommand<T>()` (`…Infrastructure/Extensions.cs:89-90`) |
| Identity source | `Correlation-Context` header (`…Infrastructure/Extensions.cs:100-103`) | broker message context (`…Infrastructure/Contexts/AppContextFactory.cs`) |
| De-duplication | **none effective** — a fresh `MessageId` GUID per request (§3.25) | inbox, keyed on the broker `MessageId` |
| Failure surfaced as | HTTP **400**, always (§3.26) | a **rejected event** — or nothing (§3.20) |
| Caller learns outcome | synchronously | only by polling `operations-service` |

The mapper here is only 27 lines, and three of its eight arms are wrong or lossy:

- **`UnauthorizedParcelAccessException` maps to `AddParcelRejected`**
  (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:24`) although it is thrown **only** by
  `DeleteParcelHandler` (`…Application/Commands/Handlers/DeleteParcelHandler.cs:36`). A failed delete
  is reported to the client as a failed add.
- **`ParcelNotFoundException` produces `DeleteParcelRejected(Guid.Empty, …)`** (`:21-23`) — the parcel
  id is discarded, so a client with several deletes in flight cannot tell which one failed.
- **`InvalidParcelNameException` and `InvalidParcelDescriptionException` map to `AddParcelRejected`**
  (`:19-20`), which carries **no parcel id at all**
  (`…Application/Events/Rejected/AddParcelRejected.cs`) — the same ambiguity, by design of the event
  rather than by mapper error.

### 1.4 Position in the platform

| Direction | Counterpart | Mechanism | Evidence |
| --- | --- | --- | --- |
| Inbound (sync) | `api-gateway` | HTTP, 6 routes, all `auth: true` | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, parcels block |
| Inbound (sync) | `orders-service` | `GET /parcels/{parcelId}` before adding a parcel to an order | `component-internals/orders-service.md` §3.34 |
| Inbound (async) | `api-gateway` async profile | AMQP to exchange `parcels`, routing keys `add_parcel`, `delete_parcel` | `…/ntrada-async.yml`, parcels block |
| Inbound (async) | `orders-service` | `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order` on exchange `orders` | `…Application/Events/External/*.cs` |
| Inbound (async) | `customers-service` | `customer_created` on exchange `customers` | `…Application/Events/External/CustomerCreated.cs` |
| Outbound (async) | `operations-service` | commands and events by manifest | `…Operations/src/Pacco.Services.Operations.Api/messages.json`, `parcels-service` block |
| Outbound (async) | **nobody** | `parcel_added` and `parcel_deleted` have no `SubscribeEvent<>` anywhere in the workspace (§3.19) | verified across all clones |
| Contract test | `orders-service` | Pactify file-based pact, `orders` → `parcels` | `tests/…/PACT/ParcelsApiPactProviderTests.cs` |

Note the asymmetry with `orders-service`: **`orders-service` declares its `ParcelDeleted` subscription
on the `deliveries` exchange** (`component-internals/orders-service.md` §3.33), while this service
publishes `parcel_deleted` on `parcels`. The binding never matches, so nothing in the platform reacts
to a parcel being deleted.

---

## 2. Core concepts (exhaustive)

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | `Parcel` — the entity | `…Core/Entities/Parcel.cs` | §3.1 |
| 2 | The absent aggregate root — no `AggregateRoot`, no `AggregateId`, no domain events | `…Core/Entities/` (by absence) | §3.2 |
| 3 | `Variant` — the five parcel kinds | `…Core/Entities/Variant.cs` | §3.3 |
| 4 | `Size` — the six size bands | `…Core/Entities/Size.cs` | §3.4 |
| 5 | `ParcelsService.CalculateVolume` — the side-length table and the cube law | `…Core/Services/ParcelsService.cs` | §3.5 |
| 6 | `Customer` — the id-only reference replica | `…Core/Entities/Customer.cs` | §3.6 |
| 7 | `AddedToOrder` — a derived property stored as a column | `Parcel.cs:16`; `ParcelDocument.cs:17` | §3.7 |
| 8 | `AddToOrder` / `DeleteFromOrder` — the unconditional mutators | `…Core/Entities/Parcel.cs:33-41` | §3.8 |
| 9 | Name and description validation at construction | `…Core/Entities/Parcel.cs:25-28` | §3.9 |
| 10 | Exception hierarchy and the ten error codes | `…Core/Exceptions`, `…Application/Exceptions` | §3.10 |
| 11 | `IParcelRepository` — and the single-result `GetByOrderAsync` | `…Core/Repositories/IParcelRepository.cs` | §3.11 |
| 12 | `ParcelDocument` and the three mapping functions | `…Infrastructure/Mongo/Documents/*.cs` | §3.12 |
| 13 | Enum persistence — an unresolved int-vs-string question | `ParcelDocument.cs:11-12`; `Documents/Extensions.cs:49-50` | §3.13 |
| 14 | Absence of optimistic concurrency | `…Mongo/Repositories/ParcelMongoRepository.cs:34` | §3.14 |
| 15 | `AddParcel` — the command that generates its own id and carries strings for enums | `…Application/Commands/AddParcel.cs` | §3.15 |
| 16 | `AddParcelHandler` — parse, replica check, publish | `…Application/Commands/Handlers/AddParcelHandler.cs` | §3.16 |
| 17 | The identity guard: present on delete, **absent on add** | `DeleteParcelHandler.cs:33-37` | §3.17 |
| 18 | `DeleteParcelHandler` — three guards in a fixed order | `…Application/Commands/Handlers/DeleteParcelHandler.cs` | §3.18 |
| 19 | The two integration events, and their absent subscribers | `…Application/Events/*.cs` | §3.19 |
| 20 | Rejected events and `ExceptionToMessageMapper` — three defects | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` | §3.20 |
| 21 | No `IEventMapper` — publication is hand-rolled | `…Infrastructure/Services/` (by absence) | §3.21 |
| 22 | `MessageBroker` — the publish pipeline | `…Infrastructure/Services/MessageBroker.cs` | §3.22 |
| 23 | External event subscription — five events, four silent-miss handlers | `…Application/Events/External/**` | §3.23 |
| 24 | `GetByOrderAsync` releases **one** parcel per order event | `ParcelMongoRepository.cs:26-31` | §3.24 |
| 25 | Transactional outbox and inbox | `…Infrastructure/Decorators/Outbox*.cs` | §3.25 |
| 26 | `ExceptionToResponseMapper` — everything is HTTP 400 | `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` | §3.26 |
| 27 | `IAppContext` / `IIdentityContext` and the `Correlation-Context` header | `…Infrastructure/Contexts/*.cs`; `Extensions.cs:100-103` | §3.27 |
| 28 | `GetParcel` and `GetParcels` — one unguarded, one silently empty | `…Mongo/Queries/Handlers/GetParcel{,s}Handler.cs` | §3.28 |
| 29 | `GetParcelsVolume` — the only query with domain logic behind it | `…Mongo/Queries/Handlers/GetParcelsVolumeHandler.cs` | §3.29 |
| 30 | Route ordering — `parcels/volume` must precede `parcels/{parcelId}` | `…Api/Program.cs:40-41` | §3.30 |
| 31 | `[Contract]` and `UsePublicContracts` | `…Application/ContractAttribute.cs`; `Extensions.cs:86` | §3.31 |
| 32 | Queue naming and message conventions | the `rabbitMq` section of `…Api/appsettings.json` | §3.32 |
| 33 | `MessageToLogTemplateMapper` and log redaction | `…Infrastructure/Logging/MessageToLogTemplateMapper.cs`; the `logger` section | §3.33 |
| 34 | Consul, Fabio and the **empty** `httpClient.services` map | `…Infrastructure/Extensions.cs:63-65`; the `httpClient` section | §3.34 |
| 35 | Inert JWT configuration and the committed `localhost.cer` | the `jwt` section; `…Api/certs/localhost.cer` | §3.35 |
| 36 | Vault — settings, PKI and dynamic Mongo credentials | the `vault` section; `…Api/Program.cs:47` | §3.36 |
| 37 | Environment layering — and the `tests` vs `test` mismatch | `…Api/appsettings.*.json`; `scripts/*.sh` | §3.37 |
| 38 | The PACT provider test that never runs, and its Mongo fixture | `tests/…PactProviderTests/**`; `Pacco.Services.Parcels.sln` | §3.38 |
| 39 | Redis and `IDateTimeProvider` | `Extensions.cs:70`; `…Infrastructure/Services/DateTimeProvider.cs` | §3.39 |
| 40 | Deployment identity — port, image, environment names | `Dockerfile`, `scripts/`, `.travis.yml` | §3.40 |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement**, **Extension procedure**, **Failure modes**.

### 3.1 `Parcel` — the entity

**Definition.** `public class Parcel` (`…Core/Entities/Parcel.cs:6`) — note the absence of a base
class. Eight members: `Id`, `CustomerId`, `Variant`, `Size`, `Name`, `Description`, `CreatedAt`,
`OrderId`, all with private setters (`:8-15`), plus the computed `AddedToOrder => OrderId.HasValue`
(`:16`).

**Representation & storage.** One document per parcel in the `parcels` collection of the
`parcels-service` database, registered as `AddMongoRepository<ParcelDocument, Guid>("parcels")`
(`…Infrastructure/Extensions.cs:75`) `[convey]`. The document is flat — nothing is embedded, and
nothing references another collection (§3.12).

**Lifecycle.** Created by the single public constructor
`Parcel(Guid id, Guid customerId, Variant variant, Size size, string name, string description, DateTime createdAt, Guid? orderId = null)`
(`:18-31`), which is called from exactly two places: `AddParcelHandler.cs:46` (a new parcel) and
`…Mongo/Documents/Extensions.cs:26-28` (rehydration). Mutated only by `AddToOrder`/`DeleteFromOrder`
(§3.8). Hard-deleted by `DeleteParcelHandler.cs:44`; no tombstone, no soft-delete flag.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent? |
| --- | --- | --- |
| `Name` is not null/whitespace | ctor `:25` | **Loud** — `InvalidParcelNameException` |
| `Description` is not null/whitespace | ctor `:26-28` | **Loud** — `InvalidParcelDescriptionException` |
| `Variant`/`Size` are valid members | **not here** — the ctor takes enums, so parsing is the handler's job (`AddParcelHandler.cs:31-39`) | Loud, in the handler |
| The customer exists | **not here** — `AddParcelHandler.cs:41-44` | Loud |
| A parcel on an order cannot be deleted | **not here** — `DeleteParcelHandler.cs:39-42` | Loud |
| `Id` is not `Guid.Empty` | **nowhere** — there is no `AggregateId` in this service (§3.2) | **Absent** |
| `CustomerId` is not `Guid.Empty` | **nowhere** | **Absent** |
| `CreatedAt` is meaningful | **nowhere** — whatever the caller passes | **Absent** |

The entity therefore enforces exactly two of its eight fields. Everything else is guarded — where it
is guarded at all — one layer out, which means any future code path that constructs a `Parcel`
directly inherits almost no protection.

**Extension procedure.** Add the property with a private setter, add the constructor parameter (at the
end, with a default, to avoid breaking the two call sites), add the property to `ParcelDocument`, and
extend `AsEntity`/`AsDocument` and — if client-visible — `AsDto` and `ParcelDto`. Four to six
touch-points, none of them checked by the compiler (§3.12).

**Failure modes.** `Id` and `CustomerId` accept `Guid.Empty` silently, so a malformed command produces
a stored parcel owned by nobody, which no ownership check can ever match (§3.17). `Description` being
required is a stricter rule than `orders-service` applies to its own parcel snapshot, which validates
nothing (`component-internals/orders-service.md` §3.6) — the two views of the same parcel therefore
have different validity rules.

### 3.2 The absent aggregate root

**Definition.** This is a concept by **absence**, and it is the structural fact that most distinguishes
this service from its siblings. `…Core/Entities/` contains `Parcel.cs`, `Customer.cs`, `Size.cs` and
`Variant.cs` — and **no `AggregateRoot.cs` and no `AggregateId.cs`**, both of which exist in
`orders-service`, `deliveries-service` and `availability-service`.

**Representation & storage.** Nothing is buffered and nothing is versioned. There is no `Events`
collection on the entity and no `Version` field.

**Lifecycle.** The consequence flows all the way to the wire: because there are no domain events,
there is no `IEventMapper` implementation in this service (§3.21), and integration events are
constructed by hand inside the two command handlers — `new ParcelAdded(command.ParcelId)`
(`AddParcelHandler.cs:49`) and `new ParcelDeleted(command.ParcelId)` (`DeleteParcelHandler.cs:45`).

**Invariants & enforcement.** The trade-off is worth stating plainly, because it is invisible from any
one file:

| | `orders-service` (aggregate) | `parcels-service` (plain entity) |
| --- | --- | --- |
| What publishes | `EventMapper` reads the buffer | the handler writes the event literally |
| Silent-loss channel | a missing mapper arm returns `null` and the event vanishes | none — you cannot forget to map what you wrote by hand |
| Risk | events raised but never mapped | **state changed without any event at all** |

The second risk is live here: the four external event handlers each call `UpdateAsync` and publish
**nothing** (§3.23). A parcel's `OrderId` changing is not observable to anyone. That is defensible —
`orders-service` already knows, since it caused it — but it means this service's state has no audit
trail whatsoever.

**Extension procedure.** Do **not** introduce an `AggregateRoot` here for symmetry alone; the current
shape is simpler and has fewer failure modes. If a future feature needs parcel state changes to be
observable (a customer notification, an analytics feed), publish explicitly from the handler that
mutates — the same way `AddParcelHandler` already does — rather than building an event buffer.

**Failure modes.** The absence is easy to misread as an oversight. A maintainer arriving from
`orders-service` will look for `order.Events`/`MapAll` and find nothing; the correct mental model is
"handlers publish, entities do not".

### 3.3 `Variant` — the five parcel kinds

**Definition.** `public enum Variant { Standard, Chemistry, Weapon, Animal, Organ }`
(`…Core/Entities/Variant.cs`) — ordinals 0–4.

**Representation & storage.** Held on `ParcelDocument.Variant`
(`…Mongo/Documents/ParcelDocument.cs:11`) with no `[BsonRepresentation]` attribute — so whether the
ordinal or the member name reaches disk depends on a globally registered convention, which is
**unresolved**; §3.13 lays out the evidence. Transported as a lowercase string via
`Variant.ToString().ToLowerInvariant()` (`…Mongo/Documents/Extensions.cs:49`).

**Lifecycle.** Set at construction, never changed. Parsed from the inbound command's string field by
`Enum.TryParse<Variant>(command.Variant, true, out var variant)` — case-insensitive
(`AddParcelHandler.cs:31`).

**Invariants & enforcement.** Parsing failure is **loud** — `InvalidParcelVariantException`, mapped to
`AddParcelRejected` on AMQP (`ExceptionToMessageMapper.cs:17`) and HTTP 400 `invalid_parcel_variant`.
This is one of the better-behaved failure paths in either service in this batch.

Note that `Enum.TryParse` also accepts a **numeric string**: `"3"` parses to `Animal` without ever
matching a name `[framework]`. And it accepts an out-of-range number — `"99"` parses successfully into
an undefined `Variant` value, which is then stored and rendered as `"99"` by `ToString()`. The guard
is name-based in intent and numeric-permissive in fact.

**Extension procedure.** **Append** new variants; neither reorder nor rename until §3.13's storage
question is settled (§5.3). Adding one requires no other code change — unlike `Size`, which has a hard
dependency in the volume table (§3.5). Consumers that switch on the lowercase string must be updated.

**Failure modes.** The numeric-parse hole above; and a rename changes the API's `variant` string,
which is a contract break under either storage form.

### 3.4 `Size` — the six size bands

**Definition.** `public enum Size { Tiny, Small, Normal, Large, Huge, Exclusive }`
(`…Core/Entities/Size.cs`) — ordinals 0–5.

**Representation & storage.** Same as `Variant`: an unattributed enum property on the document
(`…Mongo/Documents/ParcelDocument.cs:12`), lowercase string on the wire
(`…Mongo/Documents/Extensions.cs:50`).

**Lifecycle.** Parsed at `AddParcelHandler.cs:36`, set at construction, never changed.

**Invariants & enforcement.** Parsing failure is **loud** — `InvalidParcelSizeException` →
`AddParcelRejected` (`ExceptionToMessageMapper.cs:18`). It carries the same numeric-parse hole as
`Variant`, and here the consequence is worse: a numerically-parsed out-of-range `Size` is not a key in
the volume table, and §3.5 throws `KeyNotFoundException` on it.

**Extension procedure.** A new `Size` is a **two-file** change and the second file is easy to miss:

1. Append the member to `Size.cs`.
2. **Add its side length to `_parcelSideLengths` in `…Core/Services/ParcelsService.cs:10-18`.**

Skipping step 2 compiles cleanly and throws `KeyNotFoundException` the first time a parcel of the new
size appears in a volume query (§3.5). This is the single most important extension constraint in this
service.

**Failure modes.** The unresolved storage form makes both reordering and renaming unsafe (§3.13,
§5.3); the volume table makes even *appending* a change with a mandatory second edit.

### 3.5 `ParcelsService.CalculateVolume` — the side-length table and the cube law

**Definition.** The only domain calculation in the service
(`…Core/Services/ParcelsService.cs`):

- `_parcelSideLengths` (`:10-18`) maps each `Size` to a side length in **millimetres** —
  `Tiny` 10, `Small` 30, `Normal` 50, `Large` 75, `Huge` 100, `Exclusive` 200.
- `CalculateVolume(IEnumerable<Parcel> parcels)` (`:20-24`) projects each parcel to its side length,
  then to a volume, then sums.
- `CalculateParcelVolume(double sideLength) => Math.Pow(sideLength, 3) / 1_000_000` (`:26-27`).

**Representation & storage.** The dictionary is `IReadOnlyDictionary<Size, double>`, held on an
instance registered as a **singleton constructed eagerly at startup**:
`builder.Services.AddSingleton<IParcelsService>(new ParcelsService())`
(`…Application/Extensions.cs:19`) — note the `new`, not a type registration, so it is not
constructor-injectable and cannot take dependencies.

**Lifecycle.** Called from exactly one place: `GetParcelsVolumeHandler.cs:87` (§3.29). Nothing else in
the service, and no other service, calls it.

**Invariants & enforcement.**

- **Every `Size` member must be a key.** The indexer `_parcelSideLengths[parcel.Size]` (`:22`) throws
  `KeyNotFoundException` for a missing key — **loud**, but as a framework exception, not a domain one,
  so `ExceptionToResponseMapper` renders it as the generic HTTP 400 `{code:"error"}` (§3.26) and a
  reader of the response cannot tell what went wrong.
- The units are **undocumented in code**. Dividing the cube of a millimetre side length by 1,000,000
  yields cubic *centimetres* if the input is millimetres (mm³ ÷ 1000 = cm³ would be the conversion, so
  ÷ 1,000,000 is neither cm³ nor m³), or — reading the numbers as centimetres — cubic metres
  (cm³ ÷ 1,000,000 = m³). The latter is the only interpretation under which the constants are
  plausible: a 200 cm cube of an `Exclusive` parcel gives 8 m³. **The side lengths are therefore
  centimetres and the result is cubic metres**, but nothing in the code says so, and
  `ParcelsVolumeDto.Volume` is an undecorated `double`
  (`…Application/DTO/ParcelsVolumeDto.cs`). Recorded as **Q-3**.
- Parcels are treated as **cubes**. There is no length/width/height anywhere in the model, so the
  volume is a size-band approximation, not a measurement.

**Extension procedure.** Adding a `Size` requires adding a row here (§3.4). Changing a side length
silently changes every historical volume answer — the figures are not stored, so there is no record
of what a past query returned. If real dimensions are ever introduced, they belong on `Parcel` and
this class becomes a summation, at which point the `Size` band and the volume decouple.

**Failure modes.** `KeyNotFoundException` on an unmapped size, surfaced as an opaque 400; silent unit
ambiguity; and — because the whole calculation is in-process over a fetched list — the cost of a
volume query is the cost of loading every requested parcel document (§3.29).

### 3.6 `Customer` — the id-only reference replica

**Definition.** `public class Customer { public Guid Id { get; } }`
(`…Core/Entities/Customer.cs`) — identical in shape to `orders-service`'s
(`component-internals/orders-service.md` §3.7).

**Representation & storage.** The `customers` collection, registered as
`AddMongoRepository<CustomerDocument, Guid>("customers")` (`…Infrastructure/Extensions.cs:74`)
`[convey]`. `ICustomerRepository` exposes only `ExistsAsync` and `AddAsync`
(`…Core/Repositories/ICustomerRepository.cs`) — there is no update and no delete.

**Lifecycle.** Written once by `CustomerCreatedHandler.cs:25`; read once per `AddParcel` by
`AddParcelHandler.cs:41`. Never updated, never removed.

**Invariants & enforcement.** `CustomerCreatedHandler` throws `CustomerAlreadyExistsException` when the
id is already present (`:20-23`) — **not idempotent**. Because `CustomerAlreadyExistsException` has no
arm in `ExceptionToMessageMapper` (§3.20), a redelivered `customer_created` fails **silently**. That
outcome is benign (the replica is already correct) but the mechanism is the same one that hides real
failures. `MessageToLogTemplateMapper` does register an `OnError` template for this exception, so at
least it is logged (§3.33).

**Extension procedure.** As in `orders-service`: extend the entity, the document and the mappers,
subscribe to an update event (only `customer_created` is subscribed), and **backfill** — the replica
cannot be reconstructed from anything this service holds.

**Failure modes.** Ordering dependency (`AddParcel` fails until `customer_created` has been consumed —
loud on both transports, which is good); no repair path for a lost event
(`service-summaries.md` gap **G12**); existence-only, so a deleted or suspended customer still passes.

### 3.7 `AddedToOrder` — a derived property stored as a column

**Definition.** `public bool AddedToOrder => OrderId.HasValue;` on the entity
(`…Core/Entities/Parcel.cs:16`) — a computed property with no backing field. But `ParcelDocument` has
`public bool AddedToOrder { get; set; }` (`…Mongo/Documents/ParcelDocument.cs:17`) — a real, stored
column.

**Representation & storage.** The value is written on every save —
`AddedToOrder = entity.AddedToOrder` (`…Mongo/Documents/Extensions.cs:41`) — and **is not read back**:
`AsEntity` (`:26-28`) passes only `document.OrderId` into the constructor, so the entity always
re-derives the flag. `AsDto` does not expose it either (`:44-55`).

**Lifecycle.** Write-only from the entity's perspective; read only by one consumer.

**Invariants & enforcement.** The reason it exists is concrete and worth knowing before touching it:
**`GetParcelsHandler` filters on it server-side** — `documents.Where(p => !p.AddedToOrder)`
(`…Mongo/Queries/Handlers/GetParcelsHandler.cs:43`). A computed CLR property cannot be translated into
a Mongo query, so the flag is denormalised specifically to make that filter expressible. It is
redundant with `OrderId`, and deliberately so.

Consistency holds **as long as every write goes through `AsDocument`**, which every write in `src/`
does. It can only drift from a write that bypasses the mapper — for example a document seeded directly
by a test fixture (§3.38) or an operational script. In that case a parcel with an `OrderId` and
`AddedToOrder = false` is invisible to nothing and *visible* in the default `GET /parcels` listing that
is supposed to exclude assigned parcels; the reverse combination hides an unassigned parcel from that
listing permanently.

**Extension procedure.** If the field is ever removed, `GetParcelsHandler:43` must become
`Where(p => p.OrderId == null)`, which Mongo can translate directly — that is a strictly better design
and the removal is safe. If it is kept, never write it from anywhere but `AsDocument`.

**Failure modes.** Silent drift from out-of-band writes, and a redundant field that a maintainer may
reasonably assume is authoritative.

### 3.8 `AddToOrder` / `DeleteFromOrder` — the unconditional mutators

**Definition.** Two three-line methods (`…Core/Entities/Parcel.cs:33-41`):

```
public void AddToOrder(Guid orderId)  { OrderId = orderId; }
public void DeleteFromOrder()         { OrderId = null; }
```

Neither validates anything.

**Representation & storage.** Both mutate `OrderId`, which is then persisted along with the derived
`AddedToOrder` (§3.7).

**Lifecycle.** Called only from external event handlers: `AddToOrder` from `ParcelAddedToOrderHandler`
(`:24`), `DeleteFromOrder` from `OrderCanceledHandler` (`:24`), `OrderDeletedHandler` (`:24`) and
`ParcelDeletedFromOrderHandler` (`:24`). **No command path reaches either** (§1.2).

**Invariants & enforcement.** **None.** Specifically:

- `AddToOrder` on a parcel already attached to a *different* order overwrites silently. There is no
  `if (AddedToOrder) throw`. Since `orders-service` does not remove a parcel from its own order on
  deletion (`component-internals/orders-service.md` §3.8), the same parcel can legitimately be added
  to a second order, and this method will quietly reassign it — leaving order A's document still
  listing a parcel this service says belongs to order B.
- `DeleteFromOrder` on an already-detached parcel is a silent no-op, which makes the *detach* path
  idempotent. That is the one place where the absence of a guard helps.
- `AddToOrder(Guid.Empty)` is accepted.

**Extension procedure.** If reassignment should be refused, the guard belongs in `AddToOrder` and must
throw a new `DomainException` — which then needs an `ExceptionToMessageMapper` arm, or the refusal is
silent on the only transport these handlers use (§3.20). Before adding it, check that
`parcel_added_to_order` redelivery would not then start failing: the outbox inbox de-duplicates by
message id (§3.25), but a genuine re-publish carries a new id and would hit the new guard.

**Failure modes.** Silent reassignment, described above; no event published on either mutation (§3.2),
so neither is observable.

### 3.9 Name and description validation

**Definition.** Both are validated inline in the constructor using a conditional-throw expression
(`…Core/Entities/Parcel.cs:25-28`): `string.IsNullOrWhiteSpace(name) ? throw new InvalidParcelNameException(name) : name`,
and the same shape for `description`.

**Lifecycle.** Evaluated on every construction — including **rehydration** from Mongo
(`…Mongo/Documents/Extensions.cs:26-28`), which is the interesting case.

**Invariants & enforcement.** **Loud** — `invalid_parcel_name` / `invalid_parcel_description`, both
mapped to `AddParcelRejected` on AMQP (`ExceptionToMessageMapper.cs:19-20`) and to HTTP 400.

But because the check runs on rehydration too, **a stored document with a null or blank name cannot be
read back at all**: `ParcelMongoRepository.GetAsync` throws instead of returning the parcel, and the
exception surfaces as a generic failure from whatever operation triggered the read. Such a document is
not merely invalid — it is unreachable through the repository, and only the query handlers (which read
`ParcelDocument` directly and never construct a `Parcel`, §3.28) can still see it. The PACT provider
test seeds exactly such a document (§3.38).

**Extension procedure.** If a length limit or character set is added, add it here — and consider that
tightening a rule retroactively invalidates stored documents, which will then throw on read. Any
tightening needs a data audit first.

**Failure modes.** The rehydration trap above. Note also that `AddParcel` does **not** trim input, so a
name of `" x "` is valid and stored with its whitespace.

### 3.10 Exception hierarchy and the ten error codes

**Definition.** Two hierarchies, `DomainException` (`…Core/Exceptions/DomainException.cs`) and
`AppException` (`…Application/Exceptions/AppException.cs`), each abstract with an abstract
`string Code`.

| Layer | Exception | Code | Thrown by | AMQP mapping |
| --- | --- | --- | --- | --- |
| Core | `CannotDeleteParcelException` | `cannot_delete_parcel` | `DeleteParcelHandler:41` | `DeleteParcelRejected(ex.Id, …)` ✔ |
| Core | `ParcelNotFoundException` | `parcel_not_found` | `DeleteParcelHandler:30` | `DeleteParcelRejected(Guid.Empty, …)` — **id lost** |
| Core | `CustomerNotFoundException` | `customer_not_found` | `AddParcelHandler:43` | `AddParcelRejected` ✔ |
| Core | `InvalidParcelNameException` | `invalid_parcel_name` | `Parcel` ctor `:25` | `AddParcelRejected` ✔ |
| Core | `InvalidParcelDescriptionException` | `invalid_parcel_description` | `Parcel` ctor `:27` | `AddParcelRejected` ✔ |
| Core | `InvalidAggregateIdException` | `invalid_aggregate_id` | **nothing** — there is no `AggregateId` here (§3.2) | none |
| App | `InvalidParcelVariantException` | `invalid_parcel_variant` | `AddParcelHandler:33` | `AddParcelRejected` ✔ |
| App | `InvalidParcelSizeException` | `invalid_parcel_size` | `AddParcelHandler:38` | `AddParcelRejected` ✔ |
| App | `UnauthorizedParcelAccessException` | `unauthorized_parcel_access` | `DeleteParcelHandler:36` | **`AddParcelRejected`** — wrong operation (§3.20) |
| App | `CustomerAlreadyExistsException` | `customer_already_exists` | `CustomerCreatedHandler:22` | **none** — silent |

**Invariants & enforcement.** Nothing checks that a code appears in `operations-service`'s manifest,
and nothing checks that an exception has a mapper arm. `InvalidAggregateIdException` is a **dead
type** — it exists because the service was scaffolded from the same template as its siblings, and
nothing in this repository can throw it.

**Extension procedure.** Derive from the right base, supply a snake-case `Code`, add an
`ExceptionToMessageMapper` arm **naming the correct operation** — the existing mismatch (§3.20) shows
how easily that goes wrong when the mapper is a flat type switch with no operation context.

**Failure modes.** Two of the ten codes are mis-routed or lossy on AMQP and one is entirely silent;
one is unreachable.

### 3.11 `IParcelRepository`

**Definition.** Five methods (`…Core/Repositories/IParcelRepository.cs`), implemented by
`ParcelMongoRepository` over Convey's `IMongoRepository<ParcelDocument, Guid>` `[convey]`:

| Method | Implementation | Note |
| --- | --- | --- |
| `Task<Parcel> GetAsync(Guid id)` | `…Mongo/Repositories/ParcelMongoRepository.cs:19-24` | `parcel?.AsEntity()` |
| `Task<Parcel> GetByOrderAsync(Guid orderId)` | `:26-31` | **returns a single parcel** — see §3.24 |
| `Task AddAsync(Parcel)` | `:33` | `_repository.AddAsync(parcel.AsDocument())` |
| `Task UpdateAsync(Parcel)` | `:34` | whole-document replace (§3.14) |
| `Task DeleteAsync(Guid id)` | `:35` | hard delete |

**Representation & storage.** Registered as
`builder.Services.AddTransient<IParcelRepository, ParcelMongoRepository>()`
(`…Infrastructure/Extensions.cs:53`); the underlying collection binding is at `:75`.

**Lifecycle.** Used by both command handlers and all four order-event handlers. **The three query
handlers do not use it** — they inject `IMongoRepository<ParcelDocument, Guid>` directly (§3.28), so
the read side never constructs a `Parcel` and never runs its constructor validation (§3.9).

**Invariants & enforcement.** All reads return `null` on a miss and **every caller decides what that
means**, inconsistently:

- `DeleteParcelHandler.cs:28-31` → throws `ParcelNotFoundException` (loud).
- All four external event handlers → **`return;`** (silent, §3.23).

**Extension procedure.** Add the method to the interface and the implementation. If the new method
filters on anything but `_id`, **check the index situation first** — there is no index creation
anywhere in this repository (§5.5).

**Failure modes.** `GetByOrderAsync`'s single-result signature is the significant one and it has its
own concept (§3.24). Beyond that: `UpdateAsync` on a parcel that has been deleted concurrently is a
replace of a non-existent document — whether Convey's `UpdateAsync` upserts or no-ops is
**`Unverifiable — Missing Source Evidence`**.

### 3.12 `ParcelDocument` and the three mapping functions

**Definition.** `ParcelDocument : IIdentifiable<Guid>` (`…Mongo/Documents/ParcelDocument.cs:7`) has
**nine** properties (`:9-17`) — the entity's eight plus the denormalised `AddedToOrder` (§3.7). Three
functions in `…Mongo/Documents/Extensions.cs` move between shapes:

| Function | Direction | Notable behaviour |
| --- | --- | --- |
| `AsEntity()` | document → `Parcel` | `:26-28` — calls the constructor, so **constructor validation runs on every read** (§3.9); ignores the stored `AddedToOrder` |
| `AsDocument()` | `Parcel` → document | `:30-42` — nine assignments, including the derived `AddedToOrder` |
| `AsDto()` | document → `ParcelDto` | `:44-55` — lowercases `Variant` and `Size`; **omits `AddedToOrder`** |

Customers use the parallel pair at `:57-64`.

**Representation & storage.** Unlike `orders-service`, **every entity field round-trips**: there is no
equivalent of that service's unpersisted `CancellationReason`
(`component-internals/orders-service.md` §3.15). The mapping here is complete.

**Lifecycle.** `AsEntity` on every repository read, `AsDocument` on every write, `AsDto` on every
query.

**Invariants & enforcement.** `AsDocument` is a hand-written object initialiser with no compiler check
against the document's property set — adding a property to `ParcelDocument` and forgetting the
assignment compiles and silently stores the default. The same hazard applies to `AsEntity`, where the
constructor's parameter list *does* provide some protection: adding a required constructor parameter
breaks the call site at compile time. Prefer adding constructor parameters over optional ones for that
reason.

**Extension procedure.** Four to six touch-points per field (§3.1). Verify by reading the value back,
not by inspecting the write.

**Failure modes.** Silent field loss on `AsDocument`; and `AsEntity` throwing on a document that
violates the name/description rule (§3.9), which turns a data problem into a read failure.

### 3.13 Enum persistence — an unresolved int-vs-string question

**Definition.** `Variant` and `Size` each have three representations: the CLR enum in memory,
*something* on disk, and a lowercase string on the wire. The wire form is settled —
`ToString().ToLowerInvariant()` outbound (`…Mongo/Documents/Extensions.cs:49-50`), case-insensitive
`Enum.TryParse` inbound (`AddParcelHandler.cs:31,36`). The stored form is not.

**Representation & storage.** Two lines of evidence, pointing opposite ways:

| Evidence | Points to | Strength |
| --- | --- | --- |
| No `[BsonRepresentation]` anywhere in `…Mongo/Documents/`; the MongoDB C# driver's *unconfigured* default for an enum is its underlying integer `[framework]` | **integer ordinal** | weak — holds only if no convention is registered |
| This repository's own test fixture registers `new EnumRepresentationConvention(BsonType.String)` in a convention pack named **`convey_conventions`**, applied to all types (`tests/…/Fixtures/MongoDbFixtureInitializer.cs:46,54`), and imports `IMongoDbSeeder`/`MongoDbOptions` from `Convey.Persistence.MongoDB` — identifying it as a stand-in for what `AddMongo` does at startup. An identical file exists in `hianshul100_Pacco.Services.Availability` | **member name string** | strong |

Convey's own initializer is **not in this workspace**, so this remains inference. On balance the
string form is more likely, since a globally registered convention overrides the driver default for
every document type. Marked **`Unverifiable — Missing Source Evidence`** and carried as **Q-4**.
One `db.parcels.findOne({}, {size: 1, variant: 1})` against any deployed environment settles it, and
should be run before any change to either enum.

**Invariants & enforcement.** None either way. Inbound parsing is name-based in intent and numerically
permissive in fact (§3.3). Under integer storage, an unmatched value deserialises silently and renders
as the number; under string storage, an unmatched value **throws** on deserialisation `[framework]`,
which — because `AsEntity` runs on every repository read (§3.12) — turns one bad document into a read
failure rather than a bad value.

**Extension procedure.** **Append only**, until Q-4 is answered:

| Change | If stored as ordinal | If stored as member name |
| --- | --- | --- |
| **Append** | safe (but see §3.4 for `Size`'s mandatory second edit) | safe |
| **Reorder / insert** | silently relabels every stored parcel — a `Huge` parcel becoming `Large` changes its volume by a factor of 3.4 | harmless on disk |
| **Rename** | stored data untouched; the wire string changes in **both** directions, so a client that previously sent `"weapon"` is now rejected | stored data *and* wire string change; existing documents fail to load |

Whichever is in force, pin it before making a structural change: add an explicit
`[BsonRepresentation(...)]` to the document property so it no longer depends on a global convention,
migrate stored documents to that form, then reorder or rename.

**Failure modes.** No schema-version marker anywhere in the repository, so neither representation is
self-describing and neither migration is detectable after the fact.

### 3.14 Absence of optimistic concurrency

**Definition.** `ParcelMongoRepository.UpdateAsync(Parcel parcel) => _repository.UpdateAsync(parcel.AsDocument())`
(`…Mongo/Repositories/ParcelMongoRepository.cs:34`) — a whole-document replace keyed on `Id`
`[convey]`. There is no version field on `ParcelDocument` and no `Version` on `Parcel`.

**Lifecycle.** Every mutating path is read → mutate → replace.

**Invariants & enforcement.** **None — last writer wins, whole document.** The realistic collision here
is narrower than in `orders-service` because there are no blocking calls inside the window: the
handlers are read-mutate-write with nothing in between. The exposure is genuine but small — two order
events for the same parcel arriving concurrently (for example `parcel_added_to_order` for order B and
`order_canceled` for order A) can be applied in either order, and the loser's effect vanishes.

**Extension procedure.** Adding a version field means adding it to `Parcel` and `ParcelDocument`,
incrementing it in the mutators, and changing `UpdateAsync` to a conditional replace. All four are
required; adding the field alone changes nothing.

**Failure modes.** Silent lost update on concurrent order events for one parcel; no telemetry that
would reveal it.

### 3.15 `AddParcel` — the command that generates its own id

**Definition.** `…Application/Commands/AddParcel.cs` — `[Contract]`, `ICommand`, six get-only
properties: `ParcelId`, `CustomerId`, `Variant`, `Size`, `Name`, `Description`.

**Representation & storage.** `Variant` and `Size` are **`string`** on the command and `enum` on the
entity; the conversion is the handler's first act (§3.16).

**Lifecycle.** The constructor contains
`ParcelId = parcelId == Guid.Empty ? Guid.NewGuid() : parcelId;` — the same self-generating-id idiom
as `orders-service`'s `CreateOrder` (`component-internals/orders-service.md` §3.19), with the same
consequence: a `Guid.Empty` is silently substituted rather than rejected, and the substituted id is
never communicated back to a direct AMQP publisher. On the gateway's async route the id is generated
upstream (`resourceId: generate: true`) and this branch never fires.

**Invariants & enforcement.** **No validation attributes and no validation middleware.** `CustomerId`
is taken from the payload and is **never checked against the caller's identity** (§3.17). A command
with every field defaulted reaches the handler, where it fails on the variant parse — so the effective
first validation is `Enum.TryParse` on an empty string.

**Extension procedure.** Add the field to the command, to the handler's construction call
(`AddParcelHandler.cs:46`), to `Parcel`, to `ParcelDocument` and to the mappers. Also add it to the
gateway's async route payload if it must be settable from outside, and to
`…Operations…/messages.json` if the message shape is asserted there.

**Failure modes.** Silent id substitution; no idempotency (a retried `POST /parcels` with the same
non-empty id calls `AddAsync` twice — §3.25 explains why the inbox does not help on HTTP).

### 3.16 `AddParcelHandler`

**Definition.** `…Application/Commands/Handlers/AddParcelHandler.cs:29-50` — the service's only
creation path.

**Lifecycle.**

| Step | Line | Failure behaviour |
| --- | --- | --- |
| `Enum.TryParse<Variant>(command.Variant, true, out var variant)` | `:31` | `InvalidParcelVariantException` (`:33`) |
| `Enum.TryParse<Size>(command.Size, true, out var size)` | `:36` | `InvalidParcelSizeException` (`:38`) |
| `_customerRepository.ExistsAsync(command.CustomerId)` | `:41` | `CustomerNotFoundException` (`:43`) |
| `new Parcel(command.ParcelId, command.CustomerId, variant, size, command.Name, command.Description, _dateTimeProvider.Now)` | `:46-47` | name/description validation (§3.9) |
| `_parcelRepository.AddAsync(parcel)` | `:48` | insert |
| `_messageBroker.PublishAsync(new ParcelAdded(command.ParcelId))` | `:49` | outbox → exchange `parcels` |

**Invariants & enforcement.** Every one of the four guards is **loud on both transports** — each has an
`ExceptionToMessageMapper` arm producing `AddParcelRejected` (`ExceptionToMessageMapper.cs:16-20`).
This is the best-instrumented write path across both services in this batch, and it is worth using as
the template for new work.

The one thing it does **not** do is check who the caller is (§3.17).

**Extension procedure.** Guards run in the order above; a new guard's position determines which error
a caller sees when several inputs are wrong at once. Add the matching mapper arm in the same change.

**Failure modes.** `_dateTimeProvider.Now` is `DateTime.UtcNow` (§3.39), so `CreatedAt` is UTC —
but nothing enforces that, and the constructor accepts any value. `AddAsync` on a duplicate id has
undefined behaviour here (**`Unverifiable — Missing Source Evidence`**, Convey's repository is not in
this workspace).

### 3.17 The identity guard: present on delete, absent on add

**Definition.** One handler checks ownership. `DeleteParcelHandler.cs:33-37`:

```
var identity = _appContext.Identity;
if (identity.IsAuthenticated && identity.Id != parcel.CustomerId && !identity.IsAdmin)
{
    throw new UnauthorizedParcelAccessException(parcel.Id, identity.Id);
}
```

`AddParcelHandler` has **no `IAppContext` dependency at all** — it is not in the constructor
(`AddParcelHandler.cs:20-27`).

**Invariants & enforcement.** Two independent holes:

1. **`AddParcel` takes `CustomerId` from the request body and never compares it to the caller.** Any
   caller who can reach the service can create a parcel *on behalf of any customer*, provided that
   customer exists in the replica. The mitigation is entirely at the gateway, whose async parcels
   route binds `customerId` from the token (`ntrada-async.yml`, parcels block) `[ntrada]` — a binding
   this service neither knows about nor verifies.
2. **The delete guard is gated on `identity.IsAuthenticated &&`**, exactly as in `orders-service`
   (`component-internals/orders-service.md` §3.20). An **unauthenticated** caller — one whose
   correlation context is missing or whose `user.id` is blank — passes the condition and may delete
   any parcel.

| Caller | `IsAuthenticated` | `DeleteParcel` outcome |
| --- | --- | --- |
| Authenticated owner | true | passes |
| Authenticated non-owner | true | **blocked** |
| Authenticated admin | true | passes |
| **Unauthenticated** | **false** | **passes — deletes any parcel** |

Enforcement is **loud when it triggers** (HTTP 400 `unauthorized_parcel_access`, or — wrongly —
`AddParcelRejected` on AMQP, §3.20) and **silent when bypassed**.

**Extension procedure.** The fix is two-part and must be done together:

1. Invert the delete condition to
   `if (!identity.IsAuthenticated || (identity.Id != parcel.CustomerId && !identity.IsAdmin))`.
2. Inject `IAppContext` into `AddParcelHandler` and either reject a mismatched `CustomerId` or ignore
   the payload value and use `identity.Id`. The second is the better design; it changes the command's
   effective contract, so coordinate with the gateway binding and with `ordermaker-saga-service`,
   which does not publish `AddParcel` today but is the kind of caller that would break.

Before either, confirm what correlation context internal publishers supply — see **Q-2**.

**Failure modes.** Cross-tenant parcel creation and unauthenticated deletion, both currently mitigated
only by network topology. Note also that `identity.Id` is `Guid.Empty` when the header value is not a
parseable GUID (`…Infrastructure/Contexts/IdentityContext.cs`), so a malformed id is *authenticated*
but matches no parcel — a hard block rather than a bypass. The two malformed-input paths fail in
opposite directions.

### 3.18 `DeleteParcelHandler` — three guards in a fixed order

**Definition.** `…Application/Commands/Handlers/DeleteParcelHandler.cs:25-46`.

**Lifecycle.**

| Step | Line | Failure |
| --- | --- | --- |
| `_parcelRepository.GetAsync(command.ParcelId)` | `:27` | `ParcelNotFoundException` (`:30`) — id lost in the rejected event (§3.20) |
| ownership guard | `:33-37` | `UnauthorizedParcelAccessException` — mapped to the **wrong** rejected event (§3.20) |
| `if (parcel.AddedToOrder)` | `:39-42` | `CannotDeleteParcelException` — the only correctly mapped arm |
| `_parcelRepository.DeleteAsync(command.ParcelId)` | `:44` | hard delete |
| `PublishAsync(new ParcelDeleted(command.ParcelId))` | `:45` | outbox → exchange `parcels` |

**Invariants & enforcement.** The order matters: a caller who does not own a parcel learns
`unauthorized_parcel_access` rather than `cannot_delete_parcel`, which is the right precedence
(ownership before business rule). But `parcel_not_found` comes **first**, so a non-owner probing ids
can distinguish "exists" from "does not exist" — a minor enumeration leak that is inherent to
returning a distinct error for a missing resource.

The `AddedToOrder` guard is the domain rule that makes this service's relationship state meaningful: a
parcel committed to an order cannot be destroyed underneath it.

**Extension procedure.** New guards go between the ownership check and the business rule. Every new
exception needs a mapper arm naming `DeleteParcel`, or it becomes silent.

**Failure modes.** Two of the three guards report the wrong thing on AMQP (§3.20). And the delete is
hard: nothing retains a record that the parcel existed, so the published `parcel_deleted` is the only
trace — and it has no subscriber (§3.19).

### 3.19 The two integration events, and their absent subscribers

**Definition.** `…Application/Events/ParcelAdded.cs` and `ParcelDeleted.cs` — both `[Contract]`, both
`IEvent`, both carrying **only** `ParcelId`.

**Representation & storage.** Published to the topic exchange `parcels` with snake-case routing keys
`parcel_added` and `parcel_deleted` (§3.32).

**Lifecycle.** Constructed by hand in the two command handlers (`AddParcelHandler.cs:49`,
`DeleteParcelHandler.cs:45`) — there is no event mapper (§3.21).

**Invariants & enforcement.** **Neither event has a subscriber anywhere in this workspace.** Verified
by searching every clone for `SubscribeEvent<ParcelAdded>` and `SubscribeEvent<ParcelDeleted>`: there
are none. For `parcel_deleted` the near-miss is instructive — `orders-service` *does* define a
`ParcelDeleted` external event and a handler for it, but declares it as `[Message("deliveries")]`
(`component-internals/orders-service.md` §3.33), so its queue is bound to an exchange on which this
routing key is never published. The intended integration exists on both sides and is broken by a
one-word attribute value.

The events are not useless: `operations-service` correlates them for the client's operation poll via
its `messages.json` manifest (`component-internals/operations-service.md` §3.9).

**Extension procedure.** New event → class with `[Contract]`, publish it explicitly from the handler
that mutates, add a `messages.json` entry, add a log template (§3.33). Because publication is
hand-written, there is no mapper arm to forget — but equally, nothing reminds you to publish.

**Failure modes.** Carrying only `ParcelId` means any subscriber must call back for details — and there
is no HTTP client dependency in the other direction to make that cheap. A future subscriber will
likely need the events widened, which is a backward-compatible change (§5.6).

### 3.20 Rejected events and `ExceptionToMessageMapper`

**Definition.** `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` — 27 lines, a **flat**
switch on exception type with **no nested switch on the inbound message type**, except for one arm.
Contrast `orders-service`, whose mapper is a matrix (`component-internals/orders-service.md` §3.26).

**Lifecycle.** Called by the RabbitMQ subscriber on an unhandled exception `[convey]`. A non-null
result is published to the `parcels` exchange; **null means the exception is swallowed and the message
acknowledged**.

**Invariants & enforcement.** Three defects, all consequences of the flat shape:

1. **`UnauthorizedParcelAccessException => new AddParcelRejected(ex.Message, ex.Code)`** (`:24`). That
   exception is thrown **only** by `DeleteParcelHandler.cs:36`. A client that submits a delete over
   AMQP and is refused receives an **`add_parcel_rejected`** event. If it is also adding parcels, the
   rejection is indistinguishable from a failed add — and since `AddParcelRejected` carries no parcel
   id, it cannot be attributed to either operation. This is the clearest defect in the file.
2. **`ParcelNotFoundException` loses the id** (`:21-23`): the only arm with message-type awareness,
   it emits `new DeleteParcelRejected(Guid.Empty, ex.Message, ex.Code)` when the inbound message is a
   `DeleteParcel`, and `null` otherwise. `Guid.Empty` is passed where `command.ParcelId` was available
   — the exception itself carries the id, and the arm discards it.
3. **`CustomerAlreadyExistsException` has no arm** — it falls to `_ => null` (`:25`) and is silent
   (§3.6).

**Extension procedure.** For a new exception, add an arm and **name the operation the exception
actually comes from**. When an exception can arise from more than one operation, follow the
`ParcelNotFoundException` pattern of switching on `message` — and pass the real id. Cross-check
`…Operations…/messages.json` so `operations-service` recognises the rejected event.

**Failure modes.** A client polling `operations-service` after a failed delete is told about an add;
a client with several deletes in flight cannot attribute a rejection; and a duplicate
`customer_created` fails invisibly.

### 3.21 No `IEventMapper` — publication is hand-rolled

**Definition.** A concept by absence: `…Infrastructure/Services/` contains `DateTimeProvider.cs` and
`MessageBroker.cs` and **no `EventMapper.cs`**. `…Infrastructure/Extensions.cs:48-78` registers no
`IEventMapper`.

**Lifecycle.** Both published events are constructed literally at their publication site (§3.19).

**Invariants & enforcement.** This removes the whole class of silent event loss that
`orders-service` suffers, where `EventMapper.Map` returns `null` for an unmapped domain event and
`MessageBroker` discards it without logging
(`component-internals/orders-service.md` §3.23). Here, an event is published because someone wrote a
line to publish it.

The cost is the mirror image: **there is no mechanism that ties a state change to an event**. Four of
the six mutating paths in this service (the external event handlers, §3.23) change state and publish
nothing at all.

**Extension procedure.** When adding a mutating path, decide explicitly whether it should publish, and
write the `PublishAsync` line. Do **not** introduce an event mapper to "fix" the asymmetry — that
would require domain events, which would require an aggregate root, which is a much larger change than
the problem warrants (§3.2).

**Failure modes.** Unobservable state changes, described above.

### 3.22 `MessageBroker` — the publish pipeline

**Definition.** `…Infrastructure/Services/MessageBroker.cs` — structurally identical to
`orders-service`'s (`component-internals/orders-service.md` §3.25): every handler depends on
`IMessageBroker`, never on the RabbitMQ client.

**Lifecycle.** `PublishAsync` returns early on a null collection; builds a correlation id from the
correlation-context accessor or the HTTP context; resolves a span context from the message properties
header or the active tracer span; collects headers to forward — **only `Saga`**
(`…Infrastructure/Extensions.cs:105-119`); **skips null events**; then routes each event through the
outbox when enabled or publishes directly otherwise `[convey]`.

**Invariants & enforcement.** The null-skip is unconditional and unlogged. It is inert in this service,
because nothing here can produce a null event (§3.21) — but it is the same code, and it will become
live the moment any mapping indirection is introduced.

**Extension procedure.** To forward more headers, extend `GetHeadersToForward`
(`…Infrastructure/Extensions.cs:105-119`), which returns `null` unless a `Saga` header is present.

**Failure modes.** A publish outside any span carries an empty `span_context`, breaking the trace at
that hop; and because the outbox is per-message rather than per-transaction, a handler that publishes
two events can have the first stored and the second lost (§3.25).

### 3.23 External event subscription — five events, four silent-miss handlers

**Definition.** Five `SubscribeEvent<T>()` calls (`…Infrastructure/Extensions.cs:91-95`), each
resolving its exchange from a `[Message("<exchange>")]` attribute on the event class.

| Event | Exchange | Handler | Lookup | On miss |
| --- | --- | --- | --- | --- |
| `OrderCanceled` | `orders` | `OrderCanceledHandler` | `GetByOrderAsync(@event.OrderId)` (`:18`) | **`return;`** (`:19-22`) |
| `OrderDeleted` | `orders` | `OrderDeletedHandler` | `GetByOrderAsync(@event.OrderId)` | **`return;`** |
| `ParcelAddedToOrder` | `orders` | `ParcelAddedToOrderHandler` | `GetAsync(@event.ParcelId)` (`:18`) | **`return;`** (`:19-22`) |
| `ParcelDeletedFromOrder` | `orders` | `ParcelDeletedFromOrderHandler` | `GetAsync(@event.ParcelId)` | **`return;`** |
| `CustomerCreated` | `customers` | `CustomerCreatedHandler` | `ExistsAsync` (`:20`) | throws (silent on AMQP, §3.6) |

Each of the four order handlers is the same eight-line shape: look up, `return` on null, mutate
(`AddToOrder` or `DeleteFromOrder`), `UpdateAsync`. None publishes anything (§3.21). None logs the
miss.

**Lifecycle.** Queues follow the template in the `rabbitMq` section of `…Api/appsettings.json`,
producing names of the form `parcels-service/orders.order_canceled` (§3.32).

**Invariants & enforcement.** The silent `return` is a deliberate idempotency choice — an event for a
parcel this service has never heard of, or has already deleted, should not fail. But it is
indistinguishable from the failure case: a `parcel_added_to_order` whose `ParcelId` is wrong, or which
arrives before the parcel was created, is dropped with no trace, and the parcel is never marked as
belonging to that order. Nothing reconciles afterwards.

**Extension procedure.** Add the event class with the **producer's** exchange name — this is exactly
where `orders-service` went wrong in the opposite direction (§3.19) — add the handler, add
`SubscribeEvent<T>()`, add a log template (§3.33). Consider logging the miss rather than returning
bare; it costs one line and converts an invisible drop into a diagnosable one.

**Failure modes.** Four silent-miss paths, plus the single-result lookup problem that has its own
concept (§3.24).

### 3.24 `GetByOrderAsync` releases **one** parcel per order event

**Definition.** `Task<Parcel> GetByOrderAsync(Guid orderId)` — a **singular** return type
(`…Core/Repositories/IParcelRepository.cs`), implemented as
`_repository.GetAsync(p => p.OrderId == orderId)` (`…Mongo/Repositories/ParcelMongoRepository.cs:28`),
which returns the first matching document `[convey]`.

**Representation & storage.** `OrderId` is not unique in the `parcels` collection — an order routinely
has several parcels, since `orders-service` supports adding many
(`component-internals/orders-service.md` §3.1).

**Lifecycle.** Called by `OrderCanceledHandler.cs:18` and `OrderDeletedHandler.cs:18`. Each handler
then detaches **that one parcel** and returns.

**Invariants & enforcement.** The invariant the handlers intend — "when an order is cancelled or
deleted, every parcel on it is released" — **does not hold**. Concretely: an order with three parcels
is deleted; `order_deleted` arrives once; one parcel is detached; **two remain with `OrderId` pointing
at an order that no longer exists**. Those two:

- are excluded from the default `GET /parcels` listing, which filters `!AddedToOrder` (§3.28) — so the
  customer cannot see them;
- cannot be deleted, because `DeleteParcelHandler.cs:39-42` refuses to delete a parcel that is
  `AddedToOrder` (§3.18);
- can never be released, because the only event that would release them has already been consumed.

They are **permanently stranded**. There is no repair path in the service and no reconciliation job.
This is the highest-severity defect in this component.

**Extension procedure.** The fix is a repository change plus two handler changes:

1. Change the interface to `Task<IEnumerable<Parcel>> GetByOrderAsync(Guid orderId)` and implement it
   with `_repository.FindAsync(p => p.OrderId == orderId)` — `FindAsync` is already used by
   `GetParcelsVolumeHandler.cs:85`, so the plural form is available `[convey]`.
2. Change `OrderCanceledHandler` and `OrderDeletedHandler` to iterate, calling `DeleteFromOrder` and
   `UpdateAsync` per parcel. Note this turns one write into N writes with no transaction; a partial
   failure leaves some parcels stranded, so the handler must be safe to redeliver — it is, because
   `DeleteFromOrder` on an already-detached parcel is a no-op (§3.8).
3. **Backfill the existing stranded parcels** (§5.4) — the code change does not repair them.

Add an index on `OrderId` in the same change (§5.5); this query is a full scan today.

**Failure modes.** Stranded parcels, invisible and undeletable, accumulating one per multi-parcel
order cancellation or deletion. Recorded as **B-1**.

### 3.25 Transactional outbox and inbox

**Definition.** Two decorators, byte-for-byte the same shape as `orders-service`'s
(`component-internals/orders-service.md` §3.29):
`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs` and `OutboxEventHandlerDecorator.cs`,
both `[Decorator] internal sealed`, registered by `TryDecorate` at
`…Infrastructure/Extensions.cs:56-57`.

**Representation & storage.** Mongo collections `inbox` and `outbox`, per the `outbox` section of
`…Api/appsettings.json`, which enables the outbox, sets `type` to sequential, an expiry of one hour, a
dispatch interval of two seconds, and — importantly — **`disableTransactions` set to `true`**. The
`local` profile disables the outbox entirely.

**Lifecycle.** Each decorator reads the inbound message id from `IMessagePropertiesAccessor` and falls
back to a fresh GUID:

```
_messageId = string.IsNullOrWhiteSpace(messageProperties?.MessageId)
    ? Guid.NewGuid().ToString("N")
    : messageProperties.MessageId;
```
(`OutboxCommandHandlerDecorator.cs:27-29`, `OutboxEventHandlerDecorator.cs:27-29`)

`HandleAsync` then wraps the inner handler in `_outbox.HandleAsync(_messageId, …)` when enabled, or
calls it directly when not (`:32-35` in both).

**Invariants & enforcement.** Two consequences worth internalising before relying on this:

1. **Idempotency is AMQP-only.** On an HTTP request there are no message properties, so a fresh GUID
   is minted per request and the inbox key never collides. A retried `POST /parcels` with the same
   `parcelId` therefore runs the handler twice. On AMQP, a redelivered `add_parcel` with the same
   `MessageId` is deduplicated `[convey]`.
2. **`disableTransactions: true` means the handler's own writes and the outbox insert are not
   atomic.** With `AddParcelHandler`, a crash between `AddAsync` (`:48`) and `PublishAsync` (`:49`)
   leaves a parcel with no `parcel_added` event. Nothing detects or repairs that. It is currently
   harmless only because `parcel_added` has no subscriber (§3.19) — which is a reason to fix the
   subscriber gap and the transaction gap together, not separately.

Enforcement of the outbox contract itself lives in Convey and is **`Unverifiable — Missing Source
Evidence`**: whether `_outbox.HandleAsync` records the id before or after the inner handler succeeds
determines whether a failing handler is retried or permanently suppressed.

**Extension procedure.** Nothing to do — the decorators are open-generic and apply to every
`ICommandHandler<>`/`IEventHandler<>` automatically. To make HTTP idempotent, derive the key from
something stable in the request (the `parcelId`, or an `Idempotency-Key` header) rather than from
`Guid.NewGuid()`.

**Failure modes.** Duplicate HTTP writes; non-atomic write-plus-publish; and an hour-long `expiry`
that silently bounds how far back deduplication reaches.

### 3.26 `ExceptionToResponseMapper` — everything is HTTP 400

**Definition.** `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` — 46 lines, identical in
structure to `orders-service`'s (`component-internals/orders-service.md` §3.30).

**Representation & storage.** `Map` (`:15-24`) returns `{code, reason}` with
`HttpStatusCode.BadRequest` for `DomainException` (`:18-19`), the same for `AppException` (`:20-21`),
and **`{code = "error", reason = "There was an error."}` — also `BadRequest`** for everything else
(`:22-23`).

`GetCode` (`:26-45`) caches per exception type in a static `ConcurrentDictionary` (`:13`), preferring
the exception's own `Code` when non-blank (`:36-38`) and otherwise deriving it as
`exception.GetType().Name.Underscore().Replace("_exception", string.Empty)` (`:39`).

**Lifecycle.** Installed by `AddErrorHandler<ExceptionToResponseMapper>()`
(`…Infrastructure/Extensions.cs:60`) and activated by `UseErrorHandler()` (`:83`) `[convey]`.

**Invariants & enforcement.** **No status code other than 400 is reachable through this mapper.**
Concretely, on `GET /parcels/{parcelId}` for an id that does not exist, no exception is thrown at all
— `GetParcelHandler.cs:24` returns `null` and the dispatcher renders that (§3.28). But a genuinely
missing parcel on `DELETE` returns **400**, not 404; an unauthorized delete returns **400**, not 403;
and an unexpected `NullReferenceException` returns **400** with `"There was an error."` rather than
500. A caller cannot distinguish "your request was wrong" from "the service is broken", and neither
can a monitoring rule built on status codes.

The catch-all is the right *shape* — it does not leak stack traces or internal messages — and the
wrong *code*.

**Extension procedure.** To return a non-400 status, add an arm before the catch-all matching the
specific exception type and naming the status. Do not change the `DomainException`/`AppException`
arms wholesale: several codes are already load-bearing for the gateway and for `operations-service`
correlation. Adding a `Code` to a new exception fixes its wire code; omitting one falls back to the
underscored type name, which is why `CannotDeleteParcelException` surfaces as
`cannot_delete_parcel`.

**Failure modes.** Server faults are indistinguishable from client errors; every alert threshold built
on 5xx rates is blind to this service.

### 3.27 `IAppContext` / `IIdentityContext` and the `Correlation-Context` header

**Definition.** Four types in `…Infrastructure/Contexts/` behind two `Application`-layer interfaces,
plus a factory:

| Type | File | Role |
| --- | --- | --- |
| `AppContext` | `Contexts/AppContext.cs` | `RequestId` + `Identity`; `Empty` static (`:26`) |
| `IdentityContext` | `Contexts/IdentityContext.cs` | `Id`, `Role`, `IsAuthenticated`, `IsAdmin`, `Claims` |
| `CorrelationContext` | `Contexts/CorrelationContext.cs` | the deserialisation target, with a nested `UserContext` (`:17-23`) |
| `AppContextFactory` | `Contexts/AppContextFactory.cs` | chooses the AMQP or HTTP source |

**Lifecycle.** `AppContextFactory.Create()` (`:19-33`) prefers the AMQP correlation context when
present — serialising it to JSON and deserialising it back into the local `CorrelationContext` type
(`:23-27`), a round-trip through strings rather than a cast — and otherwise reads the HTTP context via
`GetCorrelationContext()`, which parses the **`Correlation-Context`** header
(`…Infrastructure/Extensions.cs:100-103`). A null on either path yields `AppContext.Empty`, whose
identity is `IdentityContext.Empty` — `Id = Guid.Empty`, `Role = string.Empty`,
`IsAuthenticated = false`, `IsAdmin = false`.

**Invariants & enforcement.** Three details decide how every guard in this service behaves:

- `Id = Guid.TryParse(id, out var userId) ? userId : Guid.Empty` (`IdentityContext.cs:26`) — an
  unparseable id becomes `Guid.Empty` **silently**, and `Guid.Empty` matches no parcel's
  `CustomerId`, so a malformed id fails *closed* on delete.
- `IsAdmin = Role.Equals("admin", StringComparison.InvariantCultureIgnoreCase)` (`:29`) — a single
  hard-coded role string, compared case-insensitively. There is no role list and no policy.
- `IsAuthenticated` is **copied verbatim from the header payload** (`:28`, via
  `CorrelationContext.UserContext.IsAuthenticated`). This service performs no token validation of its
  own: it trusts whatever the gateway put in the header. Anyone who can reach the service directly can
  assert any identity, or assert none — and asserting *none* is the more useful attack, because it
  bypasses the delete guard entirely (§3.17).

**Extension procedure.** To add a claim-based rule, read `identity.Claims` — it is populated from the
header and defaults to an empty dictionary (`:30`), so a missing claim is an empty-string lookup, not
an exception. To add a second privileged role, replace the `IsAdmin` computation rather than adding a
parallel property, so every existing guard picks it up.

**Failure modes.** Full identity spoofing for any caller with network reach; and the silent
`Guid.Empty` fallback, which turns a malformed id into a hard block rather than an error message.

### 3.28 `GetParcel` and `GetParcels` — one unguarded, one silently empty

**Definition.** Two of the three query handlers, both injecting
`IMongoRepository<ParcelDocument, Guid>` **directly** rather than `IParcelRepository` (§3.11).

**`GetParcelHandler`** (`…Mongo/Queries/Handlers/GetParcelHandler.cs:20-25`) is four lines:
`_parcelRepository.GetAsync(p => p.Id == query.ParcelId)` (`:22`) then `document?.AsDto()` (`:24`).
**It has no `IAppContext` dependency and performs no ownership check** — any caller who knows a
parcel id reads that parcel in full, including its `customerId`, `description` and `orderId`. This is
the endpoint `orders-service` calls server-to-server (`component-internals/orders-service.md` §3.34)
and the one the PACT contract pins (§3.38), which is presumably why it is unguarded; the effect is
that the gateway's `parcels/{parcelId}` route is an unauthenticated read of anyone's parcel.

**`GetParcelsHandler`** (`GetParcelsHandler.cs:27-49`) builds a queryable and applies up to two
filters:

| Step | Line | Behaviour |
| --- | --- | --- |
| `_parcelRepository.Collection.AsQueryable()` | `:29` | unbounded — no `Skip`/`Take`/`Limit` |
| if `query.CustomerId.HasValue` and the identity mismatches | `:33-36` | **`return Enumerable.Empty<ParcelDto>()`** |
| else filter by `CustomerId` | `:38` | |
| if `!query.IncludeAddedToOrders` | `:41-44` | `Where(p => !p.AddedToOrder)` — the reason the flag is stored (§3.7) |
| `ToListAsync()` then `Select(AsDto)` | `:46-48` | |

**Invariants & enforcement.** The asymmetry between the two handlers is the point: the *list* endpoint
enforces ownership and the *single-item* endpoint does not. And the list endpoint enforces it
**silently** — a caller asking for another customer's parcels receives `200 OK` with `[]`,
indistinguishable from "that customer has no parcels". The guard also carries the same
`identity.IsAuthenticated &&` prefix as everywhere else (§3.17), so an unauthenticated caller passes
it; and it applies **only when `CustomerId` is supplied** — `GET /parcels` with no query string
returns *every customer's* unassigned parcels, to anyone.

**Extension procedure.** Add filters as further `Where` clauses before `ToListAsync`, so they compose
into the server-side query rather than filtering in memory — the existing two both do. Any new filter
over a field other than `_id` is a collection scan until an index exists (§5.5). When adding paging,
note that `ParcelDto` is returned as a bare array, so a page wrapper is a breaking response-shape
change.

**Failure modes.** Unbounded result sets; a silent empty list that hides an authorization failure; an
unfiltered listing with no customer scope; and an entirely unguarded single-parcel read.

### 3.29 `GetParcelsVolume` — the only query with domain logic behind it

**Definition.** `…Mongo/Queries/Handlers/GetParcelsVolumeHandler.cs:25-43`, serving
`GET /parcels/volume` and returning `ParcelsVolumeDto { double Volume }`.

**Lifecycle.**

| Step | Line | Note |
| --- | --- | --- |
| `if (query.ParcelIds is null \|\| !query.ParcelIds.Any())` → `Volume = 0` | `:27-33` | an empty request is a successful zero, not an error |
| `FindAsync(p => query.ParcelIds.Contains(p.Id))` | `:35` | translated to `$in` `[convey]` |
| `documents.Select(d => d.AsEntity())` | `:36` | **runs constructor validation on every document** (§3.9) |
| `_parcelsService.CalculateVolume(parcels)` | `:37` | the only call into `Core/Services` |

**Invariants & enforcement.** This is the only read path that goes through the domain, and it is the
only one that can *throw* on stored data: `AsEntity` runs `Parcel`'s constructor, so a document with a
blank name or description makes the whole volume query fail rather than returning a partial answer.

Three silent behaviours matter to a caller:

- **Ids that do not exist are simply absent from the result**, and the volume is computed over
  whatever was found. Asking for five parcels of which two exist returns a number, with nothing
  indicating that three were missing. A caller using this to price or size a delivery gets a
  confidently wrong answer.
- **There is no ownership check**, consistent with `GetParcel` (§3.28) and inconsistent with
  `GetParcels`.
- **There is no cap on `ParcelIds`**, so the `$in` list is caller-controlled and unbounded.

**Extension procedure.** If a caller needs to know about missing ids, return them alongside the
volume — that is an additive change to `ParcelsVolumeDto`, and additive DTO changes are safe for JSON
consumers (§5.6). Do not silently substitute zero for a missing parcel.

**Failure modes.** Silent under-counting; one malformed stored document failing the entire query; an
unbounded `$in`; and `KeyNotFoundException` if any matched parcel carries a `Size` absent from the
side-length table (§3.4).

### 3.30 Route ordering — `parcels/volume` must precede `parcels/{parcelId}`

**Definition.** Six endpoint registrations in `…Api/Program.cs:38-45`, in source order:

| Order | Line | Registration |
| --- | --- | --- |
| 1 | `:39` | `Get("")` — the root liveness response |
| 2 | `:40` | `Get<GetParcelsVolume, ParcelsVolumeDto>("parcels/volume")` |
| 3 | `:41` | `Get<GetParcel, ParcelDto>("parcels/{parcelId}")` |
| 4 | `:42` | `Get<GetParcels, IEnumerable<ParcelDto>>("parcels")` |
| 5 | `:43` | `Delete<DeleteParcel>("parcels/{parcelId}")` |
| 6 | `:44-45` | `Post<AddParcel>("parcels", afterDispatch: (cmd, ctx) => ctx.Response.Created($"parcels/{cmd.ParcelId}"))` |

**Invariants & enforcement.** `parcels/volume` is registered **before** `parcels/{parcelId}`. If the
underlying router matches in registration order — which is the behaviour this ordering implies, and
which is `[convey]`-dependent and **`Unverifiable — Missing Source Evidence`** — then reversing the
two lines makes `GET /parcels/volume` bind `parcelId = "volume"`, fail to parse as a `Guid`, and
return a null parcel instead of a volume. The literal route would become unreachable with **no error
at startup and no error at request time**: a 200 with an empty body.

This is a real constraint expressed only as line ordering, with no comment marking it. It is the kind
of thing an automatic "sort these alphabetically" cleanup destroys.

**Extension procedure.** **Register every literal sub-path before the parameterised one.** A new
`parcels/summary` route goes above `:41`, not below it. After any reordering, curl
`/parcels/volume?parcelIds=[]` and confirm it returns a `volume` field rather than an empty body.

Note also `:44-45`: `POST /parcels` returns `201 Created` with a `Location` built from the command's
own `ParcelId` — which is why `AddParcel`'s self-generating id (§3.15) is observable to an HTTP
caller even though it is invisible to an AMQP one.

**Failure modes.** A silently shadowed route; and `Get("")` at `:39` returning the app name, which is
what the Consul health ping and the `logger.excludePaths` list both target.

### 3.31 `[Contract]` and `UsePublicContracts`

**Definition.** `…Application/ContractAttribute.cs` — an empty marker
(`public class ContractAttribute : Attribute`), activated by
`app.UsePublicContracts<ContractAttribute>()` (`…Infrastructure/Extensions.cs:86`) `[convey]`, which
publishes the JSON shape of every marked type at a well-known endpoint.

**Lifecycle.** Applied to `AddParcel`, `DeleteParcel`, `ParcelAdded`, `ParcelDeleted`,
`AddParcelRejected` and `DeleteParcelRejected`.

**Invariants & enforcement.** **Unlike `orders-service`, this service's coverage is complete** — every
public message type carries the attribute, where `orders-service` has five types that lack it
(`component-internals/orders-service.md` §3.36). Nothing enforces that; it is currently true by
discipline, and the next message added is one omission away from breaking it. There is no test, no
analyzer and no build step that checks the marker.

Note what the attribute does *not* do: it has no effect on serialisation, routing or validation. A
type without it still works; it is simply invisible to whatever consumes the contracts endpoint.

**Extension procedure.** Put `[Contract]` on every new command, event and rejected event. Add it in
the same commit as the type, not afterwards.

**Failure modes.** A missing marker is undetectable from inside the service — it surfaces only as an
absent entry in a document a consumer reads.

### 3.32 Queue naming and message conventions

**Definition.** The `rabbitMq` section of `…Api/appsettings.json` fixes the async surface: a
connection name of `parcels-service`, three retries two seconds apart, **`conventionsCasing` set to
`snakeCase`**, a durable **topic** exchange named **`parcels`**, and a queue template of
`parcels-service/{{exchange}}.{{message}}`. The message context header is `message_context` and the
span header is `span_context`.

**Representation & storage.** The casing convention is what turns `AddParcel` into the routing key
`add_parcel` and `ParcelAddedToOrder` into `parcel_added_to_order` `[convey]`. The queue template
combines the **producer's** exchange with the message name, so the five subscriptions
(`…Infrastructure/Extensions.cs:89-95`) produce:

| Subscription | Queue |
| --- | --- |
| `AddParcel` | `parcels-service/parcels.add_parcel` |
| `DeleteParcel` | `parcels-service/parcels.delete_parcel` |
| `OrderCanceled` | `parcels-service/orders.order_canceled` |
| `OrderDeleted` | `parcels-service/orders.order_deleted` |
| `ParcelAddedToOrder` | `parcels-service/orders.parcel_added_to_order` |
| `ParcelDeletedFromOrder` | `parcels-service/orders.parcel_deleted_from_order` |
| `CustomerCreated` | `parcels-service/customers.customer_created` |

**Invariants & enforcement.** The exchange a subscription binds to comes from the event class's
`[Message("…")]` attribute, **not** from this configuration — the configured `parcels` exchange is
only the publish target. That separation is exactly what `orders-service` got wrong in the other
direction (§3.19). Nothing validates that a declared exchange exists or that a binding ever receives
traffic; a wrong exchange name produces a silent, permanently empty queue.

**Extension procedure.** Renaming a message type renames its routing key and therefore its queue —
every existing message in the old queue is orphaned. Renaming the exchange orphans every binding at
once. Both are deploy-ordering problems with no in-code guard; treat a rename as add-new,
dual-publish, drain, remove-old.

**Failure modes.** Silent empty queues from a wrong exchange name; orphaned messages after a rename;
and — because `declare` is true for both exchange and queue — a mistyped name creating a new,
plausible-looking, permanently idle topology rather than failing.

### 3.33 `MessageToLogTemplateMapper` and log redaction

**Definition.** `…Infrastructure/Logging/MessageToLogTemplateMapper.cs` — a static dictionary of
**seven** templates (`:12-70`), returning `null` for any unmapped type (`:75`). Wired by
`AddHandlersLogging()` (`…Infrastructure/Logging/Extensions.cs:10-19`), which registers the mapper as
a singleton (`:14`) and applies command- and event-handler logging over the `Application` assembly
(`:17-18`) `[convey]`.

**Representation & storage.** The seven entries cover both commands (`AddParcel` `:16`, `DeleteParcel`
`:23`) and all five subscribed external events (`CustomerCreated` `:30`, `OrderCanceled` `:43`,
`OrderDeleted` `:50`, `ParcelAddedToOrder` `:57`, `ParcelDeletedFromOrder` `:64`).

**Invariants & enforcement.** Coverage of *handled* messages is complete — every handler in the
service has a template. Two structural gaps remain:

1. **Only `After` is populated** on six of the seven. There is no `Before` anywhere, so a handler that
   never completes leaves no trace that it started. Only `CustomerCreated` has an `OnError` map, and
   it has exactly one entry — `CustomerAlreadyExistsException` (`:34-39`). Every other exception in
   the service passes through handler logging without a message, including all four
   `AddParcel` guards.
2. **The templates are keyed to `typeof(T)`**, so renaming a message class silently drops its
   template — the dictionary lookup simply misses and `Map` returns `null` (`:75`).

Note the two order templates say *"Parcels can be added to the new order again"* (`:46`, `:53`) —
plural, describing the intended behaviour. §3.24 establishes that only one parcel is actually
released. **The log line asserts a behaviour the code does not implement**, which is the sort of
discrepancy that keeps a bug hidden during triage.

Redaction is configured separately, in the `logger` section of `…Api/appsettings.json`: an
`excludeProperties` list covering api-key, access-key, client-id, client-secret, connection-string,
password, email, login, secret and token property names, and an `excludePaths` list of `/`, `/ping`
and `/metrics`. Note that **`Name` and `Description` are not excluded** — the customer-supplied parcel
name and description are written to logs verbatim by the `AddParcel` template's `{ParcelId}`/
`{CustomerId}` siblings and by request logging. If those fields can carry personal data, the log sink
inherits it. The `logger.seq.apiKey` is the unchanged placeholder `"secret"`; the file sink writes to
`logs/logs.txt` on a daily interval.

**Extension procedure.** Add a template for every new command and subscribed event, keyed on the type.
Add an `OnError` entry for each exception that path can throw — this is the cheapest available fix for
the silent-drop problems in §3.20, because a template entry produces a log line even when the AMQP
mapper returns `null`. If a new field can carry personal data, add its property name to
`excludeProperties`.

**Failure modes.** Silent template loss on rename; no start-of-handler trace; a log line that states
an untrue plural; and unredacted free-text fields.

### 3.34 Consul, Fabio and the **empty** `httpClient.services` map

**Definition.** `AddConsul()` and `AddFabio()` (`…Infrastructure/Extensions.cs:64-65`) plus
`AddHttpClient()` (`:63`) `[convey]`.

**Representation & storage.** The `consul` section registers the service as `parcels-service` with a
ping endpoint of `ping` every three seconds and removal after three; `fabio` registers the same
service name; `httpClient` sets `type` to `fabio`, three retries, request masking with a `*****`
template — and **an empty `services` map**.

**Invariants & enforcement.** The empty map is correct here and worth stating explicitly, because it
is the clearest structural difference between this service and its neighbours: **`parcels-service`
makes no outbound HTTP calls at all.** There is no `IParcelsServiceClient` equivalent, no
`…Application/Services/Clients/` folder, and no typed client registration. Every dependency it has is
asynchronous. Compare `orders-service`, which registers three typed clients and whose
`httpClient.services` map names them (`component-internals/orders-service.md` §3.34).

The consequence is that this service has **no synchronous failure mode from a downstream outage** —
its only coupling is to Mongo and RabbitMQ.

**Extension procedure.** Adding an outbound call means adding the typed client, registering it, and
adding its logical name to `httpClient.services` in `appsettings.json` **and** in
`appsettings.docker.json` (which has its own `httpClient` block). Missing the docker profile produces
a resolution failure only in containers.

The `address` in the `consul` section of the base profile is `docker.for.win.localhost` — a
Windows-Docker-specific hostname that resolves nowhere else. It is overridden to `parcels-service` in
the docker profile and disabled entirely in `local` and `test`, so it is inert in practice, but it
will confuse anyone running the base profile directly.

**Failure modes.** A service registered under an unroutable address if the base profile is used
as-is; and Fabio routing that depends on registration succeeding, with no fallback if it does not.

### 3.35 Inert JWT configuration and the committed `localhost.cer`

**Definition.** The `jwt` section of `…Api/appsettings.json` names a certificate location of
`certs/localhost.cer`, a valid issuer of `pacco`, audience validation off, and issuer and lifetime
validation on. The file it points at is committed at `…Api/certs/localhost.cer`.

**Representation & storage.** `AddSecurity()` is called at `…Infrastructure/Extensions.cs:77`
`[convey]`. **There is no `AddJwt()` call, no `[Authorize]` attribute, and no authentication
middleware in `…Api/Program.cs`** — the pipeline is `UseInfrastructure()` then
`UseDispatcherEndpoints(...)` then `UseLogging()` and `UseVault()` (`:36-47`).

**Invariants & enforcement.** No code in `src/` reads the `jwt` section. On that evidence the block is
**inert**, and authentication happens entirely at the gateway, which forwards its conclusions in the
`Correlation-Context` header (§3.27). Recorded as **A-3**, with the falsifier stated: a Convey
extension that reads the section implicitly under `AddSecurity()` would overturn it (**Q-1**).

The certificate is a **public** certificate (`.cer`), not a private key, so committing it discloses
nothing secret. It is cited by path only, per the folder convention. Note that `appsettings.local.json`
blanks the certificate location, which is consistent with the block being unused rather than
optional.

**Extension procedure.** If per-service token validation is ever wanted, adding `AddJwt()` and
`UseAuthentication()` is not sufficient on its own: every guard in this service reads
`identity.IsAuthenticated` from the *header*, not from a validated principal (§3.27), so the two
sources would need to be reconciled or the guards rewritten against the principal.

**Failure modes.** A configuration block that looks like enforcement and is not — the most likely
misreading of this service's security posture.

### 3.36 Vault — settings, PKI and dynamic Mongo credentials

**Definition.** `UseVault()` (`…Api/Program.cs:47`) plus the `vault` section of
`…Api/appsettings.json` `[convey]`.

**Representation & storage.** The section enables Vault at a localhost URL with token auth; a KV v2
engine at mount point `kv`, path `parcels-service/settings`; PKI with role name `parcels-service` and
common name `parcels-service.pacco.io`; and a Mongo database lease with role `parcels-service`,
auto-renewal on, and a connection-string template that interpolates `{{username}}` and `{{password}}`.
The `token` and `password` values are the unchanged placeholder literal `"secret"` — quoted here
because the fact that they are defaults *is* the evidence. Vault is disabled in the `docker`, `local`
and `test` profiles.

**Lifecycle.** At startup Vault settings are layered over the configuration; when the Mongo lease is
enabled, the connection string in the `mongo` section is replaced by dynamically issued credentials
and renewed in the background.

**Invariants & enforcement.** Three consequences a maintainer needs:

1. **The effective configuration at runtime may not match any file in the repository.** Anything read
   from `kv/parcels-service/settings` overrides the committed values, so debugging a
   configuration-dependent behaviour by reading `appsettings.json` alone can be wrong.
2. **The Mongo connection string is dynamic when the lease is on**, so a credential expiry is a
   plausible cause of sudden connection failures with no code change.
3. **Vault is off in every non-base profile**, so none of this is exercised by the docker compose
   stack or by tests. The base profile is the only one that turns it on, and it points at localhost.

**Extension procedure.** A new secret goes into the KV path, not into `appsettings.json`. When adding
one, confirm the key name matches the configuration binding exactly — a mismatch falls back to the
committed default silently.

**Failure modes.** Silent fallback to committed defaults on a key mismatch; lease expiry presenting as
an unexplained Mongo outage; and placeholder credentials that would grant broad Vault access if the
base profile ever reached a shared environment.

### 3.37 Environment layering — and the `tests` vs `test` mismatch

**Definition.** Five profiles ship: `appsettings.json` (base) plus `development` (an **empty object**),
`docker`, `local` and `test`.

**Representation & storage.**

| Profile | Selected by | What it changes |
| --- | --- | --- |
| base | default | everything above; Vault, Consul, Fabio, Jaeger, metrics, outbox all **on**, all pointed at localhost |
| `development` | `ASPNETCORE_ENVIRONMENT=development` | **nothing — the file is `{}`** |
| `docker` | `Dockerfile:10` | container hostnames (`mongo`, `rabbitmq`, `redis`, `consul`, `fabio`, `jaeger`, `seq`); Vault **off**; file logging off |
| `local` | `scripts/start.sh:2` | Consul, Fabio, Jaeger, metrics, Vault **and the outbox** off; verbose logging; blank JWT certificate path |
| `test` | `scripts/start-test.sh:2` | database `test-parcels-service`; Consul, Fabio, Jaeger, metrics, Vault off |

**Invariants & enforcement.** **`scripts/test.sh:2` exports `ASPNETCORE_ENVIRONMENT=tests` — plural —
while the profile file is `appsettings.test.json`.** No file matches `tests`, so `dotnet test` runs
against the **base** profile: the production-shaped one, with Vault enabled, Consul enabled, the
outbox enabled and the database named `parcels-service` rather than `test-parcels-service`. Since
`.travis.yml:14` invokes `./scripts/test.sh`, this is what CI would use. `scripts/start-test.sh:2`
gets it right (`test`), which is what makes the mismatch easy to miss — the two scripts disagree with
each other.

The failure is **silent**: an unmatched environment name is not an error in ASP.NET Core, it simply
layers nothing `[framework]`. Today the mismatch has no observable effect because the solution
contains no test projects to run (§3.38) — `dotnet test` finds nothing. It becomes live the moment a
test project is added to the solution, and it will then look like a test-isolation bug rather than a
one-character typo.

**Extension procedure.** Change `scripts/test.sh:2` to `test` in the same change that adds a test
project to the solution — the two are a single fix. A new profile needs a file *and* a way to select
it; the empty `development` file shows how easily one half ships without the other.

**Failure modes.** CI running against production-shaped configuration; tests writing to the live
database name; and the `development` profile silently inheriting every localhost default.

### 3.38 The PACT provider test that never runs, and its Mongo fixture

**Definition.** `tests/Pacco.Services.Parcels.PactProviderTests/` — one test class, two fixture
classes, a `.csproj` and an `appsettings.json`.

**Representation & storage.** `ParcelsApiPactProviderTests.Pact_Should_Be_Verified` (`:16-26`) inserts
a fixed `ParcelDocument` (`:30-37`, id `c68a24ea-…`, `Size.Huge`, `Variant.Weapon`) and then verifies
`PactVerifier.Create(_httpClient).Between("orders", "parcels").RetrievedFromFile("../../../../../../pacts")`
(`:21-25`) — file-based, six levels up, **no broker**. The provider is hosted in-process via
`new TestServer(Program.GetWebHostBuilder(new string[0]))` (`:46`), which is why
`GetWebHostBuilder` is `public` (`…Api/Program.cs:28`).

**Invariants & enforcement.** Five things make this test non-operational, and they compound:

1. **The test project is not in `Pacco.Services.Parcels.sln`** — the solution lists four projects
   (`:8-14`), all under `src/`. `dotnet test` at the root therefore runs nothing. This mirrors the
   consumer half in `orders-service`, which is likewise excluded from its own solution
   (`component-internals/orders-service.md` §3.45): **both ends of the contract are disabled**, so the
   pact is verified by nobody.
2. **`../../../../../../pacts` resolves outside the repository** — six levels above the test's output
   directory, i.e. a sibling of the clone. Nothing in this repository creates that directory.
3. **`MongoDbFixture` hard-codes `mongodb://localhost:27017`** (`Fixtures/MongoDbFixture.cs:21`),
   ignoring the `mongo` section entirely, and **`Dispose` calls `_client.DropDatabase(_databaseName)`**
   (`:67`). The database name is passed as `"test-parcels-service"` (`ParcelsApiPactProviderTests.cs:45`),
   so as written it drops the test database — but see §3.37: under the `tests` typo the *application*
   under test would be using `parcels-service`, and the fixture would be inserting into and dropping a
   different database than the one being read.
4. **`InitializeMongo()` is commented out** (`MongoDbFixture.cs:25`). That call is what registers the
   `convey_conventions` pack — `IgnoreExtraElements`, `EnumRepresentationConvention(BsonType.String)`
   and camel-case element names (`Fixtures/MongoDbFixtureInitializer.cs:53-55`). The fixture therefore
   depends on the application host, constructed on the next line (`:46`), having registered those
   conventions globally first. It happens to work because BSON conventions are process-global and the
   host is built before `InsertAsync` runs — but it is an ordering dependency expressed nowhere, and
   it is the *reason* §3.13's storage question is answerable at all.
5. The fixture's seeded document sets no `CustomerId`, `Description` or `OrderId`, so it would fail
   `AsEntity`'s description validation (§3.9) if read through the repository. It is read through
   `GetParcelHandler`, which maps document → DTO directly (§3.28), so it survives — an accidental
   dependency on the read path bypassing the domain.

**Extension procedure.** To revive this: add the project to the solution, fix `scripts/test.sh` (§3.37),
decide where pact files live and create that path, un-comment `InitializeMongo()` so the fixture no
longer depends on host construction order, and coordinate with `orders-service`'s consumer half —
reviving one end alone produces either an unverified pact or a missing-file failure. Whether the
exclusion is deliberate is **Q-3**.

**Failure modes.** A contract test that exists, compiles, and asserts nothing; and a fixture whose
`Dispose` drops a database whose name depends on a typo in another file.

### 3.39 Redis and `IDateTimeProvider`

**Definition.** Two small registrations that are easy to misread as more than they are.

**`AddRedis()`** (`…Infrastructure/Extensions.cs:70`) with a `redis` section giving a connection
string and the key prefix `parcels:`. **No code in `src/` reads or writes a Redis key** — there is no
`IDistributedCache` injection, no cache decorator and no rate limiter. On that evidence Redis is
registered and unused here, the same as in `orders-service`
(`component-internals/orders-service.md` §3.44); the most likely reason is that it backs a Convey
component (distributed locking or the outbox) rather than application code, which is
**`Unverifiable — Missing Source Evidence`**.

**`IDateTimeProvider`** — `…Infrastructure/Services/DateTimeProvider.cs`, three lines:
`public DateTime Now => DateTime.UtcNow` (`:8`), registered as a singleton at
`…Infrastructure/Extensions.cs:50`. Injected only by `AddParcelHandler`, which uses it for
`CreatedAt` (`AddParcelHandler.cs:47`).

**Invariants & enforcement.** The provider's value is that `CreatedAt` is injectable and therefore
testable. Note the property is named `Now` and returns `UtcNow`: a maintainer who reads the interface
without the implementation will reasonably assume local time. Nothing enforces UTC — the `Parcel`
constructor takes whatever `DateTime` it is given, with no `Kind` check, and `ParcelDocument.CreatedAt`
stores it as-is.

**Extension procedure.** Take `IDateTimeProvider` in any new handler that needs the current time
rather than calling `DateTime.UtcNow` directly; the existing singleton registration covers it with no
further wiring. If the naming is fixed, rename the member to `UtcNow` rather than changing what it
returns — changing the value silently shifts every new `CreatedAt`.

**Failure modes.** An unused Redis dependency that still has to be running for startup to succeed if
Convey resolves it eagerly; and a `Now`/`UtcNow` naming mismatch that invites a local-time bug.

### 3.40 Deployment identity — port, image, environment names

**Definition.** The facts that identify this service to the platform, spread across four files.

| Fact | Value | Source |
| --- | --- | --- |
| Service name | `parcels-service` | the `app`, `consul` and `fabio` sections |
| Jaeger service name | **`parcels`** — not `parcels-service` | the `jaeger` section |
| Host port | `5007` | `hianshul100_Pacco/compose/services.yml:87-92` (`5007:80`); `README.md`; `Pacco.Services.Parcels.rest:1` |
| Container port | `80` | `Dockerfile:9` (`ASPNETCORE_URLS http://*:80`) |
| Image | `devmentors/pacco.services.parcels` | `compose/services.yml:88`; built as `$DOCKER_USERNAME/pacco.services.parcels` by `scripts/dockerize.sh:16` |
| Container environment | `docker` | `Dockerfile:10` |
| Mongo database | `parcels-service` (`test-parcels-service` under the `test` profile) | the `mongo` section |
| Exchange | `parcels` | the `rabbitMq` section |

**Representation & storage.** The Dockerfile is a two-stage SDK-3.1 → ASP.NET-3.1 build publishing
only `src/Pacco.Services.Parcels.Api` (`:4`). CI is Travis: `build.sh`, then `test.sh`, then
`dockerize.sh` on success (`.travis.yml:12-16`), restricted to `master` and `develop` (`:6-9`).
`dockerize.sh` tags `latest` + build number on master and `dev-<build>` on develop (`:5-14`).

**Invariants & enforcement.** Four names for one service — `parcels-service` to Consul, Fabio, Mongo
and the queue template; `parcels` to Jaeger and RabbitMQ; `pacco.services.parcels` to Docker; and
`Pacco Parcels Service` as the display name. Nothing keeps them consistent, and the Jaeger/exchange
short form is the one that surprises: a trace search for `parcels-service` finds nothing.

`dockerize.sh` runs `docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD` (`:18`) from CI-injected
environment variables. No credential is committed; the variables are supplied by the Travis
environment. On any branch other than `master` or `develop` both `TAG` and `VERSION_TAG` are empty
(`:2-14`), so the script would build and push malformed tags — but the branch filter at
`.travis.yml:6-9` prevents CI from reaching it. Running the script by hand off those branches is the
hazard.

**Extension procedure.** Changing the port means changing the compose mapping, the `consul` section's
port, the README and the `.rest` file. Changing the service name touches Consul, Fabio, the queue
template, the Vault paths, the Mongo database name and `compose/services.yml` — and the database name
change requires a data migration, since nothing renames a Mongo database in place.

**Failure modes.** Name drift between the six identifiers; and a hand-run `dockerize.sh` pushing
empty-tagged images.

---

## 4. Primary control flows

Each flow is traced entry point → functions → datastore and side effects. Line citations are to the
files named in §3.

### 4.1 `POST /parcels` — create a parcel over HTTP

| # | Step | Where |
| --- | --- | --- |
| 1 | Gateway matches its sync `parcels` POST route and forwards, adding `Correlation-Context` | `…APIGateway/ntrada.yml` `[ntrada]` |
| 2 | `Post<AddParcel>("parcels", afterDispatch: …)` binds the JSON body onto the command | `…Api/Program.cs:44-45` `[convey]` |
| 3 | `AddParcel`'s constructor substitutes a fresh GUID if `ParcelId` is `Guid.Empty` | §3.15 |
| 4 | `OutboxCommandHandlerDecorator` mints a **fresh** message id (no AMQP properties on an HTTP request) and calls the inner handler through the outbox | `OutboxCommandHandlerDecorator.cs:27-34` |
| 5 | `Enum.TryParse<Variant>` → `InvalidParcelVariantException` on failure | `AddParcelHandler.cs:31-34` |
| 6 | `Enum.TryParse<Size>` → `InvalidParcelSizeException` on failure | `:36-39` |
| 7 | `_customerRepository.ExistsAsync(command.CustomerId)` → **read `customers`** → `CustomerNotFoundException` on miss | `:41-44` |
| 8 | `new Parcel(…, _dateTimeProvider.Now)` → name/description validation | `:46-47`; §3.9 |
| 9 | `_parcelRepository.AddAsync` → `AsDocument()` → **insert into `parcels`** | `:48`; `ParcelMongoRepository.cs:33` |
| 10 | `PublishAsync(new ParcelAdded(command.ParcelId))` → outbox → exchange `parcels`, key `parcel_added` | `:49`; §3.22 |
| 11 | `afterDispatch` writes `201 Created`, `Location: parcels/{ParcelId}` | `Program.cs:44-45` |

**Side effects:** two collection reads, one insert, one outbox row. **No ownership check occurs at any
step** (§3.17). Any exception from steps 5–9 becomes HTTP 400 via `ExceptionToResponseMapper` (§3.26).
Steps 9 and 10 are **not atomic** (§3.25).

### 4.2 `add_parcel` over AMQP — the same handler, a different failure surface

Steps 3–10 above are identical; only the ends differ.

| # | Step | Where |
| --- | --- | --- |
| 1 | Gateway async profile publishes to exchange `parcels`, key `add_parcel`, with `resourceId` generated upstream | `…APIGateway/ntrada-async.yml` `[ntrada]` |
| 2 | Subscriber on `parcels-service/parcels.add_parcel` deserialises and dispatches | `…Infrastructure/Extensions.cs:89`; §3.32 |
| 3 | The outbox decorator uses the **inbound `MessageId`**, so redelivery is deduplicated | `OutboxCommandHandlerDecorator.cs:27-29` |
| 4 | …handler body as §4.1 steps 5–10… | |
| 5 | On exception, `ExceptionToMessageMapper.Map` runs | `ExceptionToMessageMapper.cs:15-25` |
| 6 | All four guard exceptions map to `AddParcelRejected(reason, code)` → published to `parcels` | `:16-20` |
| 7 | `operations-service` correlates the rejection to the client's operation id | `component-internals/operations-service.md` §3.9 |

**This is the only fully instrumented failure path in the service**: every exception the handler can
raise has a mapper arm. Note the rejected event carries **no parcel id** (§3.19), so a client with
several adds in flight cannot attribute the failure — it must rely on `operations-service`'s
correlation instead.

### 4.3 `DELETE /parcels/{parcelId}` — and its two mis-reported AMQP failures

| # | Step | Where |
| --- | --- | --- |
| 1 | `Delete<DeleteParcel>("parcels/{parcelId}")` binds the route value | `…Api/Program.cs:43` `[convey]` |
| 2 | `_parcelRepository.GetAsync(command.ParcelId)` → **read `parcels`** | `DeleteParcelHandler.cs:27` |
| 3 | Null → `ParcelNotFoundException` | `:28-31` |
| 4 | Ownership guard, gated on `identity.IsAuthenticated &&` | `:33-37`; §3.17 |
| 5 | `if (parcel.AddedToOrder)` → `CannotDeleteParcelException` | `:39-42` |
| 6 | `_parcelRepository.DeleteAsync` → **hard delete from `parcels`** | `:44` |
| 7 | `PublishAsync(new ParcelDeleted(command.ParcelId))` → exchange `parcels`, key `parcel_deleted` | `:45` |

**On AMQP the failure reporting is wrong twice over** (§3.20): step 3's exception becomes
`DeleteParcelRejected(Guid.Empty, …)` — correct event, id discarded — and step 4's becomes
**`AddParcelRejected`**, an event about a different operation entirely. Only step 5 reports correctly.

**Nothing consumes the `parcel_deleted` published at step 7** (§3.19): `orders-service` declares its
matching subscription on the `deliveries` exchange, so the binding never receives it. An order
therefore continues to reference a parcel that no longer exists — although in practice step 5 makes
that unreachable, since a parcel on an order cannot be deleted. The two defects mask each other.

### 4.4 `parcel_added_to_order` — attaching a parcel

| # | Step | Where |
| --- | --- | --- |
| 1 | `orders-service` publishes on exchange `orders` after `Order.AddParcel` succeeds | `component-internals/orders-service.md` §3.24 |
| 2 | Subscriber on `parcels-service/orders.parcel_added_to_order` dispatches | `…Infrastructure/Extensions.cs:93` |
| 3 | `OutboxEventHandlerDecorator` deduplicates on the inbound message id | `OutboxEventHandlerDecorator.cs:27-34` |
| 4 | `_parcelRepository.GetAsync(@event.ParcelId)` → **read `parcels`** | `ParcelAddedToOrderHandler.cs:18` |
| 5 | Null → **`return;`** — silent, unlogged | `:19-22` |
| 6 | `parcel.AddToOrder(@event.OrderId)` — unconditional, overwrites any existing `OrderId` | `:24`; §3.8 |
| 7 | `_parcelRepository.UpdateAsync` → **whole-document replace**, `AddedToOrder` recomputed to `true` | `:25`; §3.7 |

**No event is published.** `parcel_deleted_from_order` is the exact mirror, calling `DeleteFromOrder`.
Step 6 is the reason a parcel can be silently moved between orders: nothing checks whether `OrderId`
is already set.

### 4.5 `order_canceled` / `order_deleted` — where parcels get stranded

| # | Step | Where |
| --- | --- | --- |
| 1 | `orders-service` publishes on exchange `orders` | `component-internals/orders-service.md` §3.24 |
| 2 | Subscriber on `parcels-service/orders.order_canceled` (resp. `…order_deleted`) dispatches | `…Infrastructure/Extensions.cs:91-92` |
| 3 | `_parcelRepository.GetByOrderAsync(@event.OrderId)` → **`GetAsync(p => p.OrderId == orderId)`, a collection scan returning ONE document** | `OrderCanceledHandler.cs:18`; `ParcelMongoRepository.cs:28` |
| 4 | Null → **`return;`** | `:19-22` |
| 5 | `parcel.DeleteFromOrder()` → `OrderId = null` | `:24` |
| 6 | `UpdateAsync` → replace; `AddedToOrder` recomputed to `false` | `:25` |
| 7 | Handler returns. **The event is consumed. Any remaining parcels on that order are never visited.** | §3.24 |

For an order carrying *n* parcels, *n − 1* are left with a dangling `OrderId`: hidden from the default
listing, undeletable, and unreachable by any future event. This is **B-1**; the fix and the required
backfill are in §5.4 and §7.3.

The log line emitted for this handler reads *"Parcels can be added to the new order again"* — plural
(`MessageToLogTemplateMapper.cs:46,53`), asserting behaviour the code does not implement (§3.33).

### 4.6 `GET /parcels/volume` — the only read that touches the domain

| # | Step | Where |
| --- | --- | --- |
| 1 | `Get<GetParcelsVolume, ParcelsVolumeDto>("parcels/volume")`, registered **before** `parcels/{parcelId}` | `…Api/Program.cs:40`; §3.30 |
| 2 | Empty or null `ParcelIds` → `Volume = 0`, returned immediately | `GetParcelsVolumeHandler.cs:27-33` |
| 3 | `FindAsync(p => query.ParcelIds.Contains(p.Id))` → **`$in` scan over `parcels`**, unbounded | `:35` |
| 4 | `documents.Select(d => d.AsEntity())` → constructor validation on every document | `:36`; §3.9 |
| 5 | `_parcelsService.CalculateVolume(parcels)` → side-length lookup, cube, sum | `:37`; §3.5 |
| 6 | `new ParcelsVolumeDto { Volume = volume }` | `:39-42` |

**Ids that match nothing are silently absent** — the caller receives a confident number computed over
a subset (§3.29). Step 4 throws if any stored document has a blank name or description; step 5 throws
`KeyNotFoundException` if any `Size` is missing from the table (§3.4). Both surface as HTTP 400.

### 4.7 `customer_created` — seeding the replica

| # | Step | Where |
| --- | --- | --- |
| 1 | `customers-service` publishes on exchange `customers` | `component-internals/customers-service.md` §3.20 |
| 2 | Subscriber on `parcels-service/customers.customer_created` dispatches | `…Infrastructure/Extensions.cs:95` |
| 3 | `_customerRepository.ExistsAsync(@event.CustomerId)` → **read `customers`** | `CustomerCreatedHandler.cs:20` |
| 4 | Exists → `throw new CustomerAlreadyExistsException` | `:20-23` |
| 5 | Otherwise `AddAsync(new Customer(@event.CustomerId))` → **insert into `customers`** | `:25` |

Step 4 has **no arm in `ExceptionToMessageMapper`**, so a redelivered `customer_created` throws, is
mapped to `null`, and is **silently acknowledged** (§3.20). The one saving grace is that
`MessageToLogTemplateMapper` does carry an `OnError` entry for exactly this exception
(`:34-39`), so it appears in the log even though it produces no rejected event — the only exception in
the service with that treatment (§3.33).

Until this flow runs for a given customer, **that customer cannot create parcels**: `AddParcelHandler`
step 7 (§4.1) fails with `CustomerNotFoundException`. The replica is a hard prerequisite, not a cache.

### 4.8 `GET /parcels` and `GET /parcels/{parcelId}`

Both are thin, and their asymmetry is the point (§3.28).

- **`GET /parcels/{parcelId}`** (`Program.cs:41`) → `GetParcelHandler.cs:22` →
  `GetAsync(p => p.Id == query.ParcelId)` → `document?.AsDto()`. **No identity check.** A miss returns
  a null DTO, rendered as an empty body — not a 404.
- **`GET /parcels`** (`Program.cs:42`) → `GetParcelsHandler.cs:29-48` → `AsQueryable()`, an optional
  ownership-gated `CustomerId` filter, an optional `!AddedToOrder` filter, `ToListAsync()`,
  `Select(AsDto)`. An ownership mismatch returns **`[]` with `200 OK`**; omitting `customerId`
  entirely skips the check and lists every customer's unassigned parcels.

Neither path constructs a `Parcel`, so neither runs domain validation, and neither is paged.

---

## 5. Persistence & schema evolution

### 5.1 What is stored

Mongo database `parcels-service` (`test-parcels-service` under the `test` profile), per the `mongo`
section of `…Api/appsettings.json`. Four collections, two of them owned by Convey:

| Collection | Bound at | Document | Written by |
| --- | --- | --- | --- |
| `parcels` | `…Infrastructure/Extensions.cs:75` | `ParcelDocument` — 9 fields (`ParcelDocument.cs:9-17`) | `AddParcelHandler`, `DeleteParcelHandler`, all four order-event handlers |
| `customers` | `:74` | `CustomerDocument` — **id only** | `CustomerCreatedHandler` |
| `inbox` | the `outbox` section | Convey-owned | the two handler decorators `[convey]` |
| `outbox` | the `outbox` section | Convey-owned | `MessageBroker` `[convey]` |

There is **no schema-version field** on either application document, no migrations folder, and no
seeding (`mongo.seed` is `false` in every profile). `IgnoreExtraElements` is part of the convention
pack (`tests/…/MongoDbFixtureInitializer.cs:53`) `[convey]`, so **removing a property from a document
type does not fail on read — the stored field is silently ignored and then lost on the next write**.

### 5.2 Adding a field to `Parcel`

Six edits, in this order; the compiler catches only the first three.

1. `…Core/Entities/Parcel.cs` — property with a `private set`, and a **constructor parameter**. Adding
   a constructor parameter rather than an optional property is deliberate: it breaks `AsEntity` at
   compile time, which is the only automatic reminder in this list.
2. `…Mongo/Documents/ParcelDocument.cs` — the storage property.
3. `…Mongo/Documents/Extensions.cs:26-28` — `AsEntity`, forced by step 1.
4. `…Mongo/Documents/Extensions.cs:30-42` — **`AsDocument`. Silent if forgotten**: the field is never
   persisted and reads back as its default on the next load. This is exactly the failure
   `orders-service` shipped with `CancellationReason` (`component-internals/orders-service.md` §3.15).
5. `…Application/DTO/ParcelDto.cs` and `Extensions.cs:44-55` — `AsDto`, if the field is public.
6. `…Application/Commands/AddParcel.cs` and `AddParcelHandler.cs:46-47`, if the field is settable.

**Existing documents are not backfilled.** A non-nullable value type reads back as `0`/`false`, which
is indistinguishable from a legitimately-zero value. If the distinction matters, make the field
nullable and backfill by hand — there is no migration mechanism in this repository.

### 5.3 Changing `Variant` or `Size`

**Answer Q-4 first** (§3.13): run `db.parcels.findOne({}, {size: 1, variant: 1})` against a deployed
environment to establish whether enums are stored as ordinals or as member names. Until then, only
appending is safe.

| Change | If stored as ordinal | If stored as member name |
| --- | --- | --- |
| **Append** | safe — but a new `Size` **also requires** the side-length entry (§3.4) | same |
| **Reorder / insert** | silently relabels every stored parcel; a `Huge` parcel read as `Large` changes its computed volume by a factor of ~3.4 (§3.5) | harmless on disk |
| **Rename** | stored data untouched; the wire string changes in **both** directions, so clients sending the old string are rejected | stored data *and* wire string change; existing documents fail to deserialise |

Before any structural change, pin the representation: add an explicit `[BsonRepresentation(...)]` to
the document property so it stops depending on a globally registered convention, migrate every stored
document to that form, then make the change.

### 5.4 Repairing the stranded parcels

Fixing `GetByOrderAsync` (§3.24, §7.3) stops new stranding; it does **not** repair existing documents,
because the events that would have released them are long consumed. A one-off repair is required, and
it needs a source of truth this service does not hold:

1. Enumerate distinct `orderId` values in `parcels` where `orderId` is not null.
2. For each, ask `orders-service` whether the order still exists and is neither canceled nor deleted.
   There is no bulk endpoint for this — `GET /orders/{orderId}` per id
   (`component-internals/orders-service.md` §6.1) — and no HTTP client exists in this service, so the
   repair is an out-of-band script, not application code.
3. For every order that is gone or canceled, set `orderId` to `null` and `addedToOrder` to `false` on
   its parcels. **Both fields must be written**, because `AddedToOrder` is stored, not computed on
   read (§3.7); updating only `orderId` leaves the parcel hidden from the default listing.

Run the repair **after** deploying the fix, or newly-stranded parcels will appear between the two
steps.

### 5.5 Indexes

**There are none.** No `CreateIndex` call, no index attribute and no init script anywhere in this
repository. Every query except a by-id lookup is a full collection scan:

| Query | Predicate | Cost |
| --- | --- | --- |
| `GetParcelHandler` | `p.Id == …` | `_id` index `[framework]` |
| `GetByOrderAsync` | `p.OrderId == …` | **scan** — and it runs on every `order_canceled`/`order_deleted` |
| `GetParcelsHandler` | `p.CustomerId == …`, `!p.AddedToOrder` | **scan** |
| `GetParcelsVolumeHandler` | `$in` over `_id` | `_id` index |

The two that matter are `orderId` (hot on the event path) and `customerId` (hot on the listing path).
Whether indexes are created out of band is **Q-7**; nothing in the repository suggests they are. Add
them alongside the §7.3 fix, not after it — that change makes the `orderId` scan return *more*
documents, not fewer.

### 5.6 Evolving DTOs and message contracts

- **`ParcelDto`** is serialised straight to JSON. **Adding** a field is backward-compatible; removing
  or renaming one breaks any consumer reading it, including `orders-service`, which deserialises this
  exact shape when it calls `GET /parcels/{parcelId}`
  (`component-internals/orders-service.md` §3.34), and the PACT file, which pins it (§3.38) — though
  nothing verifies that pact today.
- **`ParcelsVolumeDto`** carries a single `Volume`. Adding a "missing ids" field alongside it (§3.29)
  is additive and safe.
- **`ParcelAdded` / `ParcelDeleted`** carry only `ParcelId`. Widening them is safe *because* they have
  no subscribers (§3.19) — this is the cheapest moment in the service's life to fix their shape.
- **`AddParcelRejected` / `DeleteParcelRejected`** are read by `operations-service` via its
  `messages.json` manifest. Renaming either, or changing its routing key, requires a matching edit
  there (§7.6).

Adding a field to any `[Contract]` type also changes what `UsePublicContracts` publishes (§3.31),
which is documentation-only and has no runtime effect.

### 5.7 The customer replica

`customers` holds ids only (§3.6) and is fed exclusively by `customer_created`. Adding a field means
(a) extending `Customer`, `CustomerDocument` and the mappers at `Documents/Extensions.cs:57-64`,
(b) subscribing to whatever event carries updates, and (c) **backfilling** — this service cannot
reconstruct the replica, because `customers-service` publishes no snapshot or replay event
(`component-internals/customers-service.md` §3.20). In practice a backfill means a one-off read
against `customers-service`'s API.

There is also **no deletion path**: nothing subscribes to a customer-deleted event, so a removed
customer remains able to have parcels created against them indefinitely.

---

## 6. Surface → internals map

Every published entry point, classified **read-only**, **mutating** or **absent**, and mapped to the
internals that serve it.

### 6.1 HTTP routes

| Route | Kind | Handler | Guarded? | Notes |
| --- | --- | --- | --- | --- |
| `GET /` | read-only | `Program.cs:39` | — | app name; the Consul ping and `logger.excludePaths` target |
| `GET /parcels/volume` | read-only | `GetParcelsVolumeHandler` | **no** | must stay registered above `parcels/{parcelId}` (§3.30) |
| `GET /parcels/{parcelId}` | read-only | `GetParcelHandler` | **no** | full parcel, any caller (§3.28) |
| `GET /parcels` | read-only | `GetParcelsHandler` | **partial** | only when `customerId` is supplied; failure is a silent `[]` |
| `POST /parcels` | **mutating** | `AddParcelHandler` | **no** | `201 Created` + `Location` (§3.30) |
| `DELETE /parcels/{parcelId}` | **mutating** | `DeleteParcelHandler` | **yes**, with the `IsAuthenticated &&` bypass (§3.17) | |
| `/ping`, `/metrics`, `/docs` | read-only | Convey/AppMetrics/Swagger `[convey]` | — | `swagger.routePrefix` is `docs`; `includeSecurity` is on |

### 6.2 Inbound AMQP commands

| Routing key | Kind | Handler | Rejection event |
| --- | --- | --- | --- |
| `add_parcel` | **mutating** | `AddParcelHandler` | `AddParcelRejected` — all four guards covered |
| `delete_parcel` | **mutating** | `DeleteParcelHandler` | `DeleteParcelRejected` for one guard; **`AddParcelRejected` for the ownership guard**; id lost on not-found (§3.20) |

### 6.3 Inbound AMQP events

| Exchange | Routing key | Kind | Effect on miss |
| --- | --- | --- | --- |
| `orders` | `order_canceled` | **mutating** | silent `return` — and only one parcel released (§3.24) |
| `orders` | `order_deleted` | **mutating** | silent `return` — same |
| `orders` | `parcel_added_to_order` | **mutating** | silent `return` |
| `orders` | `parcel_deleted_from_order` | **mutating** | silent `return` |
| `customers` | `customer_created` | **mutating** | throws on duplicate; **silently acknowledged** (§4.7) |

### 6.4 Outbound events

| Routing key | Payload | Consumers |
| --- | --- | --- |
| `parcel_added` | `ParcelId` | **none** — `operations-service` correlates it by manifest, nothing subscribes (§3.19) |
| `parcel_deleted` | `ParcelId` | **none** — `orders-service`'s matching handler is bound to the `deliveries` exchange (§3.19) |
| `add_parcel_rejected` | reason, code — **no id** | `operations-service` |
| `delete_parcel_rejected` | parcel id, reason, code | `operations-service` |

### 6.5 Absent surface

Things a reader might reasonably expect and which **do not exist**:

- **No update endpoint.** A parcel's name, description, size or variant cannot be changed after
  creation. The only mutable field is `OrderId`, and only events can change it (§1.1).
- **No endpoint or command that attaches a parcel to an order.** That is `orders-service`'s decision,
  reflected here (§3.23).
- **No outbound HTTP client of any kind** (§3.34).
- **No paging, sorting or filtering beyond `customerId` and `includeAddedToOrders`** (§3.28).
- **No health endpoint beyond `GET /`**, no readiness probe distinct from liveness.
- **No `parcel_updated` event**, and no event at all for the four order-driven state changes (§3.21).

---

## 7. Change/extension guide

### 7.1 Add a field to a parcel

Follow §5.2's six edits in order. The one that fails silently is `AsDocument`; verify by writing a
parcel and **reading it back through the API**, not by inspecting the write path.

### 7.2 Add a `Size` or a `Variant`

For `Variant`: append the member. Nothing else is required.

For `Size`: append the member **and** add its side length to `_parcelSideLengths`
(`…Core/Services/ParcelsService.cs:10-18`). Skipping the second edit compiles cleanly and throws
`KeyNotFoundException` the first time a parcel of that size appears in a volume query (§3.5). This is
the single most important extension constraint in the service, and nothing in the code marks it —
consider adding a `default` case to the lookup that throws a named domain exception, so the failure is
attributable.

For either: **do not reorder or rename** until §5.3's storage question is answered.

### 7.3 Fix the stranded parcels (B-1)

The highest-value change available. Four parts, all required:

1. `…Core/Repositories/IParcelRepository.cs` — change `GetByOrderAsync` to return
   `Task<IEnumerable<Parcel>>`.
2. `ParcelMongoRepository.cs:26-31` — implement with `FindAsync(p => p.OrderId == orderId)`, already
   used at `GetParcelsVolumeHandler.cs:35` `[convey]`.
3. `OrderCanceledHandler.cs` and `OrderDeletedHandler.cs` — iterate, calling `DeleteFromOrder` and
   `UpdateAsync` per parcel. There is no transaction, so a partial failure must be safe to redeliver:
   it is, because `DeleteFromOrder` on an already-detached parcel is a no-op (§3.8).
4. Add an index on `orderId` (§5.5) — this change makes the scan return more documents, not fewer.

Then run the §5.4 backfill. Deploy the code first, repair second.

### 7.4 Close the identity holes

Two independent changes; do them together, because fixing one alone leaves a plausible-looking but
still-open surface.

1. **`DeleteParcelHandler.cs:34`** — invert to
   `if (!identity.IsAuthenticated || (identity.Id != parcel.CustomerId && !identity.IsAdmin))`. Then
   do the same in `GetParcelsHandler.cs:33`, where the same prefix appears.
2. **`AddParcelHandler`** — inject `IAppContext` and stop trusting the payload's `CustomerId`: either
   reject a mismatch or ignore the payload and use `identity.Id`. The second is better and changes the
   command's effective contract, so coordinate with the gateway's `customerId:@user_id` binding
   `[ntrada]`.

Consider also adding an ownership check to `GetParcelHandler` — but **not** without first confirming
that `orders-service`'s server-to-server call carries a correlation context that would satisfy it
(**Q-2**). Adding the guard blind will break `AddParcelToOrder`
(`component-internals/orders-service.md` §3.34).

Before any of this, establish what identity internal publishers actually supply (**Q-2**); every one
of these guards depends on it.

### 7.5 Fix the AMQP rejection reporting

Three edits in `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs`:

1. `:24` — `UnauthorizedParcelAccessException` must produce a **`DeleteParcelRejected`**, carrying the
   exception's parcel id. It is thrown only by `DeleteParcelHandler`.
2. `:21-23` — pass the exception's own id instead of `Guid.Empty`.
3. Add an arm for `CustomerAlreadyExistsException`, or accept that a duplicate `customer_created` is
   silent by design and say so in a comment.

Then check `…Operations…/messages.json` recognises anything renamed. Adding an `OnError` entry to
`MessageToLogTemplateMapper` (§3.33) is the cheap complement: it produces a log line even for
exceptions the AMQP mapper drops.

### 7.6 Add a new command

1. `…Application/Commands/<Name>.cs` — `[Contract]`, `ICommand`, get-only properties.
2. `…Application/Commands/Handlers/<Name>Handler.cs` — order the guards deliberately; ownership before
   business rules (§3.18).
3. `…Api/Program.cs` — register the route, **literal paths above parameterised ones** (§3.30).
4. `…Infrastructure/Extensions.cs` — `SubscribeCommand<T>()` if it should also be accepted over AMQP.
5. A rejected event per failure mode, `[Contract]` and `IRejectedEvent`, **carrying the resource id**.
6. A mapper arm per exception in `ExceptionToMessageMapper` — omissions are silent (§3.20).
7. A `MessageToLogTemplateMapper` entry, with `OnError` for each exception (§3.33).
8. An entry in `…Operations…/messages.json` so the operation is correlatable.
9. The gateway route, sync or async.

Steps 5–8 are the ones that get skipped; each is invisible from inside the service.

### 7.7 Add a subscription to another service's event

1. `…Application/Events/External/<Name>.cs` — `[Message("<producer's exchange>")]`, `IEvent`. **The
   exchange name is the producer's, not `parcels`.** This is precisely the mistake `orders-service`
   made in the opposite direction (§3.19), and it produces a silent, permanently empty queue.
2. The handler — decide explicitly what a lookup miss means, and prefer logging it over a bare
   `return` (§3.23).
3. `SubscribeEvent<T>()` in `…Infrastructure/Extensions.cs`.
4. A log template (§3.33).

Verify by checking that the queue `parcels-service/<exchange>.<routing_key>` exists **and has a
non-zero message count** after a known publish. A wrong exchange name produces a queue that looks
correct and is never fed.

### 7.8 Make the read side safe under load

In rough order of value:

1. Add indexes on `orderId` and `customerId` (§5.5).
2. Page `GET /parcels` (§3.28) — note this changes the response from a bare array to a wrapper, so
   version it or add paging as opt-in query parameters that default to the current behaviour.
3. Cap `GetParcelsVolume`'s `ParcelIds` and report ids that were not found (§3.29).
4. Require a `customerId` on `GET /parcels`, or apply the caller's identity when it is omitted —
   today the unfiltered listing is the widest read in the service.

### 7.9 Maintenance contract

**Any later phase that changes this component's internals must update this model in the same change.**
Concretely, a change to any of the following invalidates a section here and must be reflected in it:

| Change | Sections to update |
| --- | --- |
| A field on `Parcel` or `ParcelDocument`, or any mapper | §3.1, §3.12, §5.2 |
| A `Variant` or `Size` member, or the side-length table | §3.3, §3.4, §3.5, §5.3, §7.2 |
| A command, event, subscription or rejected event | §2, §3.15–§3.23, §6.2–§6.4 |
| A guard, or anything reading `IAppContext` | §3.17, §3.27, §6.1, §7.4 |
| A route, or route ordering | §3.30, §6.1 |
| `ExceptionToMessageMapper` or `ExceptionToResponseMapper` | §3.20, §3.26, §7.5 |
| Any `appsettings` section, script, Dockerfile or `.travis.yml` | §3.32–§3.37, §3.40 |
| The solution file or the test project | §3.38 |

If a defect recorded here is fixed, **remove or amend the entry rather than leaving it**; a model that
describes bugs which no longer exist is worse than no model, because it is trusted. The concept count
in `component-internals/index.md` §1 must be updated in the same change (§8.4).

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

Each is stated so it can be falsified, with the evidence it rests on.

- **A-1.** `Convey 0.4.*` behaves as its usage implies: `AddMongoRepository` binds a collection by
  name; `SubscribeCommand`/`SubscribeEvent` declare one queue per `(service, exchange, message)`;
  `UseDispatcherEndpoints` binds route values and the request body onto the command and matches routes
  in registration order; a `null` from `IExceptionToMessageMapper` means "acknowledge and drop";
  `_outbox.HandleAsync(id, …)` deduplicates on `id`. **None of this is verifiable in this workspace** —
  the package has no source here. Falsified by reading Convey 0.4.x's source.
- **A-2.** `AddMongo()` registers the same convention pack the test fixture reproduces —
  `IgnoreExtraElements`, `EnumRepresentationConvention(BsonType.String)` and camel-case element names
  (`tests/…/Fixtures/MongoDbFixtureInitializer.cs:53-55`). Held with moderate confidence only; §3.13
  and **Q-4** record the contrary evidence. The `IgnoreExtraElements` half is the safer of the two,
  since it matches the observed tolerance for the un-mapped `AddedToOrder` field.
- **A-3.** The `jwt` configuration block is inert. Asserted from the absence of any reader in `src/`
  and of `AddJwt()`/`UseAuthentication()`; falsified by a Convey extension that reads the section
  implicitly under `AddSecurity()` — see **Q-1**.
- **A-4.** The gateway is the only externally reachable path to this service, so the identity holes in
  §3.17 are currently mitigated by network topology. Asserted from `compose/services.yml:87-94`,
  which places the service on the internal `pacco` network — but it also maps host port `5007`, so on
  a developer machine the service *is* directly reachable. Falsified by any ingress that exposes the
  container outside the compose network.
- **A-5.** Redis is registered for a Convey component rather than for application code. Asserted from
  the absence of any `IDistributedCache` or cache-decorator usage in `src/` (§3.39).
- **A-6.** `parcel_added` and `parcel_deleted` have no subscriber **anywhere in the platform**.
  Asserted from a search for `SubscribeEvent<ParcelAdded>`/`SubscribeEvent<ParcelDeleted>` across every
  clone in this workspace. Falsified by a consumer outside these repositories.

### 8.2 Blockers

Defects that a maintainer must know about before changing this component. Each states its trigger and
its consequence.

- **B-1. `GetByOrderAsync` releases one parcel per order event.** For an order with *n* parcels,
  `order_canceled`/`order_deleted` strands *n − 1* of them: hidden from the default listing,
  undeletable, unreachable by any future event. Highest severity in the component. §3.24, §4.5;
  fix in §7.3, repair in §5.4.
- **B-2. `AddParcel` performs no ownership check at all.** `CustomerId` is taken from the request body
  and never compared to the caller, so any caller with network reach can create parcels for any
  existing customer. §3.17.
- **B-3. Every ownership guard is gated on `identity.IsAuthenticated &&`.** An *unauthenticated*
  caller passes `DeleteParcelHandler`'s guard and `GetParcelsHandler`'s filter. Identity is read from
  a header this service never validates (§3.27), so asserting "not authenticated" is trivial. §3.17,
  §3.28.
- **B-4. `UnauthorizedParcelAccessException` is mapped to `AddParcelRejected`.** A refused AMQP
  *delete* is reported as a failed *add*, on an event that carries no parcel id. `ParcelNotFoundException`
  additionally discards the id it holds. `CustomerAlreadyExistsException` is unmapped and silent.
  §3.20, §4.3.
- **B-5. `parcel_added` and `parcel_deleted` have no subscriber.** `orders-service` declares its
  matching `ParcelDeleted` handler on the `deliveries` exchange, so the intended integration exists on
  both sides and is broken by one attribute value. §3.19,
  `component-internals/orders-service.md` §3.33.
- **B-6. No indexes exist.** `GetByOrderAsync` scans the collection on **every** order-cancel and
  order-delete event, and `GetParcels` scans on every listing. §5.5.
- **B-7. `scripts/test.sh` exports `ASPNETCORE_ENVIRONMENT=tests`, and no such profile exists.** CI
  (`.travis.yml:14`) therefore runs against the base profile — Vault on, Consul on, database
  `parcels-service`. Currently masked because the solution contains no test projects. §3.37.
- **B-8. The PACT provider test is excluded from `Pacco.Services.Parcels.sln`, as is its consumer
  counterpart in `orders-service`.** The `orders → parcels` contract is verified by nobody. The pact
  path also resolves outside the repository, and the fixture's `InitializeMongo()` is commented out.
  §3.38.
- **B-9. A new `Size` requires a second, uncompiled edit.** Appending to `Size.cs` without adding a
  side length to `_parcelSideLengths` throws `KeyNotFoundException` at the first volume query for that
  size. §3.4, §3.5, §7.2.
- **B-10. Handler writes and event publication are not atomic.** The `outbox` section sets
  `disableTransactions` to `true`, so a crash between `AddAsync` and `PublishAsync` leaves a parcel
  with no `parcel_added`. Harmless only while B-5 holds. §3.25.

### 8.3 Open questions

- **Q-1.** Does `AddSecurity()` — or any other Convey extension called from
  `…Infrastructure/Extensions.cs:48-78` — implicitly consume the `jwt` configuration section? A-3 and
  §3.35 depend on the answer. This restates `service-summaries.md` G14/Q11 rather than adding a new
  gap.
- **Q-2.** What correlation context reaches this service on the AMQP path, and on `orders-service`'s
  server-to-server `GET /parcels/{parcelId}` call? Every guard in §3.17 and every fix in §7.4 depends
  on whether internal callers present an authenticated identity, an unauthenticated one, or none.
- **Q-3.** Is the file-based PACT exchange between `orders` and `parcels` still intended? Both halves
  are excluded from their solutions and the shared `pacts` directory exists in neither repository
  (§3.38). Reviving one end alone is worse than reviving neither.
- **Q-4.** Are `Variant` and `Size` stored as integer ordinals or as member names? The document type
  applies no `[BsonRepresentation]`, implying ordinals, but the repository's own test fixture
  registers a global `EnumRepresentationConvention(BsonType.String)` (§3.13). The two answers imply
  **opposite** prohibitions — never reorder, versus never rename — so the question blocks any
  structural change to either enum. Settled by one `db.parcels.findOne()`.
- **Q-5.** Who consumes `GET /parcels/volume`, and in what units? §3.5 establishes the arithmetic —
  side lengths in centimetres, cubed, divided by 1,000,000, giving cubic metres — but no caller for
  this endpoint exists in any clone in this workspace, and no unit is documented anywhere. A consumer
  that assumed litres would be wrong by 1000×.
- **Q-6.** Is the absence of an ownership check on `AddParcel` (B-2) deliberate — relying entirely on
  the gateway's `customerId:@user_id` binding — or an omission? `DeleteParcel` has the check, which
  suggests omission rather than design.
- **Q-7.** Are Mongo indexes created out of band, by an ops runbook or an init script outside these
  repositories? Nothing in this repository creates any (§5.5). This is the same question
  `orders-service.md` **Q-7** asks of its own collections.
- **Q-8.** Should this service react to a customer being deleted? There is no such subscription
  (§5.7), so a removed customer can still have parcels created against them indefinitely — assuming
  `customers-service` publishes such an event at all.

### 8.4 Cross-references

| Topic | Where |
| --- | --- |
| The producer of all four order events, and the consumer of `GET /parcels/{parcelId}` | `component-internals/orders-service.md` |
| The producer of `customer_created` | `component-internals/customers-service.md` |
| The upstream half of every route and async publish | `component-internals/api-gateway.md` |
| Correlation of rejected events to client operations | `component-internals/operations-service.md` §3.9 |
| The saga that drives the order lifecycle these events come from | `component-internals/ordermaker-saga-service.md` |
| Surface catalogue for this service | `baselines/api-inventory.md`; `baselines/service-summaries.md` §2.3 |
| Repository contents | `repo-summary/Pacco.Services.Parcels.md` |

**Patterns this component instantiates.** Links use the `[[pattern-file-name]]` form, matching
`../patterns/`.

| Pattern | Where it appears here |
| --- | --- |
| [[four-project-layering]] | `Core` → `Application` → `Infrastructure` → `Api`, inward dependencies only |
| [[cqrs-command-query-separation]] | commands in `Application`, query handlers in `Infrastructure/Mongo/Queries` |
| [[dual-transport-command-surface]] | `AddParcel`/`DeleteParcel` over both HTTP and AMQP (§4.1, §4.2) |
| [[document-entity-mapping]] | `AsEntity`/`AsDocument`/`AsDto` (§3.12) |
| [[reference-data-replica]] | the id-only `customers` collection (§3.6) |
| [[transactional-outbox]] | `AddMessageOutbox(o => o.AddMongo())` and the two decorators (§3.25) |
| [[handler-decorator-idempotency]] | message-id-keyed inbox (§3.25) |
| [[exception-to-response-mapping]] | `{code, reason}` at HTTP 400 (§3.26) |
| [[exception-to-message-mapping]] | rejected events over AMQP (§3.20) |
| [[rejected-event-convention]] | `AddParcelRejected`, `DeleteParcelRejected` |
| [[correlation-context-identity]] | `Correlation-Context` header → `IAppContext` (§3.27) |
| [[topic-exchange-per-service]] | exchange `parcels`, snake-case routing keys (§3.32) |
| [[queue-naming-convention]] | `parcels-service/{{exchange}}.{{message}}` (§3.32) |
| [[public-contracts-marker]] | `[Contract]` + `UsePublicContracts` (§3.31) |
| [[handler-log-templates]] | `MessageToLogTemplateMapper` (§3.33) |
| [[service-discovery-consul-fabio]] | `AddConsul`/`AddFabio` (§3.34) |
| [[vault-backed-configuration]] | KV, PKI and dynamic Mongo leases (§3.36) |
| [[environment-profile-layering]] | five `appsettings` profiles (§3.37) |
| [[distributed-tracing-jaeger]] | `AddJaeger` + the RabbitMQ plugin, `span_context` header |
| [[metrics-prometheus]] | `AddMetrics`, `prometheusEnabled` |
| [[injected-clock]] | `IDateTimeProvider` (§3.39) |
| [[pact-contract-testing]] | the Pactify provider test (§3.38) |
| [[two-stage-dotnet-dockerfile]] | SDK → ASP.NET runtime, `ASPNETCORE_ENVIRONMENT=docker` (§3.40) |

Patterns this component **deliberately does not** instantiate, and where that matters:

| Pattern | Why it is absent |
| --- | --- |
| [[aggregate-root-domain-events]] | no `AggregateRoot`, no event buffer — §3.2 sets out the trade-off |
| [[domain-event-to-integration-event-mapping]] | no `IEventMapper`; publication is hand-written (§3.21) |
| [[typed-http-service-client]] | `httpClient.services` is empty; no outbound HTTP at all (§3.34) |

---

*Component-internals model for `parcels-service`, batch 5 of 7. Registered in
`component-internals/index.md` §1. Any change to this component's internals must update this document
in the same change (§7.9).*
