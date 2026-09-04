# Component internals — `availability-service`

| | |
| --- | --- |
| **Component** | `availability-service` |
| **Source repository** | `hianshul100_Pacco.Services.Availability` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository) |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 1 of 7 |
| **Status** | New artifact — no prior `component-internals/availability-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. The existing `baselines/service-summaries.md` §2.3 remains valid and is **complemented**, not replaced: that document catalogues the surface, this one models the internals. |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full
> (`src/Pacco.Services.Availability.{Core,Application,Infrastructure,Api}` plus five test projects).
> `Convey 0.4.*` — which supplies the CQRS dispatcher, the Mongo repository, the RabbitMQ client,
> the outbox and the WebApi endpoint mapping — is a NuGet reference with **no source in this
> workspace**. Mechanisms owned by Convey are marked `[convey]` and, where their exact semantics
> change a conclusion, flagged `Unverifiable — Missing Source Evidence`. The upstream half of every
> inbound contract is modelled in `component-internals/api-gateway.md`.

---

## 1. Purpose & boundary

### 1.1 What this component is responsible for

`availability-service` owns **one aggregate — `Resource`** — and answers exactly one business
question: *is this resource free on this day, and who has it?* It is the platform's calendar-slot
allocator. Its distinguishing rule is **priority-based expropriation**: a higher-priority
reservation may evict a lower-priority one already holding the same day.

| Responsibility | Where it lives |
| --- | --- |
| Model a resource as an id + a tag set + a set of reservations | `src/…Core/Entities/Resource.cs` |
| Enforce one-reservation-per-calendar-day, with priority expropriation | `…Core/Entities/Resource.cs:57-80` (`AddReservation`) |
| Validate that a resource always has at least one non-blank tag | `…Core/Entities/Resource.cs:37-48` (`ValidateTags`) |
| Buffer domain events and expose them for publication | `…Core/Entities/AggregateRoot.cs` |
| Accept 4 commands over HTTP and the same 4 over AMQP | `…Api/Program.cs:38-46`; `…Infrastructure/Extensions.cs:107-110` |
| Answer 2 queries from a Mongo read model | `…Application/Queries/{GetResource,GetResources}.cs` |
| React to 2 external events from other services | `…Application/Events/External/{CustomerCreated,VehicleDeleted}.cs` |
| Verify the caller's customer state against `customers-service` | `…Infrastructure/Services/Clients/CustomersServiceClient.cs` |
| Translate domain events into integration events and publish them | `…Infrastructure/Services/{EventMapper,EventProcessor,MessageBroker}.cs` |
| Translate exceptions into HTTP responses **and** into rejected events | `…Infrastructure/Exceptions/ExceptionTo{Response,Message}Mapper.cs` |
| Persist resources in MongoDB with optimistic concurrency | `…Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:30-32` |
| Publish reliably via a Mongo-backed transactional outbox | `…Infrastructure/Decorators/Outbox*Decorator.cs`; `…Api/appsettings.json:102-110` |
| Register with Consul, emit Jaeger spans, expose Prometheus metrics | `…Infrastructure/Extensions.cs:79-88`; `…Api/appsettings.json:7-22`, `:70-96` |
| Fetch its own secrets and dynamic Mongo credentials from Vault | `…Api/Program.cs:48` (`UseVault`); `…Api/appsettings.json:171-200` |

### 1.2 What this component explicitly is **not**

- **Not an authenticator.** It performs **no JWT validation whatsoever.** A search across
  `src/` for `AddJwt`, `AddAuth`, `UseAuthentication`, `UseAuthorization` and `[Authorize]` returns
  **nothing**. The `jwt` block in `…Api/appsettings.json:79-87` (with
  `certificate.location: certs/localhost.cer`) is **inert configuration — dead weight that no code
  reads.** Identity arrives as a JSON header manufactured by the gateway
  (`component-internals/api-gateway.md` §3.14) and is **believed unconditionally**.
- **Not an authorization engine in general.** Exactly **one** handler checks ownership
  (`ReserveResourceHandler.cs:29-33`), and that check is itself skipped when the caller is
  unauthenticated. `DeleteResourceHandler`, `AddResourceHandler` and
  `ReleaseResourceReservationHandler` contain **no authorization code at all**.
- **Not the owner of customers or vehicles.** It holds no copy of a customer beyond an id
  (`CustomerCreatedHandler` does nothing but log — `…Application/Events/External/Handlers/CustomerCreatedHandler.cs`),
  and it treats a vehicle id as a resource id by convention only (§3.21).
- **Not a scheduler or a time authority.** It stores days, never times (§3.10), and has no
  background job that expires or reconciles reservations. `MetricsJob`
  (`…Infrastructure/Metrics/MetricsJob.cs`) is the only `BackgroundService`, and it only emits gauges.
- **Not a saga coordinator.** It forwards a `Saga` header when calling other services
  (`…Infrastructure/Extensions.cs:122-136`) but contains no saga definition; the orchestration lives
  in `orders-service` / a saga host outside this repository.
- **Not multi-tenant, not paginated on writes, and not soft-delete.** `DeleteAsync` is a hard
  `DeleteOneAsync` (`ResourcesMongoRepository.cs:34-35`).

### 1.3 The dual-transport boundary

Every write command has **two entry points with different failure semantics**, and this is the
single most important thing to hold in mind when reading the rest of this document:

| | HTTP path | AMQP path |
| --- | --- | --- |
| Entry | `UseDispatcherEndpoints` (`…Api/Program.cs:38-46`) | `SubscribeCommand<T>()` (`…Infrastructure/Extensions.cs:107-110`) |
| Identity source | `Correlation-Context` header (`…Infrastructure/Extensions.cs:117-120`) | broker message context (`AppContextFactory.cs:19-33`) |
| De-duplication | **none effective** — a fresh `MessageId` GUID per request (§3.27) | inbox, keyed on the broker `MessageId` |
| Failure surfaced as | HTTP **400** (almost always — §3.18) | a **rejected event** published to the broker (§3.16) |
| Failure when unmapped | generic 400 `{code:"error"}` | **nothing at all** — silent |
| Caller learns outcome | synchronously | only by polling `operations-service` |

A behaviour that "works" on one path can be silently broken on the other. §4 traces both.

---

## 2. Core concepts (exhaustive)

Every distinct internal mechanism this service implements. Owner column names the file that
*defines* the concept.

| # | Concept | Owner |
| --- | --- | --- |
| 1 | `Resource` aggregate root | `…Core/Entities/Resource.cs` |
| 2 | `AggregateRoot` — event buffer + `Version` | `…Core/Entities/AggregateRoot.cs` |
| 3 | `AggregateId` value wrapper | `…Core/Entities/AggregateId.cs` |
| 4 | Tag set & tag validation | `…Core/Entities/Resource.cs:37-48` |
| 5 | `Reservation` value object (a `struct`) | `…Core/ValueObjects/Reservation.cs` |
| 6 | Reservation priority & expropriation | `…Core/Entities/Resource.cs:57-80` |
| 7 | Domain event | `…Core/Events/*.cs`, `IDomainEvent` |
| 8 | Domain exception hierarchy | `…Core/Exceptions/*.cs` |
| 9 | `ResourceDocument` / `ReservationDocument` | `…Infrastructure/Mongo/Documents/*.cs` |
| 10 | Days-since-epoch date encoding | `…Infrastructure/Mongo/Documents/Extensions.cs:40-44` |
| 11 | Optimistic concurrency on `Version` | `…Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:30-32` |
| 12 | Command + command handler | `…Application/Commands/**` |
| 13 | Query + read model | `…Application/Queries/**`, `…Infrastructure/Mongo/Queries/**` |
| 14 | Integration event & `EventMapper` | `…Infrastructure/Services/EventMapper.cs` |
| 15 | `EventProcessor` — the publish pipeline | `…Infrastructure/Services/EventProcessor.cs` |
| 16 | Rejected event & `ExceptionToMessageMapper` | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` |
| 17 | Transactional outbox / inbox decorators | `…Infrastructure/Decorators/Outbox*Decorator.cs` |
| 18 | `ExceptionToResponseMapper` — HTTP status mapping | `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` |
| 19 | `IAppContext` / `IdentityContext` | `…Infrastructure/Contexts/*.cs` |
| 20 | `Correlation-Context` header ingestion | `…Infrastructure/Extensions.cs:117-120` |
| 21 | External event subscription & `[Message]` attributes | `…Application/Events/External/*.cs` |
| 22 | Queue naming & message conventions | `…Api/appsettings.json:111-152` |
| 23 | `[Contract]` and `UsePublicContracts` | `…Api/Program.cs`; `…Application/Commands/*.cs` |
| 24 | Dispatcher-bound HTTP endpoints | `…Api/Program.cs:38-46` |
| 25 | `CustomersServiceClient` + PKI client certificates | `…Infrastructure/Services/Clients/CustomersServiceClient.cs` |
| 26 | Customer state check (`CustomerStateDto.IsValid`) | `…Application/DTO/CustomerStateDto.cs:8` |
| 27 | `MessageBroker` — span-context-aware publisher | `…Infrastructure/Services/MessageBroker.cs` |
| 28 | Log-message templating (`MessageToLogTemplateMapper`) | `…Infrastructure/Logging/*.cs` |
| 29 | `CustomMetricsMiddleware` + `MetricsJob` | `…Infrastructure/Metrics/{CustomMetricsMiddleware,MetricsJob}.cs` |
| 30 | Consul registration & Fabio addressing | `…Infrastructure/Extensions.cs:79-80`; `…Api/appsettings.json:7-33` |
| 31 | Vault: KV secrets, PKI, dynamic Mongo credentials | `…Api/Program.cs:48`; `…Api/appsettings.json:171-200` |
| 32 | Environment layering (`appsettings.{local,docker,tests}.json`) | `…Api/appsettings.*.json` |
| 33 | Mongo collection naming & the `resources` literal | `…Infrastructure/Extensions.cs:90`; `…Mongo/Queries/Handlers/*.cs` |
| 34 | `IDomainEventHandler<>` — a live but **unused** extension point | `…Infrastructure/Extensions.cs`; zero implementations |
| 35 | Redis registration — **configured, no consumer** | `…Infrastructure/Extensions.cs:85`; `…Api/appsettings.json:153-156` |
| 36 | Test topology (Unit / Integration / EndToEnd / Performance / Shared) | `tests/**` |

---

## 3. Per concept

Paths below are relative to `src/`, with the `Pacco.Services.Availability.` prefix elided:
`Core/…`, `Application/…`, `Infrastructure/…`, `Api/…`.

### 3.1 `Resource` — the aggregate root

**Definition.** The only aggregate in the service. A `Resource` is *an id, a non-empty set of tags,
and a set of reservations*. It has no name, no owner, no type discriminator and no state field —
`Core/Entities/Resource.cs:10-35` is the entire shape. What kind of thing a resource *is* (a
vehicle, a courier, a warehouse slot) is expressed **only** through its tags.

**Representation & storage.**

```csharp
public class Resource : AggregateRoot {
    private ISet<string> _tags = new HashSet<string>();
    private ISet<Reservation> _reservations = new HashSet<Reservation>();
    public IEnumerable<string> Tags { get => _tags; private set => _tags = new HashSet<string>(value); }
    public IEnumerable<Reservation> Reservations { get => _reservations; private set => _reservations = new HashSet<Reservation>(value); }
```
(`Core/Entities/Resource.cs:12-25`)

Both collections are exposed as `IEnumerable<>` with **private setters that re-wrap into a
`HashSet`** — so callers cannot mutate the collections, and every assignment (including
rehydration from Mongo) re-applies set semantics. The `HashSet<Reservation>` is the enforcement
point for "one reservation per calendar day", because `Reservation.GetHashCode()` hashes on the
date alone (§3.5).

Persisted as `ResourceDocument` in the Mongo collection `resources`
(`Api/appsettings.json` `mongo.database: availability-service`; collection name at
`Infrastructure/Extensions.cs` — `.AddMongoRepository<ResourceDocument, Guid>("resources")`).

**Lifecycle.**

| Phase | Trigger | Code |
| --- | --- | --- |
| Created | `AddResource` command | `Resource.Create` → `Core/Entities/Resource.cs:50-55`, raising `ResourceCreated` |
| Rehydrated | any read | `ResourceDocument.AsEntity()` → `Infrastructure/Mongo/Documents/Extensions.cs` |
| Reserved | `ReserveResource` | `AddReservation` (`:57-80`) |
| Released | `ReleaseResourceReservation` | `ReleaseReservation` (`:82-90`) |
| Deleted | `DeleteResource`, or `VehicleDeleted` from vehicles-service | `Delete()` (`:92-100`) — **hard delete** |

`Delete()` is worth reading closely: it raises a `ReservationCanceled` for **every** existing
reservation and then a single `ResourceDeleted` (`:94-99`). So deleting a resource with three
reservations buffers four domain events. That matters because of the `Version` rule in §3.2 — all
four are one version increment — and because `ReservationCanceled` maps to a real integration event
that other services act on (§3.14).

**Invariants & enforcement.**

| Invariant | Enforced at | Violation |
| --- | --- | --- |
| Tags non-null and non-empty | `ValidateTags` (`:37-48`), called from the constructor `:30` | **Loud** — `MissingResourceTagsException` |
| No blank/whitespace tag | `ValidateTags` (`:44-47`) | **Loud** — `InvalidResourceTagsException` |
| Id ≠ `Guid.Empty` | `AggregateId` ctor (§3.3) | **Loud** — `InvalidAggregateIdException` |
| At most one reservation per calendar day | `HashSet` + `AddReservation`'s collision scan (`:59-79`) | **Loud** on equal-or-higher priority; silent expropriation otherwise (§3.6) |
| Releasing a reservation that exists | `ReleaseReservation` (`:84-87`) | **SILENT** — early `return`, no event, no error |

Because `ValidateTags` is called from the constructor and the constructor is also what
`AsEntity()` uses, **a document already in Mongo with an empty tag array cannot be read back** —
the read throws `MissingResourceTagsException`, which `ExceptionToResponseMapper` turns into a
**400** (§3.18). A data-integrity problem therefore surfaces to a client as a bad-request on a
`GET`. There is no way to repair such a document through this service.

Also note what is **not** invariant: nothing constrains tag count, tag length, tag vocabulary,
reservation count, or how far into the future/past a reservation may be. A resource may hold an
unbounded number of reservations, and `ReserveResource` accepts a date in the year 1900 as readily
as 2100.

**Extension procedure — add a field to `Resource` (e.g. `Name`).**
1. Add the property with a `private set` and assign it in the constructor
   (`Core/Entities/Resource.cs:27-35`); add it to `Create` if it is required at creation.
2. Add the matching property to `ResourceDocument` and both mapping directions in
   `Infrastructure/Mongo/Documents/Extensions.cs` (`AsEntity` / `AsDocument`). **Omitting one
   direction is silent**: `IgnoreExtraElements` is on (§3.33), so a missing element deserializes to
   `default` with no error.
3. Add it to `ResourceDto` (`Application/DTO/ResourceDto.cs`) and the document→DTO mapping if the
   read model must expose it.
4. Add it to the `AddResource` command and to the gateway's `bind:`/body if clients must supply it
   (`component-internals/api-gateway.md` §3.9).
5. If it belongs on the wire, add it to the integration event contract in
   `Pacco.Services.Availability.Contracts` (§3.23) — a **cross-repository, versionless** change.

**Failure modes.** Silent field loss on an incomplete mapping (above). Unbounded reservation growth
on a long-lived resource makes the document grow without limit — Mongo's 16 MB document cap is the
only ceiling, and hitting it produces a write error at `UpdateAsync`, not a domain error.

---

### 3.2 `AggregateRoot` — the event buffer and `Version`

**Definition.** The base class giving every aggregate an id, an in-memory list of pending domain
events, and an integer version used for optimistic concurrency.

**Representation & storage.** `Core/Entities/AggregateRoot.cs` in its entirety is 25 lines:

```csharp
private readonly List<IDomainEvent> _events = new List<IDomainEvent>();
public IEnumerable<IDomainEvent> Events => _events;
public AggregateId Id { get; protected set; }
public int Version { get; protected set; }

protected void AddEvent(IDomainEvent @event) {
    if (!_events.Any()) { Version++; }      // ← only the FIRST event bumps the version
    _events.Add(@event);
}
public void ClearEvents() => _events.Clear();
```

**The `if (!_events.Any())` guard is the single most subtle line in this repository.** `Version` is
incremented **once per loaded-and-modified instance**, not once per event. That is deliberate and
correct: one HTTP request = one aggregate load = one persisted version bump, regardless of how many
domain events the operation produced. `Resource.Delete()` raising four events still moves `Version`
by exactly one.

**Lifecycle.** `Version` is set from the document on rehydration (`Resource` ctor's `version`
parameter, `Core/Entities/Resource.cs:28,34`), incremented in memory by the first `AddEvent`, and
written back by `UpdateAsync` under the concurrency filter (§3.11). `_events` lives only for the
duration of the request; it is never persisted. `ClearEvents()` exists but — verified by search —
**is never called anywhere in `src/`**; the buffer is discarded with the object.

**Invariants & enforcement.**
- `Version` is monotonic per aggregate *only if* every mutation goes through `AddEvent`. A mutator
  that changes state without raising an event leaves `Version` unchanged, and §3.11's filter
  (`r.Version < resource.Version`) then **matches zero documents — the write is silently
  discarded.** This is not hypothetical: it is exactly what happens in
  `ReleaseResourceReservation` for a non-existent reservation (§3.12, and the flow in §4.4).
- `Events` is exposed as `IEnumerable<IDomainEvent>` over the private `List`, so callers can
  enumerate but not add. There is no cap on buffer size.
- The event buffer is **not thread-safe** (`List<T>`, no synchronisation). Each request loads its
  own instance, so this is safe as used; it would break if an aggregate were ever cached.

**Extension procedure — add a new aggregate.** Derive from `AggregateRoot`, call `AddEvent` from
every mutator, add a `Version` column to the document, and reproduce the concurrency filter in the
new repository (`Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:30-32` is the
template — it is hand-written, not provided by Convey).

**Failure modes.** State change without an event → silent lost update. Calling `ClearEvents()`
before `EventProcessor.ProcessAsync` → all integration events silently dropped. Two events raised
across two separate load/save cycles in one request → two version bumps, and the second `UpdateAsync`
will fail its filter unless the instance was reloaded.

---

### 3.3 `AggregateId`

**Definition.** A `Guid` wrapper that makes "empty id" unrepresentable.

**Representation & storage.** `Core/Entities/AggregateId.cs`. Key mechanics:

```csharp
public AggregateId() : this(Guid.NewGuid()) { }
public AggregateId(Guid value) {
    if (value == Guid.Empty) { throw new InvalidAggregateIdException(value); }
    Value = value; }
public static implicit operator Guid(AggregateId id) => id.Value;
public static implicit operator AggregateId(Guid id) => new AggregateId(id);
```

**Lifecycle.** Created implicitly. The two implicit conversions mean `Guid` and `AggregateId` are
interchangeable at every call site — `Resource`'s constructor takes a `Guid` and assigns it to an
`AggregateId Id` (`Core/Entities/Resource.cs:31`), and repository methods take `AggregateId` but are
called with `Guid`s (`Core/Repositories/IResourcesRepository.cs:8-12` vs.
`Application/Commands/Handlers/*.cs`).

**Invariants & enforcement.**
- `Guid.Empty` → **loud** `InvalidAggregateIdException` (`Code = "invalid_aggregate_id"`).
- **But the implicit operator makes the throw site invisible.** Writing
  `await _repository.GetAsync(command.ResourceId)` where `ResourceId` is `Guid.Empty` throws from a
  conversion the reader cannot see in the source. On the HTTP path this is survivable —
  `InvalidAggregateIdException` is a `DomainException`, so `ExceptionToResponseMapper` returns
  `{code:"invalid_aggregate_id", reason:"Invalid aggregate id: 00000000-…"}` with **400**
  (§3.18). On the AMQP path it is not: `ExceptionToMessageMapper` has **no case for it**
  (`Infrastructure/Exceptions/ExceptionToMessageMapper.cs:55`, `_ => null`), so the message fails
  with **no rejected event at all** and an async caller waits forever (§3.16).
- `Equals` compares by `Value` and by exact type (`:30-35`), so `AggregateId` is a proper value
  object; `GetHashCode` delegates to the `Guid`.

**Extension procedure.** Use `AggregateId` for every new aggregate's id. If you need a typed id per
aggregate (to prevent passing a resource id where a customer id belongs), that would mean removing
the implicit operators — a wide, breaking change across all handlers.

**Failure modes.** Empty-GUID input yields an unhelpful generic 400 and a silent no-op on the bus.
The mitigation used elsewhere is to default the id at the command boundary, which `AddResource`
does (`Application/Commands/AddResource.cs` — substitutes a new GUID for `Guid.Empty`) but
`ReserveResource`, `ReleaseResourceReservation` and `DeleteResource` do **not**.

---

### 3.4 Tags

**Definition.** A free-form set of strings that is the *only* classification a resource carries.
Tags are how `orders-service` and the rest of the platform decide whether a resource is suitable.

**Representation & storage.** `ISet<string>` in the aggregate (`Core/Entities/Resource.cs:12`),
`IEnumerable<string>` in `ResourceDocument`, a BSON array in Mongo.

**Lifecycle.** Set **once, at construction** (`Core/Entities/Resource.cs:32`). There is **no
mutator**: no `AddTag`, no `RemoveTag`, no `UpdateTags` anywhere in the aggregate, no command for
it, and no HTTP route. **A resource's tags are immutable for its entire life** — to change them you
must delete and recreate the resource, which changes its id and cancels every reservation.
This is a genuine functional gap, not an oversight in this document: `baselines/service-summaries.md`
G7 records the same absence from the surface side.

**Invariants & enforcement.** `ValidateTags` (`Core/Entities/Resource.cs:37-48`), both checks
**loud**:
- `tags is null || !tags.Any()` → `MissingResourceTagsException` (`Code = "missing_resource_tags"`)
- `tags.Any(string.IsNullOrWhiteSpace)` → `InvalidResourceTagsException`
  (`Code = "invalid_resource_tags"`)

Both are mapped to **HTTP 400** by `ExceptionToResponseMapper`, and both are mapped on the AMQP path
to `AddResourceRejected` — but with **`Guid.Empty` as the resource id**
(`Infrastructure/Exceptions/ExceptionToMessageMapper.cs`), because the exception carries no id. An
async client therefore receives a rejection it cannot correlate to its request except through the
message context. That is a real defect of the rejected-event contract (§3.16).

What is **not** enforced: duplicates are silently collapsed by the `HashSet`; case is significant
(`"vehicle"` and `"Vehicle"` are different tags); there is no length limit, no character
restriction, and no controlled vocabulary. Cross-service tag agreement is by convention only —
nothing in this repository declares the tags other services search for.

**Extension procedure — make tags mutable.**
1. Add `UpdateTags(IEnumerable<string> tags)` to `Resource`, calling `ValidateTags` then `AddEvent`
   with a new `ResourceTagsUpdated` domain event.
2. Add the domain event to `Core/Events/`, an integration event to the Contracts assembly, and a
   **case in `EventMapper.Map`** — omitting the mapper case means the event is **silently dropped**
   (§3.14/§3.15).
3. Add an `UpdateResource` command + handler, subscribe it in `Infrastructure/Extensions.cs`, map a
   `Put<UpdateResource>` endpoint in `Api/Program.cs`, and add the route to all four gateway
   manifests.

**Failure modes.** Immutability forces delete-and-recreate, which cascades `ReservationCanceled`
events to every downstream consumer — an operationally expensive way to fix a typo.

---

### 3.5 `Reservation` — the value object

**Definition.** A `(DateTime, Priority)` pair. It is a **`struct`**, and that choice drives two of
this service's most important silent-failure paths.

**Representation & storage.** `Core/ValueObjects/Reservation.cs` in full:

```csharp
public struct Reservation : IEquatable<Reservation> {
    public DateTime DateTime { get; }
    public int Priority { get; }
    public Reservation(DateTime dateTime, int priority) => (DateTime, Priority) = (dateTime, priority);
    public bool Equals(Reservation reservation)
        => Priority.Equals(reservation.Priority) && DateTime.Date.Equals(reservation.DateTime.Date);
    public override int GetHashCode() => DateTime.Date.GetHashCode();
}
```

**Note the deliberate `Equals`/`GetHashCode` asymmetry.** `GetHashCode` uses **only the date**;
`Equals` uses **date *and* priority**. This is legal (equal objects still hash equally) and it is
what makes the `HashSet<Reservation>` behave as "a bucket per calendar day": two reservations for
the same day always collide into the same bucket, and `AddReservation`'s explicit priority logic
(§3.6) then decides which survives. If `GetHashCode` included `Priority`, same-day reservations with
different priorities would land in different buckets, `HashSet.Add` would accept both, and the
one-per-day invariant would be enforced only by the scan — a much more fragile design. **Do not
"fix" this asymmetry.**

The three critical consequences of `struct`:
1. `Reservation` has an implicit parameterless constructor producing
   `default(Reservation)` = `{DateTime: 0001-01-01 00:00:00, Priority: 0}`, which is a *valid,
   addable* value.
2. `FirstOrDefault` on `IEnumerable<Reservation>` returns that default instead of `null` when
   nothing matches, and **there is no way to distinguish "found a midnight-of-year-1 reservation"
   from "found nothing"**. This is the root of the silent release bug (§3.12, §4.4).
3. It is copied by value, so `Resource.Reservations` cannot be mutated through an enumerated item —
   a genuine safety benefit.

**Lifecycle.** Constructed in `ReserveResourceHandler`
(`Application/Commands/Handlers/ReserveResourceHandler.cs` — `new Reservation(command.DateTime,
command.Priority)`), stored in the aggregate's `HashSet`, mapped to `ReservationDocument` for
persistence (§3.9/§3.10), and echoed into domain events by value.

**Invariants & enforcement.**
- **None on `DateTime`.** No past/future bound, no null (it is a value type), no timezone
  normalisation. Whatever the client sends is what is compared.
- **None on `Priority`.** Negative priorities are accepted. There is no enum, no constant, no
  documented scale, and no upper bound anywhere in the repository. What "priority 3" means is
  entirely a cross-service convention with no artefact defining it — see ABQ Q-3.
- Time-of-day is **structurally significant in memory but lost on persistence** (§3.10). A
  reservation constructed with `2026-09-04T14:30` is equal (by `Equals`) to one at
  `2026-09-04T09:00` of the same priority, is published in the integration event with `14:30`, and
  is read back from Mongo as `2026-09-04T00:00`. **Three different representations of the same
  reservation exist simultaneously.**

**Extension procedure — add a field (e.g. `CustomerId`).** Add the property and the constructor
parameter; **update `Equals` and `GetHashCode` deliberately** (adding `CustomerId` to `Equals`
would allow two same-day reservations by different customers to be considered distinct — but they
would still collide in the hash bucket and be rejected by `AddReservation`'s scan, so the net effect
is only to change which exception fires); update `ReservationDocument` and both mapping directions;
update the `ResourceReserved` contract. Note the aggregate currently **does not know who holds a
reservation** — `CustomerId` exists only on the command and the integration event, never in the
stored aggregate. Adding it here would be the single highest-value change to this component
(see §7.3).

**Failure modes.** `default(Reservation)` treated as a real value (§3.12). Time-of-day loss (§3.10).
Priority collisions with no tie-break other than "the incumbent wins" (§3.6).

---

### 3.6 Priority & expropriation — the core business rule

**Definition.** When two reservations want the same calendar day, the higher priority wins and the
incumbent is evicted. This is the reason this service exists.

**Representation & storage.** `Core/Entities/Resource.cs:57-80`, `AddReservation`:

```csharp
var hasCollidingReservation = _reservations.Any(HasTheSameReservationDate);
if (hasCollidingReservation) {
    var collidingReservation = _reservations.First(HasTheSameReservationDate);
    if (collidingReservation.Priority >= reservation.Priority) {
        throw new CannotExpropriateReservationException(Id, reservation.DateTime.Date);
    }
    if (_reservations.Remove(collidingReservation)) {
        AddEvent(new ReservationCanceled(this, collidingReservation));
    }
}
if (_reservations.Add(reservation)) { AddEvent(new ReservationAdded(this, reservation)); }
bool HasTheSameReservationDate(Reservation r) => r.DateTime.Date == reservation.DateTime.Date;
```

**The exact semantics, stated precisely:**

| Incumbent priority vs. incoming | Outcome | Events raised |
| --- | --- | --- |
| incumbent **>** incoming | rejected | none — **loud** `CannotExpropriateReservationException` |
| incumbent **==** incoming | rejected | none — **loud**, same exception |
| incumbent **<** incoming | incumbent evicted, new one added | `ReservationCanceled` *then* `ReservationAdded` |
| no incumbent | added | `ReservationAdded` |

The `>=` at line 63 is the load-bearing comparison: **equal priority does not expropriate.**
First-come-wins at equal priority. This is also what makes a retried `POST` reservation fail
(`component-internals/api-gateway.md` §3.20): the second attempt sees its own successful first
attempt as an equal-priority incumbent and throws.

**Lifecycle.** Evaluated synchronously inside the aggregate, entirely in memory, before any I/O.
The scan is `O(n)` over the reservation set — twice, because `Any` and `First` are separate passes
over `HasTheSameReservationDate`.

**Invariants & enforcement.**
- At most one reservation per calendar day, enforced twice over: by the `HashSet`'s date-only hash
  and by the explicit scan. **Loud** on violation.
- Eviction is **not** announced to the evicted party by this service directly — it publishes
  `ReservationCanceled` → the integration event `ResourceReservationCanceled` (§3.14) and relies on
  a downstream consumer to notify. **This service does not know who held the evicted reservation**
  (§3.5), so the published event carries the resource id and the date but **not the victim's
  customer id.** Any consumer wanting to notify the evicted customer must have recorded the
  reservation itself. That is the most consequential modelling gap in this component.
- The eviction is **not transactional across services**: the `ReservationCanceled` and
  `ReservationAdded` events are published together after the Mongo write (§3.15), through the
  outbox, so a consumer may observe them out of order relative to other services' state.
- `if (_reservations.Remove(...))` and `if (_reservations.Add(...))` guard both events, so the
  events are raised **only when the set actually changed**. `Add` returning false (an exactly-equal
  reservation — same date *and* same priority) is unreachable here, because that case would have
  been caught by the `>=` throw first.

**Extension procedure — change the tie-break to last-write-wins at equal priority.** Change `>=` to
`>` at `Core/Entities/Resource.cs:63`. Be aware this makes the operation non-idempotent under the
gateway's blind retries: a retried request would then evict its own successful first attempt and
publish a spurious `ReservationCanceled`. The safer change is to make the operation idempotent —
compare the incoming reservation against the incumbent and return without events if they are
identical.

**Extension procedure — add a reason to the eviction.** Add fields to
`Core/Events/ReservationCanceled.cs`, to the `ResourceReservationCanceled` contract, and to
`EventMapper`'s case. All three, or the field is silently absent on the wire.

**Failure modes.** Equal-priority rejection surfaces as HTTP 400 `cannot_expropriate_reservation`
(mapped — §3.18) or, on AMQP, as a **`ReleaseResourceReservationRejected`** event — the *wrong
event name*, because no `ReserveResourceRejected` type exists (§3.16). An async client cannot tell a
failed reserve from a failed release.

---

### 3.7 Domain events

**Definition.** In-process notifications that something happened inside the aggregate. They are
**not** the wire contract; §3.14 converts them.

**Representation & storage.** `Core/Events/`. `IDomainEvent` is an **empty marker interface**
(`Core/Events/IDomainEvent.cs` — three lines, no members). Five implementations, all immutable
classes with getter-only properties and a tuple-assignment constructor:

| Event | Payload | Raised by |
| --- | --- | --- |
| `ResourceCreated` | `Resource` | `Resource.Create` (`Resource.cs:53`) |
| `ReservationAdded` | `Resource`, `Reservation` | `AddReservation` (`:76`) |
| `ReservationCanceled` | `Resource`, `Reservation` | `AddReservation` on eviction (`:70`); `Delete()` per reservation (`:96`) |
| `ReservationReleased` | `Resource`, `Reservation` | `ReleaseReservation` (`:89`) |
| `ResourceDeleted` | `Resource` | `Delete()` (`:99`) |

**Every event holds a reference to the live `Resource` object**, not a snapshot. Since the events
are consumed within the same request (`EventProcessor.ProcessAsync(resource.Events)`), and the
aggregate is not mutated further after the handler completes, this is safe as written — but it means
an event's `Resource.Reservations` reflects the state **at publication time**, not at the moment the
event was raised. For `ReservationCanceled` raised during expropriation, `Resource` therefore
already shows the *new* reservation and not the cancelled one. The cancelled `Reservation` is
carried separately, which is why the event has two fields.

**Lifecycle.** Raised → buffered in `AggregateRoot._events` → after the repository write, passed to
`EventProcessor.ProcessAsync` (§3.15) → mapped (§3.14) → published (§3.27). Never persisted. There
is **no event store** and no replay: this is a state-stored aggregate that happens to emit events,
not event sourcing.

**Invariants & enforcement.** `IDomainEvent` being an empty marker means there is **no compile-time
requirement** for an event to carry anything, to be mappable, or to be handled. Nothing checks that
a new `IDomainEvent` has a case in `EventMapper`. This is the enforcement hole behind §3.14's silent
drop.

**Extension procedure — add a domain event.**
1. Add the class to `Core/Events/` implementing `IDomainEvent`.
2. Raise it via `AddEvent` from the aggregate mutator.
3. **Add a `case` to `Infrastructure/Services/EventMapper.cs`** — this is the step that is easy to
   forget and produces no error.
4. Add the corresponding integration event to the Contracts assembly (§3.23) with a `[Contract]`
   attribute.
5. If in-process reaction is wanted, implement `IDomainEventHandler<TEvent>` — see §3.34 for the
   caveat that no such handler currently exists.

**Failure modes.** An event with no `EventMapper` case is **silently swallowed** (§3.15). An event
raised outside `AddEvent` bypasses the `Version` bump (§3.2).

---

### 3.8 Domain exception hierarchy

**Definition.** The typed failures the domain raises, each carrying a stable string `Code` intended
as the client-facing error identifier.

**Representation & storage.** `Core/Exceptions/`. `DomainException` is
`abstract class DomainException : Exception` with `public virtual string Code { get; }`
(`Core/Exceptions/DomainException.cs`). Four concrete types:

| Type | `Code` | Extra data | Thrown at |
| --- | --- | --- | --- |
| `MissingResourceTagsException` | `missing_resource_tags` | — | `Resource.cs:41` |
| `InvalidResourceTagsException` | `invalid_resource_tags` | — | `Resource.cs:46` |
| `CannotExpropriateReservationException` | `cannot_expropriate_reservation` | `ResourceId`, `DateTime` | `Resource.cs:65` |
| `InvalidAggregateIdException` | `invalid_aggregate_id` | `Id` | `AggregateId.cs:18` |

A **second, parallel hierarchy** lives outside Core, in `Application/Exceptions/`:
`AppException` (structurally identical to `DomainException` — `abstract class AppException :
Exception` with `public virtual string Code { get; }`) and five subclasses:

| Type | `Code` | Extra data | Thrown at |
| --- | --- | --- | --- |
| `ResourceNotFoundException` | `resource_not_found` | `Id` | `DeleteResourceHandler.cs:26`, `ReserveResourceHandler.cs:38`, `ReleaseResourceReservationHandler.cs:27` |
| `ResourceAlreadyExistsException` | `resource_already_exists` | `Id` | `AddResourceHandler.cs:25` |
| `CustomerNotFoundException` | `customer_not_found` | `Id` | `ReserveResourceHandler.cs:44` |
| `InvalidCustomerStateException` | `invalid_customer_state` | `Id`, `State` | `ReserveResourceHandler.cs:49` |
| `UnauthorizedResourceAccessException` | `unauthorized_resource_access` | `ResourceId`, `CustomerId` | `ReserveResourceHandler.cs:32` |

The two hierarchies are **not related by inheritance** — `AppException` does not derive from
`DomainException`. Every consumer must therefore handle both, and both mappers do so with two
separate top-level cases (`ExceptionToResponseMapper.cs:18-21`).

**Lifecycle.** Thrown inside the aggregate or handler; caught by Convey's exception middleware on
the HTTP path (mapped by §3.18) or by the message-broker pipeline on the AMQP path (mapped by
§3.16).

**Invariants & enforcement.** `Code` is `virtual` on both bases with **no value and no `abstract`
requirement** — a subclass that forgets to override it compiles fine and yields `null`. That case is
handled, but not obviously: `ExceptionToResponseMapper.GetCode` (`:34-40`) uses the declared `Code`
only `when !string.IsNullOrWhiteSpace(...)`, and otherwise falls back to
`exception.GetType().Name.Underscore().Replace("_exception", string.Empty)`. So:

- With `Code` overridden (all nine current types) → the declared string wins; **renaming the class
  does not change the wire code.**
- With `Code` forgotten → the code is derived from the **type name**, and a later rename then
  **silently changes the client-visible error code.**

The result is memoised per exception type in a `static ConcurrentDictionary<Type,string> Codes`
(`:13`, `:42`), so the fallback is computed at most once per type per process — a micro-optimisation
with one consequence worth knowing: the dictionary is `static` and never evicted, so it is shared
across all requests and all test cases in a process.

**Extension procedure.** Derive from `DomainException` (a rule the aggregate enforces) or
`AppException` (a rule a handler enforces) — the choice determines nothing functionally today, since
both map to 400, but it is the layering signal. **Always override `Code`.** Then add a case to
`ExceptionToMessageMapper` (§3.16) for every command that can raise it, or the AMQP path produces
nothing.

**Failure modes.** Unmapped exception → generic 400 on HTTP, **nothing** on AMQP. A subclass
without a `Code` override gets a type-name-derived code that changes on rename.

---

### 3.9 `ResourceDocument` / `ReservationDocument`

**Definition.** The persistence-layer shape, deliberately separate from the aggregate so that
storage concerns do not leak into `Core`.

**Representation & storage.** `Infrastructure/Mongo/Documents/ResourceDocument.cs` and
`ReservationDocument.cs`. `ResourceDocument` implements Convey's `IIdentifiable<Guid>` and carries exactly four fields —
`Id` (`Guid`), `Version` (`int`), `Tags` (`IEnumerable<string>`), `Reservations`
(`IEnumerable<ReservationDocument>`). `ReservationDocument` carries **`TimeStamp` as an `int`**
(note the name: it is *not* called `DateTime`, and it holds days, not a timestamp — §3.10) and
`Priority` as an `int`. Both are `internal sealed`.

Mapping lives in `Infrastructure/Mongo/Documents/Extensions.cs` — a single file with
`AsEntity()`, `AsDocument()` and `AsDto()` extension methods, plus the two date helpers. The
mapping is **hand-written in both directions**; there is no AutoMapper, no convention, and no test
asserting round-trip fidelity.

**Lifecycle.** `AsDocument()` on every write; `AsEntity()` on every aggregate read; `AsDto()` on
every query read (§3.13 — the query path **bypasses the aggregate entirely**, going document → DTO
without constructing a `Resource`).

That bypass has a concrete consequence: **`GET /resources` and `GET /resources/{id}` do not run
`ValidateTags`** if they use `AsDto()` directly from the Mongo query, whereas anything that loads
through `IResourcesRepository.GetAsync` does. Read `Infrastructure/Mongo/Queries/` alongside
`Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs` before assuming which path a given
endpoint takes.

**Invariants & enforcement.** None at the document level — no schema validation, no required
fields, no Mongo JSON-schema validator configured (`mongo.seed: false`,
`Api/appsettings.json`; nothing in `Infrastructure/` creates indexes or validators). Every
constraint is re-applied on the way *out*, by `Resource`'s constructor, which is why a malformed
document fails on read rather than on write (§3.1).

**Extension procedure.** See §3.1's five steps. The specific trap: `IgnoreExtraElements` is
registered as a Mongo convention (§3.33), so **removing a property from the document class does not
error on read** — the stored data is simply ignored, and re-saving the aggregate deletes it.

**Failure modes.** Asymmetric mapping (a property mapped in `AsDocument` but not `AsEntity`, or
vice versa) is **completely silent** and manifests as data that appears to save but reads back as
`default`. There is no round-trip test in `tests/` covering this.

---

### 3.10 Days-since-epoch date encoding

**Definition.** Reservation dates are stored as an integer count of days since `DateTime.MinValue`,
not as a BSON date.

**Representation & storage.** `Infrastructure/Mongo/Documents/Extensions.cs:40-44`:

```csharp
internal static int AsDaysSinceEpoch(this DateTime dateTime) => (dateTime - new DateTime()).Days;
internal static DateTime AsDateTime(this int daysSinceEpoch) => new DateTime().AddDays(daysSinceEpoch);
```

`new DateTime()` is `0001-01-01T00:00:00`, so this is **days since 0001-01-01**, not the Unix
epoch — the method name is misleading. `(a - b).Days` truncates toward zero, discarding the
time-of-day component entirely.

**Lifecycle.** Applied in `AsDocument()` on write and `AsDateTime()` on read, for every
`ReservationDocument.DateTime`.

**Invariants & enforcement — the round-trip is lossy, and nothing warns.**

| Stage | Value |
| --- | --- |
| Client sends / command carries | `2026-09-04T14:30:00` |
| In-memory `Reservation.DateTime` | `2026-09-04T14:30:00` |
| Domain event / **published integration event** | `2026-09-04T14:30:00` |
| Stored in Mongo | `739_XXX` (an `int`) |
| Read back | `2026-09-04T00:00:00` |

So **the value another service receives on the bus differs from the value this service will report
on a subsequent read.** A consumer that stores the event's `DateTime` and later compares it to a
`GET /resources/{id}` response will find them unequal. Nothing detects or reconciles this.

The encoding is *consistent with* the domain rule (§3.5 compares on `.Date` throughout), so it does
not break any invariant — it is a correct-but-surprising optimisation. But it also means:
- Sub-day reservations are **impossible by construction**, and the API silently accepts a time it
  cannot honour. This is not documented on any endpoint.
- `DateTimeKind` is discarded. A UTC input and a local input for the same instant near midnight
  store as **different days**. There is no normalisation anywhere in the pipeline —
  `ReserveResource` takes the client's `DateTime` verbatim.
- The stored `int` is not human-readable in Mongo; ad-hoc queries against `reservations.dateTime`
  need the same arithmetic.

**Extension procedure — support time-of-day.** Change `ReservationDocument.DateTime` to `DateTime`,
drop both helpers, and **write a migration**: every existing document's `reservations[].dateTime`
is an `int` that will not deserialize into a `DateTime`. There is no migration framework in this
repository (§5), so this is a hand-written script. You must also revisit `Reservation.Equals` and
`GetHashCode` (§3.5), which deliberately compare on `.Date` only, and the collision scan at
`Core/Entities/Resource.cs:79`.

**Extension procedure — the cheaper fix.** Normalise at the boundary instead: truncate to `.Date`
in `ReserveResourceHandler` when constructing the `Reservation`. This makes in-memory, event and
stored values agree, requires no migration, and is a one-line change. Recommended.

**Failure modes.** Timezone-dependent off-by-one-day. Event/read divergence. Negative day counts
for dates before `0001-01-01` are unrepresentable but also unreachable (`DateTime` cannot go
earlier).

---

### 3.11 Optimistic concurrency

**Definition.** The mechanism preventing two concurrent reservations on the same resource from
overwriting each other.

**Representation & storage.**
`Infrastructure/Mongo/Repositories/ResourcesMongoRepository.cs:30-32`:

```csharp
public Task UpdateAsync(Resource resource)
    => _repository.Collection.ReplaceOneAsync(
        r => r.Id == resource.Id && r.Version < resource.Version,
        resource.AsDocument());
```

Note this **drops down to `_repository.Collection`** — Convey's `IMongoRepository<>` abstraction
does not offer a filtered replace, so the concurrency check is this repository's own code, written
by hand. The rest of the type uses the abstraction (`GetAsync`, `ExistsAsync`, `AddAsync`,
`DeleteAsync`).

**Lifecycle.** Every mutating handler calls `UpdateAsync` after mutating the aggregate. The filter
compares the *stored* version against the *in-memory, already-incremented* version (§3.2).

**Invariants & enforcement — and the silent hole.** The filter is correct: a stale write (whose
`Version` is not greater than what is stored) matches nothing and does not overwrite. **But
`ReplaceOneAsync`'s result is discarded.** `UpdateAsync` returns `Task`, not
`Task<ReplaceOneResult>`, and no caller inspects `ModifiedCount` or `MatchedCount`. Therefore:

> **A lost update is completely silent.** The handler continues, publishes its integration events
> as if the write had succeeded, and the caller receives HTTP 200 / an async success. The database
> is unchanged and the rest of the platform has been told it changed.

Two distinct situations trigger this:
1. **Genuine concurrency.** Two simultaneous reservations for different days on the same resource:
   both load `Version: 5`, both compute `Version: 6`, the first write succeeds, the second matches
   nothing. **One reservation is lost, `ResourceReserved` is published for it anyway.**
2. **No state change at all.** `ReleaseResourceReservation` for a non-existent reservation raises no
   event, so `Version` stays at 5, so `r.Version < 5` is false, so zero documents match. Here
   nothing is lost — but the same silent path is exercised (§3.12).

There is also **no retry**: nothing reloads the aggregate and re-applies the operation. And there is
no Mongo transaction wrapping the domain write with the outbox insert
(`outbox.disableTransactions: true`, `Api/appsettings.json:129`), so even a successful write and its
outbox record are not atomic (§3.17).

**Extension procedure — make lost updates loud.** Change the signature to return the result and
throw on `ModifiedCount == 0`:

```csharp
var result = await _repository.Collection.ReplaceOneAsync(
    r => r.Id == resource.Id && r.Version < resource.Version, resource.AsDocument());
if (result.ModifiedCount == 0) { throw new ConcurrentResourceUpdateException(resource.Id); }
```

Then add the new exception to **both** `ExceptionToResponseMapper` (→ 409 would be correct, though
see §3.18: the current mapper only ever emits 400) and `ExceptionToMessageMapper`. Note this makes
the no-op release path (§3.12) start throwing, so fix that first or the two changes interact.

**Failure modes.** Silent lost update (above). Version overflow at `int.MaxValue` — unreachable in
practice. A handler that calls `UpdateAsync` twice on the same instance: the second call's filter
fails because the stored version already equals the in-memory one.

---

### 3.12 Commands & command handlers

**Definition.** The four write operations. A command is an immutable DTO implementing Convey's
`ICommand`; its handler is an `internal sealed class` implementing `ICommandHandler<T>`.

**Representation & storage.** `Application/Commands/` and `Application/Commands/Handlers/`.

| Command | Fields | Handler | Distinguishing behaviour |
| --- | --- | --- | --- |
| `AddResource` | `ResourceId`, `Tags` | `AddResourceHandler` | **Normalises its own input in the constructor** (below) |
| `ReserveResource` | `ResourceId`, `DateTime`, `Priority`, `CustomerId` | `ReserveResourceHandler` | The only handler with an authorization check and the only one making an outbound HTTP call |
| `ReleaseResourceReservation` | `ResourceId`, `DateTime` | `ReleaseResourceReservationHandler` | The silent no-op path |
| `DeleteResource` | `ResourceId` | `DeleteResourceHandler` | **No authorization of any kind** |

**`AddResource` is the only command that defends itself** (`Application/Commands/AddResource.cs:14-16`):

```csharp
public AddResource(Guid resourceId, IEnumerable<string> tags)
    => (ResourceId, Tags) = (resourceId == Guid.Empty ? Guid.NewGuid() : resourceId,
        tags ?? Enumerable.Empty<string>());
```

Two normalisations with opposite effects: substituting a fresh GUID for `Guid.Empty` makes the
command safe (and is why `POST /resources` works without the gateway's `resourceId:` generation —
`component-internals/api-gateway.md` §3.11); but replacing `null` tags with an **empty** enumerable
converts a null-reference into a `MissingResourceTagsException`, which is the *intended* loud
failure. The other three commands perform no normalisation, so `Guid.Empty` reaches the aggregate
and throws from the implicit conversion (§3.3).

**Lifecycle of a handler.** Constructed per request by the DI container (`AddCommandHandlers()`,
`Application/Extensions.cs:13`, which is Convey's assembly scan) and **wrapped in three decorators**
before execution (§3.17): Outbox → Jaeger → Logging.

Every handler follows the same four-step shape, and the ordering is the important part:

```
1. guard              (only ReserveResourceHandler has one)
2. load               _repository.GetAsync / ExistsAsync   → throw if wrong
3. mutate + persist   resource.X(); _repository.Add/Update/DeleteAsync(resource)
4. publish            _eventProcessor.ProcessAsync(resource.Events)
```

**Step 3 and step 4 are not atomic** in any handler. The outbox mitigates this only if
`outbox.enabled` (§3.17) — and even then `disableTransactions: true` means the domain write and the
outbox insert are separate operations. See §3.17 for exactly what this does and does not buy.

**Invariants & enforcement, handler by handler.**

- `AddResourceHandler.cs:23-26` — `ExistsAsync` then throw `ResourceAlreadyExistsException`.
  **Loud.** This is a check-then-act race: two concurrent adds of the same id can both pass
  `ExistsAsync`. `AddAsync` is an unconditioned insert, and there is **no unique index** on the
  collection (nothing in `Infrastructure/` creates indexes; `mongo.seed: false`), so the second
  insert succeeds and **two documents with the same `Id` exist**. Every subsequent read returns one
  of them arbitrarily. This is a genuine, silent corruption path.
- `DeleteResourceHandler.cs:22-31` — loads, throws `ResourceNotFoundException` if absent
  (**loud**), calls `resource.Delete()` to raise the events, then `DeleteAsync`. Note the ordering:
  the aggregate is deleted from Mongo *before* the events are published, and `DeleteAsync` uses no
  version filter — so a delete always wins over a concurrent update. **There is no authorization
  check: any caller who can reach this handler can delete any resource.**
- `ReleaseResourceReservationHandler.cs:30-33` — **the silent path**:

  ```csharp
  var reservation = resource.Reservations.FirstOrDefault(r => r.DateTime.Date == command.DateTime.Date);
  resource.ReleaseReservation(reservation);
  await _repository.UpdateAsync(resource);
  await _eventProcessor.ProcessAsync(resource.Events);
  ```

  Because `Reservation` is a **struct** (§3.5), `FirstOrDefault` returns `default(Reservation)` —
  not `null` — when nothing matches, and the code does not check. The chain is then:
  `ReleaseReservation(default)` → `HashSet.Remove` returns false → early `return`, **no event** →
  `Version` unchanged → `UpdateAsync`'s filter `r.Version < resource.Version` matches **zero
  documents** → `resource.Events` is empty → `EventProcessor` returns early. **Four consecutive
  silent no-ops, and the caller receives HTTP 200 (or an async success) for an operation that did
  nothing.** There is no code path by which a release of a non-existent reservation can be
  reported. This is the single most consequential defect in the component.
- `ReserveResourceHandler.cs:29-33` — the ownership guard:

  ```csharp
  var identity = _appContext.Identity;
  if (identity.IsAuthenticated && identity.Id != command.CustomerId && !identity.IsAdmin)
      throw new UnauthorizedResourceAccessException(command.ResourceId, identity.Id);
  ```

  Read the first conjunct carefully. **The guard is a no-op when `IsAuthenticated` is false.** An
  unauthenticated caller is not denied; it is *exempted*. Since `IsAuthenticated` comes entirely
  from the gateway-supplied header (§3.19/§3.20), a request that reaches port 5001 without a
  `Correlation-Context` header — which any process on the network can make — reserves on behalf of
  **any** `CustomerId` it names. Written as fail-open, this is the security posture of the whole
  service.

**Extension procedure — add a command.**
1. Add the DTO to `Application/Commands/` implementing `ICommand`, with `[Contract]` (§3.23).
2. Add the handler to `Application/Commands/Handlers/` as `internal sealed`; Convey's
   `AddCommandHandlers()` finds it by scan — **no registration needed, and no error if the handler
   is missing.** A command with no handler dispatched over HTTP produces a Convey-level failure;
   over AMQP, `Unverifiable — Missing Source Evidence` whether the message is nacked or dropped.
3. Map the HTTP verb in `Api/Program.cs` (§3.24).
4. Add `.SubscribeCommand<TCommand>()` in `Infrastructure/Extensions.cs`'s `UseInfrastructure`
   (§3.21) — **omitting this leaves the async route publishing into a void.**
5. Add the exception→rejected-event cases (§3.16) for every failure the handler can produce.
6. Add the gateway route in all four manifests.

**Failure modes.** Silent release no-op (above). Duplicate-id race on add (above). Unauthenticated
exemption on reserve (above). Unauthorized delete (above). A handler that throws after
`UpdateAsync` but before `ProcessAsync` persists state without publishing — the outbox does not
cover this, because the outbox entry is written *inside* `ProcessAsync` (§3.15 → §3.27).

---

### 3.13 Queries & the read model

**Definition.** The two read operations, served directly from Mongo documents without loading the
aggregate.

**Representation & storage.** `Application/Queries/`:

| Query | Result | Parameters |
| --- | --- | --- |
| `GetResource : IQuery<ResourceDto>` | one resource or `null` | `Guid ResourceId` |
| `GetResources : IQuery<IEnumerable<ResourceDto>>` | a flat list | `IEnumerable<string> Tags`, `bool MatchAllTags` |

Note both use `{ get; set; }` (mutable) rather than the constructor-immutability of the commands —
that is deliberate: Convey binds query properties from the **query string**, which requires setters.

The DTOs (`Application/DTO/`) are deliberately thin: `ResourceDto { Id, Tags, Reservations }` and
`ReservationDto { DateTime, Priority }`. **`Version` is not exposed**, so a client cannot implement
its own optimistic concurrency against this service.

Handlers live in `Infrastructure/Mongo/Queries/` (not in `Application/`) because they depend on the
Mongo driver — an intentional layering choice consistent with the rest of the service.

**Lifecycle.** Bound from the URL by the dispatcher (§3.24), executed against the `resources`
collection, mapped document → DTO by `AsDto()` (`Infrastructure/Mongo/Documents/Extensions.cs`).

**Invariants & enforcement.**
- **The read path never constructs a `Resource`.** It therefore never runs `ValidateTags` and never
  applies any domain rule. A document that the *write* path would reject reads back fine here. This
  asymmetry is easy to miss and matters when reasoning about data repair.
- **`GetResource` returning `null` is what produces the only non-400 error this service emits.**
  Convey's dispatcher endpoint returns **404** for a `null` query result — confirmed by the E2E test
  (`tests/…Tests.EndToEnd/`, which asserts `HttpStatusCode.NotFound` for a missing resource). This
  is Convey behaviour, not this repository's code: `Unverifiable — Missing Source Evidence` for the
  exact mechanism, but the *observable* is verified by a test in this repository.
- **`MatchAllTags` semantics** — the flag switches the Mongo filter between "resource has all of
  these tags" and "resource has any of these tags". Read
  `Infrastructure/Mongo/Queries/GetResourcesHandler.cs` before relying on the default:
  `bool` defaults to `false`, so **omitting the parameter gives ANY-match**, which is the more
  permissive of the two. A client that forgets the flag gets a wider result set, silently.
- **No pagination.** `GetResources` returns every match. There is no `page`/`results` parameter, no
  limit, and no `Total-Count` header — even though the gateway's CORS config exposes that header
  (`component-internals/api-gateway.md` §3.18) because *other* services paginate. A tag matching
  100 000 resources returns 100 000 documents in one response.
- **No authorization on either query.** Any caller reaching the service can enumerate every
  resource and every reservation date. Because reservations carry no customer id (§3.5), this leaks
  *when* resources are booked but not *by whom* — a meaningful mitigation, and an accidental one.

**Extension procedure — add a query.** Add the `IQuery<T>` type with settable properties, add the
handler under `Infrastructure/Mongo/Queries/`, map it in `Api/Program.cs` with
`.Get<TQuery, TResult>("route")`, and add the gateway route. To add pagination, Convey provides
`IPagedQuery`/`PagedResult`; adopting it changes the response envelope shape and is therefore a
breaking change for existing clients.

**Failure modes.** Unbounded result sets. Silent ANY-vs-ALL confusion. **Duplicate ids crash the
read**: `GetResourceHandler.cs:23` uses `SingleOrDefaultAsync`, which *throws* when more than one
document matches — so the check-then-act race in `AddResourceHandler` (§3.12) turns into a permanent
generic 400 on every subsequent `GET /resources/{id}` for that id, with no indication of the real
cause. `GetResourcesHandler` uses `ToListAsync` and is unaffected, so the resource still appears in
list results — twice.

---

### 3.14 Integration events & `EventMapper`

**Definition.** The public, cross-service wire contract. Integration events are distinct types from
domain events (§3.7), live in `Application/Events/`, implement Convey's `IEvent`, and carry only
primitives.

**Representation & storage.** Five integration events, each `[Contract]`-attributed:

| Integration event | Fields | Mapped from |
| --- | --- | --- |
| `ResourceAdded` | `ResourceId` | `Core.Events.ResourceCreated` |
| `ResourceDeleted` | `ResourceId` | `Core.Events.ResourceDeleted` |
| `ResourceReserved` | `ResourceId`, `DateTime` | `Core.Events.ReservationAdded` |
| `ResourceReservationReleased` | `ResourceId`, `DateTime` | `Core.Events.ReservationReleased` |
| `ResourceReservationCanceled` | `ResourceId`, `DateTime` | `Core.Events.ReservationCanceled` |

The mapping is a single hand-written `switch` expression —
`Infrastructure/Services/EventMapper.cs:17-26`:

```csharp
public IEvent Map(IDomainEvent @event)
    => @event switch {
        ResourceCreated e => (IEvent) new ResourceAdded(e.Resource.Id),
        ResourceDeleted e => new Application.Events.ResourceDeleted(e.Resource.Id),
        ReservationAdded e => new ResourceReserved(e.Resource.Id, e.Reservation.DateTime),
        ReservationReleased e => new ResourceReservationReleased(e.Resource.Id, e.Reservation.DateTime),
        ReservationCanceled e => new ResourceReservationCanceled(e.Resource.Id, e.Reservation.DateTime),
        _ => null };
```

Note lines 7-8 of the file: `using ReservationCanceled = …Core.Events.ReservationCanceled;` and
`using ResourceDeleted = …Core.Events.ResourceDeleted;`. **Domain and integration events share type
names**, and the file disambiguates with aliases plus one fully-qualified reference
(`Application.Events.ResourceDeleted` at `:21`). Any new event whose name collides must repeat this
dance; getting it wrong produces a *compile* error, which is the one place in this pipeline where
the failure is loud.

**Lifecycle.** Called once per domain event by `EventProcessor.HandleDomainEvents`
(`EventProcessor.cs:62`), immediately after the in-process `IDomainEventHandler<>` fan-out.

**Invariants & enforcement — the silent drop.** `_ => null` at `:25` combined with
`EventProcessor.cs:63-66`:

```csharp
var integrationEvent = _eventMapper.Map(@event);
if (integrationEvent is null) { continue; }
```

A domain event with no case in the switch is **silently discarded**. No log line (the `LogTrace` at
`:54` fires *before* the map and says only "Handling domain event: X"), no exception, no metric.
Adding a domain event and forgetting the mapper case produces a system that appears to work and
never tells the rest of the platform anything.

Two further properties of the mapping that matter to consumers:
- **The published `DateTime` is the in-memory value, with time-of-day intact** — while a subsequent
  read of the same reservation returns midnight (§3.10). Consumers that persist the event value and
  later reconcile against a `GET` will find a mismatch.
- **Expropriation publishes two events, in raise order**: `ResourceReservationCanceled` then
  `ResourceReserved` (from `Resource.AddReservation`'s ordering, `Core/Entities/Resource.cs:70`
  then `:76`). `MessageBroker.PublishAsync` iterates the list in order (`MessageBroker.cs:67`), so
  ordering is preserved *at publish*. Whether it survives the outbox and the broker is
  `Unverifiable — Missing Source Evidence` — `outbox.type: "sequential"`
  (`Api/appsettings.json:127`) suggests it does, but nothing in this repository proves it.
- **Neither event names the customer** (§3.6), so a consumer cannot tell who was evicted.

**Extension procedure.** Add the integration event class with `[Contract]`, add the `switch` case,
and — if the event is part of the published contract package — add it to
`Pacco.Services.Availability.Contracts` too (§3.23). Three places; the compiler enforces none of
the linkage.

**Failure modes.** Missing switch case → silent, total loss of the event. Renaming an integration
event class → the RabbitMQ routing key changes (it is derived from the type name under
`snakeCase` conventions, §3.22) → **every subscriber silently stops receiving it**, with no error
on either side.

---

### 3.15 `EventProcessor` — the publish pipeline

**Definition.** The single funnel through which every domain event leaves the aggregate. It does
two jobs: dispatch in-process domain-event handlers, and collect mapped integration events for
publication.

**Representation & storage.** `Infrastructure/Services/EventProcessor.cs`, 74 lines.

```csharp
public async Task ProcessAsync(IEnumerable<IDomainEvent> events) {
    if (events is null) { return; }                          // :31-34
    _logger.LogTrace("Processing domain events...");
    var integrationEvents = await HandleDomainEvents(events);
    if (!integrationEvents.Any()) { return; }                // :38-41
    _logger.LogTrace("Processing integration events...");
    await _messageBroker.PublishAsync(integrationEvents); }
```

`HandleDomainEvents` (`:47-72`) opens **one DI scope for the whole batch**
(`using var scope = _serviceScopeFactory.CreateScope()`, `:50`) and, per event:

```csharp
var handlerType = typeof(IDomainEventHandler<>).MakeGenericType(eventType);
dynamic handlers = scope.ServiceProvider.GetServices(handlerType);
foreach (var handler in handlers) { await handler.HandleAsync((dynamic) @event); }
```

**This is reflection plus `dynamic` dispatch** (`:55-60`). Three consequences a maintainer must
know:
1. The handler lookup is by **exact runtime type**, not by assignability — a handler registered for
   a base type is never invoked for a derived event.
2. `GetServices` returning an empty sequence is normal and silent; today it *always* returns empty
   (§3.34).
3. `(dynamic) @event` defers binding to runtime. A handler whose `HandleAsync` signature does not
   match throws `RuntimeBinderException` at the call site — a **runtime**, not compile-time,
   failure, and one that reads as an unrelated internal error.

Handlers run **before** the mapping, sequentially, and **inside the same `await` chain as the
request** — an expensive domain-event handler directly extends request latency, and a throwing one
aborts the whole batch so that **no integration event is published for any event in it**, including
those already mapped.

**Lifecycle.** Called at the end of every command handler with `resource.Events`. Never called
anywhere else — verified by search.

**Invariants & enforcement.**
- `events is null` → silent return (`:31-34`). Unreachable in practice: `AggregateRoot.Events`
  is never null.
- **Empty batch → silent return** (`:38-41`). This is the mechanism by which the release no-op
  (§3.12) publishes nothing.
- No transaction, no ordering guarantee across batches, and no failure isolation between the
  in-process handlers and the outbound publish.
- `ClearEvents()` is **not** called afterwards. Harmless as used (the aggregate is discarded), but
  it means calling `ProcessAsync` twice on the same aggregate **republishes every event**.

**Extension procedure — react to a domain event in-process.** Implement
`IDomainEventHandler<TEvent>` (`Application/Events/IDomainEventHandler.cs`) and register it in the
container. Registration is via the assembly scan in `Infrastructure/Extensions.cs`, which
enumerates types implementing the open generic — see §3.34, which explains why zero such handlers
exist today and what to check when you add the first one.

**Failure modes.** Throwing handler → whole batch lost. Missing map case → single event lost
(§3.14). Double invocation → duplicate publication.

---

### 3.16 Rejected events & `ExceptionToMessageMapper`

**Definition.** The AMQP failure contract. When a command consumed from RabbitMQ throws, Convey
asks this mapper to convert the exception into an `IRejectedEvent` and publishes that instead. It is
the async equivalent of an HTTP error response, and it is what `operations-service` turns into the
terminal state a polling client sees.

**Representation & storage.** Four rejected-event types in `Application/Events/Rejected/`:

| Type | Fields |
| --- | --- |
| `AddResourceRejected` | `ResourceId`, `Reason`, `Code` |
| `DeleteResourceRejected` | `ResourceId`, `Reason`, `Code` |
| `ReleaseResourceRejected` | `ResourceId`, `DateTime`, `Reason`, `Code` |
| `ReleaseResourceReservationRejected` | `ResourceId`, `DateTime`, `Reason`, `Code` |

**There is no `ReserveResourceRejected`.** That absence is the defect the mapper is built around.

`Infrastructure/Exceptions/ExceptionToMessageMapper.cs:12-56` is a nested `switch` on
`(exception, message)` — the incoming *command* is the second discriminant, which is what lets one
exception type produce different rejections for different commands. The complete table:

| Exception | Incoming command | Rejected event published |
| --- | --- | --- |
| `MissingResourceTagsException` | *any* | `AddResourceRejected(**Guid.Empty**, …)` |
| `InvalidResourceTagsException` | *any* | `AddResourceRejected(**Guid.Empty**, …)` |
| `ResourceAlreadyExistsException` | *any* | `AddResourceRejected(ex.Id, …)` |
| `CannotExpropriateReservationException` | `ReserveResource` | **`ReleaseResourceReservationRejected`** |
| `CannotExpropriateReservationException` | anything else | **`null`** |
| `CustomerNotFoundException` | `ReserveResource` | **`ReleaseResourceReservationRejected`** |
| `InvalidCustomerStateException` | `ReserveResource` | **`ReleaseResourceReservationRejected`** |
| `ResourceNotFoundException` | `DeleteResource` | `DeleteResourceRejected` |
| `ResourceNotFoundException` | `ReserveResource` | **`ReleaseResourceReservationRejected`** |
| `ResourceNotFoundException` | `ReleaseResourceReservation` | `ReleaseResourceRejected` |
| `UnauthorizedResourceAccessException` | `ReserveResource` | **`ReleaseResourceReservationRejected`** |
| anything else | *any* | **`null`** |

**Three distinct problems are visible in that table, all of them silent:**

1. **Every reserve failure is reported under a release event name.** Five of the eleven mapped
   rows publish `ReleaseResourceReservationRejected` in response to a `ReserveResource` command. A
   consumer cannot distinguish "your reservation was refused" from "your release was refused" by
   event type — only by reading `Code`. This is not a naming quibble: a subscriber that binds only
   to reserve-related routing keys **never learns that a reservation failed.**
2. **Tag failures lose the resource id.** `MissingResourceTagsException` and
   `InvalidResourceTagsException` carry no id (§3.8), so the mapper hard-codes `Guid.Empty`
   (`:15-18`). An async caller receives a rejection it cannot correlate to its request except
   through the message context.
3. **Unmapped combinations publish nothing.** Every `_ => null` — six of them, including the final
   catch-all at `:55` — means the command fails and **no event is published at all.** The message
   is consumed, the work does not happen, and `operations-service` is never told, so the client's
   operation stays pending indefinitely. Notable unmapped cases: `InvalidAggregateIdException`
   (§3.3), any `MongoException`, any `HttpRequestException` from `CustomersServiceClient` (§3.25),
   and `UnauthorizedResourceAccessException` arriving on any command other than `ReserveResource`.

`ReleaseResourceRejected` and `ReleaseResourceReservationRejected` are structurally identical and
differ only in name; only the former is used for actual releases, and only in one row.

**Lifecycle.** Registered as `IExceptionToMessageMapper` in `Infrastructure/Extensions.cs`; invoked
by Convey's RabbitMQ subscriber when a handler throws. Whether Convey nacks/requeues the original
message in addition to publishing the rejection is `Unverifiable — Missing Source Evidence`.

**Extension procedure — add a rejection.**
1. Add the `IRejectedEvent` type in `Application/Events/Rejected/` with `[Contract]`, carrying
   `Reason` and `Code` plus enough identity to correlate.
2. Add a `case` for **every (exception, command) pair** it can arise from — the nested switch means
   one case per pair, not one per exception.
3. Add it to the Contracts package (§3.23).
4. **Fix the reserve naming while you are there**: adding `ReserveResourceRejected` and
   re-pointing the five reserve rows is a contract-breaking change for any current subscriber, so it
   must be coordinated — publish both for a transition period.

**Failure modes.** All three problems above. Additionally: because the mapper returns `object`
(untyped), a case returning a non-`IRejectedEvent` compiles fine and fails at runtime inside Convey.

---

### 3.17 Transactional outbox & inbox

**Definition.** The mechanism intended to make "state changed" and "the world was told" reliable
across a process crash.

**Representation & storage.** Configuration at `Api/appsettings.json:102-110`:

| Key | Value | Meaning |
| --- | --- | --- |
| `outbox.enabled` | `true` (`false` in `local`) | master switch |
| `outbox.type` | `sequential` | ordering discipline |
| `outbox.expiry` | `3600` | seconds before a sent message is purged |
| `outbox.intervalMilliseconds` | `2000` | dispatcher poll period |
| `outbox.inboxCollection` | `inbox` | consumed-message ledger |
| `outbox.outboxCollection` | `outbox` | pending-publication ledger |
| `outbox.disableTransactions` | **`true`** | **no Mongo multi-document transaction** |

Wiring is in `Infrastructure/Extensions.cs`, using Scrutor decoration:

```csharp
.AddMessageOutbox(o => o.AddMongo())
.TryDecorate(typeof(ICommandHandler<>), typeof(OutboxCommandHandlerDecorator<>))
.TryDecorate(typeof(IEventHandler<>), typeof(OutboxEventHandlerDecorator<>))
```

Together with the Jaeger and Logging decorators Convey adds, every command handler executes inside a
three-layer wrapper — **Outbox → Jaeger → Logging → handler** — none of which is visible at the
handler's own call site.

**Lifecycle.**
1. The decorator computes a message id for the inbox check.
2. If that id is already in `inbox`, the handler is **skipped** (idempotent consumption).
3. The handler runs; any `_outbox.SendAsync` inside it (from `MessageBroker`, §3.27) writes to
   `outbox` instead of publishing directly.
4. A background dispatcher polls `outbox` every 2000 ms and publishes.
5. Sent rows are purged after `expiry`.

**Invariants & enforcement — what this does and does not guarantee.**

- **`disableTransactions: true` means the domain write and the outbox insert are *not* atomic.**
  The handler writes the `resources` document (§3.11), then `ProcessAsync` → `MessageBroker` →
  `_outbox.SendAsync` writes the `outbox` document. A crash between them leaves state changed and
  **no event ever published**. That is precisely the dual-write failure an outbox exists to prevent,
  and the configuration disables the prevention. The setting is presumably there because Mongo
  transactions require a replica set, which the `docker-compose` topology does not provide —
  `Unverifiable — Missing Source Evidence` for the intent, but the trade-off is real either way.
- **The inbox is effective only on the AMQP path.** Its key is the broker `MessageId`. On the HTTP
  path there is no incoming message, and `MessageBroker.PublishAsync` mints
  `Guid.NewGuid().ToString("N")` per event (`Infrastructure/Services/MessageBroker.cs:74`), so
  every HTTP invocation looks new. **A retried HTTP POST is not de-duplicated** — which matters
  because the gateway retries blindly (`component-internals/api-gateway.md` §3.20).
- **`outbox.enabled: false` in `appsettings.local.json`** means local development exercises a
  *different* code path — direct publish (`MessageBroker.cs:83`) rather than
  `_outbox.SendAsync` (`:78`). Behaviour that works locally may differ in Docker. This is a real
  environment-divergence trap.
- Ordering: `type: sequential` implies the dispatcher preserves insertion order.
  `Unverifiable — Missing Source Evidence` (Convey internals) — do not depend on it for the
  cancel-then-reserve pair (§3.14) without testing.
- **At-least-once, not exactly-once.** A crash after publishing but before marking the row sent
  republishes. Consumers must be idempotent; nothing here makes them so.

**Extension procedure.**
- To enable transactions: provision Mongo as a replica set and set `disableTransactions: false`.
  Verify against every environment file — `local` and `docker` each override the outbox block.
- To de-duplicate the HTTP path: derive the message id from the gateway's `RequestId`
  (`IAppContext.RequestId`, available in every handler) instead of a fresh GUID, in
  `MessageBroker.PublishAsync`. That single change would make retried HTTP writes idempotent.
- To add a decorator: `TryDecorate` in `Infrastructure/Extensions.cs`. **Order matters and is
  registration-order dependent** — a new decorator registered before the outbox one runs outside
  the inbox check.

**Failure modes.** Lost event on a crash between the two writes (above). Unbounded `outbox` growth
if the dispatcher is stopped while the service keeps writing. Duplicate publication on redelivery.
Local/docker behavioural divergence.

---

### 3.18 `ExceptionToResponseMapper` — HTTP status mapping

**Definition.** The HTTP failure contract: how an exception becomes a status code and a body.

**Representation & storage.** `Infrastructure/Exceptions/ExceptionToResponseMapper.cs:15-24`:

```csharp
public ExceptionResponse Map(Exception exception)
    => exception switch {
        DomainException ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.BadRequest),
        AppException ex    => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.BadRequest),
        _                  => new ExceptionResponse(new {code = "error", reason = "There was an error."}, HttpStatusCode.BadRequest) };
```

**Every branch returns `HttpStatusCode.BadRequest`.** There is no 401, no 403, no 404, no 409, no
500 anywhere in this file. The full HTTP error vocabulary of this service is therefore:

| Situation | Status | Body |
| --- | --- | --- |
| Any `DomainException` | **400** | `{code: <ex.Code>, reason: <ex.Message>}` |
| Any `AppException` | **400** | `{code: <ex.Code>, reason: <ex.Message>}` |
| Anything else (Mongo down, HTTP client failure, NRE, …) | **400** | `{code: "error", reason: "There was an error."}` |
| A query returning `null` | **404** | (empty — produced by Convey's dispatcher, not this mapper) |

Concretely: `UnauthorizedResourceAccessException` → **400, not 403**.
`ResourceNotFoundException` → **400, not 404** — while a *missing* resource on a `GET` gives 404
from the dispatcher. So `GET /resources/{unknown}` returns 404 and
`DELETE /resources/{unknown}` returns 400, for the same underlying condition. **Mongo being
unreachable also returns 400**, telling a client its request was malformed when the service is
simply down; nothing distinguishes a client error from an outage, and no monitor keyed on 5xx will
ever fire.

The generic branch is, at least, correct about disclosure: it deliberately does **not** leak the
exception message for unmapped types. (Contrast the gateway, whose
`customErrors.includeExceptionMessage: true` will happily relay whatever it receives.)

`GetCode` (`:26-45`) is described in §3.8.

**Lifecycle.** Registered as `IExceptionToResponseMapper`; invoked by Convey's
`UseErrorHandler`/exception middleware (`Infrastructure/Extensions.cs`, `UseInfrastructure`).

**Invariants & enforcement.** None — this file *is* the enforcement, and it enforces uniformity.
Nothing validates that a status is appropriate.

**Extension procedure — return meaningful statuses.** Add cases before the catch-all:

```csharp
UnauthorizedResourceAccessException ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.Forbidden),
ResourceNotFoundException ex          => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.NotFound),
ResourceAlreadyExistsException ex     => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message}, HttpStatusCode.Conflict),
```

and change the final branch to `InternalServerError`. **Check the E2E tests first**
(`tests/…Tests.EndToEnd/`) — they assert specific statuses, and changing the catch-all to 500 is
the kind of change that a green build will not catch, because that repository's `.travis.yml` does
not run the tests (§3.36).

**Failure modes.** Infrastructure outages indistinguishable from client errors. Clients cannot
branch on status and must parse `code`. Retry logic in any HTTP client sees 400 (non-retryable) for
transient failures — arguably a lucky accident, since it prevents retry storms, but not by design.

---

### 3.19 `IAppContext` / `IIdentityContext`

**Definition.** The in-process representation of "who is calling and under what request id" — the
only identity this service has.

**Representation & storage.** Interfaces in `Application/` (so handlers depend on abstractions),
implementations in `Infrastructure/Contexts/` (all `internal sealed`).

```csharp
public interface IAppContext { string RequestId { get; }  IIdentityContext Identity { get; } }
public interface IIdentityContext { Guid Id; string Role; bool IsAuthenticated; bool IsAdmin; IDictionary<string,string> Claims; }
```

`IdentityContext`'s constructor (`Infrastructure/Contexts/IdentityContext.cs:24-31`) is where every
defensive decision lives:

```csharp
Id = Guid.TryParse(id, out var userId) ? userId : Guid.Empty;   // :26
Role = role ?? string.Empty;                                    // :27
IsAuthenticated = isAuthenticated;                              // :28  ← taken verbatim
IsAdmin = Role.Equals("admin", StringComparison.InvariantCultureIgnoreCase);  // :29
Claims = claims ?? new Dictionary<string,string>();             // :30
```

**Every field is defended except the one that matters.** A malformed user id becomes `Guid.Empty`
rather than throwing; a null role becomes `""`; null claims become an empty dictionary. But
`IsAuthenticated` is **copied straight from the JSON the gateway sent**, with no verification of any
kind. That is the trust boundary, and it is a boolean in a header.

Two further details:
- `IsAdmin` is a **case-insensitive** comparison against the literal `"admin"`. The gateway's claim
  gate is a **case-sensitive** exact match against `admin`
  (`component-internals/api-gateway.md` §3.8). This service is therefore the more permissive of the
  two — `Admin` passes here and fails there.
- `Guid.TryParse` failing silently means a garbled user id produces `Id = Guid.Empty`, which then
  compares unequal to any real `CustomerId` — so a malformed identity **fails closed** on the
  ownership check (assuming `IsAuthenticated` is true). That is the right behaviour, achieved
  incidentally.

`AppContext` (`Infrastructure/Contexts/AppContext.cs`) has three constructors: a parameterless one
minting `Guid.NewGuid().ToString("N")` as the request id with `IdentityContext.Empty` (`:11`), one
taking a `CorrelationContext` (`:15-18`, which itself falls back to `IdentityContext.Empty` when
`context.User is null`), and the primary. `AppContext.Empty` (`:26`) is the value used whenever no
correlation context is present — **an unauthenticated, admin-less identity with `Id = Guid.Empty`.**

**Lifecycle.** Produced per request by `AppContextFactory.Create()` (§3.20) and injected into
handlers that ask for `IAppContext` — today only `ReserveResourceHandler`.

**Invariants & enforcement.**
- `IIdentityContext` is never null: every path yields either a populated `IdentityContext` or
  `IdentityContext.Empty`. Handlers need no null check, and none has one.
- **`AppContext.Empty` is fail-open in the one place it is consulted** (§3.12,
  `ReserveResourceHandler.cs:30`): `IsAuthenticated == false` short-circuits the guard.
- `RequestId` is available to every handler but is used **only** for logging; it is not the outbox
  message id (§3.17), which is the missed opportunity that leaves the HTTP path un-de-duplicated.

**Extension procedure — make identity fail closed.** Change the guard to

```csharp
if (!identity.IsAuthenticated) { throw new UnauthorizedResourceAccessException(command.ResourceId, identity.Id); }
if (identity.Id != command.CustomerId && !identity.IsAdmin) { throw new …; }
```

and apply the same pattern to `DeleteResourceHandler`, which has no guard at all. **Check the
tests before doing so**: the unit tests construct handlers with a substituted `IAppContext`, and the
E2E tests drive the service through `PaccoApplicationFactory` **without** a `Correlation-Context`
header — so making identity mandatory will fail every end-to-end test until they are updated. That
is a signal about the current posture, not an argument against the change.

**Extension procedure — add a field.** Add it to `IIdentityContext`, to `IdentityContext`, to
`CorrelationContext.UserContext` (§3.20), **and** to the gateway's own duplicate class. Four files
across two repositories; three of them and it silently reads as `default`.

**Failure modes.** Forged header → full impersonation, including `Role: admin`. Absent header →
unauthenticated exemption. Case-mismatched role → admin at this service but not at the gateway.

---

### 3.20 `Correlation-Context` header ingestion

**Definition.** The one place this service reads the caller's identity off the wire.

**Representation & storage.** Two cooperating pieces.

*The header read* — `Infrastructure/Extensions.cs:117-120`:

```csharp
internal static CorrelationContext GetCorrelationContext(this IHttpContextAccessor accessor)
    => accessor.HttpContext?.Request.Headers.TryGetValue("Correlation-Context", out var json) is true
        ? JsonConvert.DeserializeObject<CorrelationContext>(json.FirstOrDefault())
        : null;
```

*The transport-agnostic factory* — `Infrastructure/Contexts/AppContextFactory.cs:19-33`:

```csharp
public IAppContext Create() {
    if (_contextAccessor.CorrelationContext is {}) {                       // AMQP path
        var payload = JsonConvert.SerializeObject(_contextAccessor.CorrelationContext);
        return string.IsNullOrWhiteSpace(payload) ? AppContext.Empty
            : new AppContext(JsonConvert.DeserializeObject<CorrelationContext>(payload)); }
    var context = _httpContextAccessor.GetCorrelationContext();            // HTTP path
    return context is null ? AppContext.Empty : new AppContext(context); }
```

The AMQP branch does a **serialize-then-deserialize round trip** (`:23`, `:27`) to convert Convey's
loosely-typed correlation context into this service's own `CorrelationContext` class. It is a
JSON-shaped structural cast — expensive, and dependent on property names matching on both sides.

`Infrastructure/Contexts/CorrelationContext.cs` is the local DTO: `CorrelationId`, `SpanContext`,
`User { Id, IsAuthenticated, Role, Claims }`, `ResourceId`, `TraceId`, `ConnectionId`, `Name`,
`CreatedAt` — **field-for-field identical to the gateway's
`src/Pacco.APIGateway/Infrastructure/CorrelationContext.cs`**, hand-copied, `internal sealed`, with
no shared package and no version marker. Neither side references the other.

Registration makes this per-request-transparent:
`builder.Services.AddTransient(ctx => ctx.GetRequiredService<IAppContextFactory>().Create())`
(`Infrastructure/Extensions.cs:64`) — so injecting `IAppContext` anywhere yields a freshly built
context.

**Lifecycle.** Built on demand, per resolution of `IAppContext`. Note that `AddTransient` means
**two injections in one request build it twice**, re-running the JSON round trip. Today only one
handler injects it, so this is latent.

**Invariants & enforcement.**
- Header absent → `null` → `AppContext.Empty` → unauthenticated (§3.19). **Silent, and fail-open at
  the one guard that consults it.**
- Header present but not valid JSON → `JsonConvert.DeserializeObject` **throws** →
  `ExceptionToResponseMapper`'s catch-all → generic **400** (§3.18). So a *malformed* header fails
  loudly-ish while an *absent* one passes silently — the opposite of what security requires.
- Multiple `Correlation-Context` headers → `json.FirstOrDefault()` takes the first, ignoring the
  rest, silently.
- **No signature, no MAC, no mutual TLS on the inbound path.** The header is a bearer assertion of
  identity that anyone who can open a TCP connection to port 5001 can write. The service's Vault
  PKI certificates (§3.25) protect its *outbound* calls only. This is the platform's central
  security assumption: the network is trusted and only the gateway can reach the services.
- The forwarded header set is deliberately narrow — `GetHeadersToForward`
  (`Infrastructure/Extensions.cs:122-136`) forwards **only `Saga`**, returning `null` for
  everything else. Nothing else propagates automatically.
- `GetSpanContext` (`:138-151`) reads the span header and **requires the value to be `byte[]`**
  (`:145`); a `string`-valued header yields `string.Empty` and the trace silently breaks at that hop.

**Extension procedure — add a forwarded header.** Add it to `GetHeadersToForward`. Note it returns
`null` (not an empty dictionary) when nothing matches — a shape that callers must tolerate;
`MessageBroker.cs:63` passes the result straight through.

**Extension procedure — authenticate the inbound path.** Two options, in increasing order of
correctness: (a) require the gateway's client certificate on inbound calls too —
`.AddCertificateAuthentication()` / `.UseCertificateAuthentication()` are **already wired**
(`Infrastructure/Extensions.cs:92`, `:105`), so this may be a configuration change rather than a
code change; (b) sign the correlation envelope at the gateway and verify here, which requires the
matching change in `component-internals/api-gateway.md` §3.14. Option (a) is the smaller step and
uses machinery that already exists in the composition root.

**Failure modes.** Forgery, absence-as-anonymity, first-header-wins, malformed-JSON-as-400.
Silent divergence from the gateway's copy of the DTO whenever either side adds a field.

---

### 3.21 External event subscription

**Definition.** How this service consumes events published by other services, and how the four
commands become AMQP-consumable.

**Representation & storage.** The complete subscription list is
`Infrastructure/Extensions.cs:106-112`:

```csharp
.UseRabbitMq()
.SubscribeCommand<AddResource>()
.SubscribeCommand<DeleteResource>()
.SubscribeCommand<ReleaseResourceReservation>()
.SubscribeCommand<ReserveResource>()
.SubscribeEvent<CustomerCreated>()
.SubscribeEvent<VehicleDeleted>()
```

Six subscriptions: four own commands (matching the gateway's four `availability` routing keys —
`component-internals/api-gateway.md` §3.16) and two foreign events.

The foreign events declare **which exchange they come from** via `[Message]`:

| Type | Attribute | Fields | Handler |
| --- | --- | --- | --- |
| `CustomerCreated` | `[Message("customers")]` | `CustomerId` | `CustomerCreatedHandler` |
| `VehicleDeleted` | `[Message("vehicles")]` | `VehicleId` | `VehicleDeletedHandler` |

Both are **local re-declarations** of types owned by other services
(`Application/Events/External/`). There is no shared contracts package for inbound events: this
service defines its own class with the properties it needs, and deserialization matches by name.
**A rename in `customers-service` silently produces `Guid.Empty` here**, not an error.

**Handler behaviour.**

- `CustomerCreatedHandler.cs:10-11` — `=> Task.CompletedTask`, with a comment explaining that
  customer data *could* be cached locally. **It is a deliberate no-op.** The subscription exists,
  the queue is declared and drained, and nothing happens. Worth knowing before assuming this service
  has a customer projection: it does not, which is why §3.25 makes a synchronous HTTP call instead.
- `VehicleDeletedHandler.cs:17` — a one-liner with real consequences:
  ```csharp
  public Task HandleAsync(VehicleDeleted @event) => _dispatcher.SendAsync(new DeleteResource(@event.VehicleId));
  ```
  **It assumes `resourceId == vehicleId`.** That coupling is asserted nowhere else — not in a
  comment, not in a test, not in a contract. It works only because `vehicles-service` and whatever
  creates the corresponding resource agree to use the same GUID.

  The failure path is worse than the assumption. If no resource with that id exists,
  `DeleteResourceHandler` throws `ResourceNotFoundException` → `ExceptionToMessageMapper` looks at
  the incoming message, which is **`VehicleDeleted`, not `DeleteResource`** → the `message switch`
  at `ExceptionToMessageMapper.cs:39-48` matches no case → `null` → **no rejected event is
  published.** So deleting a vehicle that has no availability resource fails completely silently on
  the bus. This is the clearest example in the codebase of why the mapper's second discriminant is
  the incoming *message*, not the *dispatched* command, and why that distinction is a trap.

  Note also that `SendAsync` re-enters the command pipeline **in-process**, so the `DeleteResource`
  runs inside the `VehicleDeleted` consumption — inside the `OutboxEventHandlerDecorator`, keyed on
  the `VehicleDeleted` message id.

**Lifecycle.** Queues are declared at startup by `UseRabbitMq()` using the template in §3.22.
Messages are consumed, decorated (outbox/inbox), dispatched, and — on failure — mapped to a rejected
event (§3.16).

**Invariants & enforcement.** A subscribed type with no handler, or a handler with no subscription,
both compile. `SubscribeEvent<T>` without an `IEventHandler<T>` produces a queue that fills and is
never drained correctly — `Unverifiable — Missing Source Evidence` for Convey's exact behaviour.

**Extension procedure — consume a new external event.**
1. Declare the event class in `Application/Events/External/` with `[Message("<owning exchange>")]`
   and **property names matching the publisher's exactly**.
2. Add an `IEventHandler<T>` in `.../Handlers/` (found by `AddEventHandlers()`,
   `Application/Extensions.cs:14`).
3. Add `.SubscribeEvent<T>()` to `UseInfrastructure`.
4. If the handler can fail in a way the publisher should learn about, add
   `ExceptionToMessageMapper` cases keyed on **that event type**, not on any command it dispatches.

**Failure modes.** Silent property-name drift with the publisher. Silent rejection loss when a
handler dispatches a command (the `VehicleDeleted` case). Missing `[Message]` attribute → the
subscriber binds to this service's own exchange (`availability`) instead of the publisher's, so the
event is never received — silently.

---

### 3.22 Queue naming & message conventions

**Definition.** The naming rules that decide which physical queue and routing key each subscription
and publication uses. They are pure configuration, and they are the contract with every other
service.

**Representation & storage.** `Api/appsettings.json`, `rabbitMq` section:

| Key | Value | Effect |
| --- | --- | --- |
| `exchange.name` | `availability` | this service's own exchange |
| `exchange.type` | `topic` | matches the gateway's `type: topic` |
| `exchange.declare` / `durable` | `true` / `true` | survives broker restart |
| `exchange.autoDelete` | `false` | persists with no consumers |
| `queue.template` | `availability-service/{{exchange}}.{{message}}` | queue naming |
| `queue.declare` / `durable` | `true` / `true` | |
| `queue.autoDelete` / `exclusive` | `false` / `false` | shared across instances |
| `conventionsCasing` | `snakeCase` | **type name → routing key** |
| `context.enabled` / `context.header` | `true` / `message_context` | carries the correlation context between services (`§3.20`) |
| `spanContextHeader` | `span_context` | matches the gateway (`ntrada-async.yml:81`) |
| `retries` / `retryInterval` | `3` / `2` | broker **connection** retry, not message redelivery |
| `connectionName` | `availability-service` | identifies this connection in the RabbitMQ UI |

There is **no** `messageProcessor` key and **no** `requeueFailedMessages` key in any of the four
settings files — redelivery on handler failure is whatever Convey's default is, and it is
`Unverifiable — Missing Source Evidence` from this workspace.

**The two rules that matter:**

1. **`conventionsCasing: snakeCase` turns a C# type name into a routing key.** `AddResource` →
   `add_resource`; `ReserveResource` → `reserve_resource`; `ResourceReserved` →
   `resource_reserved`. This is the *entire* binding contract with the gateway and with every
   subscriber. **Renaming a class renames a routing key**, and nothing anywhere fails at build
   time.
2. **`queue.template: availability-service/{{exchange}}.{{message}}`** produces, for each
   subscription, a queue named after the consumer, the source exchange, and the message:

| Subscription | Source exchange | Queue name |
| --- | --- | --- |
| `AddResource` | `availability` | `availability-service/availability.add_resource` |
| `DeleteResource` | `availability` | `availability-service/availability.delete_resource` |
| `ReserveResource` | `availability` | `availability-service/availability.reserve_resource` |
| `ReleaseResourceReservation` | `availability` | `availability-service/availability.release_resource_reservation` |
| `CustomerCreated` | `customers` | `availability-service/customers.customer_created` |
| `VehicleDeleted` | `vehicles` | `availability-service/vehicles.vehicle_deleted` |

Prefixing with the **consumer** name is what allows several services to consume the same event
independently — each gets its own queue and its own copy. It also means renaming this service
orphans every existing queue (they persist, `autoDelete: false`, accumulating messages nobody
reads).

**Mismatch to be aware of:** the gateway publishes `release_resource`
(`ntrada-async.yml`, `routing_key: release_resource`) while the command type is
`ReleaseResourceReservation`, whose snake-case form is `release_resource_reservation`. Whether these
bind depends on Convey's exact routing-key derivation and on any `[Message]` attribute on the
command — **`Unverifiable — Missing Source Evidence`**, since the commands carry `[Contract]` but
no `[Message]`, and Convey's convention provider is not in this workspace. This is exactly the
class of drift the naming convention makes invisible, and it is recorded as ABQ Q-1: it should be
verified against a live broker before relying on the async release path.

**Lifecycle.** Exchanges and queues are declared at startup (`declare: true`) and outlive the
process.

**Invariants & enforcement.** None in code — every one of these is a string in a JSON file matched
against a string in another repository's JSON file. The only enforcement is a message that never
arrives.

**Extension procedure.** Adding a subscription automatically creates its queue from the template.
**Changing `queue.template` or `conventionsCasing` is a breaking change for every existing queue**
and requires draining the old ones first. Adding an environment override means editing
`appsettings.docker.json` and `appsettings.local.json` too — both redefine `rabbitMq`.

**Failure modes.** Routing-key drift after a class rename (silent). Queue orphaning after a service
rename (silent, plus unbounded growth). Exchange-name typo → messages published into a
newly-declared, unconsumed exchange (silent).

---

### 3.23 `[Contract]` and `UsePublicContracts`

**Definition.** A marker that publishes a type's JSON shape at a documentation endpoint, so other
teams can discover the wire format without reading this repository.

**Representation & storage.** `Application/ContractAttribute.cs` is **four lines** —
`public class ContractAttribute : Attribute { }`, with no properties, no `AttributeUsage`
restriction and no version field. It is applied to:

| Category | Types |
| --- | --- |
| Commands | `AddResource`, `DeleteResource`, `ReserveResource`, `ReleaseResourceReservation` |
| Integration events | `ResourceAdded`, `ResourceDeleted`, `ResourceReserved`, `ResourceReservationReleased`, `ResourceReservationCanceled` |
| Rejected events | `AddResourceRejected`, `DeleteResourceRejected`, `ReleaseResourceRejected`, `ReleaseResourceReservationRejected` |

Thirteen types. Activated by `.UsePublicContracts<ContractAttribute>()`
(`Infrastructure/Extensions.cs:102`), which is Convey middleware that serves the marked types'
schemas over HTTP.

**Notably not marked:** `GetResource`, `GetResources`, `ResourceDto`, `ReservationDto`,
`CustomerStateDto`, `CustomerCreated`, `VehicleDeleted`. So **the read contract and the inbound
external-event contracts are not published** — only the write contract is. A consumer discovering
this service through the contracts endpoint learns how to command it but not what it returns.

**Lifecycle.** Reflected over at startup; served for the process lifetime.

**Invariants & enforcement.** None. The attribute is inert metadata — omitting it on a new command
does not break anything, it just makes the command undiscoverable. There is **no versioning**: the
published shape is whatever the current build's class looks like, so a field rename changes the
published contract with no deprecation path and no notice to consumers.

The exact endpoint path and response shape are Convey's — **`Unverifiable — Missing Source
Evidence`**. What is verifiable here is which types are marked and that the middleware is wired.

**Extension procedure.** Add `[Contract]` to every new command, integration event and rejected
event. Consider marking the DTOs too if consumers need the read shape. For a genuine versioning
story, the attribute would need a version property and the middleware would need to honour it —
neither exists.

**Failure modes.** Silent contract drift (a renamed field changes the published schema with no
signal). Unmarked new types are invisible to consumers, who then reverse-engineer the shape from
observed messages.

---

### 3.24 Dispatcher-bound HTTP endpoints

**Definition.** The service's entire HTTP surface, declared as a mapping from route to CQRS type.
**There are no controllers, no attributes and no action methods in this repository.**

**Representation & storage.** `Api/Program.cs:38-46`:

```csharp
.UseEndpoints(endpoints => endpoints
    .Get("", ctx => ctx.Response.WriteAsync("Pacco Availability Service"))
    .Get("ping", ctx => ctx.Response.WriteAsync("pong"))
    .Get<GetResources, IEnumerable<ResourceDto>>("resources")
    .Get<GetResource, ResourceDto>("resources/{resourceId}")
    .Post<AddResource>("resources", afterDispatch: (cmd, ctx) => ctx.Response.Created($"resources/{cmd.ResourceId}"))
    .Post<ReserveResource>("resources/{resourceId}/reservations")
    .Delete<ReleaseResourceReservation>("resources/{resourceId}/reservations/{dateTime}")
    .Delete<DeleteResource>("resources/{resourceId}"))
```

(The exact call shape is per `Convey.WebApi.CQRS`'s `UseDispatcherEndpoints`; the routes, types and
the `afterDispatch` are this repository's.)

**Lifecycle.** Registered once at startup. Per request: the dispatcher binds the route/query/body
onto the command or query type, resolves the handler, invokes it through the decorator stack
(§3.17), and writes a conventional response.

**Invariants & enforcement.**
- **Response conventions are Convey's, and they are the only reason this service ever returns
  anything other than 400.** `Get<TQuery,TResult>` returns 200 with the result or **404** when the
  result is `null`; `Post<TCommand>` returns 202 unless an `afterDispatch` overrides it. The single
  `afterDispatch` here makes `POST /resources` return **201 with `Location: resources/{id}`** —
  verified by the E2E test, which asserts both the status and the header.
- Binding is by name: `resources/{resourceId}` fills `AddResource.ResourceId`,
  `GetResource.ResourceId`, etc. **A route-parameter name that does not match a property is
  silently ignored** and the property keeps its default. The `{dateTime}` segment on the release
  route binds `ReleaseResourceReservation.DateTime` — a `DateTime` parsed from a URL segment, with
  culture/format handling entirely inside Convey (`Unverifiable — Missing Source Evidence`).
- `.Get("", …)` and `.Get("ping", …)` are raw handlers writing plain text; `ping` is the liveness
  probe (and is in the gateway's `logger.excludePaths`).
- **Nothing here is authenticated or authorized.** No route declares a policy; the only
  authorization in the whole service is the one conditional inside `ReserveResourceHandler`.
- Only **four** of the seven CQRS endpoints mutate. There is no `PUT`, so no update-in-place
  operation exists for tags (§3.4).

**Extension procedure — add an endpoint.** Add the line to `UseEndpoints`, matching the CQRS type's
property names to the route template. Add the corresponding gateway route in all four manifests. To
control the response, pass `afterDispatch` — the `POST /resources` line is the template.

**Failure modes.** Silent parameter-name mismatch. A route registered for a command with no handler
fails at dispatch time. Route ordering: `resources/{resourceId}` and a hypothetical
`resources/search` would collide; today no such overlap exists.

---

### 3.25 `CustomersServiceClient` & PKI client certificates

**Definition.** The service's only synchronous outbound dependency: it asks `customers-service`
whether a customer is in good standing before allowing a reservation.

**Representation & storage.** `Infrastructure/Services/Clients/CustomersServiceClient.cs`, 40 lines:

```csharp
public CustomersServiceClient(IHttpClient httpClient, HttpClientOptions options,
    ICertificatesService certificatesService, VaultOptions vaultOptions, SecurityOptions securityOptions) {
    _httpClient = httpClient;
    _url = options.Services["customers"];                                   // :20
    if (!vaultOptions.Enabled || vaultOptions.Pki?.Enabled != true) { return; }   // :21-24
    var certificate = certificatesService.Get(vaultOptions.Pki.RoleName);   // :26
    if (certificate is null) { return; }                                    // :27-30
    var header = securityOptions.Certificate.GetHeaderName();
    _httpClient.SetHeaders(h => h.Add(header, certificate.GetRawCertDataString())); }

public Task<CustomerStateDto> GetStateAsync(Guid id)
    => _httpClient.GetAsync<CustomerStateDto>($"{_url}/customers/{id}/state");
```

**Lifecycle.** Registered `AddTransient` (`Infrastructure/Extensions.cs:60`), so the constructor —
**including the certificate fetch** — runs on every resolution, i.e. on every `ReserveResource`.
`certificatesService.Get` is presumably cached by Convey (`Unverifiable — Missing Source
Evidence`); if it is not, every reservation performs a Vault round trip.

The URL comes from `httpClient.services.customers`, and `httpClient.type: fabio`
(`Api/appsettings.json`) means the value is a **service name**, not a URL:

| Environment | `services.customers` | Resolution |
| --- | --- | --- |
| base (`appsettings.json`) | `customers-service` | via Fabio/Consul |
| `local` | `http://localhost:5002` | direct |
| `docker` | container name | direct/Fabio |
| **`tests`** | **`{}` — the whole `services` map is empty** | **`KeyNotFoundException`** |

**Invariants & enforcement — three silent behaviours in eleven lines.**

1. **`options.Services["customers"]` is an unguarded indexer.** Under `tests`, where the map is
   empty, constructing this client throws `KeyNotFoundException`. That exception is not a
   `DomainException` or `AppException`, so on HTTP it becomes the generic 400 (§3.18) and on AMQP
   it produces **no rejected event** (§3.16). A missing configuration key therefore presents as a
   malformed-request error. Using `TryGetValue` with a clear startup failure would be strictly
   better.
2. **The client certificate is skipped silently whenever Vault or PKI is off** (`:21-24`). That is
   the case in `local`, `docker` **and** `tests` — i.e. in every environment represented in this
   repository. So mTLS-style caller authentication between services is a production-only behaviour
   that no test and no local run exercises. If `customers-service` ever *requires* the certificate,
   this service works everywhere except production; if it does not, the certificate is decorative.
3. **A null certificate from Vault is also silently skipped** (`:27-30`) — the call proceeds
   unauthenticated rather than failing.

The header name comes from `security.certificate.header: "Certificate"` (`Api/appsettings.json`),
and the value is `GetRawCertDataString()` — the raw certificate, not a signature over the request.
It proves possession of a Vault-issued certificate, nothing about the request contents.

**Failure semantics of the call itself.** `GetAsync<T>` returning `null` (a 404, or a non-2xx that
Convey swallows) is interpreted by `ReserveResourceHandler.cs:42-45` as
`CustomerNotFoundException`. **So "customers-service is down" and "this customer does not exist" are
indistinguishable**, and both deny the reservation. There is no timeout, retry, circuit breaker or
fallback configured here — the reservation simply fails.

**Extension procedure — add another service client.** Copy this class, add the key to
`httpClient.services` in **all four** appsettings files (the `tests` omission above is exactly the
bug this causes), inject `IHttpClient`, and register it in `AddInfrastructure`. Fix the indexer to
`TryGetValue` with an explicit startup exception while you are there.

**Failure modes.** All three silent paths above, plus: an outage of `customers-service` blocks
every reservation platform-wide, because this is a synchronous dependency on the write path. The
`CustomerCreated` subscription (§3.21) exists precisely to make a local projection possible and
remove this coupling — but it is a no-op, so the coupling stands.

---

### 3.26 Customer state validation

**Definition.** The rule deciding whether a customer may reserve.

**Representation & storage.** The whole rule is one expression —
`Application/DTO/CustomerStateDto.cs:8`:

```csharp
public string State { get; set; }
public bool IsValid => State.Equals("valid", StringComparison.InvariantCultureIgnoreCase);
```

Consumed at `ReserveResourceHandler.cs:47-50`:

```csharp
if (!customerState.IsValid) { throw new InvalidCustomerStateException(command.ResourceId, customerState?.State); }
```

**Invariants & enforcement.**
- The **only** valid state is the literal string `"valid"`, case-insensitively. Every other value —
  `"incomplete"`, `"suspended"`, `"banned"`, `""` — denies the reservation identically. This service
  has no knowledge of the customer state machine that `customers-service` owns; it knows one word.
- **`State` is not null-guarded.** If `customers-service` returns a JSON body that deserializes to a
  `CustomerStateDto` with a null `State` — which happens for `{}` or `{"state": null}` — then
  `State.Equals(...)` throws a `NullReferenceException`. That is not a `DomainException` or
  `AppException`, so it becomes a generic **400** on HTTP and **no rejected event** on AMQP. The
  handler's own defensive style makes this ironic: line 42 checks `customerState is null` and line
  49 uses `customerState?.State` with a null-conditional — but line 8 of the DTO dereferences
  `State` unconditionally, one call earlier. The guard is on the wrong object.
  The fix is one character class: `State?.Equals("valid", …) == true`, or
  `string.Equals(State, "valid", StringComparison.InvariantCultureIgnoreCase)`.
- `InvariantCultureIgnoreCase` is the right choice here (an ordinal-ignore-case would also do);
  it avoids the Turkish-I class of bug on a literal ASCII comparison.

**Lifecycle.** Evaluated once per `ReserveResource`, on the synchronous outbound call's result.
The value is never cached — every reservation attempt hits `customers-service`.

**Extension procedure — support more states.** Replace the boolean with a set membership check, or
move the decision to `customers-service` and have it return a boolean. Either way, add the null
guard first: it is a one-line fix for a real crash path, and `tests/…Tests.Unit` has no case
covering it.

**Failure modes.** NRE on a null state (above). A rename of the `"valid"` state in
`customers-service` silently blocks **all** reservations platform-wide, with the error surfacing
here as `invalid_customer_state` — pointing the investigator at the customer, not at the rename.

---

### 3.27 `MessageBroker` — the span-context-aware publisher

**Definition.** The single seam through which every integration event leaves this service. It is
the concrete implementation of the Application-layer port `IMessageBroker`
(`src/…Application/Services/IMessageBroker.cs`), and it is the *only* place in the component that
knows Convey's `IBusPublisher` and `IMessageOutbox` exist.

**Representation.** `src/…Infrastructure/Services/MessageBroker.cs`, an `internal sealed class`
registered as `Transient` in `Extensions.cs:59`. Its constructor
(`MessageBroker.cs:28-43`) takes six collaborators — `IBusPublisher`, `IMessageOutbox`,
`ICorrelationContextAccessor`, `IHttpContextAccessor`, `IMessagePropertiesAccessor`, `ITracer` —
plus `RabbitMqOptions` and a logger. Only one constructor line has behaviour:

```csharp
_spanContextHeader = string.IsNullOrWhiteSpace(options.SpanContextHeader)
    ? DefaultSpanContextHeader   // "span_context", MessageBroker.cs:18
    : options.SpanContextHeader;
```

`rabbitMq.spanContextHeader` is set to `"span_context"` in `Api/appsettings.json:151`, so the
default and the configured value coincide — the fallback exists but is never exercised in any
shipped environment.

**Lifecycle — what `PublishAsync` actually does** (`MessageBroker.cs:47-86`):

1. **Null-batch guard** (`:49-52`). `events is null` → `return`. Silent.
2. **Ambient-context capture** (`:54-65`), done **once for the whole batch**, before the loop:
   - `originatedMessageId` = the inbound AMQP `MessageId`, or `null` on the HTTP path
     (`_messagePropertiesAccessor.MessageProperties?.MessageId`).
   - `correlationId` = the inbound AMQP `CorrelationId`, or `null` on the HTTP path.
   - `spanContext` = the inbound `span_context` header; **if blank**, it falls back to
     `_tracer.ActiveSpan?.Context.ToString()` (`:58-61`). This is the join between the two
     tracing worlds: an HTTP request has no AMQP header but does have an active Jaeger span, so
     the trace survives the HTTP→AMQP hop. If Jaeger is disabled (`jaeger.enabled: false` in
     `appsettings.local.json` / `appsettings.tests.json`) `ActiveSpan` is null and the published
     span context is `string.Empty` — the trace simply stops here rather than erroring.
   - `headers` = `messageProperties.GetHeadersToForward()` — the extension at
     `Infrastructure/Extensions.cs:122-136`, which forwards **exactly one** header, `Saga`.
   - `correlationContext` = `_contextAccessor.CorrelationContext` (AMQP path) **or**
     `_httpContextAccessor.GetCorrelationContext()` (HTTP path, `Extensions.cs:117-120`). The
     null-coalesce is what makes one publisher serve both transports.
3. **Per-event loop** (`:67-85`):
   - `@event is null` → `continue`. Silent skip; a null in the middle of a batch does not abort
     the rest.
   - `var messageId = Guid.NewGuid().ToString("N")` (`:74`) — **a fresh identifier per event,
     unconditionally**. See the failure modes below; this single line is why outbox de-duplication
     cannot protect the HTTP path.
   - `if (_outbox.Enabled)` → `_outbox.SendAsync(@event, originatedMessageId, messageId,
     correlationId, spanContext, correlationContext, headers)` then `continue` (`:76-81`).
   - Otherwise `_busPublisher.PublishAsync(…)` with the same seven arguments (`:83-84`).

The two branches take identical arguments, so switching `outbox.enabled` changes *when* the
message hits RabbitMQ, never *what* is on the wire.

**Invariants & enforcement.**

| Invariant | Enforcement | Loud / silent |
|---|---|---|
| A null batch publishes nothing | `:49-52` early return | **Silent** |
| A null event inside a batch is skipped | `:69-72` `continue` | **Silent** |
| Every published event carries a unique `MessageId` | `:74` fresh GUID | Enforced, but see below |
| Ambient context is captured once per batch | `:54-65` outside the loop | Structural |
| Outbox and direct publish are behaviourally interchangeable | identical argument lists `:78` / `:83` | Convention only — no test asserts it |

**Extension procedure — add a forwarded header.** Edit `GetHeadersToForward` in
`Infrastructure/Extensions.cs:122-136`. It currently builds a dictionary containing only
`Saga` when the inbound properties carry it. Adding a header is a two-line change there; nothing
else in the component needs to know. Note the asymmetry: headers are forwarded **out** from an
inbound AMQP message, but there is no equivalent path lifting an HTTP header into the outbound
message — an HTTP-originated publish always has an empty forwarded-header set.

**Extension procedure — publish from a new place.** Do not inject `IBusPublisher`. Inject
`IMessageBroker` (the Application port) and call `PublishAsync`. Every current caller reaches it
through `EventProcessor` (`§3.15`); the only direct callers of `IMessageBroker` outside
`EventProcessor` are the outbox decorators (`§3.17`), which publish rejected events.

**Failure modes.**

- **Fresh `MessageId` per publish defeats inbox de-duplication.** Because `:74` mints a new GUID
  every time, a retried HTTP `POST /resources` produces a *different* `MessageId` for the
  resulting `ResourceAdded` than the first attempt did. Downstream consumers using Convey's inbox
  will treat the two as distinct messages. De-duplication only works on the AMQP path, and only
  because the *inbound* `MessageId` is stable — which is a property of the sender, not of this
  class. Cross-reference `§3.17` and ABQ Q-5.
- **Batch-level context, event-level publish.** All events in one batch share `spanContext`,
  `correlationId` and `correlationContext`. For `Resource.Delete()`, which raises N
  `ReservationCanceled` plus one `ResourceDeleted` (`Core/Entities/Resource.cs:92-100`), that is
  correct and desirable. It would be wrong only if a batch ever mixed causally unrelated events —
  no current code path does.
- **`_tracer.ActiveSpan` is read at publish time, not at handle time.** The decorator stack
  (`§3.17`) puts the Jaeger span *outside* the handler, so `ActiveSpan` is the
  `handling-{command}` span during publish. If the decorator order in `Extensions.cs:67-68` were
  changed so the outbox decorator ran outside Jaeger's, the published span context would silently
  become the parent span or empty — a tracing regression with no test coverage.
- **`_logger.LogTrace` at `:75`** is the only publish-time observability. With
  `logger.level: "information"` (`appsettings.json:35`) it is off in every environment except
  `local`, whose `appsettings.local.json` sets `"level": "verbose"`.

---

### 3.28 `MessageToLogTemplateMapper` — per-message log phrasing

**Definition.** A lookup table that gives Convey's logging decorator a human-readable sentence to
emit before and/or after each handled message, with the handler's own properties interpolated.

**Representation.** `src/…Infrastructure/Logging/MessageToLogTemplateMapper.cs`, an
`internal sealed class` implementing Convey's `IMessageToLogTemplateMapper`. It is wired by
`.AddCommandHandlersLogging()` and `.AddEventHandlersLogging()` in
`Infrastructure/Extensions.cs` (the fluent chain at `:74-93`). The table itself is a
**property, not a field** (`:12-13`: `private static IReadOnlyDictionary<…> MessageTemplates =>`),
so a new `Dictionary` is allocated on **every** `Map` call. Harmless at this scale; worth knowing
before someone grows the table.

**The five mapped messages** (`:15-31`):

| Message | `Before` | `After` | `OnError` |
|---|---|---|---|
| `AddResource` | — | `Added a resource with id: {ResourceId}. ` | `ResourceAlreadyExistsException` → `Resource with id: {ResourceId} already exists.` |
| `DeleteResource` | — | `Deleted a resource with id: {ResourceId}.` | — |
| `ReleaseResourceReservation` | — | `Released a resource with id: {ResourceId}.` | — |
| `ReserveResource` | — | `Reserved a resource with id: {ResourceId} priority: {Priority}, date: {DateTime}.` | — |
| `VehicleDeleted` | `Vehicle with id: {VehicleId} has been deleted.` | — | — |

**Lifecycle.** `Map<TMessage>` (`:33-37`) keys on `message.GetType()` — **exact runtime type, not
assignability**. A hit returns the template; a miss returns `null`, and Convey's decorator treats
`null` as "log nothing extra".

**Invariants & enforcement.**

- The placeholder names must match **public property names on the message type**, because Serilog
  destructures the message object against the template. `{ResourceId}`, `{Priority}`,
  `{DateTime}` and `{VehicleId}` all exist on their respective commands/events
  (`Application/Commands/*.cs`, `Application/Events/External/VehicleDeleted.cs`). Nothing enforces
  this — a typo produces a log line with a literal `{Typo}` in it. **Silent.**
- `VehicleDeleted` is the only entry using `Before`, and it is the only *inbound external* event
  in the table. That is deliberate: `§3.21` documents that `VehicleDeletedHandler` dispatches a
  `DeleteResource` whose own `After` template then fires, so a successful vehicle deletion emits
  two log lines and a failed one emits only the `Before`.

**Extension procedure.** Add a dictionary entry keyed on the message type. There is no
registration step — the mapper is a single class and Convey resolves it once. Three rules:
(1) use the *exact* type, not a base type; (2) placeholders must be real property names;
(3) if the message can fail in a way worth naming, add an `OnError` entry — that is the only way
to turn a stack trace into a sentence.

**Failure modes.**

- **Unmapped messages log nothing descriptive.** All four commands are mapped, but
  `CustomerCreated` and both queries (`GetResource`, `GetResources`) fall through `:36`'s
  `TryGetValue` to `null`. Operators reading Seq will see generic Convey handler logs for those and
  rich sentences for the five above — an inconsistency that reads as "the logs are broken" rather
  than "the table is incomplete."
- **The `ReleaseResourceReservation` template says the wrong noun.** `"Released a resource with
  id: {ResourceId}."` (`:27`) describes releasing a **resource**; the command releases a
  **reservation** on one. This is the same resource/reservation naming slippage that produces
  `ReleaseResourceRejected` for a `ReleaseResourceReservation` failure (`§3.16`) — so an operator
  who greps the logs for the release path finds a sentence that matches the *rejected event's*
  vocabulary rather than the command's. ABQ Q-1.
- **Redaction interacts here.** `logger.excludeProperties` (`Api/appsettings.json:37-50`) strips
  `Password`, `Token`, `Email`, `ConnectionString`, etc. from log events. None of the four
  placeholders above is on that list, so all five templates render fully. Adding a template that
  interpolates a redacted property name would silently render it as the redaction placeholder.

---

### 3.29 `CustomMetricsMiddleware` and `MetricsJob` — the two metric producers

**Definition.** Two independent, hand-written metric sources layered on top of Convey's
`AddMetrics()` / `UseMetrics()` (App.Metrics + a Prometheus scrape endpoint). One counts a fixed
pair of HTTP request shapes; the other samples process health on a timer.

#### 3.29.1 `CustomMetricsMiddleware`

**Representation.** `src/…Infrastructure/Metrics/CustomMetricsMiddleware.cs`, registered as a
**singleton** (`Extensions.cs:66`) and inserted into the pipeline by
`.UseMiddleware<CustomMetricsMiddleware>()` (`Extensions.cs:104`), immediately after
`.UseMetrics()`. It implements `IMiddleware` (the factory-activated form), which is why the
singleton registration is required — `UseMiddleware<T>` for an `IMiddleware` resolves `T` from the
container per request.

**The lookup table** (`:18-22`) has exactly **two** entries:

| Key (`"{METHOD}:{path}"`) | Counter emitted |
|---|---|
| `GET:/resources` | `queries` counter, tag `query=GetResources` |
| `POST:/resources` | `commands` counter, tag `command=AddResource` |

**Lifecycle** (`InvokeAsync`, `:30-48`):
1. `!_enabled` (from `MetricsOptions.Enabled`, i.e. `metrics.enabled`) → `next(context)` with no
   measurement.
2. `GetKey(request.Method, request.Path.ToString())` (`:50-51`) → **exact string match** against
   the dictionary. A miss → `next(context)`.
3. On a hit: create a DI scope, resolve `IMetricsRoot`, increment the counter, then `next`.

**Invariants & enforcement.**

- The counter fires **before** the request is handled and **regardless of outcome**. A `POST`
  that ends in a 400 from `ExceptionToResponseMapper` (`§3.18`) still increments
  `commands{command=AddResource}`. The counter measures *arrivals*, not *successes* — a
  distinction nothing in the code or the metric name records. **Silent.**
- Matching is exact and case-sensitive on the path. `GET /resources/` (trailing slash) or
  `GET /Resources` does not match. ASP.NET Core does not normalise either for you here.
- The counter names (`"commands"`, `"queries"`) and tag values (`typeof(GetResources).Name`,
  `typeof(AddResource).Name`) are derived from the **type name at compile time** — renaming the
  `AddResource` class silently renames the Prometheus tag value, breaking any dashboard query
  pinned to it. **Silent.**

**Extension procedure.** Add a `[GetKey("VERB", "/path")] = Command(typeof(X).Name)` row. Two
traps: (1) parameterised routes cannot be expressed — `GET:/resources/{resourceId}` would need a
literal path, so the four remaining endpoints in `Api/Program.cs:38-46` are simply
uncountable by this mechanism; (2) if you want per-outcome counters you must move the increment
after `await next(context)` and switch `InvokeAsync` to `async`.

**Failure modes.**

- **Five of seven endpoints are invisible.** `GET /`, `GET /resources/{resourceId}`,
  `POST /resources/{resourceId}/reservations/{dateTime}`,
  `DELETE /resources/{resourceId}/reservations/{dateTime}` and `DELETE /resources/{resourceId}`
  produce no custom counter. An operator reading the `commands` counter would conclude
  `AddResource` is the only command the service receives.
- **AMQP-delivered commands are never counted.** The middleware is HTTP-only. `AddResource`
  arriving over RabbitMQ (the path exercised by `tests/…Tests.Integration/Async/AddResourceTests.cs`)
  bypasses it entirely, so the same logical command is counted on one transport and not the other.
- **`metrics.enabled: false`** in `appsettings.local.json` and `appsettings.tests.json` means
  neither counter nor gauge exists in local development or in the test environment — metric
  regressions cannot be caught before `docker`.

#### 3.29.2 `MetricsJob`

**Representation.** `src/…Infrastructure/Metrics/MetricsJob.cs`, a `BackgroundService`
registered via `AddHostedService<MetricsJob>()` (`Extensions.cs:65`). It publishes two gauges,
`threads` and `working_set` (`:15-23`).

**Lifecycle** (`ExecuteAsync`, `:36-57`):
- `!_options.Enabled` → logs `"Metrics are disabled."` at Information and **returns**, ending the
  hosted service permanently for the process lifetime. Loud, once.
- Otherwise loop until cancellation: new DI scope → `IMetricsRoot` →
  `Gauge.SetValue(_threads, Process.GetCurrentProcess().Threads.Count)` and
  `Gauge.SetValue(_workingSet, process.WorkingSet64)` → `await Task.Delay(5000, stoppingToken)`.

**Invariants & enforcement.**

- **The sampling interval is hardcoded.** `Task.Delay(5000, stoppingToken)` at `:55` ignores
  `metrics.interval` (`Api/appsettings.json:95`, value `5`). The two currently agree by
  coincidence — `interval: 5` seconds and 5000 ms — which is exactly why the bug is invisible.
  Changing `metrics.interval` to `30` changes nothing. **Silent.** See ABQ Q-6.
- `Process.GetCurrentProcess()` is allocated inside the loop and never disposed (`:50`). `Process`
  is `IDisposable`; on Linux this leaks a small handle per tick. At one tick per five seconds it
  is a slow leak, not an outage, but it is a real defect.
- A new DI scope per tick (`:46`) is correct — `IMetricsRoot` is resolved from a scope, and the
  `using` block disposes it.

**Extension procedure — honour the configured interval.** Replace `5000` with
`(int)TimeSpan.FromSeconds(_options.Interval).TotalMilliseconds`, guarding against `0`
(a zero interval would spin the loop). `_options` is already injected (`:33`), so this is a
one-line change with no registration impact.

**Extension procedure — add a gauge.** Add a `GaugeOptions` field and one `SetValue` call inside
the existing scope. Do **not** add a second `BackgroundService` for it; the scope and the
`IMetricsRoot` resolution are the expensive parts, and they are already paid for here.

**Failure modes.** If `IMetricsRoot` is not registered — which happens when `AddMetrics()` is
removed from the chain at `Extensions.cs:86` but `metrics.enabled` stays `true` —
`GetRequiredService<IMetricsRoot>()` throws inside a `BackgroundService`. On .NET Core 3.1 an
unhandled exception in `ExecuteAsync` does **not** stop the host; the task faults silently and
gauges simply stop updating with no log line. **Silent**, and the most likely way for this job to
die unnoticed.

---

### 3.30 Consul registration & Fabio addressing

**Definition.** How the service makes itself findable, and how it finds `customers-service`
without hardcoding a host. Two Convey extensions, `.AddConsul()` and `.AddFabio()`
(`Infrastructure/Extensions.cs:79-80`), driven entirely by configuration; there is no
registration code in this repository. `[convey]`

**Representation & storage — the inbound half (Consul).** `Api/appsettings.json:7-17`:

| Key | `appsettings.json` | `appsettings.docker.json` | `local` / `tests` |
|---|---|---|---|
| `consul.enabled` | `true` | `true` | `false` |
| `consul.url` | `http://localhost:8500` | `http://consul:8500` | inherited |
| `consul.service` | `availability-service` | `availability-service` | inherited |
| `consul.address` | `docker.for.win.localhost` | `availability-service` | `localhost` (tests) |
| `consul.port` | `5001` | `80` | `5001` (tests) |
| `consul.pingEnabled` | `true` | `true` | inherited |
| `consul.pingEndpoint` | `ping` | `ping` | `""` (local) |
| `consul.pingInterval` | `3` | `3` | inherited |
| `consul.removeAfterInterval` | `3` | `3` | inherited |

On startup Convey registers `{consul.service}` at `{consul.address}:{consul.port}` with a health
check that polls `/{consul.pingEndpoint}` every `pingInterval` seconds and deregisters the
instance `removeAfterInterval` **minutes** after the check starts failing. The `ping` endpoint
itself is supplied by `.UseConvey()` (`Extensions.cs:101`), not by anything in this repo — which is
why it does not appear in the `UseDispatcherEndpoints` block in `Api/Program.cs:38-46` but is
nevertheless excluded from logging and tracing (`logger.excludePaths`,
`jaeger.excludePaths`, both `["/", "/ping", "/metrics"]`).

**Representation & storage — the outbound half (Fabio).** `httpClient.type: "fabio"`
(`appsettings.json:24`) tells Convey's HTTP client to rewrite a *logical* service name into a
Fabio-routed URL against `fabio.url`. The service map is a single entry:

```json
"httpClient": { "services": { "customers": "customers-service" } }   // appsettings.json:26-28
```

`CustomersServiceClient` reads it as `options.Services["customers"]`
(`Infrastructure/Services/Clients/CustomersServiceClient.cs:20`) and issues
`GET {_url}/customers/{customerId}/state`. With `type: "fabio"` the literal
`customers-service` is resolved by Fabio via Consul; with `appsettings.local.json`'s
`"type": ""` and `"customers": "http://localhost:5002"` the same code path issues a direct
absolute-URL call. **The client code is identical in both cases** — the indirection lives wholly
in configuration.

**Lifecycle.** Registration happens once at host start; deregistration is Consul's responsibility
via the failing health check, not an explicit shutdown call. `appsettings.docker.json` overrides
`address` to the compose service name and `port` to `80`, matching the container's
`ASPNETCORE_URLS http://*:80` (`Dockerfile:9`).

**Invariants & enforcement.**

- `consul.address`/`consul.port` must describe how **other containers** reach this one, not how
  the process binds. Nothing validates the pair. A wrong value registers a healthy-looking service
  at an unreachable address; the failure surfaces in *callers* as connection refused. **Silent
  here, loud there.**
- `appsettings.json`'s `address: "docker.for.win.localhost"` is a Windows-Docker-specific host
  name. On Linux or macOS the base file's Consul registration points at a name that does not
  resolve. It is masked in practice because `docker` overrides it and `local`/`tests` disable
  Consul — but anyone running the base profile on Linux hits it. ABQ Q-7.
- `appsettings.local.json` sets `pingEndpoint: ""` while leaving `pingEnabled: true`. With Consul
  disabled this is inert; if someone enables `consul` locally without fixing the endpoint the
  health check targets the root path.
- `httpClient.services` is keyed by an arbitrary short name (`"customers"`). The key is a
  **string literal in C#** (`CustomersServiceClient.cs:20`) with no constant and no guard — see
  `§3.25` for the `KeyNotFoundException` this produces under `appsettings.tests.json`, whose
  `services` is `{}`.

**Extension procedure — call a new downstream service.** (1) Add a `httpClient.services` entry in
`appsettings.json` **and every environment override that redefines the `services` object** —
`docker`, `local` and `tests` each redeclare it wholesale, so a key added only to the base file is
absent in all three. (2) Define an Application-layer port under `Application/Services/Clients/`.
(3) Implement it in `Infrastructure/Services/Clients/` taking `IHttpClient` and `HttpClientOptions`.
(4) Register it in `Extensions.cs` alongside `ICustomersServiceClient` (`:60`).

**Failure modes.**

- **Fabio is a single point of failure for all synchronous calls.** `httpClient.retries: 3`
  retries the request, but every retry goes back through the same Fabio instance
  (`fabio.url: http://fabio:9999`). There is no circuit breaker in the configuration.
- **A stale Consul entry keeps traffic flowing to a dead instance** for up to
  `pingInterval` × failure threshold before deregistration. The synchronous reservation path
  (`§3.25`) turns that into `null` from `GetStateAsync` and then a
  `CustomerNotFoundException` — i.e. an infrastructure outage is reported as a business error.
- **`requestMasking`** (`appsettings.json:29-32`, `enabled: true`, `maskTemplate: "*****"`) is set
  in the base file only; `docker`, `local` and `tests` redeclare `httpClient` without it, so
  outbound request logging is unmasked in every environment that actually runs.

---

### 3.31 Vault — KV settings, PKI certificates, and dynamic Mongo credentials

**Definition.** Three distinct Vault integrations, all enabled by a single `.UseVault()` call in
`Api/Program.cs:48` and configured under one `vault` block
(`Api/appsettings.json:171-200`). `.UseVault()` runs as a **configuration provider**, so
everything it fetches is merged into `IConfiguration` *before* the options objects the rest of
this document describes are bound. `[convey]`

**Representation & storage.**

| Sub-feature | Config | What it does |
|---|---|---|
| Authentication | `vault.url: http://localhost:8200`, `authType: "token"`, `token: "secret"` | How the process authenticates to Vault |
| KV secrets | `kv.enabled: true`, `engineVersion: 2`, `mountPoint: "kv"`, `path: "availability-service/settings"` | Overlays a KV-v2 secret onto `IConfiguration` at startup |
| PKI | `pki.enabled: true`, `roleName: "availability-service"`, `commonName: "availability-service.pacco.io"` | Issues a short-lived X.509 certificate used for mTLS |
| Dynamic Mongo credentials | `lease.mongo.type: "database"`, `roleName: "availability-service"`, `enabled: true`, `autoRenewal: true`, `templates.connectionString: "mongodb://{{username}}:{{password}}@localhost:27017"` | Vault mints a Mongo user; the template becomes `mongo.connectionString` |

**Lifecycle.**

1. **Startup.** `.UseVault()` authenticates, reads `kv/availability-service/settings`, and merges
   it into configuration. Because it is a provider, a KV key named `mongo:connectionString` (or
   `Mongo__ConnectionString`) **overrides** the value in `appsettings.json:98`.
2. **Lease acquisition.** With `lease.mongo.enabled: true`, Vault issues a username/password pair
   against the `database` secrets engine, substitutes them into
   `templates.connectionString`, and writes the result into configuration — which is what
   `.AddMongo()` (`Extensions.cs:84`) then binds.
3. **Renewal.** `autoRenewal: true` starts a background renewer. When the lease expires and cannot
   be renewed, the credentials become invalid *while the process is running*.
4. **PKI.** The issued certificate is what `.AddCertificateAuthentication()` /
   `.UseCertificateAuthentication()` (`Extensions.cs:92`, `:105`) and the outbound client-certificate
   path in `CustomersServiceClient` (`§3.25`) are built around.

**Invariants & enforcement.**

- **The dynamic connection-string template hardcodes `localhost:27017`**
  (`appsettings.json:196`). Only the credentials are dynamic; the host is not. In any environment
  where Mongo is not on localhost, enabling `lease.mongo` produces a connection string that
  cannot connect — and it *overwrites* the correct `mongo.connectionString` from
  `appsettings.docker.json:…` (`mongodb://mongo:27017`). This is why `appsettings.docker.json`
  disables Vault wholesale (`"vault": { "enabled": false, … }`). ABQ Q-8.
- **Vault is disabled in every environment file in this repository.** `local`, `docker` and
  `tests` all set `vault.enabled: false` (and redundantly disable `kv`, `pki` and `lease.mongo`).
  Only the base `appsettings.json` — which is never the effective profile, since
  `Dockerfile:10` sets `ASPNETCORE_ENVIRONMENT docker` and `scripts/start.sh:2` sets `local` —
  has it on. **Consequence: no code path in this repository that depends on Vault is exercised by
  anything runnable from this repository.**
- `vault.token: "secret"`, `vault.password: "secret"` and `logger.seq.apiKey: "secret"` are
  placeholder literals in a committed file. They are not live credentials, but they are the exact
  shape of a real one and there is no mechanism preventing a real value being committed in the
  same slot. The `logger.excludeProperties` list (`appsettings.json:37-50`) redacts `Token`,
  `Secret`, `Password`, `ApiKey` from **log output** — it does nothing for configuration files.
- A Vault outage at startup fails the host: the configuration provider cannot supply
  `mongo.connectionString`. This is the correct behaviour (fail fast, loudly) and is the one place
  in the component where a dependency outage is not silently degraded.

**Extension procedure — move a secret into Vault.** Write it at
`kv/availability-service/settings` using the configuration key path with `:` or `__` separators
(e.g. `rabbitMq:password`). Remove the plaintext from `appsettings.json`. Do **not** add a new
config section for it — the whole point of the KV provider is that consuming code sees no
difference.

**Extension procedure — make the Mongo lease usable.** Replace the hardcoded `localhost:27017`
in `templates.connectionString` with the environment's host, per environment file, and re-enable
`vault.lease.mongo` there. Until that is done, `autoRenewal` is renewing credentials for a
connection string nobody can use.

**Failure modes.**

- **Lease expiry mid-flight.** If renewal fails, existing Mongo operations begin returning
  authentication errors. `ResourcesMongoRepository` does not catch them
  (`§3.11`), so they surface as unhandled exceptions → `ExceptionToResponseMapper`'s
  generic branch → a `400 Bad Request` with `{"code":"error"}` on the HTTP path (`§3.18`), and as
  a nacked message on the AMQP path. **A database credential outage is reported to the client as a
  bad request.** This is the single most misleading failure mapping in the component.
- **KV overlay is invisible.** A key set in Vault silently wins over the same key in
  `appsettings.json`. There is no log line naming the winner, so "the config file says X but the
  service behaves like Y" is undiagnosable from this repository alone.
- **PKI certificate rotation** replaces the certificate under a running process. Anything caching
  the old certificate — including the outbound HTTP client if it pins one — keeps presenting an
  expired identity. `Unverifiable — Missing Source Evidence`: Convey's rotation behaviour is not
  in this workspace.

---

### 3.32 Environment layering — `appsettings.{local,docker,tests,development}.json`

**Definition.** The rule that decides which values the component actually runs with. There is no
custom configuration code: `WebHost.CreateDefaultBuilder(args)` (`Api/Program.cs:29`) applies the
standard ASP.NET Core order — `appsettings.json`, then `appsettings.{ASPNETCORE_ENVIRONMENT}.json`,
then environment variables, then command-line args — and `.UseVault()` (`:48`) layers Vault on top
(`§3.31`).

**Representation & storage.** Five files in `src/Pacco.Services.Availability.Api/`:

| File | Selected by | Size / shape |
|---|---|---|
| `appsettings.json` | always (base) | 200 lines, every section |
| `appsettings.local.json` | `scripts/start.sh:2` (`export ASPNETCORE_ENVIRONMENT=local`); both `Properties/launchSettings.json` profiles | ~50 lines, **disable switches only** |
| `appsettings.docker.json` | `Dockerfile:10` (`ENV ASPNETCORE_ENVIRONMENT docker`) | ~90 lines, host rewrites |
| `appsettings.tests.json` | `PaccoApplicationFactory.CreateWebHostBuilder` → `.UseEnvironment("tests")` (`tests/…Tests.Shared/Factories/PaccoApplicationFactory.cs:9`) | ~110 lines, a near-full copy of the base |
| `appsettings.development.json` | the ASP.NET Core default when `ASPNETCORE_ENVIRONMENT` is unset in a dev host | **`{}`** — empty |

**Lifecycle — what each profile actually turns off.**

| Concern | base | `local` | `docker` | `tests` |
|---|---|---|---|---|
| Consul / Fabio | on | **off** | on (`consul:8500`, `fabio:9999`) | **off** |
| Jaeger | on | **off** | on (`jaeger:6831`) | **off** |
| Metrics | on | **off** | on (`env: "docker"`) | **off** (`prometheusEnabled` stays `true`) |
| Outbox | **on** | **off** | *not overridden → on* | **off** |
| Vault (all three) | on | **off** | **off** | **off** |
| Mongo database | `availability-service` | inherited | `availability-service` | **`resource-test-db`** |
| RabbitMQ host | `localhost` | inherited | `rabbitmq` | `localhost` |
| Seq | `localhost:5341` | **off** | `seq:5341` | `localhost:5341` |
| Log level | `information` | **`verbose`** | inherited | inherited |
| `httpClient.services.customers` | `customers-service` | `http://localhost:5002` | `customers-service` | **`{}` (empty)** |

**Invariants & enforcement.**

- **JSON configuration merges per *leaf key*, not per object** — except that a redeclared object
  replaces only the keys it names. This is the source of three real discrepancies visible in the
  table: `appsettings.docker.json` never mentions `outbox`, so **the outbox is enabled in Docker**
  with the base file's `disableTransactions: true` (see `§3.17`); `appsettings.tests.json` and
  `appsettings.docker.json` both redeclare `httpClient` without `requestMasking`, disabling
  outbound-log masking; and `appsettings.tests.json` redeclares `httpClient.services` as `{}`,
  which is what makes `CustomersServiceClient`'s unguarded indexer throw (`§3.25`).
- **`appsettings.tests.json` is a near-complete duplicate of the base file, not an overlay.** It
  restates the whole `rabbitMq` block verbatim (`:…` — 40 lines identical to
  `appsettings.json:111-152`). Any change to the base RabbitMQ conventions must be made twice or
  the test environment silently diverges. Nothing detects the divergence. This is the single
  highest-maintenance-cost fact about the configuration.
- **`appsettings.development.json` is empty**, so an unset `ASPNETCORE_ENVIRONMENT` yields the base
  file unmodified — meaning Consul, Fabio, Jaeger, metrics, the outbox **and Vault** all switch on
  and point at `localhost`. That is the least-safe profile and it is the accidental default.
  ABQ Q-9.
- `appsettings.tests.json` uses 4-space indentation while every other file uses 2. Cosmetic, but a
  reliable signal that it was authored separately rather than derived.

**Extension procedure — add a configuration key.** (1) Add it to `appsettings.json` with a safe
default. (2) Check each of `local`, `docker`, `tests` for a **redeclaration of the parent object**;
if the parent is redeclared there, the new key must be added there too or it will be missing.
(3) If a test binds the section through `OptionsHelper.GetOptions<T>("section")`
(`tests/…Tests.Shared/Helpers/OptionsHelper.cs:7-17`), remember that helper reads
`appsettings.tests.json` **from disk directly**, not through the host — so it sees only that file,
with no base-file merge at all.

**Failure modes.**

- **`OptionsHelper` does not layer.** It builds a fresh `ConfigurationBuilder` over
  `appsettings.tests.json` alone (`:20-23`), with `optional: true`. If the file is missing or the
  section is absent, it returns a **default-constructed options object** — `MongoDbFixture` would
  then call `new MongoClient(null)` and fail with an opaque driver error rather than a
  configuration error. **Silent until it isn't.**
- **`prometheusEnabled: true` with `enabled: false`** in `tests` and (partially) `local` is a
  contradictory pair. `MetricsJob` short-circuits on `Enabled` (`Metrics/MetricsJob.cs:38-42`), so
  the Prometheus flag has no effect — but it reads as though metrics work.
- **Environment variables override everything below Vault.** Convey's Mongo/RabbitMQ options can
  be set via `Mongo__ConnectionString` etc. with no trace in any file, so a deployment can differ
  from every profile documented here.

---

### 3.33 Mongo collection naming and the `"resources"` literal

**Definition.** How a C# type reaches a specific MongoDB collection. There is no naming convention
class in this repository; the collection name is a **string literal repeated in three places**.

**Representation.**

| Location | Literal | Purpose |
|---|---|---|
| `Infrastructure/Extensions.cs:90` | `.AddMongoRepository<ResourceDocument, Guid>("resources")` | Registers `IMongoRepository<ResourceDocument, Guid>` bound to the collection |
| `Infrastructure/Mongo/Queries/Handlers/GetResourceHandler.cs:21` | `_database.GetCollection<ResourceDocument>("resources")` | Read side, single document |
| `Infrastructure/Mongo/Queries/Handlers/GetResourcesHandler.cs:24` | `_database.GetCollection<ResourceDocument>("resources")` | Read side, filtered list |

Plus two test-side repetitions: `MongoDbFixture<ResourceDocument, Guid>("resources")` in
`tests/…Tests.EndToEnd/Sync/AddResourceTests.cs`, `…/GetResourceTests.cs` and
`tests/…Tests.Integration/Async/AddResourceTests.cs`.

The **write** side never uses the literal directly — `ResourcesMongoRepository`
(`§3.11`) takes the registered `IMongoRepository<ResourceDocument, Guid>`, which already carries
the name. The **read** side bypasses the repository and goes to `IMongoDatabase` for query
efficiency, which is why it must repeat the literal.

**Storage — the other two collections.** `outbox.inboxCollection: "inbox"` and
`outbox.outboxCollection: "outbox"` (`Api/appsettings.json:107-108`) are managed entirely by
Convey's `AddMessageOutbox(o => o.AddMongo())` (`Extensions.cs:82`). No code in this repository
names or reads them. The database itself is `mongo.database` — `availability-service` in base and
docker, `resource-test-db` under `tests`.

**Lifecycle.** MongoDB creates a collection lazily on first write, so there is no provisioning
step. `mongo.seed: false` in every profile means Convey's seeder is inert; there is no
`IMongoDbSeeder` implementation in this repository.

**Invariants & enforcement.**

- The five occurrences of `"resources"` must agree. **Nothing enforces this.** Changing the
  registration at `Extensions.cs:90` without changing both query handlers produces a service whose
  writes land in one collection and whose reads return empty from another — a
  `GET /resources/{id}` returning `404` immediately after a `201 Created`. Silent, and it would
  survive the unit tests (which never touch Mongo) while failing the E2E tests (which pin the
  literal independently, so they would fail for the *right* reason but point at the wrong place).
- Field names inside the document come from Mongo's default POCO conventions plus whatever Convey
  registers; there are no `[BsonElement]` attributes anywhere in
  `Infrastructure/Mongo/Documents/`. Property renames therefore rename stored fields. See `§5`.
- `IIdentifiable<Guid>` supplies `Id`, which the driver maps to `_id`. `Guid` is stored as a
  BSON binary subtype, not a string — relevant to anyone querying the collection by hand.

**Extension procedure — add a collection.** (1) Define the document type implementing
`IIdentifiable<TKey>` under `Infrastructure/Mongo/Documents/`. (2) Add
`.AddMongoRepository<TDoc, TKey>("name")` to the chain in `Extensions.cs`. (3) Introduce a
`const string` for the name and use it in the registration **and** in any query handler, rather
than extending the current three-literal pattern.

**Failure modes.** Beyond the divergence above: because both read paths construct their own
`IMongoCollection` from `IMongoDatabase`, they do not participate in whatever configuration
`AddMongoRepository` applies to the registered repository (read preference, write concern). Today
Convey applies none, so the two agree — but that is coincidence, not design.

---

### 3.34 `IDomainEventHandler<T>` — a live but unused extension point

**Definition.** A per-domain-event handler hook that runs **inside** `EventProcessor`, between the
aggregate raising the event and the integration event being published. It is the intended place
for reactions that must happen in the same logical unit of work as the domain change, but that do
not belong in the aggregate.

**Representation.** `src/…Application/Events/IDomainEventHandler.cs` — nine lines:

```csharp
public interface IDomainEventHandler<in T> where T : class, IDomainEvent
{
    Task HandleAsync(T @event);
}
```

The generic parameter is **contravariant** (`in T`), so a handler for a base event type would also
be resolvable for derived ones — irrelevant today, since all five domain events
(`Core/Events/*.cs`) derive directly from the empty `IDomainEvent` marker with no hierarchy.

**Wiring.** Two halves, both already in place:

1. **Registration** — `Infrastructure/Extensions.cs:69-72` scans **every assembly in the current
   AppDomain** with Scrutor, finds classes assignable to `IDomainEventHandler<>`, registers them
   `AsImplementedInterfaces()` with `WithTransientLifetime()`.
2. **Invocation** — `Infrastructure/Services/EventProcessor.cs:55-60`:

```csharp
var handlerType = typeof(IDomainEventHandler<>).MakeGenericType(eventType);
dynamic handlers = scope.ServiceProvider.GetServices(handlerType);
foreach (var handler in handlers) { await handler.HandleAsync((dynamic) @event); }
```

Reflection to build the closed generic, then `dynamic` dispatch to call it — because the static
type is `IDomainEvent`, not `T`.

**Lifecycle.** Handlers run inside the **single DI scope** created at `EventProcessor.cs:50`, once
per event, in the order the container returns them (registration order, which is assembly-scan
order — effectively unspecified). They run **before** `_eventMapper.Map(@event)` for that event
(`:62`), and **before** any integration event is published (publication happens after the whole
loop, at `:44`).

**Current population: zero.** A repository-wide search for implementations finds only the
interface declaration, the `EventProcessor` invocation and the Scrutor registration — no
implementing class exists in `src/` or `tests/`. `GetServices` therefore returns an empty
sequence on every event, and the `foreach` is a no-op.

**Invariants & enforcement.**

- A handler that throws **aborts the entire batch**. There is no try/catch in
  `HandleDomainEvents`, so the exception propagates out of `ProcessAsync` into the command handler,
  which has already called `UpdateAsync` — the write is committed, the events are not published.
  This is the same dual-write hazard described in `§3.17`, and adding the first
  `IDomainEventHandler` implementation is the change most likely to trigger it. **Loud** (the
  exception surfaces), but the resulting state is inconsistent.
- Handlers see the domain event, **not** the aggregate. `Resource` is not available to them; if a
  handler needs the aggregate it must reload it from `IResourcesRepository`, which will return the
  already-updated state.
- `AddClasses(...).AsImplementedInterfaces()` registers a handler for **every** interface it
  implements, not just `IDomainEventHandler<T>`. A class implementing both
  `IDomainEventHandler<ResourceCreated>` and, say, `IDisposable` gets registered under both. Keep
  handler classes single-purpose.

**Extension procedure.** Create a class in the **Application** layer implementing
`IDomainEventHandler<TEvent>` for one of the five events in `Core/Events/`. No registration call
is needed — the assembly scan finds it, provided the assembly is loaded into the AppDomain by the
time `AddInfrastructure` runs. Wrap anything that can fail in its own error handling if a failure
must not abort publication.

**Failure modes.**

- **`AppDomain.CurrentDomain.GetAssemblies()` only sees loaded assemblies.** .NET lazily loads
  assemblies on first use. `AddInfrastructure` runs early in `ConfigureServices`
  (`Api/Program.cs:34`), and the Application assembly is already loaded because `AddApplication()`
  (`:33`) ran first — but a handler placed in a *new* assembly referenced only by that handler
  would not be found. The failure is a handler that silently never runs. **Silent.**
- **`dynamic` dispatch defers all type errors to runtime.** A handler whose `HandleAsync`
  signature does not match — wrong parameter type, non-`Task` return — compiles fine and throws
  `RuntimeBinderException` at `:58` on the first event, inside the command handler.
- **Ordering is unspecified.** Two handlers for the same event have no defined order and no way to
  declare one.

---

### 3.35 Redis — configured with no consumer

**Definition.** A registered but entirely unused persistence dependency.

**Evidence.** Three occurrences, and only three:

| Location | Content |
|---|---|
| `Infrastructure/Pacco.Services.Availability.Infrastructure.csproj:23` | `<PackageReference Include="Convey.Persistence.Redis" Version="0.4.*" />` |
| `Infrastructure/Extensions.cs:20` | `using Convey.Persistence.Redis;` |
| `Infrastructure/Extensions.cs:85` | `.AddRedis()` in the fluent chain |

Configuration exists in `Api/appsettings.json:153-156`
(`connectionString: "localhost"`, `instance: "availability:"`) and is overridden in
`appsettings.docker.json` (`"redis"`) and `appsettings.tests.json` (`"localhost"`).

**No consumer.** A repository-wide search for `IDistributedCache`, `ICacheService`,
`IConnectionMultiplexer` or any Redis type outside those three lines returns nothing in `src/` or
`tests/`. Nothing reads or writes Redis.

**Lifecycle.** `.AddRedis()` registers the client and, in Convey 0.4, typically establishes the
connection eagerly or on first resolution. `[convey]` — the exact eagerness is not verifiable from
this workspace. Because nothing resolves the registered services, the practical effect today is a
package reference and a config block.

**Invariants & enforcement.** None. There is no invariant here; that is the point of documenting
it.

**Extension procedure — actually use it.** Inject Convey's Redis abstraction into an
Infrastructure-layer class (never into Application or Core — the port/adapter split in `§1.3`
requires the Application layer to see an interface it owns). The natural first use is caching
`GetResources` with no tags, which is currently an unbounded `Find(_ => true)` over the whole
collection (`GetResourcesHandler.cs:28`). Invalidation would hook the five domain events via
`IDomainEventHandler<T>` (`§3.34`) — the two unused extension points compose.

**Extension procedure — remove it.** Delete the three lines above and the `redis` block from all
four settings files. Nothing else depends on them. This is the lower-risk option and should be the
default unless a caching requirement is imminent.

**Failure modes.**

- **A Redis outage may or may not affect startup.** If `.AddRedis()` connects eagerly, an
  unreachable Redis fails the host for a dependency the service does not use. Marked
  `Unverifiable — Missing Source Evidence`; resolving it is the reason to care about this
  otherwise inert registration. ABQ Q-10.
- **`redis.instance: "availability:"`** is a key prefix. If Redis is later shared across services,
  removing or changing this prefix would collide with other services' keys — a cross-component
  failure originating in a config value nobody is currently reading.

---

### 3.36 Test topology — Unit, Integration, EndToEnd, Performance, Shared

**Definition.** Five test projects under `tests/`, each pinned to a different seam of the
component. Knowing which seam each one covers is what tells you whether a change is protected.

**Representation.**

| Project | Files | Seam exercised | Infrastructure required |
|---|---|---|---|
| `…Tests.Unit` | `Core/Entities/CreateResourceTests.cs`, `Applications/Handlers/ReserveResourceHandlerTests.cs` | Domain rules; one handler with all four dependencies substituted | none |
| `…Tests.Integration` | `Async/AddResourceTests.cs` | AMQP command in → Mongo document out | RabbitMQ + Mongo |
| `…Tests.EndToEnd` | `Sync/AddResourceTests.cs`, `Sync/GetResourceTests.cs` | HTTP in → Mongo | Mongo (+ whatever `tests` profile leaves on) |
| `…Tests.Performance` | `PerformanceTests.cs` | `GET /resources` throughput | a **running** service at `http://localhost:5001` |
| `…Tests.Shared` | `Factories/`, `Fixtures/`, `Helpers/` | no tests — support code | — |

**Lifecycle — the shared machinery.**

- **`PaccoApplicationFactory<TEntryPoint>`** (`tests/…Tests.Shared/Factories/PaccoApplicationFactory.cs:6-10`)
  is a `WebApplicationFactory<T>` whose only override is
  `base.CreateWebHostBuilder().UseEnvironment("tests")` (`:9`). That single line is what selects
  `appsettings.tests.json` and therefore everything in `§3.32`'s `tests` column. It substitutes
  **nothing** — the real `Program` runs with the real Mongo, real RabbitMQ and real
  `CustomersServiceClient`.
- **`MongoDbFixture<TEntity,TKey>`** (`…/Fixtures/MongoDbFixture.cs`) opens its own `MongoClient`
  from `OptionsHelper.GetOptions<MongoDbOptions>("mongo")` (`:22`), i.e. **bypassing the host's DI
  entirely**, and on `Dispose` calls `_client.DropDatabase(_databaseName)` (`:68`). Under the
  `tests` profile that database is `resource-test-db`, which is why dropping it is safe — and why
  changing `mongo.database` in `appsettings.tests.json` to the real name would make the test suite
  delete the development database. Note `InitializeMongo()` (`:31-33`) exists but its call site is
  commented out at `:27`.
- **`RabbitMqFixture`** (`…/Fixtures/RabbitMqFixture.cs`) does the same for AMQP: its own
  `ConnectionFactory` from `OptionsHelper`, publishing with a **locally reimplemented**
  `SnakeCase` (`:91-94`) rather than Convey's conventions, and `SubscribeAndGet` (`:52-89`)
  declaring a `test_{snake_case_message}` queue bound to the `availability` exchange.
- **`OptionsHelper`** (`…/Helpers/OptionsHelper.cs:7-23`) — see `§3.32`; it reads
  `appsettings.tests.json` from disk with `optional: true` and no layering.

**What the tests actually assert.**

- `CreateResourceTests` — `Resource.Create` with a valid id and one tag produces exactly one
  buffered event, of type `ResourceCreated` (`:29-32`); empty tags throw
  `MissingResourceTagsException` (`:41-44`). This pins `§3.1` and `§3.4`, and indirectly the
  `Version`/event-buffer rule of `§3.2`.
- `ReserveResourceHandlerTests` — an unknown resource id throws `ResourceNotFoundException`
  (`:22-29`); a known resource plus a customer whose `State == "valid"` (`:38-42`) results in
  `UpdateAsync(resource)` **and** `ProcessAsync(resource.Events)` both being received (`:46-47`).
  Critically, `_appContext` is a bare `Substitute.For<IAppContext>()` (`:63`), so
  `Identity.IsAuthenticated` is `false` — **the test passes precisely because the authorization
  guard is short-circuited**, which is the behaviour flagged in `§3.12`. The one authorization
  rule in the component is not covered by any test; it is *neutralised* by the only test that
  touches it.
- `Sync/AddResourceTests` — `201 Created` (`:24-32`), `Location: resources/{ResourceId}` (`:34-45`,
  pinning the `afterDispatch` convention of `Api/Program.cs:42-43`), and the document landing in
  the `"resources"` collection with matching `Id` and `Tags`.
- `Sync/GetResourceTests` — `404` for a missing document (`:20-27`), and a correct DTO for an
  inserted one (`:29-40`). Its `InsertResourceAsync` (`:48-61`) writes
  `TimeStamp = DateTime.UtcNow.AsDaysSinceEpoch()`, making it the only test that touches the lossy
  date encoding of `§3.10` — and it never asserts on the round-tripped date, so the lossiness is
  untested.
- `Async/AddResourceTests` — publishes `AddResource` to the `availability` exchange
  (`:16`, `Exchange = "availability"` at `:38`), subscribes for `ResourceAdded`, and asserts the
  Mongo document. This is the only test covering the AMQP transport, the `EventMapper` and the
  publish pipeline end to end.
- `PerformanceTests.get_resources` — NBomber, 1 concurrent copy, 3 seconds, asserting
  `RPS >= 100` and `OkCount >= 300` against a hardcoded `http://localhost:5001/resources`.

**Invariants & enforcement.**

- **The test suite is not run in CI.** `.travis.yml:12-15` runs `./scripts/build.sh`
  (`dotnet build -c release`) and then `./scripts/dockerize.sh` on success. `scripts/test.sh`
  exists (`dotnet test`) and is **never invoked**. Every assertion above is advisory unless a
  developer runs it locally. **Silent** — a red suite cannot break the build. This is the single
  most consequential fact in this section.
- The three infrastructure-dependent projects have no container orchestration in this repository —
  no `docker-compose.yml`, no fixture-managed containers. They assume Mongo on
  `localhost:27017` and RabbitMQ on `localhost:5672` per `appsettings.tests.json`.
- `PerformanceTests` is an `xUnit` `[Fact]`, so `dotnet test` would attempt to run it — against a
  service that is not running — making a naive `scripts/test.sh` fail. That is a plausible reason
  the script is not wired into CI, and it is fixable by moving the fact behind a trait filter.
- Two different `InternalsVisibleTo` targets exist —
  `Pacco.Services.Availability.Tests.Unit` (`Application/Extensions.cs:6`) and
  `Pacco.Services.Availability.UnitTests` (`Application/Tests/EnableInternalsTesting.cs`). Only
  the first matches a real project name; the second grants access to an assembly that does not
  exist. Harmless, but it is dead configuration that will mislead the next person who adds a test
  project.

**Extension procedure — add a unit test.** Put it in `…Tests.Unit`, substitute every dependency
with NSubstitute, assert with Shouldly. Follow the file's `#region Arrange` convention: the
constructor wires substitutes, a private `Act(...)` names the operation under test. For handler
tests, **set up `IAppContext.Identity` explicitly** — the default substitute silently disables the
authorization path.

**Extension procedure — add an E2E test.** Take `PaccoApplicationFactory<Program>` via
`IClassFixture`, set `factory.Server.AllowSynchronousIO = true` (required by the dispatcher
endpoints' synchronous body writes), create a `MongoDbFixture<TDoc,TKey>` with the collection
name, and implement `IDisposable` to dispose it — otherwise the test database is not dropped and
the next run starts dirty.

**Failure modes.**

- **`MongoDbFixture.Dispose` drops the whole database, not the collection.** Two E2E test classes
  running in parallel would drop the database out from under each other. xUnit runs classes in
  parallel by default and there is no `[Collection]` attribute on any of them — so
  `AddResourceTests` and `GetResourceTests` in `…Tests.EndToEnd` are already at risk of exactly
  this. ABQ Q-11.
- **`RabbitMqFixture` reimplements snake-casing.** Its `SnakeCase` (`:91-94`) and Convey's
  `snakeCase` convention (`rabbitMq.conventionsCasing`) are independent implementations. They agree
  for every current message name, but a name they disambiguate differently — anything with
  consecutive capitals — would make the integration test publish to a routing key nothing consumes,
  and the test would hang on `await tcs.Task` rather than fail with a useful message.
- **`RabbitMqFixture` never disposes in the test.** `Async/AddResourceTests.Dispose` (`:49-52`)
  disposes only `_mongoDbFixture`; the AMQP channel and connection leak per test class.
- **No test covers a rejected event, the outbox, `ExceptionToMessageMapper`, the
  `Correlation-Context` header, or any of the four AMQP-subscribed commands other than
  `AddResource`.** The areas with the most silent failure modes in this document are the areas
  with zero coverage.

---

## 4. Primary control flows

Each trace below runs entry point → key functions → datastore / side-effects. Frames provided by
Convey rather than by this repository are marked `[convey]`; every other frame cites a file and
line in `hianshul100_Pacco.Services.Availability`.

### 4.1 `POST /resources` — create a resource over HTTP

```
HTTP POST /resources  {resourceId?, tags[]}
 │
 ├─ Kestrel → ASP.NET Core pipeline
 │   ├─ UseErrorHandler                       Extensions.cs:98            [convey] wraps everything below
 │   ├─ UseSwaggerDocs / UseJaeger            Extensions.cs:99-100        [convey] starts the request span
 │   ├─ UseConvey / UsePublicContracts        Extensions.cs:101-102       [convey]
 │   ├─ UseMetrics                            Extensions.cs:103           [convey] Prometheus endpoint
 │   ├─ CustomMetricsMiddleware.InvokeAsync   Metrics/CustomMetricsMiddleware.cs:30
 │   │     key "POST:/resources" hits  →  counter commands{command=AddResource}++
 │   ├─ UseCertificateAuthentication          Extensions.cs:105           [convey]
 │   └─ UseDispatcherEndpoints .Post<AddResource>("resources")   Api/Program.cs:42
 │         binds body → AddResource; ctor normalises:
 │           ResourceId == Guid.Empty → Guid.NewGuid()      Commands/AddResource.cs
 │           Tags       == null       → Enumerable.Empty<string>()
 │
 ├─ ICommandDispatcher.SendAsync(cmd)                                     [convey]
 │   └─ ICommandHandler<AddResource>  — resolved through the decorator stack:
 │        OutboxCommandHandlerDecorator      Decorators/OutboxCommandHandlerDecorator.cs
 │          _messageId = inbound MessageId ?? Guid.NewGuid().ToString("N")   :27-29
 │          _outbox.Enabled? → _outbox.HandleAsync(_messageId, inner)  else → inner
 │        JaegerCommandHandlerDecorator      Jaeger/JaegerCommandHandlerDecorator.cs
 │          span "handling-add_resource"; on throw → tag error, rethrow
 │        LoggingCommandHandlerDecorator                                   [convey]
 │          template from MessageToLogTemplateMapper.cs:16-25
 │        AddResourceHandler.HandleAsync      Commands/Handlers/AddResourceHandler.cs:21
 │
 ├─ _repository.ExistsAsync(command.ResourceId)     AddResourceHandler.cs:23
 │     └─ ResourcesMongoRepository → IMongoRepository → db "availability-service", coll "resources"
 │     hit → throw ResourceAlreadyExistsException  :25
 │
 ├─ Resource.Create(command.ResourceId, command.Tags)   Core/Entities/Resource.cs:50
 │     ├─ implicit Guid → AggregateId          Core/Entities/AggregateId.cs  (Guid.Empty → throw)
 │     ├─ ValidateTags                          Resource.cs:37-48  (null/empty → MissingResourceTagsException;
 │     │                                        any null/whitespace tag → InvalidResourceTagsException)
 │     └─ AddEvent(new ResourceCreated(this))   AggregateRoot.cs  → Version 0 → 1, buffer = [ResourceCreated]
 │
 ├─ _repository.AddAsync(resource)              AddResourceHandler.cs:29
 │     └─ resource.AsDocument()  Mongo/Documents/Extensions.cs:15-26  → InsertOne into "resources"
 │            Reservations → ReservationDocument{ TimeStamp = DateTime.AsDaysSinceEpoch(), Priority }
 │                                                        ↑ LOSSY, see §3.10
 │
 └─ _eventProcessor.ProcessAsync(resource.Events)   AddResourceHandler.cs:30
       └─ EventProcessor.ProcessAsync            Services/EventProcessor.cs
            :31-34  events null → return                        (silent)
            :50     one DI scope for the whole batch
            :55-60  IDomainEventHandler<ResourceCreated> → none registered → no-op  (§3.34)
            :62-66  EventMapper.Map(ResourceCreated) → ResourceAdded
                    null → continue                              (silent drop, §3.14)
            :38-41  empty result → return                        (silent)
            :44     MessageBroker.PublishAsync(integrationEvents)
                      Services/MessageBroker.cs:54-65  capture spanContext / correlationId /
                                                       correlationContext / Saga header
                      :74   messageId = new GUID  (per event, always fresh)
                      :76-81 outbox enabled → _outbox.SendAsync(...)  → Mongo "outbox" collection
                      :83    else            → _busPublisher.PublishAsync(...)
                                               exchange "availability", routing key "resource_added"

HTTP response ← afterDispatch  Api/Program.cs:43  →  201 Created, Location: resources/{ResourceId}
```

**Side-effect summary.** One document in `resources`; one row in `outbox` (or one AMQP publish);
one counter increment; one Jaeger span; one Seq log line.

**The ordering hazard.** `AddAsync` (`:29`) commits before `ProcessAsync` (`:30`) publishes. With
`outbox.disableTransactions: true` (`Api/appsettings.json:109`) the outbox write is not in a
transaction with the document write either, so a crash between them yields a resource that exists
with no `ResourceAdded` ever published. See `§3.17`.

**The check-then-act race.** `ExistsAsync` at `:23` and `AddAsync` at `:29` are separate round
trips with no unique index declared anywhere in this repository. Two concurrent `POST`s with the
same explicit `resourceId` can both pass the check. The second insert may or may not fail
depending on Mongo's `_id` uniqueness — `_id` **is** unique, so the second `InsertOne` throws a
duplicate-key `MongoWriteException`, which no handler catches and which
`ExceptionToResponseMapper` maps to a generic `400 {"code":"error"}` rather than the
`409`-flavoured `resource_already_exists` the same collision produces one line earlier. ABQ Q-4.

### 4.2 `POST /resources/{resourceId}/reservations/{dateTime}` — reserve, with expropriation

```
HTTP POST /resources/{id}/reservations/{dateTime}   body: {priority, customerId}
 │
 ├─ pipeline as §4.1 (CustomMetricsMiddleware does NOT match this path — no counter)
 ├─ UseDispatcherEndpoints .Post<ReserveResource>(...)   Api/Program.cs:44
 │     route values {resourceId},{dateTime} + body → ReserveResource
 ├─ decorator stack (Outbox → Jaeger "handling-reserve_resource" → Logging)
 └─ ReserveResourceHandler.HandleAsync    Commands/Handlers/ReserveResourceHandler.cs:27
      │
      ├─ var identity = _appContext.Identity                                :29
      │    IAppContext resolved via AddTransient(ctx => IAppContextFactory.Create())  Extensions.cs:64
      │    AppContextFactory.Create()  Contexts/AppContextFactory.cs:19-33
      │      HTTP path → CorrelationContext from "Correlation-Context" header (Extensions.cs:117-120)
      │      AMQP path → serialize/deserialize round trip of the message context
      │
      ├─ GUARD  if (identity.IsAuthenticated && identity.Id != CustomerId && !identity.IsAdmin)  :30
      │            → throw UnauthorizedResourceAccessException                                  :32
      │         ⚠ IsAuthenticated is false whenever the header is absent  →  guard is skipped
      │
      ├─ _repository.GetAsync(command.ResourceId)      :35   → Mongo "resources" → AsEntity()
      │      null → throw ResourceNotFoundException    :38
      │
      ├─ _customersServiceClient.GetStateAsync(CustomerId)   :41
      │      Services/Clients/CustomersServiceClient.cs
      │        _url = options.Services["customers"]   (unguarded indexer, §3.25)
      │        GET {_url}/customers/{id}/state   via Fabio (httpClient.type: "fabio")
      │        non-2xx / transport failure → returns null  (silent, §3.25)
      │      null → throw CustomerNotFoundException    :44
      │
      ├─ !customerState.IsValid  :47  → throw InvalidCustomerStateException(..., customerState?.State)  :49
      │      CustomerStateDto.IsValid dereferences State unconditionally → NRE if State is null (§3.26)
      │
      ├─ new Reservation(command.DateTime, command.Priority)   :52     (struct, Core/ValueObjects/Reservation.cs)
      ├─ resource.AddReservation(reservation)                  :53
      │      Core/Entities/Resource.cs:57-80
      │        same-date reservation exists?
      │          new.Priority >= existing.Priority  → expropriate:
      │              remove existing, add new, AddEvent(ReservationCanceled(existing))
      │                                            AddEvent(ReservationAdded(new))
      │          else → throw CannotExpropriateReservationException
      │        no clash → add + AddEvent(ReservationAdded)
      │      Version increments ONLY on the first AddEvent of the request  (AggregateRoot.cs)
      │
      ├─ _repository.UpdateAsync(resource)                     :54
      │      ResourcesMongoRepository.cs:30-32
      │        ReplaceOneAsync(r => r.Id == id && r.Version < resource.Version, doc)
      │        ⚠ ReplaceOneResult DISCARDED — a lost update is indistinguishable from success (§3.11)
      │
      └─ _eventProcessor.ProcessAsync(resource.Events)         :55
           → ReservationCanceled → ReservationCanceled(integration)   EventMapper.cs:17-26
             ReservationAdded    → ResourceReserved
           → published as ONE batch, sharing one span/correlation context
```

**The two failure paths that matter.** `CannotExpropriateReservationException` is a
`DomainException` with `Code = "cannot_expropriate_reservation"`, so HTTP callers get a
`400 {"code":"cannot_expropriate_reservation"}` (`§3.18`). Over AMQP, the same exception is mapped
by `ExceptionToMessageMapper.cs:12-56` to a **`ReleaseResourceReservationRejected`** event — not
to a `ReserveResourceRejected`, which does not exist (`§3.16`). A saga listening for a reserve
failure must listen on the release-rejection routing key. ABQ Q-2.

### 4.3 AMQP command consumption — the same handlers, a different envelope

```
RabbitMQ exchange "availability", routing key e.g. "reserve_resource"
 queue "availability-service/availability.reserve_resource"       (template, appsettings.json:145)
 │
 ├─ UseRabbitMq() + SubscribeCommand<T>()      Extensions.cs:106-110         [convey]
 ├─ Jaeger RabbitMQ plugin        AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())  Extensions.cs:81
 │     reads header "span_context" → continues the distributed trace
 ├─ message context header "message_context" (appsettings.json:147-150) → ICorrelationContextAccessor
 ├─ Inbox: OutboxCommandHandlerDecorator → _outbox.HandleAsync(inbound MessageId, inner)
 │     ⚠ de-duplication works here because the inbound MessageId is stable (unlike §4.1)
 ├─ Jaeger + Logging decorators (identical to §4.1)
 └─ the SAME ICommandHandler<T> as the HTTP path
       AppContextFactory.Create() takes the AMQP branch → identity from the message context

on success  → integration events published exactly as in §4.1
on throw    → ExceptionToMessageMapper.Map(exception, message)   Exceptions/ExceptionToMessageMapper.cs:12-56
                match → an IRejectedEvent is published, message is ACKed
                no match (six `_ => null` arms) → NO rejected event; the message is nacked/dropped
                                                  and the caller waits forever  (silent, §3.16)
```

All four command types are subscribed: `AddResource`, `DeleteResource`,
`ReleaseResourceReservation`, `ReserveResource` (`Extensions.cs:107-110`) — the set matches
`Application/Commands/` exactly, so no command is reachable over HTTP but not over AMQP, or vice
versa. **There is no `ReleaseResource` command**, yet `Application/Events/Rejected/ReleaseResourceRejected.cs`
exists and is published — for a `ResourceNotFoundException` raised while handling a
`ReleaseResourceReservation` (`ExceptionToMessageMapper.cs:45-46`). The rejected event is named
after a command that does not exist, while the command that does exist has its *other* failures
reported as `ReleaseResourceReservationRejected` — which is itself the name used for every
`ReserveResource` failure (`§3.16`). ABQ Q-1.

### 4.4 `VehicleDeleted` — an external event that mutates this component's state

```
RabbitMQ exchange "vehicles"   ([Message("vehicles")] on Application/Events/External/VehicleDeleted.cs)
 queue "availability-service/vehicles.vehicle_deleted"
 │
 ├─ SubscribeEvent<VehicleDeleted>()            Extensions.cs:112
 ├─ OutboxEventHandlerDecorator (inbox de-dupe) Decorators/OutboxEventHandlerDecorator.cs
 ├─ LoggingEventHandlerDecorator → Before: "Vehicle with id: {VehicleId} has been deleted."
 │                                  MessageToLogTemplateMapper.cs:30
 └─ VehicleDeletedHandler.HandleAsync           Events/External/Handlers/VehicleDeletedHandler.cs:17
      └─ _dispatcher.SendAsync(new DeleteResource(@event.VehicleId))
           ⚠ ASSUMES resourceId == vehicleId. Nothing in this repository establishes that.
         └─ DeleteResourceHandler.HandleAsync   Commands/Handlers/DeleteResourceHandler.cs:20
              ├─ _repository.GetAsync(id)                    :22
              │     null → throw ResourceNotFoundException   :26
              ├─ resource.Delete()                Core/Entities/Resource.cs:92-100
              │     AddEvent(ReservationCanceled) per reservation, then AddEvent(ResourceDeleted)
              ├─ _repository.DeleteAsync(resource.Id)        :30   → DeleteOne from "resources"
              └─ _eventProcessor.ProcessAsync(resource.Events)  :31
                    → N × ReservationCanceled + 1 × ResourceDeleted, one batch
```

**No authorization anywhere on this path.** `DeleteResourceHandler` does not take `IAppContext`;
`DELETE /resources/{resourceId}` (`Api/Program.cs:46`) reaches the same handler. Deletion is
governed entirely by the API gateway's edge rules, documented in
[`api-gateway.md`](./api-gateway.md).

**The rejection blind spot.** If `DeleteResourceHandler` throws, the exception unwinds through
`VehicleDeletedHandler`. `ExceptionToMessageMapper` is invoked with the **inbound** message —
`VehicleDeleted` — not the dispatched `DeleteResource`. No arm of the mapper keys on
`VehicleDeleted`, so it returns `null` and the failure is silently lost. A vehicle deleted in
`vehicles-service` whose resource does not exist here produces no rejected event, no compensating
action, and no alert.

### 4.5 `GET /resources` and `GET /resources/{resourceId}` — the read path

```
HTTP GET /resources?tags=a&tags=b&matchAllTags=true
 ├─ CustomMetricsMiddleware  →  "GET:/resources" hits  →  queries{query=GetResources}++
 ├─ .Get<GetResources, IEnumerable<ResourceDto>>("resources")   Api/Program.cs:40
 ├─ IQueryDispatcher (in-memory)   AddInMemoryQueryDispatcher()  Extensions.cs:77
 └─ GetResourcesHandler.HandleAsync   Mongo/Queries/Handlers/GetResourcesHandler.cs:22
      collection = _database.GetCollection<ResourceDocument>("resources")     :24
      Tags null or empty → Find(_ => true).ToListAsync()   :28   ⚠ UNBOUNDED full-collection scan
      MatchAllTags ? All(...) : Any(...)                   :34-36
      → d.AsDto()  Mongo/Documents/Extensions.cs:28-38 (null-safe on Tags and Reservations)

HTTP GET /resources/{resourceId}
 ├─ NO metric (path is parameterised)
 ├─ .Get<GetResource, ResourceDto>("resources/{resourceId}")   Api/Program.cs:41
 └─ GetResourceHandler.HandleAsync   Mongo/Queries/Handlers/GetResourceHandler.cs:19
      Find(r => r.Id == query.ResourceId).SingleOrDefaultAsync()   :21-23
        ⚠ Single, not First — duplicate _id values would throw rather than return one
      document?.AsDto()   :25   → null → Convey returns 404
```

The read path **never touches `Resource`, `IResourcesRepository`, `AggregateRoot` or any domain
rule**. It goes document → DTO directly. That is the CQRS split, and it is why the days-since-epoch
date loss (`§3.10`) shows up in query results even though the domain object that produced the
event held a full `DateTime`.

### 4.6 Startup composition

```
Program.Main → CreateWebHostBuilder            Api/Program.cs:28
 ├─ WebHost.CreateDefaultBuilder(args)         :29   appsettings.json + appsettings.{env}.json + env vars
 ├─ ConfigureServices                          :30-35
 │    AddConvey() → AddWebApi() → AddApplication() → AddInfrastructure() → Build()
 │      AddApplication()     Application/Extensions.cs   command handlers, events, in-memory dispatcher
 │      AddInfrastructure()  Infrastructure/Extensions.cs:55-94
 │        :57-66  concrete registrations (EventMapper singleton, broker, repo, client, contexts, metrics)
 │        :67-68  TryDecorate ICommandHandler<> / IEventHandler<> with the outbox decorators
 │        :69-72  Scrutor scan for IDomainEventHandler<>  (finds nothing today)
 │        :74-93  Convey chain: error handler, queries, http, consul, fabio, rabbit(+jaeger plugin),
 │                outbox(mongo), exception-to-message mapper, mongo, redis, metrics, jaeger,
 │                jaeger decorators, handlers logging, mongo repository "resources",
 │                swagger docs, certificate auth, security
 ├─ Configure                                  :36-46
 │    UseInfrastructure()   Extensions.cs:96-115   pipeline + 4 SubscribeCommand + 2 SubscribeEvent
 │    UseDispatcherEndpoints(...)               the 7 routes of §3.24
 ├─ UseLogging()                               :47   Serilog → console/file/Seq/ELK, excludeProperties redaction
 └─ UseVault()                                 :48   KV overlay + PKI + dynamic Mongo lease (§3.31)
```

**Decorator order is registration order.** `TryDecorate` at `:67` wraps the handler with the
outbox decorator; `AddJaegerDecorators()` at `:88` and `AddHandlersLogging()` at `:89` wrap
further. The resulting call order is **Outbox → Jaeger → Logging → handler** — which is why
`MessageBroker` sees the `handling-{command}` span as `ActiveSpan` (`§3.27`). Reordering these
three lines silently changes what span context is stamped on every published message.

---

## 5. Persistence & schema evolution

### 5.1 What is stored, and where

| Store | Database | Collection | Owner | Written by |
|---|---|---|---|---|
| MongoDB | `mongo.database` — `availability-service` (base/docker), `resource-test-db` (tests) | `resources` | this component | `ResourcesMongoRepository` |
| MongoDB | same database | `outbox` (`outbox.outboxCollection`) | Convey | `AddMessageOutbox(o => o.AddMongo())` `[convey]` |
| MongoDB | same database | `inbox` (`outbox.inboxCollection`) | Convey | outbox/inbox decorators `[convey]` |
| Redis | `redis.instance` prefix `availability:` | — | registered, **no keys written** | nothing (`§3.35`) |

Nothing else is persisted. There is no relational database, no file storage, no local cache.

### 5.2 The `resources` document shape

`Infrastructure/Mongo/Documents/ResourceDocument.cs`:

```csharp
internal sealed class ResourceDocument : IIdentifiable<Guid>
{
    public Guid Id { get; set; }                                  // → BSON _id, binary UUID
    public int Version { get; set; }                              // optimistic-concurrency token
    public IEnumerable<string> Tags { get; set; }
    public IEnumerable<ReservationDocument> Reservations { get; set; }
}

internal sealed class ReservationDocument   // ReservationDocument.cs
{
    public int TimeStamp { get; set; }      // days since DateTime.MinValue — NOT a Unix timestamp
    public int Priority { get; set; }
}
```

Reservations are stored **embedded**, not in a separate collection. The whole aggregate is one
document, which is what makes the single-document `ReplaceOneAsync` in `§3.11` an atomic
aggregate write.

**There are no mapping attributes.** No `[BsonElement]`, no `[BsonId]`, no
`[BsonIgnoreExtraElements]` anywhere in `Infrastructure/Mongo/Documents/`. Three consequences:

1. **Field names are property names.** Renaming `Version` renames the stored field; every existing
   document then reads back `Version = 0`, and `§3.11`'s `r.Version < resource.Version` predicate
   starts matching documents it should not.
2. **Unknown fields are a hard error by default.** Without `[BsonIgnoreExtraElements]`, the
   C# driver throws on a document containing a field the class does not declare. Removing a
   property from `ResourceDocument` therefore breaks reads of every document still carrying it.
   **Loud, at read time, per document.**
3. `TimeStamp` is an `int`. A value written as a `long` or `double` by any other writer
   deserialises with an overflow or format exception.

### 5.3 The three mapping functions

All in `Infrastructure/Mongo/Documents/Extensions.cs`:

| Function | Direction | Null-safety |
|---|---|---|
| `AsEntity` `:11-13` | document → `Resource` | **None.** `document.Reservations.Select(...)` NREs if the field is absent |
| `AsDocument` `:15-26` | `Resource` → document | N/A (the entity guarantees non-null collections) |
| `AsDto` `:28-38` | document → `ResourceDto` | **Full.** `Tags ?? Empty`, `Reservations?.Select(...) ?? Empty` |

The asymmetry is load-bearing and easy to miss: **the read path tolerates a partial document and
the write path does not.** A document inserted by hand without a `Reservations` array is returned
happily by `GET /resources/{id}` and crashes any command that loads it through
`ResourcesMongoRepository.GetAsync` (`:18-22`, which calls `AsEntity`). Callers see a
`400 {"code":"error"}`.

### 5.4 The date encoding — the schema decision with the widest blast radius

```csharp
internal static int AsDaysSinceEpoch(this DateTime dateTime) => (dateTime - new DateTime()).Days;   // :40-41
internal static DateTime AsDateTime(this int daysSinceEpoch) => new DateTime().AddDays(daysSinceEpoch);  // :43-44
```

`new DateTime()` is `0001-01-01T00:00:00`, so the stored integer is days since year 1, not since
1970. Round-tripping **truncates to whole days**: a reservation for `2026-09-04T14:30:00`
persists as `739_763`-ish and reads back as `2026-09-04T00:00:00`.

This is consistent with the domain, where `Reservation.Equals` compares `DateTime.Date` and
`GetHashCode` uses `DateTime.Date` alone (`§3.5`) — the domain genuinely treats a reservation as
day-granular. But the **integration events carry the full un-truncated `DateTime`** taken from the
in-memory aggregate, while the **read model returns the truncated one**. A consumer that stores
the event payload and later re-reads via `GET /resources/{id}` sees two different times for the
same reservation. `§3.10` covers this; the schema-evolution consequence is stated in `§5.5`.

### 5.5 How to evolve the schema safely

There are **no migrations** in this repository — no migration framework, no version marker on the
document, no `mongo.seed` implementation (`seed: false` in all four profiles). Schema evolution is
therefore entirely manual, and the safe procedure is:

**Adding a field.**
1. Add the property to `ResourceDocument` (or `ReservationDocument`).
2. Set it in `AsDocument` (`:15-26`).
3. Read it defensively in `AsEntity` (`:11-13`) — existing documents will not have it, and the
   driver supplies `default(T)`, which is silently valid for `int`/`Guid` and `null` for
   references.
4. If it must reach clients, add it to `ResourceDto`/`ReservationDto` and to `AsDto` (`:28-38`).
5. No backfill is required as long as step 3 treats the default as meaningful.

**Renaming a field.** Do not. A rename is a breaking read for every existing document (the old
field becomes an unmapped extra element → driver throws, per `§5.2`). If it must happen:
(a) add `[BsonElement("oldName")]` to preserve the wire name, **or** (b) add
`[BsonIgnoreExtraElements]`, deploy, backfill with an `$rename` aggregation, then remove the
attribute.

**Removing a field.** Add `[BsonIgnoreExtraElements]` to the class **first**, deploy, then remove
the property, then `$unset` it from existing documents. Removing the property first breaks reads.

**Changing the date encoding.** This is the change most likely to be wanted (`§5.4`) and it is the
most dangerous, because the stored integers are indistinguishable from a Unix-epoch encoding by
inspection alone. The safe route is additive: add a `DateTimeUtc` (`DateTime`) field alongside
`TimeStamp`, write both in `AsDocument`, read `DateTimeUtc` when non-default and fall back to
`TimeStamp.AsDateTime()` otherwise in `AsEntity`/`AsDto`, backfill, then remove `TimeStamp` via the
removal procedure above. Changing `AsDaysSinceEpoch`'s epoch in place silently reinterprets every
existing reservation as a date roughly 1970 years off.

**Changing `Version` semantics.** `Version` is written by `AsDocument` from
`AggregateRoot.Version`, which increments at most once per request (`§3.2`). The concurrency
predicate `r.Version < resource.Version` (`ResourcesMongoRepository.cs:30-32`) depends on
monotonicity. Do not reset it, do not make it a timestamp, and do not increment it more than once
per unit of work — a double increment makes the predicate skip a generation and permits a lost
update.

### 5.6 Indexes

**No index declarations exist in this repository.** The only guaranteed index is Mongo's implicit
unique index on `_id`. Two query shapes have no supporting index:

| Query | Source | Shape |
|---|---|---|
| by tag, any | `GetResourcesHandler.cs:36` | `Tags` array contains any of N |
| by tag, all | `GetResourcesHandler.cs:35` | `Tags` array contains all of N |

Both are collection scans. `GET /resources` with no tags (`:28`, `Find(_ => true)`) is an
unbounded scan with no `Limit` and no pagination anywhere in the query, the DTO, or the endpoint
(`Api/Program.cs:40`). The performance test (`§3.36`) asserts 100 RPS on exactly this endpoint —
against a database the test never populates, so it measures the empty-collection case.

**To add an index**, use Convey's Mongo initialiser hook or an out-of-band script; there is no
in-repository seam for it today, which is itself worth recording. ABQ B-2.

---

## 6. Surface → internals map

Every externally reachable entry point, with the internal code it actually runs and whether it
mutates state. "Absent" rows record surfaces a reader might reasonably expect and which do **not**
exist — those are as important as the present ones.

### 6.1 HTTP endpoints

Declared in one block, `Api/Program.cs:38-46`. There are no controllers.

| Method | Path | Read/Mutate | Internals reached | Auth enforced here? |
|---|---|---|---|---|
| `GET` | `/` | read-only | inline lambda → `AppOptions.Name` (`Program.cs:39`) | no |
| `GET` | `/resources` | read-only | `GetResources` → `GetResourcesHandler.cs:22` → Mongo `resources` | no |
| `GET` | `/resources/{resourceId}` | read-only | `GetResource` → `GetResourceHandler.cs:19` → Mongo `resources` | no |
| `POST` | `/resources` | **mutating** | `AddResource` → `AddResourceHandler.cs:21` → insert + `ResourceAdded` | no |
| `POST` | `/resources/{resourceId}/reservations/{dateTime}` | **mutating** | `ReserveResource` → `ReserveResourceHandler.cs:27` → `customers-service`, replace, `ResourceReserved` (+ `ResourceReservationCanceled`) | **partially** — the guard at `:30` is inert unless a `Correlation-Context` header is present |
| `DELETE` | `/resources/{resourceId}/reservations/{dateTime}` | **mutating** | `ReleaseResourceReservation` → `ReleaseResourceReservationHandler.cs:21` → replace, `ResourceReservationReleased` | **no** |
| `DELETE` | `/resources/{resourceId}` | **mutating** | `DeleteResource` → `DeleteResourceHandler.cs:20` → delete, N × `ResourceReservationCanceled` + `ResourceDeleted` | **no** |

**Authorization summary: one guard, on one endpoint, that a missing header disables.** Every other
mutating endpoint trusts the caller completely. The component's real access control lives at the
edge — see [`api-gateway.md`](./api-gateway.md) §3.7–§3.9.

### 6.2 Non-route HTTP surface (supplied by middleware, not by `Program.cs`)

| Path | Source | Purpose |
|---|---|---|
| `/docs` | `swagger.routePrefix: "docs"` (`appsettings.json:163`) + `.AddWebApiSwaggerDocs()` / `.UseSwaggerDocs()` (`Extensions.cs:91`, `:99`) | Swagger UI; `includeSecurity: true`, `reDocEnabled: false` |
| `/metrics` | `.AddMetrics()` / `.UseMetrics()` (`Extensions.cs:86`, `:103`), `metrics.prometheusEnabled: true` | Prometheus scrape |
| `/ping` | `.UseConvey()` (`Extensions.cs:101`); referenced by `consul.pingEndpoint` | Consul health check |

All three are excluded from Serilog and Jaeger via `logger.excludePaths` and `jaeger.excludePaths`
(`["/", "/ping", "/metrics"]`) — note `/docs` is **not** excluded, so Swagger UI requests are
logged and traced.

### 6.3 AMQP surface — consumed

| Exchange | Routing key | Type | Read/Mutate | Internals reached |
|---|---|---|---|---|
| `availability` | `add_resource` | command | **mutating** | `AddResourceHandler.cs:21` |
| `availability` | `delete_resource` | command | **mutating** | `DeleteResourceHandler.cs:20` |
| `availability` | `release_resource_reservation` | command | **mutating** | `ReleaseResourceReservationHandler.cs:21` |
| `availability` | `reserve_resource` | command | **mutating** | `ReserveResourceHandler.cs:27` |
| `customers` | `customer_created` | event | **none** | `CustomerCreatedHandler.cs:10` → `Task.CompletedTask` (deliberate no-op) |
| `vehicles` | `vehicle_deleted` | event | **mutating** | `VehicleDeletedHandler.cs:17` → dispatches `DeleteResource` |

### 6.4 AMQP surface — published

Exchange `availability` in all cases. Success events come from `EventMapper.cs:17-26`; rejections
from `ExceptionToMessageMapper.cs:12-56`.

| Event | Routing key | Raised by |
|---|---|---|
| `ResourceAdded` | `resource_added` | `ResourceCreated` domain event |
| `ResourceDeleted` | `resource_deleted` | `ResourceDeleted` domain event |
| `ResourceReserved` | `resource_reserved` | `ReservationAdded` domain event |
| `ResourceReservationReleased` | `resource_reservation_released` | `ReservationReleased` domain event |
| `ResourceReservationCanceled` | `resource_reservation_canceled` | `ReservationCanceled` domain event |
| `AddResourceRejected` | `add_resource_rejected` | tag/duplicate failures |
| `DeleteResourceRejected` | `delete_resource_rejected` | `ResourceNotFound` + `DeleteResource` |
| `ReleaseResourceRejected` | `release_resource_rejected` | `ResourceNotFound` + `ReleaseResourceReservation` |
| `ReleaseResourceReservationRejected` | `release_resource_reservation_rejected` | **all five `ReserveResource` failure modes** (`§3.16`) |

Routing keys are derived from the type name by the `snakeCase` convention (`§3.22`); none of these
types carries a `[Message]` attribute, so the derivation is the only naming authority.

### 6.5 Outbound calls

| Target | Trigger | Code |
|---|---|---|
| `GET {customers}/customers/{customerId}/state` | every `ReserveResource` | `Services/Clients/CustomersServiceClient.cs` |
| MongoDB | every command and query | `ResourcesMongoRepository`, `GetResource(s)Handler` |
| RabbitMQ | every published event | `Services/MessageBroker.cs:83` (or via the outbox collection) |
| Consul | startup registration + `/ping` health checks | `.AddConsul()` `[convey]` |
| Vault | startup + lease renewal | `.UseVault()` `[convey]` |
| Jaeger | every request and handled message | `.AddJaeger()`, `AddJaegerRabbitMqPlugin()` `[convey]` |
| Seq / ELK / file / console | every log event | `.UseLogging()` `[convey]` |
| Redis | **never** | registered, unused (`§3.35`) |

### 6.6 Absent surfaces

| Expected surface | Status |
|---|---|
| `PUT`/`PATCH` on a resource (edit tags) | **absent** — tags are set at creation and never change |
| Pagination / `limit` on `GET /resources` | **absent** — unbounded scan (`§5.6`) |
| A `ReserveResourceRejected` event | **absent** — reserve failures use the release name (`§3.16`) |
| JWT validation | **absent in code.** The `jwt` block (`appsettings.json:79-87`) and `certs/localhost.cer` exist, but no `AddJwt()`/`UseAuthentication()` call appears anywhere. The block is inert configuration; identity arrives via `Correlation-Context` (`§3.20`) |
| An HTTP health/readiness endpoint distinct from `/ping` | **absent** |
| Any `IDomainEventHandler<T>` implementation | **absent** (`§3.34`) |
| Any database migration or seeder | **absent** (`§5.5`) |
| Any index declaration | **absent** (`§5.6`) |
| An admin surface for expropriation policy | **absent** — priority comparison is hardcoded (`§3.6`) |

---

## 7. Change/extension guide

### 7.1 Add a new command (e.g. `ExtendReservation`)

1. `Application/Commands/ExtendReservation.cs` — an immutable class implementing `ICommand`, marked
   `[Contract]`. Follow `AddResource`'s constructor-normalisation pattern only if the field
   genuinely has a safe default; `ReserveResource` deliberately does not normalise.
2. `Application/Commands/Handlers/ExtendReservationHandler.cs` — `internal sealed`, implementing
   `ICommandHandler<ExtendReservation>`. It is discovered by `AddApplication()`'s scan; no explicit
   registration.
3. Load the aggregate, mutate it through a **method on `Resource`** (never mutate collections from
   the handler — `§3.1`), `UpdateAsync`, then `ProcessAsync(resource.Events)`. That four-step shape
   is identical in all four existing handlers; deviating from it is the main way to break the
   version/event contract.
4. Add the domain event to `Core/Events/`, the integration event to `Application/Events/`, and a
   **new arm to `EventMapper.cs:17-26`** — omitting the arm makes the event vanish silently
   (`§3.14`).
5. Add `.SubscribeCommand<ExtendReservation>()` in `Extensions.cs:107-110` if it should be
   reachable over AMQP, and a route in `Api/Program.cs:38-46` if over HTTP. These are independent
   — either, both, or neither.
6. Add rejection arms to `ExceptionToMessageMapper.cs` for **every (exception, command) pair**
   (`§3.16`).
7. Add a log template to `MessageToLogTemplateMapper.cs:15-31`.
8. If it should be counted, add a row to `CustomMetricsMiddleware.cs:18-22` — only possible for a
   literal, non-parameterised path (`§3.29.1`).
9. Unit-test the handler in `tests/…Tests.Unit`, **explicitly configuring `IAppContext.Identity`**
   if authorization matters (`§3.36`).

### 7.2 Add a new domain rule

Put it on `Resource` in `Core/Entities/Resource.cs`, as a private validation method called from
the public mutator, throwing a new `DomainException` subclass from `Core/Exceptions/` with an
explicit `Code` override. Then:
- add the code to whatever the API gateway or clients switch on;
- add an `ExceptionToMessageMapper` arm, or the AMQP path fails silently;
- `ExceptionToResponseMapper` needs **no** change — it reads `Code` off any `DomainException`
  (`§3.18`) — but note every mapped exception becomes a `400`, so a rule that should be a `409` or
  `403` cannot express that today.

### 7.3 Subscribe to a new external event

1. `Application/Events/External/TheEvent.cs` with `[Message("<source-exchange>")]`. Do **not** add
   `[Contract]` — this component does not own the contract.
2. `Application/Events/External/Handlers/TheEventHandler.cs` implementing `IEventHandler<TheEvent>`.
   Keep it thin: `VehicleDeletedHandler.cs:17` is one line that dispatches a command, and that is
   the pattern to copy.
3. `.SubscribeEvent<TheEvent>()` in `Extensions.cs:111-112`.
4. **Decide the rejection story explicitly.** `ExceptionToMessageMapper` receives the *inbound*
   event, so if your handler dispatches a command, failures of that command are invisible to the
   mapper unless you add an arm keyed on the inbound event type (`§4.4`).

### 7.4 Add a downstream HTTP dependency

See `§3.30`. The trap worth repeating: `httpClient.services` is redeclared wholesale in
`appsettings.docker.json`, `appsettings.local.json` and `appsettings.tests.json`, so a key added
only to `appsettings.json` is missing in every environment that runs — and
`CustomersServiceClient`-style unguarded indexing turns that into a `KeyNotFoundException` at
request time. Use `TryGetValue` in the new client.

### 7.5 Change the persistence shape

Follow `§5.5`. In one line: **add, never rename or remove**, because there are no
`[BsonIgnoreExtraElements]` attributes and no migrations.

### 7.6 Introduce real authorization

The pieces are already present and unused:
- `IIdentityContext` exposes `Id`, `Role`, `IsAuthenticated`, `IsAdmin`
  (`Infrastructure/Contexts/IdentityContext.cs`).
- `AddTransient(ctx => IAppContextFactory.Create())` (`Extensions.cs:64`) makes `IAppContext`
  injectable into any handler.
- `.AddCertificateAuthentication()` / `.UseCertificateAuthentication()` (`Extensions.cs:92`,
  `:105`) are already wired for mTLS.

The change is: inject `IAppContext` into `DeleteResourceHandler` and
`ReleaseResourceReservationHandler`, and **remove `identity.IsAuthenticated &&` from
`ReserveResourceHandler.cs:30`** so an anonymous caller is denied rather than allowed. Doing the
last part alone will break the existing unit test (`§3.36`), which is the correct signal.

### 7.7 Make failures visible

Ranked by value per line changed:
1. Check the `ReplaceOneResult` in `ResourcesMongoRepository.cs:30-32` and throw a concurrency
   exception on `ModifiedCount == 0` (`§3.11`).
2. Throw, or log a warning, in `Resource.ReleaseReservation` when the reservation is not found
   instead of returning silently (`Core/Entities/Resource.cs:86`, `§3.1`).
3. Log a warning on every `_ => null` arm of `EventMapper` and `ExceptionToMessageMapper`
   (`§3.14`, `§3.16`).
4. Null-guard `CustomerStateDto.IsValid` (`§3.26`).
5. Distinguish transport failure from "not found" in `CustomersServiceClient` (`§3.25`).

### 7.8 Wire the test suite into CI

`scripts/test.sh` exists and is never called. Add it to `.travis.yml:12-13` between `build.sh` and
`after_success`. Two prerequisites: the infrastructure-dependent projects need Mongo and RabbitMQ
services declared in the CI config, and `PerformanceTests` must be excluded (it targets a running
service at `localhost:5001`) — e.g. `dotnet test --filter Category!=Performance` after adding the
trait. `§3.36`.

### 7.9 What not to change without coordination

| Change | Breaks |
|---|---|
| Renaming any `[Contract]` type | its routing key, and therefore every subscriber (`§3.22`) |
| Changing `conventionsCasing` or `queue.template` | every existing queue binding |
| Changing `exchange.name` | every publisher and subscriber of `availability` |
| Changing the `"resources"` literal | must change in **five** places (`§3.33`) |
| Changing `AsDaysSinceEpoch`'s epoch | reinterprets every stored reservation date (`§5.4`) |
| Changing decorator registration order (`Extensions.cs:67-68`, `:88-89`) | the span context stamped on every published message (`§4.6`) |
| Adding a second increment to `AggregateRoot.Version` | the optimistic-concurrency predicate (`§3.2`, `§5.5`) |

---

## 8. Assumptions, Blockers & Open Questions (ABQ)

### 8.1 Assumptions (`A-n`) — stated so they can be falsified

| ID | Assumption | Basis | If wrong |
|---|---|---|---|
| **A-1** | Convey `0.4.*` behaves as the calling code implies — `TryDecorate` order, `AddMongoRepository` collection binding, `snakeCase` routing-key derivation, `SubscribeCommand`'s ack/nack policy, `UseVault`'s provider ordering. | The package references in `*.csproj`; call sites throughout `Infrastructure/Extensions.cs`. Convey's own source is **not in this workspace**. | Every frame marked `[convey]` in `§4` may execute differently. The 14 `Unverifiable — Missing Source Evidence` markers in this document are exactly the points where that matters. |
| **A-2** | The `Correlation-Context` header is set by the API gateway and cannot be set by an external client. | `Infrastructure/Extensions.cs:117-120` reads it unconditionally; the gateway's `on_success.headers` block ([`api-gateway.md`](./api-gateway.md) §3.14) writes it. | If the header reaches the service from outside, `§3.19`'s identity — and therefore the only authorization guard in the component — is forgeable. This is the single highest-severity assumption here. |
| **A-3** | `resourceId == vehicleId` for every resource created from a vehicle. | `VehicleDeletedHandler.cs:17` passes `@event.VehicleId` straight into `new DeleteResource(...)`. Nothing in this repository creates that correspondence. | `vehicle_deleted` either deletes nothing (`ResourceNotFoundException`, silently unmapped — `§4.4`) or deletes an unrelated resource. |
| **A-4** | Reservations are meaningfully day-granular. | `Reservation.Equals`/`GetHashCode` use `DateTime.Date`; `AsDaysSinceEpoch` truncates (`§3.5`, `§5.4`). | Sub-day booking is impossible without a schema change, and the full-precision `DateTime` on published events is misleading. |
| **A-5** | `customers-service` returns the literal `"valid"` for a bookable customer. | `CustomerStateDto.IsValid` compares against that one string (`§3.26`). | All reservations fail with `invalid_customer_state`, pointing the investigator at customers rather than at the contract. |
| **A-6** | The `outbox`/`inbox` collections live in the same Mongo database as `resources`. | `outbox.inboxCollection`/`outboxCollection` are bare names with no database key (`appsettings.json:107-108`). | `MongoDbFixture.Dispose`'s `DropDatabase` (`§3.36`) would also drop outbox state during tests — currently harmless because `outbox.enabled: false` under `tests`. |

### 8.2 Blockers (`B-n`) — things that cannot be resolved from this workspace

| ID | Blocker | What would resolve it |
|---|---|---|
| **B-1** | Convey 0.4 source is unavailable, so ack/nack semantics, requeue policy, outbox drain interval behaviour, Redis connection eagerness and Vault lease-renewal failure handling are all unverifiable. | The `Convey` repository at the pinned version, or a live broker/Vault to observe. |
| **B-2** | There is no in-repository seam for declaring Mongo indexes, and no evidence of indexes being created out of band. | Access to the deployed database, or an infrastructure repository holding provisioning scripts. |
| **B-3** | The `customers-service` contract (`GET /customers/{id}/state` response shape and its state vocabulary) is defined in another repository. | The `Pacco.Services.Customers` repository — out of scope for batch 1. |
| **B-4** | No `docker-compose.yml` or deployment manifest exists in this repository, so the actual runtime values of `ASPNETCORE_ENVIRONMENT`, Consul addresses and Vault enablement in a real deployment are unknown. `Dockerfile:10` sets `docker`, but an orchestrator can override it. | The deployment repository. |

### 8.3 Open questions (`Q-n`)

| ID | Question | Where it bites | Suggested resolution |
|---|---|---|---|
| **Q-1** | Is the release path's naming intentional? There is **no `ReleaseResource` command**, yet `ReleaseResourceRejected` exists and is published for a `ReleaseResourceReservation` failure (`ExceptionToMessageMapper.cs:45-46`); the gateway publishes `release_resource` while the command's snake-case key is `release_resource_reservation`. | `§3.16`, `§3.22`, `§3.28`, `§4.3` | Inspect a live broker's bindings; then either rename the rejected event or add a `[Message]` attribute pinning the key. |
| **Q-2** | Should `ReserveResourceRejected` exist? Five distinct reserve failures are published as `ReleaseResourceReservationRejected`. | `§3.16`, `§4.2` | Add the type, publish both for a transition window, then retire the misnamed one. Contract-breaking — needs cross-service coordination. |
| **Q-3** | What is the priority scale, and who owns it? `Reservation.Priority` is an unconstrained `int`; expropriation uses `>=` so an **equal** priority wins (`Core/Entities/Resource.cs:63`). Higher-is-stronger is implied but never stated. | `§3.5`, `§3.6` | Define the range in the domain (a value object or a guard in the `Reservation` constructor) and document whether ties expropriate. |
| **Q-4** | Should `resources` have a unique constraint beyond `_id`, and should the check-then-act in `AddResourceHandler.cs:23-29` be collapsed? | `§4.1` | Rely on `_id` uniqueness and catch the duplicate-key error, mapping it to `ResourceAlreadyExistsException` — this removes the race and fixes the misleading `400 {"code":"error"}`. |
| **Q-5** | Is a fresh `MessageId` per publish (`MessageBroker.cs:74`) intended? It defeats downstream inbox de-duplication for HTTP-originated events. | `§3.27`, `§3.17` | Derive the id deterministically from the originating message/request id when one exists. |
| **Q-6** | `MetricsJob` hardcodes `Task.Delay(5000)` and ignores `metrics.interval`. Bug or deliberate? | `§3.29.2` | One-line fix; the config key currently lies. |
| **Q-7** | `consul.address: "docker.for.win.localhost"` in the base `appsettings.json` is Windows-Docker-specific. | `§3.30` | Confirm no environment runs the base profile; if any does, parameterise. |
| **Q-8** | `vault.lease.mongo.templates.connectionString` hardcodes `localhost:27017`, so enabling the dynamic lease anywhere else produces an unusable connection string that **overrides** the correct one. | `§3.31` | Per-environment template, or interpolate the host from `mongo.connectionString`. |
| **Q-9** | `appsettings.development.json` is empty, making the accidental default profile the one with Consul, Fabio, Jaeger, metrics, the outbox and Vault all enabled against `localhost`. | `§3.32` | Either populate it as a safe-local profile or remove it so a missing environment fails visibly. |
| **Q-10** | Does `.AddRedis()` (`Extensions.cs:85`) connect eagerly? Nothing uses Redis, so an eager connection makes an unused dependency a startup risk. | `§3.35` | Verify against Convey; if unused, delete the registration and the config blocks. |
| **Q-11** | `MongoDbFixture.Dispose` drops the entire test **database**, and the two `…Tests.EndToEnd` classes have no xUnit `[Collection]` grouping — so they may run in parallel and drop the database under each other. | `§3.36` | Add a shared `[Collection]`, or drop only the collection. |
| **Q-12** | `Application/Tests/EnableInternalsTesting.cs` grants `InternalsVisibleTo("Pacco.Services.Availability.UnitTests")`, an assembly that does not exist; `Application/Extensions.cs:6` grants it to `Pacco.Services.Availability.Tests.Unit`, which does. | `§3.36` | Delete the dead one. |
| **Q-13** | Should any mutating endpoint other than `POST .../reservations/...` enforce identity? Today `DELETE /resources/{id}` and `DELETE /resources/{id}/reservations/{dateTime}` reach handlers with no `IAppContext` at all. | `§6.1`, `§7.6` | Confirm the gateway's rules are the intended sole control, and record that decision — or add the guards. |

---

## 9. Cross-references

- [`api-gateway.md`](./api-gateway.md) — the edge that terminates authentication, mints the
  `Correlation-Context` header this component trusts (`§3.20`, A-2), and publishes commands onto the
  `availability` exchange. Its §3.7–§3.9 (authentication gate, claim gate, token-subject binding)
  and §3.14 (correlation-context envelope) are the other half of `§3.19` here.
- **Not modelled in this batch** (batch 1 of 7 covers `api-gateway` and `availability-service`
  only): `customers-service` (B-3, A-5), `vehicles-service` (A-3, the `vehicle_deleted` producer),
  `operations-service` (the consumer of every rejected event in `§6.4`), and the identity service
  behind the gateway's JWT issuance. Claims in this document that depend on those components are
  marked as assumptions or blockers above rather than asserted.

---

*End of component-internals model for `availability-service`.*
