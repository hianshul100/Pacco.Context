# Component internals — `customers-service`

| | |
| --- | --- |
| **Component** | `customers-service` |
| **Source repository** | `hianshul100_Pacco.Services.Customers` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository) |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 2 of 7 |
| **Status** | New artifact — no prior `component-internals/customers-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` and `repo-summary/Pacco.Services.Customers.md` remain valid and are **complemented**, not replaced: those catalogue the surface, this document models the internals. |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full
> (`src/Pacco.Services.Customers.{Core,Application,Infrastructure,Api}`). It contains **no test
> project of any kind** — `find . -name '*.csproj'` returns exactly the four production projects,
> and there is no `tests/` directory. `Convey 0.4.*` — which supplies the CQRS dispatchers, the
> Mongo repository, the RabbitMQ client, the outbox, certificate authentication and the WebApi
> endpoint mapping — is a NuGet reference with **no source in this workspace**. Mechanisms owned by
> Convey are marked `[convey]` and, where their exact semantics change a conclusion, flagged
> `Unverifiable — Missing Source Evidence`. The upstream half of every inbound contract is modelled
> in `component-internals/api-gateway.md`.

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

`customers-service` owns **one aggregate — `Customer`** — and answers two questions for the rest of
the platform: *may this person transact?* (the `State` machine) and *is this person a VIP?* (the
loyalty flag). It is a **projection of identity into commerce**: `identity-service` decides who
exists; this service decides what that person is allowed to be, commercially.

Its two distinguishing rules are:

1. A customer is **created without being usable**. Sign-up produces a customer in `State.Incomplete`
   that publishes **nothing**; only when the person supplies a full name and address does the
   customer become `Valid` and the platform-wide `customer_created` event fire (§3.21, §3.14).
2. **VIP is earned, never granted.** No command sets VIP. It is derived, on every completed order,
   from a hard-coded threshold of 20 distinct completed orders (§3.5).

| Responsibility | Where it lives |
| --- | --- |
| Model a customer as id + email + name + address + state + VIP + completed-order set | `src/Pacco.Services.Customers.Core/Entities/Customer.cs:9-102` |
| Enforce the registration-completion rule (non-blank name & address, must be `Incomplete`) | `…Core/Entities/Customer.cs:44-65` (`CompleteRegistration`) |
| Enforce the state machine and emit a state-change event on every transition | `…Core/Entities/Customer.cs:67-80` |
| Derive VIP status from completed-order count | `…Core/Services/VipPolicy.cs:8-21` |
| Buffer domain events and expose them for publication | `…Core/Entities/AggregateRoot.cs` |
| Accept 2 commands over HTTP **and** the same 2 over AMQP | `…Api/Program.cs:38-41`; `…Infrastructure/Extensions.cs:93-94` |
| Answer 3 queries from a Mongo read model | `…Infrastructure/Mongo/Queries/Handlers/*.cs` |
| React to 2 external events from other services | `…Application/Events/External/{SignedUp,OrderCompleted}.cs` |
| Translate domain events into integration events and publish them | `…Infrastructure/Services/{EventMapper,MessageBroker}.cs` |
| Translate exceptions into HTTP responses **and** into rejected events | `…Infrastructure/Exceptions/ExceptionTo{Response,Message}Mapper.cs` |
| Persist customers in MongoDB (`customers` collection, `customers-service` DB) | `…Infrastructure/Mongo/Repositories/CustomerMongoRepository.cs`; `…Api/appsettings.json:94-98` |
| Publish reliably via a Mongo-backed transactional outbox | `…Infrastructure/Decorators/Outbox*Decorator.cs`; `…Api/appsettings.json:99-107` |
| Gate inbound HTTP on a client certificate ACL granting `availability-service` `customers:read` | `…Infrastructure/Extensions.cs:79,91`; `…Api/appsettings.json:163-182` |
| Register with Consul, emit Jaeger spans, expose Prometheus metrics | `…Infrastructure/Extensions.cs:67-75`; `…Api/appsettings.json:7-22,82-93` |
| Fetch its own secrets and dynamic Mongo credentials from Vault | `…Api/Program.cs:43` (`UseVault`); `…Api/appsettings.json:183-212` |

### 1.2 What this component explicitly is **not**

- **Not an authenticator.** There is **no JWT validation anywhere**. Grepping `src/` for `AddJwt`,
  `UseAuthentication`, `UseAuthorization` and `[Authorize]` returns nothing; the only security
  registration is `AddCertificateAuthentication()` / `UseCertificateAuthentication()`
  (`…Infrastructure/Extensions.cs:79,91`). The `jwt` block in `…Api/appsettings.json` is **inert
  configuration no code reads** (§3.26). Identity arrives as a JSON header manufactured by the
  gateway (`component-internals/api-gateway.md` §3.14) and is **believed unconditionally**.
- **Not an authorization engine.** No handler in this service inspects `IAppContext` or
  `IIdentityContext`. `AppContextFactory` is registered (`…Infrastructure/Extensions.cs:57-58`) and
  an `IAppContext` is constructed per request, but **no command handler, event handler or query
  handler injects it** — the entire caller-identity apparatus is dead weight inside this service
  (§3.19). Who may change a customer's state is decided *only* at the gateway
  (`ntrada.yml`, `claims: role: admin`).
- **Not the owner of user accounts or credentials.** It stores an `Email` copied from
  `SignedUp` (`…Application/Events/External/Handlers/SignedUpHandler.cs:39`) and never mutates it.
  Passwords, tokens and roles live in `identity-service`.
- **Not the source of truth for orders.** `CompletedOrders` is an append-only set of order ids fed
  by `OrderCompleted` (§3.6); it is a counter substrate for the VIP rule, not an order history.
- **Not a deleter.** `ICustomerRepository` exposes **`GetAsync` / `AddAsync` / `UpdateAsync` only**
  (`…Infrastructure/Mongo/Repositories/CustomerMongoRepository.cs:19-27`). There is **no
  `DeleteAsync`, no soft-delete flag, and no GDPR erase path** in this service. A customer document,
  once created, is immortal.
- **Not concurrency-safe.** `AggregateRoot.Version` exists (`…Core/Entities/AggregateRoot.cs:10`)
  but is **never incremented and never used in a filter**. Every write is a whole-document
  last-writer-wins replace (§3.11).
- **Not tested.** There is no test project in the repository. Contrast `availability-service`, which
  ships five (`component-internals/availability-service.md` §3.36). Every invariant in this document
  is enforced only by the code path that raises it — nothing pins it.
- **Not a saga coordinator.** It forwards a `Saga` header when publishing
  (`…Infrastructure/Extensions.cs:106-120`) but defines no saga.

### 1.3 The dual-transport boundary

Both write commands have **two entry points with different failure semantics**. This is the single
most important thing to hold in mind while reading the rest of the document:

| | HTTP path | AMQP path |
| --- | --- | --- |
| Entry | `UseDispatcherEndpoints` (`…Api/Program.cs:33-41`) | `SubscribeCommand<T>()` (`…Infrastructure/Extensions.cs:93-94`) |
| Identity source | `Correlation-Context` header (`…Infrastructure/Extensions.cs:101-104`) | broker message context (`AppContextFactory.cs:19-33`) |
| De-duplication | **none effective** — a fresh `MessageId` GUID per request (§3.17) | inbox, keyed on the broker `MessageId` |
| Failure surfaced as | HTTP **400**, always (§3.18) | a **rejected event** published to the broker (§3.16) |
| Failure when unmapped | generic 400 `{code:"error"}` | **nothing at all** — silent drop |
| Caller learns outcome | synchronously | only by polling `operations-service` |

Two concrete asymmetries specific to this service, both derived in §3.16:

- `InvalidRoleException` and `CustomerAlreadyCreatedException` — the only two failures the
  `SignedUp` handler can raise — are **not in `ExceptionToMessageMapper` at all**. A non-`user`
  sign-up therefore fails **silently** on the only transport that can trigger it.
- `CustomerNotFoundException` raised while handling `ChangeCustomerState` is mapped to a
  **`CompleteCustomerRegistrationRejected`** event — the wrong rejection type for the command that
  failed (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:25`).

### 1.4 Position in the platform

| Direction | Counterpart | Mechanism | Evidence |
| --- | --- | --- | --- |
| Inbound (sync) | `api-gateway` | HTTP, 6 routes, admin-gated except `GET /customers/me` and `POST /customers` | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` |
| Inbound (async) | `api-gateway` async profile | AMQP to exchange `customers`, routing keys `complete_customer_registration`, `change_customer_state` | `…/ntrada-async.yml` |
| Inbound (sync) | `availability-service` | `GET /customers/{id}/state` with a Vault PKI client certificate header | `hianshul100_Pacco.Services.Availability/…/Clients/CustomersServiceClient.cs` |
| Inbound (sync) | `pricing-service` | `GET /customers/{id}` — **without** a certificate header | `hianshul100_Pacco.Services.Pricing/…/Clients/CustomersServiceClient.cs` |
| Inbound (async) | `identity-service` | event `SignedUp` on exchange `identity` | `…Application/Events/External/SignedUp.cs:7` |
| Inbound (async) | `orders-service` | event `OrderCompleted` on exchange `orders` | `…Application/Events/External/OrderCompleted.cs:7` |
| Outbound (async) | `availability-service`, `orders-service`, `parcels-service` | event `customer_created` | consumer handlers in those three repos |
| Outbound (async) | **nobody** | events `customer_state_changed`, `customer_became_vip` | no `CustomerStateChanged` / `CustomerBecameVip` handler exists in any clone (§3.14) |

---

## 2. Core concepts (exhaustive)

Every distinct internal mechanism this service implements. The Owner column names the file that
*defines* the concept.

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | `Customer` — the aggregate root | `…Core/Entities/Customer.cs` | §3.1 |
| 2 | `AggregateRoot` — the event buffer and the unused `Version` | `…Core/Entities/AggregateRoot.cs` | §3.2 |
| 3 | `AggregateId` — the id value object | `…Core/Entities/AggregateId.cs` | §3.3 |
| 4 | `State` — the customer state machine | `…Core/Entities/State.cs` | §3.4 |
| 5 | VIP status and `VipPolicy` — the threshold of 20 | `…Core/Services/VipPolicy.cs` | §3.5 |
| 6 | `CompletedOrders` — the de-duplicating order set | `…Core/Entities/Customer.cs:11,93-101` | §3.6 |
| 7 | Domain events — the three internal signals | `…Core/Events/*.cs` | §3.7 |
| 8 | Exception hierarchy and error codes | `…Core/Exceptions`, `…Application/Exceptions` | §3.8 |
| 9 | `CustomerDocument` and the three mapping functions | `…Infrastructure/Mongo/Documents/*.cs` | §3.9 |
| 10 | Enum-as-integer persistence vs lowercase-string transport | `Documents/CustomerDocument.cs:15`, `Documents/Extensions.cs:29,41` | §3.10 |
| 11 | Absence of optimistic concurrency — last-writer-wins | `…Mongo/Repositories/CustomerMongoRepository.cs` | §3.11 |
| 12 | Commands and command handlers | `…Application/Commands/**` | §3.12 |
| 13 | Queries and the read model (incl. the unbounded scan) | `…Infrastructure/Mongo/Queries/Handlers/*.cs` | §3.13 |
| 14 | Integration events and `EventMapper` (null = silent skip) | `…Infrastructure/Services/EventMapper.cs` | §3.14 |
| 15 | `MessageBroker` — the publish pipeline | `…Infrastructure/Services/MessageBroker.cs` | §3.15 |
| 16 | Rejected events and `ExceptionToMessageMapper` | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` | §3.16 |
| 17 | Transactional outbox and inbox | `…Infrastructure/Decorators/Outbox*.cs` | §3.17 |
| 18 | `ExceptionToResponseMapper` — everything is HTTP 400 | `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` | §3.18 |
| 19 | `IAppContext` / `IIdentityContext` — built, never consumed | `…Infrastructure/Contexts/*.cs` | §3.19 |
| 20 | `Correlation-Context` header ingestion | `…Infrastructure/Extensions.cs:101-104` | §3.20 |
| 21 | External event subscription — `SignedUp`, `OrderCompleted` | `…Application/Events/External/Handlers/*.cs` | §3.21 |
| 22 | Queue naming and message conventions | `…Api/appsettings.json:108-149` | §3.22 |
| 23 | `[Contract]` and `UsePublicContracts` | `…Application/Commands/*.cs`; `Extensions.cs:89` | §3.23 |
| 24 | Dispatcher-bound HTTP endpoints | `…Api/Program.cs:33-41` | §3.24 |
| 25 | Certificate authentication and the `customers:read` ACL | `…Api/appsettings.json:163-182` | §3.25 |
| 26 | Inert JWT configuration | `…Api/appsettings.json:77-81` | §3.26 |
| 27 | `MessageToLogTemplateMapper` — per-message log phrasing | `…Infrastructure/Logging/MessageToLogTemplateMapper.cs` | §3.27 |
| 28 | Log redaction and path exclusion | `…Api/appsettings.json:32-68` | §3.28 |
| 29 | Consul registration and Fabio addressing | `…Api/appsettings.json:7-31` | §3.29 |
| 30 | Vault — KV settings, PKI certificate, dynamic Mongo credentials | `…Api/appsettings.json:183-212` | §3.30 |
| 31 | Environment layering — `appsettings.{local,docker}.json` | `…Api/appsettings.*.json` | §3.31 |
| 32 | Mongo collection naming and the `"customers"` literal | `…Infrastructure/Extensions.cs:77` | §3.32 |
| 33 | Redis — configured with no consumer | `…Infrastructure/Extensions.cs:73`; `appsettings.json:150-153` | §3.33 |
| 34 | `IDateTimeProvider` — the only clock seam | `…Infrastructure/Services/DateTimeProvider.cs` | §3.34 |
| 35 | Metrics and tracing wiring | `…Infrastructure/Extensions.cs:69,74-75` | §3.35 |
| 36 | Absence of a test suite | repository layout | §3.36 |
| 37 | Deployment identity — ports, images, process names | `Dockerfile`, `scripts/`, `hianshul100_Pacco/compose` | §3.37 |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement**, **Extension procedure**, **Failure modes**.

### 3.1 `Customer` — the aggregate root

**Definition.** The single aggregate of this service. A `Customer` is a person's *commercial*
record: identity id, contact email, postal identity (`FullName`, `Address`), a lifecycle `State`, a
loyalty flag `IsVip`, a creation timestamp, and the set of orders they have completed
(`…Core/Entities/Customer.cs:9-24`).

**Representation & storage.** In memory: a class deriving `AggregateRoot`, all setters `private`, so
**every mutation must go through a named method**. `_completedOrders` is a `HashSet<Guid>` behind an
`IEnumerable<Guid>` property whose setter re-wraps any assigned sequence into a fresh `HashSet`
(`Customer.cs:11,20-24`) — that setter is the rehydration seam used by
`Documents/Extensions.cs:8-10`. On disk: one `CustomerDocument` per customer in the `customers`
collection of the `customers-service` database (§3.9, §3.32). The Mongo `_id` is the customer's
`Guid`, which **is the `identity-service` user id** — there is no separate customer id anywhere
(`SignedUpHandler.cs:39` passes `@event.UserId` straight into the constructor).

**Lifecycle.**

| Phase | Trigger | Path | Registration vs implicit |
| --- | --- | --- | --- |
| Created | `SignedUp` event from `identity-service` | `SignedUpHandler.HandleAsync` → `new Customer(UserId, Email, Now)` → `AddAsync` (`SignedUpHandler.cs:39-40`) | **Implicit** — no command, no HTTP route, no operator action. The only creation path. |
| Completed | `CompleteCustomerRegistration` (HTTP `POST /customers` or AMQP) | `CompleteCustomerRegistrationHandler.cs:26-44` → `Customer.CompleteRegistration` | Explicit command |
| State-changed | `ChangeCustomerState` (HTTP `PUT` or AMQP) | `ChangeCustomerStateHandler.cs:26-65` | Explicit command, admin-gated at the gateway only |
| VIP-promoted | `OrderCompleted` event | `OrderCompletedHandler.cs:36-37` → `VipPolicy.ApplyVipStatusIfEligible` | **Implicit** — derived, never commanded |
| Read | 3 queries | `…Mongo/Queries/Handlers/*.cs` — bypass the aggregate entirely, project the document | Read model |
| Versioned | — | **never** (§3.11) | — |
| Deleted | — | **no path exists** (§1.2) | — |

The two-constructor shape is load-bearing. `Customer(Guid, string, DateTime)` (`Customer.cs:26-29`)
delegates to the full constructor with `fullName = string.Empty`, `address = string.Empty`,
`isVip = false`, `state = State.Incomplete`, `completedOrders = empty`. **This is the only place
`State.Incomplete` is chosen as a starting state**, and it is chosen in code, not in configuration.
The full constructor (`Customer.cs:31-42`) is the *rehydration* constructor and applies **no
validation at all** — it will happily reconstruct a customer with a null email or `State.Unknown` if
that is what the database holds.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| A registration cannot complete with a blank full name | `Customer.cs:46-49` → `InvalidCustomerFullNameException` | **Loud** — HTTP 400 `invalid_customer_fullname`; AMQP `complete_customer_registration_rejected` |
| A registration cannot complete with a blank address | `Customer.cs:51-54` → `InvalidCustomerAddressException` | **Loud** — HTTP 400 `invalid_customer_address` |
| A registration can only complete from `State.Incomplete` | `Customer.cs:56-59` → `CannotChangeCustomerStateException` | **Loud** |
| A customer exists before any command touches it | `CompleteCustomerRegistrationHandler.cs:29-32`, `ChangeCustomerStateHandler.cs:29-32` → `CustomerNotFoundException` | **Loud** on HTTP; **mis-typed** on AMQP (§3.16) |
| A customer is created at most once per user id | `SignedUpHandler.cs:33-37` → `CustomerAlreadyCreatedException` | **Silent on AMQP** — unmapped (§3.16). This is the *only* transport this check runs on. |
| Only `role == "user"` sign-ups become customers | `SignedUpHandler.cs:13,28-31` → `InvalidRoleException` | **Silent on AMQP** — unmapped |
| `CompletedOrders` never contains `Guid.Empty` | `Customer.cs:95-98` — early `return` | **Silent** — the order is dropped with no log and no event |
| VIP is monotonic (never revoked) | `Customer.cs:82-91` — `SetVip` returns early if already VIP; no `UnsetVip` exists | Structural |

Note the asymmetry: **rules that protect the write model are loud; rules that protect data quality
are silent.** A caller who sends `orderId = Guid.Empty` in an `OrderCompleted` event gets a 100 %
successful handler run that changed nothing.

**Extension procedure.** To add a field to `Customer`:

1. Add the property with a `private set` in `Customer.cs`, and a named mutator that raises a domain
   event if the change is externally visible.
2. Add the parameter to the **full** constructor (`Customer.cs:31-42`) — the rehydration path — and
   decide the default for the short constructor (`Customer.cs:26-29`).
3. Add the field to `CustomerDocument` (`…Mongo/Documents/CustomerDocument.cs`).
4. Add it to **all three** mapping functions in `…Mongo/Documents/Extensions.cs`: `AsEntity`,
   `AsDocument`, and whichever DTO projections should expose it (`AsDto`, `AsDetailsDto`).
5. If it should be published, add a case to `EventMapper.Map` (§3.14) **and** a message to
   `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` so the
   operations UI can name it.

**What silently rejects the extension:** step 4 is the trap. `AsEntity` and `AsDocument` are
hand-written positional/object-initializer mappings with **no compiler pressure** — omitting a field
compiles cleanly and silently drops the value on the next write. There is no test suite to catch it
(§3.36). Adding a domain event without step 5's `EventMapper` case means `Map` returns `null` and
`MessageBroker` skips it silently (§3.14).

**Failure modes.**

- A `SignedUp` event arriving with `Role != "user"` throws on every delivery, is retried by the
  broker, and never produces a rejection message — the failure is visible only in logs.
- Because the full constructor validates nothing, a document written by an older or buggier version
  of the service is rehydrated as-is; `State` deserialized from an out-of-range integer becomes an
  undefined enum value that no `switch` case in `ChangeCustomerStateHandler.cs:44-60` matches.
- Two concurrent writes to the same customer (e.g. an `OrderCompleted` and a `ChangeCustomerState`)
  overwrite each other wholesale (§3.11).

### 3.2 `AggregateRoot` — the event buffer and the unused `Version`

**Definition.** The base class every aggregate in this service (there is exactly one) derives from.
It provides an in-memory list of domain events raised during the current unit of work, plus an `Id`
and a `Version` (`…Core/Entities/AggregateRoot.cs:5-18`).

**Representation & storage.** `private readonly List<IDomainEvent> _events`, exposed as
`IEnumerable<IDomainEvent> Events`; `AggregateId Id { get; protected set; }`;
`int Version { get; protected set; }`. **Nothing here is persisted.** `CustomerDocument` has no
`Version` field and no events array (`…Mongo/Documents/CustomerDocument.cs:10-17`). The event buffer
lives only for the duration of a handler call.

**Lifecycle.** `AddEvent` is `protected` and called only from inside `Customer`
(`Customer.cs:64,79,90`). `Events` is read by each command/event handler immediately after the
repository write — `_eventMapper.MapAll(customer.Events)` — and then handed to the broker
(`CompleteCustomerRegistrationHandler.cs:42-43`, `ChangeCustomerStateHandler.cs:63-64`,
`OrderCompletedHandler.cs:40-41`). `ClearEvents()` exists (`AggregateRoot.cs:17`) but **has no
caller in this repository**; it does not need one, because the aggregate instance is loaded fresh
from Mongo on every handler invocation and discarded afterwards.

**Invariants & enforcement.** None are enforced here. The class is a plain buffer. Two properties of
the design are worth naming because they are invariants *by construction*:

- Events are ordered by insertion, so a handler that performs two mutations publishes them in the
  order they occurred.
- Because the aggregate is never cached between requests, the buffer can never leak events from a
  previous unit of work — which is why the absent `ClearEvents()` call is harmless *today* and would
  become a live bug the moment any caching or in-process reuse is introduced.

**`Version` is dead.** It is declared, never assigned (no `Version++`, no `IncrementVersion`
anywhere in `src/`), never written to the document, and never used in a Mongo filter. Compare
`deliveries-service`, which adds an `IncrementVersion()` method that is *also* never called
(`component-internals/deliveries-service.md` §3.2), and `availability-service`, which actually uses
its version for optimistic concurrency
(`component-internals/availability-service.md` §3.11). Customers is the weakest of the three.

**Extension procedure.** To make domain events durable (e.g. for an event store), you would add
persistence in the repository (`CustomerMongoRepository.UpdateAsync`) *and* start calling
`ClearEvents()` after publication; today, publishing and persistence are two independent statements
in the handler, and nothing binds them (§3.15 failure modes).

**Failure modes.** If a handler mutates the aggregate twice and publishes once (the normal shape),
all buffered events go out together. If a future handler forgets `MapAll(customer.Events)`, the
mutation persists and **no event is ever published** — there is no framework-level "unpublished
events" check.

### 3.3 `AggregateId`

**Definition.** A value object wrapping a `Guid`, with implicit conversions to and from `Guid` and
value equality (`…Core/Entities/AggregateId.cs`).

**Representation & storage.** Never persisted as itself: `Customer.Id` is an `AggregateId`, and
`AsDocument` assigns it to a `Guid` field via the implicit conversion
(`…Mongo/Documents/Extensions.cs:15`). The document's `Id` is a plain `Guid`
(`CustomerDocument.cs:10`).

**Lifecycle.** Constructed either parameterlessly (new `Guid.NewGuid()`) or from a supplied `Guid`,
which rejects `Guid.Empty` with `InvalidAggregateIdException`
(code `invalid_aggregate_id`, `…Core/Exceptions/InvalidAggregateIdException.cs:5`).

**Invariants & enforcement.** `Guid.Empty` is not a valid aggregate id — **loud**, a domain
exception. Note where this *does not* fire: `CompleteCustomerRegistration` and `ChangeCustomerState`
carry a `Guid CustomerId` that is used as a lookup key, not to construct an id, so an empty GUID in
a command produces `CustomerNotFoundException`, not `InvalidAggregateIdException`.

**Extension procedure.** Nothing to extend; if a second aggregate is ever added it should reuse this
type rather than a bare `Guid`, to inherit the empty-guid guard.

**Failure modes.** The guard only protects the *construction* path, which in this service is reached
exactly once — `new Customer(@event.UserId, …)` in `SignedUpHandler.cs:39`. A `SignedUp` event with
an empty `UserId` therefore fails loudly at construction… and then, because
`InvalidAggregateIdException` is **not** in `ExceptionToMessageMapper` (§3.16), fails **silently**
from the publisher's point of view.

### 3.4 `State` — the customer state machine

**Definition.** A five-member enum describing what the platform will let a customer do
(`…Core/Entities/State.cs:3-10`):

```csharp
public enum State { Unknown, Valid, Incomplete, Suspicious, Locked }
```

**Representation & storage.** Three different representations, and knowing which is which prevents a
whole class of bug:

| Representation | Where | Evidence |
| --- | --- | --- |
| The **integer** ordinal (`Unknown = 0`, `Valid = 1`, `Incomplete = 2`, `Suspicious = 3`, `Locked = 4`) | MongoDB — `CustomerDocument.State` is typed `State`, not `string` | `…Mongo/Documents/CustomerDocument.cs:15` |
| A **lowercase string** | every outbound DTO and every published event | `Documents/Extensions.cs:29,41`; `GetCustomerStateHandler.cs:29`; `EventMapper.cs:22-23` |
| A **case-insensitively parsed string** | inbound, on `ChangeCustomerState.State` | `ChangeCustomerStateHandler.cs:34` — `Enum.TryParse<State>(command.State, true, out var state)` |

`Unknown = 0` is the **default**, which matters: a document field that is missing or null on read
deserializes to `Unknown`, not to `Incomplete`.

**Lifecycle.**

- **Entry:** every customer starts at `Incomplete` (`Customer.cs:26-29`).
- **`Incomplete → Valid`** happens on `CompleteRegistration` (`Customer.cs:63`) — the only
  business-meaningful transition, and the only one that emits `CustomerRegistrationCompleted`.
- **Any → any** happens through `ChangeCustomerState`, which routes to `SetValid`, `SetIncomplete`,
  `MarkAsSuspicious` or `Lock` (`ChangeCustomerStateHandler.cs:44-60`). All four funnel into the
  private `SetState` (`Customer.cs:75-80`), which records `previousState`, assigns, and emits
  `CustomerStateChanged`.
- **No transition is forbidden.** There is no transition table. `Locked → Valid`,
  `Valid → Incomplete`, `Suspicious → Valid` are all permitted. The state machine is a *set of
  labels with an audit event*, not a graph with edges.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| The requested state string must name a member of `State` | `ChangeCustomerStateHandler.cs:34-37` → `CannotChangeCustomerStateException(customer.Id, State.Unknown)` | **Loud** |
| A no-op transition does nothing | `ChangeCustomerStateHandler.cs:39-42` — `if (customer.State == state) return;` | **Silent** — see below |
| `Unknown` cannot be reached through the switch | `ChangeCustomerStateHandler.cs:58-59` `default:` → `CannotChangeCustomerStateException` | **Loud**, but only reachable for the literal input `"unknown"` |
| Registration completion requires `Incomplete` | `Customer.cs:56-59` | **Loud** |

Two of these deserve emphasis:

1. **The silent no-op.** `PUT /customers/{id}/state/valid` on a customer already `Valid` returns
   **HTTP 204 No Content** (`…Api/Program.cs:40-41` sets `NoContent` unconditionally in
   `afterDispatch`), writes nothing, and publishes **no** `customer_state_changed`. The caller
   cannot distinguish "I changed it" from "it was already that". Over AMQP the effect is identical
   and even less observable: no event of any kind is emitted, so a saga waiting for
   `customer_state_changed` will wait forever.
2. **The `"unknown"` string is accepted by the parser.** `Enum.TryParse` with `ignoreCase: true`
   parses `"unknown"` to `State.Unknown` successfully, so it passes line 34, passes the equality
   check on line 39 (unless the customer is already `Unknown`), and reaches the `default:` case,
   which throws `CannotChangeCustomerStateException`. The `default:` arm is therefore **not dead
   code** — it is the guard for exactly one input value. Note the ordering hazard: a customer whose
   stored state *is* `Unknown` (see §3.10) and who receives `"unknown"` returns silently at line 41
   instead.

**Extension procedure.** To add a state — say `Banned`:

1. Append it to the enum **at the end** (`State.cs`). *Do not insert it in the middle*: the stored
   representation is the ordinal (§3.10), so inserting shifts the meaning of every existing document.
2. Add a mutator on `Customer` (`public void Ban() => SetState(State.Banned);`).
3. Add a `case State.Banned:` to `ChangeCustomerStateHandler.cs:44-60` — **omitting this is the
   silent rejection**: `Enum.TryParse` will accept `"banned"`, the equality check will pass, and the
   `default:` arm will throw `CannotChangeCustomerStateException`, i.e. the new state looks
   *invalid* rather than *unimplemented*.
4. Decide whether downstream services need to know. `customer_state_changed` already carries the
   state as a lowercase string, so no contract change is needed — but see §3.14: **no service in
   the workspace consumes that event**, so "downstream reacts to the new state" is not currently a
   thing that happens anywhere.
5. Consider `availability-service`'s reading of state: it calls `GET /customers/{id}/state` and
   compares against its own expectations
   (`component-internals/availability-service.md` §3.26) — a new state string may be treated as
   "not valid" there without any code change, which is usually the desired default.

**Failure modes.**

- Reordering the enum silently rewrites history for every stored document.
- A client sending `"Valid"`, `"VALID"` or `"valid"` all succeed (case-insensitive parse); a client
  sending `"active"` gets HTTP 400 `cannot_change_customer_state` — a code that names the *outcome*
  but not the *cause*, since the same code is used for the "wrong current state" failure.
- The `ChangeCustomerStateRejected` event carries `ex.State.ToString().ToLowerInvariant()`
  (`ExceptionToMessageMapper.cs:17-18`), which for a parse failure is always `"unknown"` — the
  rejection tells the consumer the *attempted* state was unknown, never what was actually sent.

### 3.5 VIP status and `VipPolicy` — the threshold of 20

**Definition.** `IsVip` is a boolean on the aggregate (`Customer.cs:16`) that no command can set.
`VipPolicy` is a domain service (`…Core/Services/VipPolicy.cs:6-22`) implementing the sole rule that
grants it.

**Representation & storage.** `bool IsVip` on the entity, `bool IsVip` on the document
(`CustomerDocument.cs:14`), surfaced only on `CustomerDetailsDto`
(`Documents/Extensions.cs:40`) — i.e. `GET /customers/{customerId}`. It is **not** on `CustomerDto`
(the list projection) and **not** on `CustomerStateDto`. A caller listing customers cannot see who
is a VIP.

**The rule, verbatim** (`VipPolicy.cs:8-21`):

```csharp
public void ApplyVipStatusIfEligible(Customer customer)
{
    if (customer.IsVip) return;
    if (customer.CompletedOrders.Count() < 20) return;
    customer.SetVip();
}
```

Three properties follow directly:

- **The threshold is the literal `20`, hard-coded in `Core`.** It is not in `appsettings.json`, not
  in Vault KV, and not behind an options class. Changing it is a code change and a redeploy.
- **It is `< 20` on `Count()`, i.e. 20 *distinct* completed orders**, because `CompletedOrders` is a
  `HashSet` (§3.6). Twenty deliveries of the same order id promote nobody.
- **`Count()` is `System.Linq.Enumerable.Count()` over an `IEnumerable<Guid>`** (note the `using
  System.Linq` at `VipPolicy.cs:1`) — it enumerates the whole set on every completed order. At
  realistic customer sizes this is irrelevant; it is called out only because the set is unbounded
  and grows forever (§3.6).

**Lifecycle.** `VipPolicy` is registered as a **singleton**
(`…Infrastructure/Extensions.cs:56` — `AddSingleton<IVipPolicy, VipPolicy>()`), which is safe
because it is stateless. It is invoked from exactly one place: `OrderCompletedHandler.cs:37`,
immediately after `AddCompletedOrder`. `SetVip` (`Customer.cs:82-91`) is idempotent — it returns
early if already VIP — so `CustomerBecameVip` is emitted **at most once per customer, ever**.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| VIP is granted only by the policy | No command or endpoint calls `SetVip`; grep for `SetVip` yields `Customer.cs:82` and `VipPolicy.cs:20` only | Structural |
| VIP is never revoked | No `UnsetVip`/`RemoveVip` member exists | Structural |
| `customer_became_vip` fires exactly once | `Customer.cs:84-87` early return | **Silent** by design (a second call is a no-op) |
| The 20th *distinct* order triggers promotion | `HashSet` semantics + `Count() < 20` | Silent |

**A dead local worth knowing about.** `OrderCompletedHandler.cs:38` computes
`var vipApplied = !isVip && customer.IsVip;` and **never uses it** — `isVip` is captured on line 35
purely to feed this expression. The variable is a vestige of a conditional-publish design that was
not completed. It is harmless (the events list already contains `CustomerBecameVip` only when the
promotion actually happened, because `SetVip` guards itself), but it is the clearest signal in the
repository that the publish path was intended to be conditional and is not.

**Extension procedure.** To make the threshold configurable:

1. Add an options class in `Application` (Core references nothing, so it cannot read configuration —
   `…Core/Pacco.Services.Customers.Core.csproj` has no package or project references beyond the
   framework).
2. Either move `VipPolicy` to `Application`/`Infrastructure`, or inject a small
   `IVipThresholdProvider` into it from `Core`.
3. Bind the options in `…Infrastructure/Extensions.cs` alongside the other registrations and add the
   key to `…Api/appsettings.json` **and** to the Vault KV payload if it should be environment-
   specific (§3.30).

**What silently rejects the extension:** adding a `vip` section to `appsettings.json` alone does
nothing — no code reads it. There is no configuration-binding validation in this service, so a typo
in a config key never surfaces.

To add a second promotion rule (e.g. total spend), add it to `ApplyVipStatusIfEligible`; do **not**
add a second `IVipPolicy` implementation, because DI registers exactly one and the last
registration would win silently.

**Failure modes.**

- Because promotion happens inside the `OrderCompleted` handler and the handler publishes
  `customer.Events` unconditionally, a customer who is promoted **and** has no other pending event
  publishes exactly `customer_became_vip` — which, per §3.14, **no service in the workspace
  consumes**. VIP status is therefore effectively write-only outside this service: the only way to
  observe it is `GET /customers/{customerId}`, which the gateway gates behind `role: admin`.
- If an `OrderCompleted` event is redelivered with the same order id, `HashSet` dedupes and the count
  does not advance — correct behaviour, achieved by data structure rather than by the inbox.
- If `OrderCompleted` arrives with `OrderId == Guid.Empty`, `AddCompletedOrder` silently drops it
  (`Customer.cs:95-98`) and the count never advances for that order. No warning is logged.

### 3.6 `CompletedOrders` — the de-duplicating order set

**Definition.** The set of order ids this customer has completed; the only input to the VIP rule.

**Representation & storage.** `private ISet<Guid> _completedOrders = new HashSet<Guid>()`
(`Customer.cs:11`) exposed as `IEnumerable<Guid> CompletedOrders` whose setter re-wraps into a new
`HashSet` (`Customer.cs:20-24`). Persisted as `IEnumerable<Guid> CompletedOrders` on the document
(`CustomerDocument.cs:17`) — i.e. **a BSON array of GUIDs embedded in the customer document**, with
no separate collection and no cap.

It is exposed on `CustomerDetailsDto` in full (`Documents/Extensions.cs:43`), so
`GET /customers/{customerId}` returns **every order id the customer has ever completed** —
an unbounded array in a synchronous response.

**Lifecycle.** Appended only, by `Customer.AddCompletedOrder(Guid)` (`Customer.cs:93-101`), called
only from `OrderCompletedHandler.cs:36`. Never removed, never trimmed, never archived. Rehydrated
through the property setter on `AsEntity` (`Documents/Extensions.cs:9-10`), which passes
`document.CompletedOrders` — if that field is absent in the stored document the value is `null`, and
the *entity* constructor coalesces it (`Customer.cs:40` — `completedOrders ?? Enumerable.Empty<Guid>()`).
This null-guard is the reason a legacy document without the field does not crash on read.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| No duplicate order ids | `HashSet<Guid>` (`Customer.cs:11`) | **Silent** — `Add` returns `false`, which is discarded |
| No empty GUIDs | `Customer.cs:95-98` early return | **Silent** |
| Null on rehydration becomes an empty set | `Customer.cs:40` | Silent, and correct |

**Extension procedure.** If completed-order data ever needs to carry more than an id (a timestamp,
an amount), the type changes from `Guid` to a value object — which changes the BSON array element
shape and therefore requires a migration (§5), because there is **no migration framework in this
repository or in any sibling service** (`patterns/index.md`).

**Failure modes.**

- **Unbounded growth.** A high-volume customer's document grows without limit inside a single BSON
  document, which has a hard 16 MB ceiling in MongoDB. At 16 bytes per GUID plus BSON array
  overhead, this is far from an operational concern at Pacco's scale, but it is a real ceiling with
  no code that watches for it. `Unverifiable — Missing Source Evidence`: no monitoring or alerting
  on document size exists in this repository.
- **Whole-document rewrite on every completed order.** `UpdateAsync` replaces the entire document
  (§3.11), so each completed order rewrites the whole array.
- **Privacy surface.** `GET /customers/{customerId}` discloses the full order-id history; the
  gateway restricts that route to `role: admin`, but `pricing-service` calls the *same route*
  service-to-service without a certificate header (§3.25), so the restriction is a gateway policy,
  not a service-enforced one.

### 3.7 Domain events — the three internal signals

**Definition.** In-process notifications raised by the aggregate, implementing the service-local
`IDomainEvent` marker (`…Core/IDomainEvent.cs`). They are **not** the messages that leave the
service; §3.14 covers that translation.

| Domain event | Raised by | Payload | Evidence |
| --- | --- | --- | --- |
| `CustomerRegistrationCompleted` | `Customer.CompleteRegistration` | the `Customer` | `…Core/Events/CustomerRegistrationCompleted.cs`; raised at `Customer.cs:64` |
| `CustomerStateChanged` | `Customer.SetState` (all four public mutators) | the `Customer` **and** `PreviousState` | `…Core/Events/CustomerStateChanged.cs:5-15`; raised at `Customer.cs:79` |
| `CustomerBecameVip` | `Customer.SetVip` | the `Customer` | `…Core/Events/CustomerBecameVip.cs`; raised at `Customer.cs:90` |

**Representation & storage.** Immutable classes holding a **reference to the live aggregate**, not a
snapshot. This is significant: `EventMapper` reads `e.Customer.State` *after* the handler has
finished mutating (`EventMapper.cs:22-23`), so `CustomerStateChanged` reports the state as of
*mapping time*, not as of raise time. With one mutation per handler these coincide; with two
mutations in one handler they would not.

`PreviousState` is the exception — it is captured by value at raise time
(`Customer.cs:77` — `var previousState = State;`), so it is genuinely historical.

**Lifecycle.** Raised → buffered in `AggregateRoot._events` → read once by the handler → mapped →
published → discarded with the aggregate instance. Never persisted, never replayed.

**A missing event, and it is the most important fact in this section.** Customer **creation** raises
**no domain event**. `SignedUpHandler.cs:39-40` constructs a `Customer` and calls `AddAsync`, and
then **the handler ends** — it does not call `MapAll`, does not touch `IMessageBroker`, and does not
even inject one (`SignedUpHandler.cs:14-16` injects only the repository, a clock and a logger).
Consequently:

- The `customer_created` integration event is produced from `CustomerRegistrationCompleted`
  (`EventMapper.cs:19`), i.e. **when registration is completed, not when the customer is created**.
- Between sign-up and registration completion, the customer exists in this service's database and is
  **invisible to every other service on the platform**.

**Invariants & enforcement.** None are enforced on events themselves. The one structural rule is
that only `Customer` can raise them, because `AddEvent` is `protected` (`AggregateRoot.cs:12`).

**Extension procedure.**

1. Add a class in `…Core/Events/` implementing `IDomainEvent`.
2. Raise it from a `Customer` mutator via `AddEvent`.
3. **Add a `case` to `EventMapper.Map`** (`…Infrastructure/Services/EventMapper.cs:17-24`) — without
   this, `Map` falls through to `return null` (line 26) and `MessageBroker` skips nulls silently
   (`MessageBroker.cs:69-72`). The mutation persists; nothing is published; nothing logs a warning.
   **This is the single most likely silent failure when extending this service.**
4. Add the integration event class in `…Application/Events/` (no `[Contract]` needed — that
   attribute is for commands, §3.23).
5. Register the message name in `Operations`' `messages.json` so it appears in the operations UI.

**Failure modes.** A domain event whose payload references the aggregate can observe *later*
mutations (see above). A domain event with no `EventMapper` case is silently swallowed. A domain
event raised but never read — a handler that forgets `customer.Events` — is silently dropped.

### 3.8 Exception hierarchy and error codes

**Definition.** Two parallel base types split by layer:

- `…Core/Exceptions/DomainException` — business-rule violations in `Core`.
- `…Application/Exceptions/AppException` — orchestration failures in `Application`.

Both expose an abstract `Code` used to build machine-readable error payloads.

**The complete inventory** (grep for `public override string Code`):

| Exception | Layer | `Code` | Raised by |
| --- | --- | --- | --- |
| `InvalidAggregateIdException` | Core | `invalid_aggregate_id` | `AggregateId` ctor |
| `CustomerNotFoundException` | Core | `customer_not_found` | `CompleteCustomerRegistrationHandler.cs:31`, `ChangeCustomerStateHandler.cs:31`, `OrderCompletedHandler.cs:32` |
| `CannotChangeCustomerStateException` | Core | `cannot_change_customer_state` | `Customer.cs:58`; `ChangeCustomerStateHandler.cs:36,59` |
| `InvalidCustomerFullNameException` | Core | `invalid_customer_fullname` | `Customer.cs:48` |
| `InvalidCustomerAddressException` | Core | `invalid_customer_address` | `Customer.cs:53` |
| `InvalidRoleException` | Application | `invalid_role` | `SignedUpHandler.cs:30` |
| `CustomerAlreadyCreatedException` | Application | `customer_already_created` | `SignedUpHandler.cs:36` |
| `CustomerAlreadyRegisteredException` | Application | `customer_already_registered` | `CompleteCustomerRegistrationHandler.cs:36` |

Note `invalid_customer_fullname` — **not** `invalid_customer_full_name`. It is a hand-written
literal, and it does not match what the fallback convention would generate.

**Representation & storage.** Codes are hand-written literals on each exception type. The
`ExceptionToResponseMapper` caches resolved codes in a static `ConcurrentDictionary<Type, string>`
(`…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:13,29-32,42`), so the code for a given
exception type is computed once per process.

**The fallback convention.** When an exception has no explicit `Code`, the mapper derives one:
`exception.GetType().Name.Underscore().Replace("_exception", string.Empty)`
(`ExceptionToResponseMapper.cs:39`). Every exception in this service sets `Code` explicitly, so the
fallback is only reachable for a future exception that forgets to.

**Lifecycle.** Thrown in `Core`/`Application` → caught by Convey's error-handling middleware
(`UseErrorHandler`, registered at `…Infrastructure/Extensions.cs:63,85`) on the HTTP path, or by the
RabbitMQ subscriber on the AMQP path → converted to a 400 response (§3.18) or to a rejected event
(§3.16).

**Invariants & enforcement.** There is **no registry** binding an exception to a transport outcome.
Both mappers are hand-written `switch` expressions with `_ => null` / `_ => generic 400` fallbacks
(`ExceptionToMessageMapper.cs:30`, `ExceptionToResponseMapper.cs:22-23`). Adding an exception
without touching both mappers compiles and runs.

**Extension procedure.**

1. Add the exception class deriving `DomainException` (Core rules) or `AppException`
   (orchestration), with an explicit `Code`.
2. Add an arm to `ExceptionToMessageMapper.Map` returning the correct rejected event **for the
   correct command** (§3.16) — otherwise AMQP failures are silent.
3. Decide whether HTTP 400 is right; if not, `ExceptionToResponseMapper` must be extended, and note
   that **it currently has no non-400 arm at all** (§3.18).

**Failure modes.** Two exception classes exist that no mapper handles at all
(`InvalidRoleException`, `CustomerAlreadyCreatedException`), and one — `InvalidAggregateIdException`
— is unreachable in practice. See §3.16 for the full matrix.

### 3.9 `CustomerDocument` and the three mapping functions

**Definition.** The persistence shape and the hand-written translation layer between it and the
domain/DTO shapes.

**Representation & storage** (`…Infrastructure/Mongo/Documents/CustomerDocument.cs:8-18`):

| Field | Type | Notes |
| --- | --- | --- |
| `Id` | `Guid` | Mongo `_id`; equals the `identity-service` user id |
| `Email` | `string` | copied from `SignedUp`, never updated afterwards |
| `FullName` | `string` | `string.Empty` until registration completes |
| `Address` | `string` | `string.Empty` until registration completes |
| `IsVip` | `bool` | |
| `State` | `State` (**enum, stored as its integer ordinal**) | §3.10 |
| `CreatedAt` | `DateTime` | from `IDateTimeProvider.Now` (§3.34) |
| `CompletedOrders` | `IEnumerable<Guid>` | embedded BSON array (§3.6) |

The class implements `Convey.Types.IIdentifiable<Guid>` (`CustomerDocument.cs:8`), which is what
`AddMongoRepository<CustomerDocument, Guid>("customers")` requires `[convey]`.

**The three mapping functions** (`…Infrastructure/Mongo/Documents/Extensions.cs`):

| Function | Direction | Lines | Fields carried |
| --- | --- | --- | --- |
| `AsEntity` | document → `Customer` | `:8-10` | all 8, via the **rehydration constructor** (no validation) |
| `AsDocument` | `Customer` → document | `:12-23` | all 8 |
| `AsDto` | document → `CustomerDto` | `:25-31` | `Id`, `State` (lowercased string), `CreatedAt` — **3 of 8** |
| `AsDetailsDto` | document → `CustomerDetailsDto` | `:33-44` | all 8 (`State` lowercased) |

Two observations that matter when changing anything here:

- **The read path never constructs a `Customer`.** The three query handlers take
  `IMongoRepository<CustomerDocument, Guid>` directly and project documents to DTOs
  (`…Mongo/Queries/Handlers/GetCustomerHandler.cs:22-24`, `GetCustomersHandler.cs:24-26`,
  `GetCustomerStateHandler.cs:22-30`). Domain invariants are therefore irrelevant to reads — a
  document that could not legally be produced by the write model is still returned happily.
- **`AsEntity` and `AsDocument` are total but unchecked.** They are positional (`AsEntity`) and
  object-initializer (`AsDocument`) mappings; the compiler will not tell you a field is missing.

**Lifecycle.** `AsDocument` runs on every `AddAsync`/`UpdateAsync`
(`CustomerMongoRepository.cs:26-27`); `AsEntity` runs on every `GetAsync`
(`CustomerMongoRepository.cs:21-23`) and is null-safe via `customer?.AsEntity()`.

**Invariants & enforcement.** The only enforced one is the `null` short-circuit on read. Everything
else is convention.

**Extension procedure.** See §3.1 step 3–4 and §5.5. The ordering rule: add to the document first,
then to `AsDocument` (so new writes carry it), then to `AsEntity` (so reads see it), then to DTOs.

**Failure modes.** A field added to the entity but not to `AsDocument` is silently lost on the next
write — and because writes are whole-document replaces (§3.11), "the next write" destroys the
previously stored value too.

### 3.10 Enum-as-integer persistence vs lowercase-string transport

**Definition.** The most consequential schema decision in this service: `State` crosses the storage
boundary as an **integer** and every other boundary as a **lowercase string**.

**Evidence.**

- Storage: `public State State { get; set; }` on the document (`CustomerDocument.cs:15`). There is
  **no** `[BsonRepresentation(BsonType.String)]` attribute anywhere in `src/`, and no custom
  serializer registration in `…Infrastructure/Extensions.cs`. The MongoDB C# driver's default
  serialization for an enum is its underlying integer value. `Unverifiable — Missing Source
  Evidence`: the driver's default is not asserted anywhere in this repository; it is inferred from
  the absence of any override. If a global convention pack were registered by Convey's `AddMongo()`
  `[convey]` it could differ — Convey has no source in this workspace.
- HTTP out: `document.State.ToString().ToLowerInvariant()` on both DTO projections
  (`Documents/Extensions.cs:29,41`) and in the state query handler
  (`GetCustomerStateHandler.cs:29`).
- AMQP out: same lowercasing, applied to both current and previous state
  (`EventMapper.cs:22-23`).
- AMQP/HTTP in: `Enum.TryParse<State>(command.State, true, …)` — a case-insensitive **string**
  parse (`ChangeCustomerStateHandler.cs:34`).

**Lifecycle.** Written as an int on every `UpdateAsync`; read back as an enum; rendered as a
lowercase string on every egress.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| Enum ordinals are stable across deployments | **nothing** — ordinals are implicit | **Silent and catastrophic if broken** |
| An out-of-range stored integer is rejected on read | **nothing** — the driver produces an undefined enum value | **Silent** |
| The wire string is always lowercase | three separate hand-written `.ToLowerInvariant()` calls | Silent (each must be remembered independently) |

**Why this is the widest-blast-radius decision.** Inserting a new member into the middle of
`State.cs` renumbers every subsequent member. Every stored `Valid` customer would read back as
whatever now occupies ordinal 1. There is no migration, no schema version field on the document, and
no test that would catch it. Contrast `availability-service`, whose equivalent hazard is its
days-since-epoch date encoding (`component-internals/availability-service.md` §3.10) — same class of
problem, different field.

**Extension procedure.** Two options:

1. **Append-only enum evolution** (what the code assumes): add members at the end, never reorder,
   never delete. Zero migration cost.
2. **Switch to string persistence**: change `CustomerDocument.State` to `string`, update `AsEntity`
   to parse and `AsDocument` to `.ToString().ToLowerInvariant()`, and **backfill every existing
   document** — an offline script, since there is no migration runner (§5.5). This buys
   reorder-safety and human-readable documents at the cost of a one-off migration.

**What silently rejects the extension:** adding `[BsonRepresentation]` to the document field without
backfilling makes new writes strings and old reads integers; the driver will fail to deserialize the
old ones — loudly at read time, but only for documents nobody has touched since the change, i.e.
your least-active customers.

**Failure modes.**

- Reordering the enum: silent data corruption across the whole collection.
- A document whose `State` is missing (written before the field existed) deserializes to `0` =
  `Unknown`, and `Unknown` is a state no `switch` arm in `ChangeCustomerStateHandler.cs:44-60`
  handles — so such a customer can *never have their state changed*: every attempt either returns
  silently (if `"unknown"` is requested) or transitions successfully to a real state (any other
  value), which is actually the recovery path.
- `GET /customers/{id}/state` on such a customer returns `{"state": "unknown"}`, which
  `availability-service` will treat as not-valid — a fail-closed outcome, which is the desirable
  direction.

### 3.11 Absence of optimistic concurrency — last-writer-wins

**Definition.** How this service handles two writers touching the same customer: it doesn't.

**Evidence** (`…Infrastructure/Mongo/Repositories/CustomerMongoRepository.cs:19-27`):

```csharp
public async Task<Customer> GetAsync(Guid id)
{
    var customer = await _repository.GetAsync(o => o.Id == id);
    return customer?.AsEntity();
}

public Task AddAsync(Customer customer) => _repository.AddAsync(customer.AsDocument());
public Task UpdateAsync(Customer customer) => _repository.UpdateAsync(customer.AsDocument());
```

`UpdateAsync` passes the whole document with no filter predicate, so the underlying Convey
`IMongoRepository.UpdateAsync` replaces by `_id` `[convey]`. Compare `availability-service`, whose
repository calls `UpdateAsync(document, d => d.Id == … && d.Version < document.Version)` and checks
the result (`component-internals/availability-service.md` §3.11). **Customers has no such filter and
checks no result.**

**Lifecycle of a write.** load whole aggregate → mutate in memory → serialize whole document →
replace by `_id` → publish events. There is no transaction spanning the write and the publish;
`disableTransactions: true` in the outbox options (`…Api/appsettings.json:106`) makes this explicit
even for the outbox itself.

**Invariants & enforcement.** None. Specifically **not** enforced:

- Lost update: two concurrent handlers both read version *n*, both write; the second overwrites the
  first's mutation entirely, including `CompletedOrders` entries.
- Read-modify-write atomicity: `AddCompletedOrder` is a set insert in application memory, not a
  Mongo `$addToSet`, so it is subject to the same loss.

**Realistic collision windows in this service:**

| Concurrent pair | Consequence |
| --- | --- |
| Two `OrderCompleted` events for the same customer | One completed order silently lost → VIP promotion delayed by one order, permanently |
| `OrderCompleted` + `ChangeCustomerState` | Either the state change or the order is lost |
| `CompleteCustomerRegistration` + `ChangeCustomerState` | Name/address or state lost |

The `OrderCompleted` case is the live one: RabbitMQ delivers to a single queue
(`customers-service/orders.order_completed`, §3.22) and Convey's subscriber concurrency is not
configured in `appsettings.json`, so whether two `OrderCompleted` events for the same customer can
be processed simultaneously depends on Convey's default prefetch/parallelism.
**`Unverifiable — Missing Source Evidence`** — Convey has no source in this workspace and no
concurrency setting appears in `…Api/appsettings.json:108-149`.

**Extension procedure.** To add optimistic concurrency, mirror `availability-service`:

1. Add `public int Version { get; set; }` to `CustomerDocument` and map it in both directions.
2. Increment the aggregate's `Version` on mutation — note `AggregateRoot.Version` has a `protected
   set`, so the increment must live inside `Customer` (Deliveries added an `IncrementVersion()` for
   exactly this and never called it — `component-internals/deliveries-service.md` §3.2).
3. Change `CustomerMongoRepository.UpdateAsync` to pass a filter including `Version <
   document.Version` and to **inspect the returned result**, throwing on a lost update.
4. Backfill `Version = 0` on existing documents, or rely on the driver's default for a missing int
   field (`0`), which makes the first update after deployment succeed.

**What silently rejects the extension:** performing steps 1–2 without step 3 gives you a version
field that increments and protects nothing — precisely the state `deliveries-service` is in today.

**Failure modes.** Silent lost updates, with no metric, no log and no exception. Nothing in this
repository would reveal that it had happened.

### 3.12 Commands and command handlers

**Definition.** The two write operations this service accepts, each reachable over both transports.

| Command | Fields | Handler | Attribute |
| --- | --- | --- | --- |
| `CompleteCustomerRegistration` | `Guid CustomerId`, `string FullName`, `string Address` | `CompleteCustomerRegistrationHandler` | `[Contract]` (`…Application/Commands/CompleteCustomerRegistration.cs:6`) |
| `ChangeCustomerState` | `Guid CustomerId`, `string State` | `ChangeCustomerStateHandler` | `[Contract]` (`…Application/Commands/ChangeCustomerState.cs:6`) |

**Representation & storage.** Immutable classes with get-only properties and a single constructor
(`CompleteCustomerRegistration.cs:9-18`, `ChangeCustomerState.cs:9-16`). They are bound from the
HTTP request by Convey's dispatcher endpoints `[convey]`, which merge the JSON body with route
values — this is how `PUT /customers/{customerId}/state/{state}` populates both properties from the
path with an empty body (`…Api/Program.cs:40`).

**Neither command has a validator.** There is no FluentValidation reference, no `IValidator`, and no
`[Required]`/data-annotation attribute in `src/`. All validation is domain validation, executed
inside the aggregate after the command has already been dispatched. The gateway declares
`payload: create_customer` and `schema: create_customer.schema` for `POST /customers`
(`ntrada.yml`), but **neither file exists in the gateway repository** — recorded in
`component-internals/api-gateway.md` §3.10 / Q-1. So `POST /customers` is unvalidated at the edge
*and* unvalidated at the service boundary; a request with a null `fullName` reaches
`Customer.CompleteRegistration` and is rejected there by the `IsNullOrWhiteSpace` check.

**Handler flow — `CompleteCustomerRegistrationHandler`** (`:26-44`):

1. `GetAsync(command.CustomerId)`; `null` → `CustomerNotFoundException`.
2. `if (customer.State is State.Valid)` → `CustomerAlreadyRegisteredException`. *This is a
   pre-check duplicating part of the aggregate's own rule* — the aggregate rejects anything that is
   not `Incomplete` (`Customer.cs:56-59`). The two differ: a `Locked` customer passes the handler's
   check and is rejected by the aggregate with `CannotChangeCustomerStateException`, while a `Valid`
   customer is rejected earlier with `CustomerAlreadyRegisteredException`. **Two different error
   codes for what a client experiences as the same situation.**
3. `customer.CompleteRegistration(FullName, Address)` — the real rule.
4. `UpdateAsync`, then `MapAll(customer.Events)`, then `PublishAsync`.

**Handler flow — `ChangeCustomerStateHandler`** (`:26-65`): covered in §3.4. Its distinguishing
feature is the silent no-op at lines 39-42.

**Lifecycle.** Registered implicitly: `AddApplication()` scans and registers `ICommandHandler<>`
implementations `[convey]`, and both handlers are then wrapped by `OutboxCommandHandlerDecorator<>`
via `TryDecorate` (`…Infrastructure/Extensions.cs:59`).

**Invariants & enforcement.** Each handler's guard clauses are its contract; see the table in §3.1.
Note that **no handler checks authorization** — §1.2.

**Extension procedure.** To add a command, e.g. `UpdateCustomerAddress`:

1. Add the command class in `…Application/Commands/` with `[Contract]` if it should appear in the
   public-contracts endpoint (§3.23).
2. Add `…Application/Commands/Handlers/UpdateCustomerAddressHandler.cs` implementing
   `ICommandHandler<UpdateCustomerAddress>`.
3. Add a mutator on `Customer` that validates and raises a domain event.
4. Map the domain event in `EventMapper` (§3.14) — else silence.
5. Add an arm to `ExceptionToMessageMapper` for the new failure(s), keyed on the **command type**
   (§3.16) — else AMQP failures are silent.
6. Wire the HTTP route in `…Api/Program.cs` inside `UseDispatcherEndpoints`.
7. Wire the AMQP subscription with `.SubscribeCommand<UpdateCustomerAddress>()` in
   `…Infrastructure/Extensions.cs:93-96`.
8. Add the route to the gateway's `ntrada.yml` (sync) and `ntrada-async.yml` (async) with the right
   `auth`/`claims`, and add the message to Operations' `messages.json`.
9. Add a log template in `MessageToLogTemplateMapper` (§3.27) if you want the handler logged.

**What silently rejects the extension:** skipping step 6 makes the command AMQP-only with no
compile error; skipping step 7 makes it HTTP-only, and the async gateway profile will publish to the
`customers` exchange where **nothing is bound**, so the message is dropped by the broker with no
error visible to the caller (which has already received `202 Accepted`).

**Failure modes.** Command binding is Convey's `[convey]`; a mismatch between the route template and
the command's property names yields a default-valued property (e.g. `Guid.Empty`), which surfaces as
`customer_not_found` rather than as a binding error.

### 3.13 Queries and the read model

**Definition.** Three read operations, all served directly from Mongo documents without touching the
aggregate.

| Query | Result | Handler | Behaviour |
| --- | --- | --- | --- |
| `GetCustomers` (no parameters) | `IEnumerable<CustomerDto>` | `…Mongo/Queries/Handlers/GetCustomersHandler.cs:22-27` | `FindAsync(_ => true)` — **the entire collection**, materialized, then projected |
| `GetCustomer(CustomerId)` | `CustomerDetailsDto` | `GetCustomerHandler.cs:20-25` | `GetAsync(p => p.Id == query.CustomerId)`, `null`-safe projection |
| `GetCustomerState(CustomerId)` | `CustomerStateDto` | `GetCustomerStateHandler.cs:20-31` | same lookup, projects `Id` + lowercase state |

**Representation & storage.** Note the **layer placement**: the query *contracts* live in
`…Application/Queries/`, but the *handlers* live in
`…Infrastructure/Mongo/Queries/Handlers/`. This is deliberate — handlers depend on
`IMongoRepository<CustomerDocument, Guid>`, which is an infrastructure type, so putting them in
`Application` would invert the dependency. They are registered by `AddQueryHandlers()` +
`AddInMemoryQueryDispatcher()` (`…Infrastructure/Extensions.cs:64-65`).

**The unbounded scan.** `GetCustomersHandler.cs:24` is:

```csharp
var customers = await _customerRepository.FindAsync(_ => true);
```

There is **no pagination, no projection pushdown, no limit and no sort**. `GetCustomers` has no
properties at all (`…Application/Queries/GetCustomers.cs`) so a caller *cannot* page even if they
wanted to. The whole `customers` collection is loaded into memory and mapped. `availability-service`
has the same shape for `GetResources` but at least filters by tags
(`component-internals/availability-service.md` §3.13); here there is not even a filter.

At the gateway, `GET /customers` requires `role: admin` (`ntrada.yml`), so the blast radius is
limited to administrative calls — but nothing in the service enforces that, and
`Convey.Persistence.MongoDB` offers a paged `BrowseAsync` `[convey]` that this service does not use.

**Lifecycle.** Read-only. No caching layer sits in front (Redis is registered but unused — §3.33),
so every call hits Mongo.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| A missing customer yields `null`, not an exception | `GetCustomerHandler.cs:24` (`document?.AsDetailsDto()`), `GetCustomerStateHandler.cs:24-25` | **Silent** — and the HTTP response is `200 OK` with an empty body, **not 404**. See below. |
| The read model is eventually the write model | trivially — same collection, same documents | — |

**The missing-404.** `Convey.WebApi.CQRS`'s `Get<TQuery, TResult>` maps a `null` result to a
response `[convey]`. Which status code it emits for `null` is **`Unverifiable — Missing Source
Evidence`** (Convey has no source in this workspace); what *is* verifiable is that this service
contributes no 404 logic of its own — `ExceptionToResponseMapper` produces only 400s (§3.18) and no
handler throws on a missing customer during a read. Callers must therefore treat an empty/absent
body as "not found". `availability-service`'s `CustomersServiceClient` does exactly that: it treats
a null state response as not-valid (`component-internals/availability-service.md` §3.26).

**Extension procedure.** To add a query:

1. Add the query class in `…Application/Queries/` implementing `IQuery<TResult>`.
2. Add the handler in `…Infrastructure/Mongo/Queries/Handlers/` implementing
   `IQueryHandler<TQuery, TResult>` — **it must be in the Infrastructure assembly**, because
   `AddQueryHandlers()` is called on the Convey builder from `…Infrastructure/Extensions.cs:64` and
   scans the calling assembly `[convey]`.
3. Add the route with `.Get<TQuery, TResult>("path")` in `…Api/Program.cs`.
4. Add the route to `ntrada.yml` (and `ntrada-async.yml` — reads stay `downstream:` in both).

**What silently rejects the extension:** putting the handler in the `Application` project. It will
compile (the interface is in `Convey.CQRS.Queries`) and simply never be registered; the dispatcher
then fails at request time with a resolution error rather than at startup.

To add pagination: add `int Page`/`int Results` to `GetCustomers`, switch the handler to Convey's
`BrowseAsync` and change the endpoint's result type — this is a **breaking change** for
`api-gateway` consumers, which currently expect a bare array.

**Failure modes.**

- `GetCustomers` scales linearly with total customers, in one round trip, with no timeout of its own
  (the HTTP client's `retries: 3` in `appsettings.json:25` applies to *outbound* calls, not to this).
- The list projection omits `IsVip`, `Email`, `FullName` and `Address`
  (`Documents/Extensions.cs:25-31`), so an admin listing customers sees only id, state and creation
  time — any UI needing more must call `GET /customers/{id}` per row (an N+1 at the gateway).

### 3.14 Integration events and `EventMapper`

**Definition.** The translation from internal domain events to the messages that leave the service,
and the single point where "we changed something" becomes "the platform is told".

**The mapping table** (`…Infrastructure/Services/EventMapper.cs:15-27`):

| Domain event (Core) | Integration event (Application) | Payload | Routing key on exchange `customers` |
| --- | --- | --- | --- |
| `CustomerRegistrationCompleted` | `Application.Events.CustomerCreated` | `Id` only | `customer_created` |
| `CustomerBecameVip` | `Application.Events.CustomerBecameVip` | `Id` only | `customer_became_vip` |
| `CustomerStateChanged` | `Application.Events.CustomerStateChanged` | `Id`, `State` (lowercase), `PreviousState` (lowercase) | `customer_state_changed` |
| *anything else* | **`null`** (`EventMapper.cs:26`) | — | — |

The routing-key names are confirmed independently by
`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`, which lists
`customers-service` with exchange `customers` and events
`customer_created`, `customer_became_vip`, `customer_state_changed` — the snake_case conversion is
`conventionsCasing: "snakeCase"` (`…Api/appsettings.json:112`) `[convey]`.

**The naming discontinuity.** The domain event is `CustomerRegistrationCompleted`; the integration
event is `CustomerCreated`. This is not a typo — it is the whole reason §3.7's "creation raises no
event" matters. **From the platform's point of view, a customer is "created" when they finish
registering, not when they sign up.** Anyone tracing `customer_created` back to a "create" operation
will look in the wrong place; the producer is `Customer.CompleteRegistration`.

**Who consumes each event** (verified by searching every clone for handlers of these types):

| Event | Consumers | Evidence |
| --- | --- | --- |
| `customer_created` | `availability-service`, `orders-service`, `parcels-service` | `CustomerCreatedHandler` in each of those repos |
| `customer_state_changed` | **none** | no handler exists in any clone |
| `customer_became_vip` | **none** | no handler exists in any clone |

So two thirds of this service's published events have **no subscriber anywhere on the platform**.
They are declared in Operations' `messages.json`, which means the operations UI can display them —
that is their only consumer.

**Representation & storage.** `EventMapper` is a stateless singleton
(`…Infrastructure/Extensions.cs:52`). `MapAll` is `events.Select(Map)` — **lazily evaluated**
(`EventMapper.cs:12-13`); each handler forces it with `.ToArray()` before publishing
(`CompleteCustomerRegistrationHandler.cs:43`, `ChangeCustomerStateHandler.cs:64`,
`OrderCompletedHandler.cs:41`). Because `Map` reads `e.Customer.State` at *evaluation* time (§3.7),
the deferred execution is load-bearing: forcing later would read later state.

**Lifecycle.** Called once per successful handler, after the repository write.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| Every domain event has an integration counterpart | **nothing** | **Silent** — unmapped → `null` → skipped at `MessageBroker.cs:69-72` |
| Published state strings are lowercase | `EventMapper.cs:23` | Silent |
| Events are published only after the write succeeds | statement ordering in each handler | Not transactional — §3.15 |

**Extension procedure.** Add a `case` to the `switch` in `Map`, add the integration event class in
`…Application/Events/`, and register the name in Operations' `messages.json`.

**What silently rejects the extension:** forgetting the `case`. There is no exhaustiveness check —
the `switch` is a statement over a class hierarchy, and the fallthrough `return null` at line 26 is
the default. Combined with `MessageBroker`'s `if (@event is null) continue;`
(`MessageBroker.cs:69-72`), a missing mapping produces **a completely successful, completely silent
non-publication**.

**Failure modes.**

- Silent non-publication, as above.
- `CustomerStateChanged`'s payload reports the *current* state at mapping time; a future handler
  performing two state changes in one unit of work would publish two events both reporting the final
  state, with different `PreviousState` values — an inconsistent history.
- Because the integration `CustomerCreated` carries **only an id**
  (`…Application/Events/CustomerCreated.cs`), the three consuming services learn nothing about the
  customer. Each stores just the id (e.g. `availability-service`'s handler only logs —
  `component-internals/availability-service.md` §3.21). Any consumer needing the email or name must
  call back over HTTP.

### 3.15 `MessageBroker` — the publish pipeline

**Definition.** The service's own thin wrapper over Convey's publisher and outbox, responsible for
propagating tracing and correlation metadata (`…Infrastructure/Services/MessageBroker.cs:16-87`).

**Representation & storage.** `internal sealed class MessageBroker : IMessageBroker`, registered
**transient** (`…Infrastructure/Extensions.cs:53`). It composes six injected services: `IBusPublisher`,
`IMessageOutbox`, `ICorrelationContextAccessor`, `IHttpContextAccessor`,
`IMessagePropertiesAccessor`, `ITracer` `[convey]`.

**The pipeline, per publish call** (`MessageBroker.cs:47-86`):

1. `events is null` → return (line 49-52).
2. Read the inbound broker message properties (`_messagePropertiesAccessor.MessageProperties`) to
   obtain `originatedMessageId` and `correlationId` (lines 54-56). On an HTTP-triggered publish
   these are `null`, because there is no inbound message.
3. Resolve the span context: prefer the inbound message's `span_context` header
   (`GetSpanContext`, `…Infrastructure/Extensions.cs:122-135` — reads the header as `byte[]` and
   UTF-8 decodes it); fall back to `_tracer.ActiveSpan.Context.ToString()`; else empty
   (`MessageBroker.cs:57-61`). The header name comes from `RabbitMqOptions.SpanContextHeader`,
   defaulting to `"span_context"` (`MessageBroker.cs:18,40-42`), and `appsettings.json:148` sets
   exactly that.
4. Forward the `Saga` header if present (`GetHeadersToForward`,
   `…Infrastructure/Extensions.cs:106-120`) — **`Saga` is the only header ever forwarded**;
   everything else on the inbound message is dropped.
5. Resolve the correlation context: the broker's, else the HTTP `Correlation-Context` header
   (`MessageBroker.cs:64-65`, §3.20).
6. For each event: **skip nulls** (lines 69-72) — the silent-drop point from §3.14; mint a fresh
   `messageId = Guid.NewGuid().ToString("N")` (line 74); log at **Trace** level (line 75); then
   either `_outbox.SendAsync(…)` if the outbox is enabled, or `_busPublisher.PublishAsync(…)`
   directly (lines 76-84).

**Lifecycle.** Constructed per resolution (transient), used once per handler.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| Nulls are never published | `MessageBroker.cs:69-72` | **Silent** |
| Every message gets a unique id | `Guid.NewGuid()` per event, line 74 | — |
| Trace context survives a hop | steps 3-4 | Silent if the header is absent or not `byte[]` (`Extensions.cs:129`) |
| The write and the publish are atomic | **nothing** | Publication happens *after* `UpdateAsync` in every handler, in a separate statement. With the outbox enabled the gap is narrowed (§3.17) but `disableTransactions: true` (`appsettings.json:106`) means the outbox insert and the domain write are **not** in one Mongo transaction. |

That last row is the important one: **this service has an outbox but not an atomic outbox.** A crash
between `UpdateAsync` and `PublishAsync` loses the event permanently, with the state change already
committed.

**Extension procedure.** To forward an additional header (say a tenant id), extend
`GetHeadersToForward` (`…Infrastructure/Extensions.cs:106-120`) — note it currently returns `null`
when the `Saga` header is absent, so a second header requires restructuring the early return, not
just adding a dictionary entry.

**Failure modes.**

- Non-atomic publish (above).
- The trace-log line (line 75) is at `LogTrace`, and `appsettings.json:33` sets
  `logger.level: "information"` — so **published-event logging is off in every environment by
  default.** The outbox is then the only record that a publish was attempted.
- `messageProperties.GetHeadersToForward()` is called on a possibly-null receiver via an extension
  method (line 63) — safe, because the extension null-checks (`Extensions.cs:109`).

### 3.16 Rejected events and `ExceptionToMessageMapper`

**Definition.** The AMQP failure contract: when a command consumed from the broker throws, Convey's
subscriber asks `IExceptionToMessageMapper` for a message to publish in its place `[convey]`. That
message is the *only* way an async caller learns the command failed.

**The rejected events available** (`…Application/Events/Rejected/`):

- `ChangeCustomerStateRejected(Guid CustomerId, string State, string Reason, string Code)`
- `CompleteCustomerRegistrationRejected(Guid CustomerId, string Reason, string Code)`

Both are listed in Operations' `messages.json` under `customers-service.rejectedEvents` as
`change_customer_state_rejected` and `complete_customer_registration_rejected`.

**The complete mapping** (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:12-31`):

| Exception | Triggering message | Produces |
| --- | --- | --- |
| `CannotChangeCustomerStateException` | `ChangeCustomerState` | `ChangeCustomerStateRejected` (state lowercased) |
| `CannotChangeCustomerStateException` | `CompleteCustomerRegistration` | `CompleteCustomerRegistrationRejected` |
| `CannotChangeCustomerStateException` | anything else | **`null`** — silent |
| `CustomerAlreadyRegisteredException` | *any* | `CompleteCustomerRegistrationRejected` |
| `CustomerNotFoundException` | *any* | `CompleteCustomerRegistrationRejected` |
| `InvalidCustomerFullNameException` | *any* | `CompleteCustomerRegistrationRejected` |
| `InvalidCustomerAddressException` | *any* | `CompleteCustomerRegistrationRejected` |
| `InvalidRoleException` | *any* | **`null`** — silent |
| `CustomerAlreadyCreatedException` | *any* | **`null`** — silent |
| `InvalidAggregateIdException` | *any* | **`null`** — silent |
| anything else | *any* | **`null`** — silent |

**Three defects follow directly from that table**, and a reader changing this service must know all
three:

1. **Wrong rejection type for `ChangeCustomerState`.** Only `CannotChangeCustomerStateException` is
   dispatched per-command. `CustomerNotFoundException` — which `ChangeCustomerStateHandler.cs:31`
   raises — falls into the flat arm at line 25 and produces a
   **`CompleteCustomerRegistrationRejected`**. An async client that sent `change_customer_state` and
   is waiting for `change_customer_state_rejected` never sees it; it sees a rejection for a command
   it never sent. `operations-service` correlates by message id, so the operation *is* marked failed
   `Unverifiable — Missing Source Evidence` (the correlation logic lives in `Pacco.Services.Operations`
   and is out of this batch's scope), but the event *name* is wrong for any other consumer.
2. **The `SignedUp` handler's two failures are unmapped.** `InvalidRoleException` (non-`user` role)
   and `CustomerAlreadyCreatedException` (duplicate) both return `null`. Since `SignedUp` arrives
   **only** over AMQP (`…Infrastructure/Extensions.cs:95`), these two rules can *only* fail
   silently. There is no `signed_up_rejected` message in Operations' manifest either — the event is
   an integration event from `identity-service`, so a rejection has nowhere natural to go. The
   practical consequence: **a sign-up with a non-`user` role produces a customer-less user, and
   nothing anywhere reports it** except an exception in the log.
3. **`OrderCompletedHandler`'s `CustomerNotFoundException` produces a
   `CompleteCustomerRegistrationRejected`** — again a rejection naming a command nobody sent, this
   time in response to an *event*.

**Representation & storage.** A stateless class implementing
`Convey.MessageBrokers.RabbitMQ.IExceptionToMessageMapper`, registered by
`AddExceptionToMessageMapper<ExceptionToMessageMapper>()` (`…Infrastructure/Extensions.cs:71`).
`Map` is a C# 8 nested `switch` expression; `message` is the *inbound* message object, which is how
per-command dispatch is possible at all.

**Lifecycle.** Invoked by Convey's subscriber pipeline on exception `[convey]`. The exact
acknowledgement semantics — whether the original message is acked, nacked or requeued after a
rejection is published — is **`Unverifiable — Missing Source Evidence`** (no Convey source in this
workspace, and `appsettings.json:108-149` sets `retries: 3` and `retryInterval: 2` without
specifying dead-lettering).

**Invariants & enforcement.** None. The `_ => null` fallbacks at lines 21 and 30 are the enforcement
gap.

**Extension procedure.**

1. Add the rejected-event class in `…Application/Events/Rejected/`.
2. Add an arm to `Map`. **Nest it by message type** if the same exception can arise from more than
   one command — follow the `CannotChangeCustomerStateException` shape at lines 15-22, not the flat
   shape at lines 23-29.
3. Add the snake_case name to Operations' `messages.json` under `rejectedEvents`, else the
   operations UI cannot resolve it.

**What silently rejects the extension:** omitting step 2 gives you a rejected-event class that is
never constructed. Omitting step 3 gives you a message on the wire that the operations UI does not
recognize.

**Failure modes.** Every `null` return is an unobservable failure for the async caller. The HTTP
caller is unaffected — that path uses `ExceptionToResponseMapper` (§3.18), which has no gaps because
its fallback still returns a 400.

### 3.17 Transactional outbox and inbox

**Definition.** Convey's Mongo-backed outbox/inbox, applied to this service through two
hand-written decorators.

**Registration** (`…Infrastructure/Extensions.cs:59-60,70`):

```csharp
builder.Services.TryDecorate(typeof(ICommandHandler<>), typeof(OutboxCommandHandlerDecorator<>));
builder.Services.TryDecorate(typeof(IEventHandler<>),   typeof(OutboxEventHandlerDecorator<>));
…
.AddMessageOutbox(o => o.AddMongo())
```

**The decorator** (`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:19-35`):

```csharp
var messageProperties = messagePropertiesAccessor.MessageProperties;
_messageId = string.IsNullOrWhiteSpace(messageProperties?.MessageId)
    ? Guid.NewGuid().ToString("N")
    : messageProperties.MessageId;

public Task HandleAsync(TCommand command)
    => _enabled
        ? _outbox.HandleAsync(_messageId, () => _handler.HandleAsync(command))
        : _handler.HandleAsync(command);
```

`_outbox.HandleAsync(messageId, handler)` is Convey's **inbox** operation: it records the message id
and skips the handler if that id was already processed `[convey]`.

**The critical consequence.** On the **HTTP path there is no inbound message**, so
`messageProperties` is `null` and the decorator mints a **fresh GUID per request**
(line 27-29). A message id that is unique per request de-duplicates nothing. Therefore:

> **De-duplication is real for AMQP-delivered commands and events; it is a no-op for HTTP requests.**

`POST /customers` submitted twice with the same body executes twice — the second is rejected by
`CustomerAlreadyRegisteredException`, so the outcome is safe, but by domain rule, not by the inbox.

**Configuration** (`…Api/appsettings.json:99-107`):

| Key | Value | Meaning |
| --- | --- | --- |
| `enabled` | `true` | outbox on (overridden to `false` in `appsettings.local.json`) |
| `type` | `sequential` | processing strategy `[convey]` |
| `expiry` | `3600` | seconds an inbox/outbox record is retained |
| `intervalMilliseconds` | `2000` | outbox dispatch poll interval |
| `inboxCollection` | `inbox` | Mongo collection, same database |
| `outboxCollection` | `outbox` | Mongo collection, same database |
| `disableTransactions` | `true` | **no multi-document transaction** — the domain write and the outbox insert are separate operations |

`expiry: 3600` sets the de-duplication window at **one hour**: a redelivery after that is processed
again as if new. For `OrderCompleted` this is harmless (the `HashSet` dedupes — §3.6); for a future
non-idempotent handler it would not be.

**Lifecycle.** Handler invoked → decorator checks/records the inbox entry → inner handler runs →
`MessageBroker` writes the event to the outbox rather than the bus (`MessageBroker.cs:76-80`) → a
Convey background processor drains the outbox to RabbitMQ every 2 s `[convey]`.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| An AMQP message is handled at most once per hour | inbox, keyed on broker `MessageId` | Silent skip on duplicate |
| An HTTP request is handled at most once | **not enforced** | Silent |
| Domain write and outbox insert are atomic | **not enforced** — `disableTransactions: true` | Silent |
| Outbox records are eventually dispatched | Convey background processor | `Unverifiable — Missing Source Evidence` — no source; failure behaviour on a permanently-unreachable broker is unknown |

**Extension procedure.** To make the HTTP path idempotent, supply a stable message id: accept a
client-provided idempotency key and thread it into the decorator instead of `Guid.NewGuid()`. Note
the decorator resolves the id **in its constructor**, so the key must be available at DI-resolution
time (i.e. from `IHttpContextAccessor`), not at `HandleAsync` time.

To enable real transactionality, set `disableTransactions: false` — which requires MongoDB to be
running as a replica set. `hianshul100_Pacco/compose/services.yml` starts a **single `mongo`
container with no replica-set configuration**, so flipping this flag in the current compose topology
would break every write.

**Failure modes.**

- Crash between the domain write and the outbox insert: state changed, event lost, permanently.
- Outbox drained but broker publish fails: retried by the processor `[convey]`; duplicate delivery
  is then absorbed by consumers' own inboxes.
- The one-hour expiry means a message parked in a dead-letter queue and replayed the next day is
  processed as new.

### 3.18 `ExceptionToResponseMapper` — everything is HTTP 400

**Definition.** The HTTP failure contract
(`…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:15-24`).

```csharp
public ExceptionResponse Map(Exception exception)
    => exception switch
    {
        DomainException ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message},
            HttpStatusCode.BadRequest),
        AppException ex => new ExceptionResponse(new {code = GetCode(ex), reason = ex.Message},
            HttpStatusCode.BadRequest),
        _ => new ExceptionResponse(new {code = "error", reason = "There was an error."},
            HttpStatusCode.BadRequest)
    };
```

**Every arm returns `BadRequest`.** There is no 404, no 409, no 403 and no 500 anywhere in this
service. Specifically:

| Situation | Natural status | Actual status | Body |
| --- | --- | --- | --- |
| `POST /customers` for a non-existent customer | 404 | **400** | `{code:"customer_not_found", reason:…}` |
| `POST /customers` for an already-registered customer | 409 | **400** | `{code:"customer_already_registered"}` |
| `PUT …/state/{state}` with a bogus state | 400 | 400 | `{code:"cannot_change_customer_state"}` |
| An unhandled `NullReferenceException` | 500 | **400** | `{code:"error", reason:"There was an error."}` |
| `GET /customers/{unknown-id}` | 404 | not an exception at all — `null` result (§3.13) | — |

The last two rows are the operationally dangerous ones. **An internal server error is reported to
the client as a 400 with a generic body**, which means monitoring that alerts on 5xx will never fire
for this service, and a client cannot distinguish "you sent bad data" from "we are broken". The
message is deliberately opaque (`"There was an error."`), which is good for information disclosure
and bad for diagnosis; the actual exception is available only in logs.

**Representation & storage.** `internal sealed`, registered via
`AddErrorHandler<ExceptionToResponseMapper>()` (`…Infrastructure/Extensions.cs:63`) and activated by
`UseErrorHandler()` (`:85`), both Convey `[convey]`. Codes are cached in a static
`ConcurrentDictionary<Type, string>` (line 13).

**Lifecycle.** Invoked by Convey's error-handling middleware for any exception escaping the
dispatcher.

**Invariants & enforcement.** The only invariant is *never leak an unhandled exception's message*,
enforced by the `_ =>` arm.

**Extension procedure.** To introduce a 404:

1. Add an arm before the `DomainException` arm:
   `CustomerNotFoundException ex => new ExceptionResponse(…, HttpStatusCode.NotFound),`. Order
   matters — `CustomerNotFoundException` derives from `DomainException`, so a later arm would never
   be reached.
2. Check every consumer. `availability-service` and `pricing-service` call this service over HTTP;
   their clients' behaviour on a 404 is defined in their own repositories and is out of scope for
   this batch — treat any status-code change as a **breaking cross-service change**.
3. Add a 500 arm for genuinely unexpected exceptions if 5xx alerting is wanted.

**What silently rejects the extension:** placing a derived-exception arm after its base-class arm.
C# `switch` expressions match top-down, and the compiler does **not** warn about an unreachable
pattern in every case here — the base arms are `DomainException`/`AppException`, so a later derived
arm is simply dead.

**Failure modes.** 5xx invisibility; ambiguous 400 semantics; no correlation id in the error body
(the response carries `code` and `reason` only), so client-side error reports cannot be tied to a
server log line without the `Correlation-Context` header the client itself supplied.

### 3.19 `IAppContext` / `IIdentityContext` — built, never consumed

**Definition.** A transport-agnostic representation of *who is calling*, assembled from either the
HTTP `Correlation-Context` header or the broker message context.

**Representation & storage** (`…Infrastructure/Contexts/`):

| Type | Shape | Evidence |
| --- | --- | --- |
| `AppContext` | `RequestId` (string), `Identity` (`IIdentityContext`) | `AppContext.cs:6-27` |
| `IdentityContext` | `Id` (Guid), `Role` (string), `IsAuthenticated` (bool), `IsAdmin` (bool), `Claims` (dictionary) | `IdentityContext.cs:7-34` |
| `CorrelationContext` | `CorrelationId`, `SpanContext`, `User`, `ResourceId`, `TraceId`, `ConnectionId`, `Name`, `CreatedAt`; nested `UserContext { Id, IsAuthenticated, Role, Claims }` | `CorrelationContext.cs:6-24` |

`IsAdmin` is **derived, not transmitted**:
`Role.Equals("admin", StringComparison.InvariantCultureIgnoreCase)` (`IdentityContext.cs:29`).
`Id` is a lenient parse — `Guid.TryParse(id, out var userId) ? userId : Guid.Empty`
(`IdentityContext.cs:26`), so a malformed user id becomes `Guid.Empty` rather than an error.

**Lifecycle** (`AppContextFactory.cs:19-33`):

1. If `ICorrelationContextAccessor.CorrelationContext` is non-null (the AMQP path), it is
   **serialized to JSON and immediately deserialized** into this service's own `CorrelationContext`
   type (lines 23-27). This round-trip exists because Convey's accessor exposes an `object`, and it
   is the only way to get the service's strongly-typed shape out of it `[convey]`.
2. Otherwise, read the HTTP `Correlation-Context` header (§3.20).
3. If neither exists, return `AppContext.Empty` — a fresh `Guid.NewGuid().ToString("N")` request id
   and `IdentityContext.Empty` (`AppContext.cs:11-13,26`), i.e. **an unauthenticated anonymous
   context, never an error**.

Registration (`…Infrastructure/Extensions.cs:57-58`):

```csharp
builder.Services.AddTransient<IAppContextFactory, AppContextFactory>();
builder.Services.AddTransient(ctx => ctx.GetRequiredService<IAppContextFactory>().Create());
```

so `IAppContext` is resolvable by any component that asks for it.

**Nothing asks for it.** Grepping `src/` for `IAppContext` finds only its definition
(`…Application/IAppContext.cs`), its implementation, its factory and these two registrations. **No
command handler, event handler, or query handler injects it.** Compare `availability-service`, where
exactly one handler uses it for an ownership check
(`component-internals/availability-service.md` §3.19). In `customers-service` the caller's identity
is *computed on every request and discarded*.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| A missing/invalid context yields an anonymous identity | `AppContext.Empty` fallbacks | **Silent** — fail-open |
| Identity is trustworthy | **nothing** — the header is accepted verbatim | The gateway is the only authenticator |
| `IsAdmin` reflects the caller's real role | `IdentityContext.cs:29`, from a header the service does not verify | **Silent** |

**The trust model, stated plainly.** Anything that can reach this service's HTTP port can set
`Correlation-Context` to `{"user":{"role":"admin","isAuthenticated":true}}`. The service will build
an admin identity from it — and then ignore it, because no code reads it. The *effective* protection
is therefore §3.25's certificate ACL (in environments where it is enabled) plus network topology,
not identity.

**Extension procedure.** To add an authorization check to a handler:

1. Inject `IAppContext` into the handler's constructor.
2. Guard on `context.Identity.IsAuthenticated` / `IsAdmin` / `Id`, throwing an `AppException`
   subclass with an explicit `Code`.
3. Add the exception to **both** mappers (§3.16, §3.18) — and note that §3.18 currently cannot
   express 403, so an unauthorized caller would receive **400**.
4. Be aware the check is only as good as the gateway: if `security.certificate.enabled` is `false`
   (which it is in both `local` and `docker` — §3.31), any network peer can forge the header.

**Failure modes.** Fail-open anonymity; unverifiable identity; a full identity apparatus whose
absence of consumers means a reviewer can easily believe authorization exists when it does not.

### 3.20 `Correlation-Context` header ingestion

**Definition.** How caller context crosses the HTTP boundary
(`…Infrastructure/Extensions.cs:101-104`):

```csharp
internal static CorrelationContext GetCorrelationContext(this IHttpContextAccessor accessor)
    => accessor.HttpContext?.Request.Headers.TryGetValue("Correlation-Context", out var json) is true
        ? JsonConvert.DeserializeObject<CorrelationContext>(json.FirstOrDefault())
        : null;
```

**Representation & storage.** A JSON document in a single HTTP header named **`Correlation-Context`**
(exact spelling, hyphenated, header lookup is case-insensitive per HTTP). Deserialized with
Newtonsoft into the internal `CorrelationContext` class (§3.19). It is produced by
`api-gateway` (`component-internals/api-gateway.md` §3.14) and, for service-to-service calls, by
Convey's HTTP client `[convey]`.

**Lifecycle.** Read at most twice per request: once by `AppContextFactory.Create()` (`:30`) and once
by `MessageBroker.PublishAsync` (`MessageBroker.cs:64-65`) to attach the context to outbound events.
It is **not** stored, and it is **not** re-emitted on the HTTP response.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| Absent header → `null` context → anonymous | `?:` at line 102 | **Silent** |
| Multi-valued header → first value wins | `json.FirstOrDefault()` (line 103) | **Silent** |
| Malformed JSON | **nothing** — `JsonConvert.DeserializeObject` throws | **Loud, but as an unmapped exception** → HTTP 400 `{code:"error"}` via the `_ =>` arm of §3.18 |

That last row is a genuine and non-obvious behaviour: **a syntactically invalid `Correlation-Context`
header fails the whole request with a generic 400**, because the deserialization happens inside
`AppContextFactory`, which runs during DI resolution of `IAppContext` — but only *if something
resolves `IAppContext`*, which in this service nothing does (§3.19). So today the malformed header
is harmless; the moment any handler injects `IAppContext`, malformed headers become 400s. Worth
knowing before doing §3.19's extension.

`MessageBroker.cs:65` also calls `GetCorrelationContext()` and would throw there — that path *is*
live on every publish. `Unverifiable — Missing Source Evidence`: whether the gateway can ever emit
malformed JSON here is not determinable from this repository.

**Extension procedure.** To add a field (e.g. a tenant id), add it to `CorrelationContext.cs` and to
the gateway's context construction. Newtonsoft ignores unknown properties by default, so adding a
field on one side only is non-breaking in the deserializing direction.

**Failure modes.** Silent anonymity when absent; whole-request failure when malformed on the publish
path; no propagation back to the response.

### 3.21 External event subscription — `SignedUp` and `OrderCompleted`

**Definition.** The two events this service consumes from other services, and the handlers that give
this service its only *creation* path and its only *VIP* path.

**Subscription** (`…Infrastructure/Extensions.cs:95-96`):

```csharp
.SubscribeEvent<SignedUp>()
.SubscribeEvent<OrderCompleted>()
```

**The contracts are re-declared locally**, not shared:

| Event | Declared at | `[Message]` exchange | Fields |
| --- | --- | --- | --- |
| `SignedUp` | `…Application/Events/External/SignedUp.cs:7-20` | `identity` | `Guid UserId`, `string Email`, `string Role` |
| `OrderCompleted` | `…Application/Events/External/OrderCompleted.cs:7-18` | `orders` | `Guid OrderId`, `Guid CustomerId` |

`[Message("identity")]` / `[Message("orders")]` tell the RabbitMQ subscriber which **producer's
exchange** to bind to `[convey]` — so this service binds queues on three exchanges: its own
(`customers`), plus `identity` and `orders`. The duplicated contract classes are the coupling
mechanism: a producer renaming a field breaks the consumer at deserialization time with no compile
error anywhere.

**`SignedUpHandler`** (`…Application/Events/External/Handlers/SignedUpHandler.cs:26-41`):

1. `if (@event.Role != RequiredRole) throw new InvalidRoleException(...)` where
   `RequiredRole = "user"` (line 13). **This is an ordinal, case-sensitive comparison** — a
   `SignedUp` carrying `"User"` throws. `identity-service`'s emitted casing is out of this batch's
   scope; treat the exact-case requirement as a live coupling.
2. `GetAsync(@event.UserId)`; non-null → `CustomerAlreadyCreatedException`.
3. `new Customer(@event.UserId, @event.Email, _dateTimeProvider.Now)` → `AddAsync`.
4. **Ends.** No `IEventMapper`, no `IMessageBroker` — they are not even injected (lines 14-16).

So: *a customer is born mute.* See §3.7 and §4.1.

**`OrderCompletedHandler`** (`…Application/Events/External/Handlers/OrderCompletedHandler.cs:27-42`):

1. `GetAsync(@event.CustomerId)`; `null` → `CustomerNotFoundException`.
2. `var isVip = customer.IsVip;` — captured for the dead local at line 38 (§3.5).
3. `customer.AddCompletedOrder(@event.OrderId)` — silent on `Guid.Empty`, silent on duplicates.
4. `_vipPolicy.ApplyVipStatusIfEligible(customer)`.
5. `UpdateAsync`, `MapAll(customer.Events)`, `PublishAsync`.

Because step 5 runs unconditionally, an `OrderCompleted` that changes nothing still performs a full
document rewrite. The publish is a no-op only because the events list is empty.

**Lifecycle.** Both handlers are wrapped by `OutboxEventHandlerDecorator<>`
(`…Infrastructure/Extensions.cs:60`), so both get real inbox de-duplication keyed on the broker
message id (§3.17) — unlike the HTTP command path.

**Invariants & enforcement.** Covered in §3.1. The decisive property is that **both of this service's
event-handler failure modes are unmapped in `ExceptionToMessageMapper`** (§3.16), so both are silent
to everyone except the log.

**Extension procedure.** To consume a third event:

1. Declare the contract class in `…Application/Events/External/` with
   `[Message("<producer-exchange>")]` matching the producer's exchange name exactly.
2. Add `…/Handlers/<Event>Handler.cs` implementing `IEventHandler<TEvent>`.
3. Add `.SubscribeEvent<TEvent>()` in `…Infrastructure/Extensions.cs`.
4. Add a log template in `MessageToLogTemplateMapper` if the handler should be logged (§3.27).
5. Map any new exception in `ExceptionToMessageMapper` — or accept silent failure.

**What silently rejects the extension:** a wrong exchange name in `[Message]`. The queue is declared
and bound to a non-existent or wrong exchange, and **no message ever arrives** — no error, no log,
nothing. The queue simply sits empty. This is the hardest failure in the whole service to diagnose
from inside the service.

**Failure modes.**

- Contract drift with `identity-service` / `orders-service` (duplicated classes).
- A `SignedUp` for a role other than `user` produces a permanent, silent gap: the user exists in
  `identity-service` and has no customer record; every later command for them returns
  `customer_not_found`.
- Redelivery outside the one-hour inbox window re-runs `SignedUpHandler`, which then throws
  `CustomerAlreadyCreatedException` — silently (§3.16). The end state is correct; the observability
  is not.

### 3.22 Queue naming and message conventions

**Definition.** How this service's AMQP topology is derived from configuration rather than declared
in code.

**Configuration** (`…Api/appsettings.json:108-149`):

| Key | Value | Effect |
| --- | --- | --- |
| `connectionName` | `customers-service` | connection label in the RabbitMQ management UI |
| `retries` / `retryInterval` | `3` / `2` | publish/consume retry policy `[convey]` |
| `conventionsCasing` | `snakeCase` | `CompleteCustomerRegistration` → routing key `complete_customer_registration` |
| `exchange.name` | `customers` | **this service owns exactly one exchange** |
| `exchange.type` | `topic` | |
| `exchange.declare` / `durable` / `autoDelete` | `true` / `true` / `false` | the service declares its own exchange at startup |
| `queue.template` | `customers-service/{{exchange}}.{{message}}` | queue name = `customers-service/<exchange>.<message>` |
| `queue.declare` / `durable` / `exclusive` / `autoDelete` | `true` / `true` / `false` / `false` | durable, shared queues |
| `context.enabled` / `context.header` | `true` / `message_context` | the correlation context travels in a header named `message_context` |
| `spanContextHeader` | `span_context` | consumed by `MessageBroker` (§3.15) |

**The resulting topology** (derived from the template, the `[Message]` attributes and the subscribe
calls):

| Queue | Bound to exchange | Routing key |
| --- | --- | --- |
| `customers-service/customers.complete_customer_registration` | `customers` | `complete_customer_registration` |
| `customers-service/customers.change_customer_state` | `customers` | `change_customer_state` |
| `customers-service/identity.signed_up` | `identity` | `signed_up` |
| `customers-service/orders.order_completed` | `orders` | `order_completed` |

The `<service>/` prefix is what makes the topology fan-out-safe: `orders-service` and
`parcels-service` each get their own queue for the same `customer_created` routing key, so all three
consumers receive every event. This is the platform-wide convention — the same template shape
appears in every service's `appsettings.json`.

**Lifecycle.** Exchanges and queues are declared at startup by `UseRabbitMq()`
(`…Infrastructure/Extensions.cs:92`) `[convey]` and are durable, so they survive broker restarts and
**persist after the service is removed** — a stopped `customers-service` still accumulates messages
in its queues.

**Invariants & enforcement.** None in this service's code; the topology is entirely
configuration-derived. A typo in `queue.template` produces a differently-named queue that binds
correctly but is not the one anyone expects.

**Extension procedure.** Changing the exchange name is a **platform-breaking change**: every
producer's `[Message("customers")]` attribute in every other repository, plus the gateway's
`ntrada-async.yml` `exchange: customers` entries, plus Operations' `messages.json` would all need to
change together.

**Failure modes.** Silent non-delivery on a wrong `[Message]` exchange (§3.21). Durable queues
accumulating unbounded messages when the service is down — there is no TTL or max-length in the
configuration.

### 3.23 `[Contract]` and `UsePublicContracts`

**Definition.** A discovery mechanism: `UsePublicContracts<ContractAttribute>()`
(`…Infrastructure/Extensions.cs:89`) exposes an HTTP endpoint listing every type annotated with the
service's own `ContractAttribute` `[convey]`, so other teams can discover the message shapes this
service accepts.

**What is annotated.** Exactly the two commands:
`CompleteCustomerRegistration` (`…Application/Commands/CompleteCustomerRegistration.cs:6`) and
`ChangeCustomerState` (`ChangeCustomerState.cs:6`). **No event, query or DTO is annotated** — so the
contracts endpoint advertises what you can *ask this service to do*, not what it will *tell you*.
For events, the platform's registry is Operations' `messages.json`.

**Lifecycle.** Read at startup by reflection over the entry assembly graph `[convey]`; served on a
route Convey chooses. The exact route and payload shape are
**`Unverifiable — Missing Source Evidence`** — Convey has no source in this workspace, and this
repository does not document the endpoint.

**Invariants & enforcement.** Annotating is voluntary; nothing checks that every command carries
`[Contract]`.

**Extension procedure.** Add `[Contract]` to any new command. Omitting it does not break dispatch —
routing works from the type, not the attribute — it only removes the command from the discovery
listing. **That is the silent rejection: the command works and is invisible.**

**Failure modes.** The contracts endpoint has no authentication of its own; it is exposed on the
same port as everything else and is reachable by anything that can reach the service (§3.25 governs
that reachability only where certificate authentication is enabled).

### 3.24 Dispatcher-bound HTTP endpoints

**Definition.** The whole HTTP surface, declared in six lines inside `UseDispatcherEndpoints`
(`…Api/Program.cs:33-41`). There are **no MVC controllers** in this service.

| Method & path | Bound to | `afterDispatch` | Evidence |
| --- | --- | --- | --- |
| `GET /` | inline lambda returning `AppOptions.Name` (`"Pacco Customers Service"`) | — | `Program.cs:34` |
| `GET /customers` | query `GetCustomers` → `IEnumerable<CustomerDto>` | — | `:35` |
| `GET /customers/{customerId}` | query `GetCustomer` → `CustomerDetailsDto` | — | `:36` |
| `GET /customers/{customerId}/state` | query `GetCustomerState` → `CustomerStateDto` | — | `:37` |
| `POST /customers` | command `CompleteCustomerRegistration` | `Created($"customers/{cmd.CustomerId}")` | `:38-39` |
| `PUT /customers/{customerId}/state/{state}` | command `ChangeCustomerState` | `NoContent()` | `:40-41` |

**Two naming observations that matter.**

- `POST /customers` **does not create a customer.** It completes the registration of a customer
  created earlier by the `SignedUp` event (§3.21). The `201 Created` with a `Location` of
  `customers/{id}` is therefore a lie in REST terms — the resource already existed. Anyone reading
  the route list alone will mis-model the lifecycle; this is precisely the kind of claim a surface
  catalogue gets wrong.
- `PUT …/state/{state}` returns `204 No Content` **unconditionally**, including for the silent
  no-op path (§3.4).

**Lifecycle.** Convey binds the route template's segments plus the JSON body onto the command/query
type `[convey]`. `afterDispatch` runs **after** the dispatcher returns without throwing; on an
exception the error handler (§3.18) has already written the response.

**Invariants & enforcement.** No route carries an authorization attribute; no route validates its
payload (§3.12). The only server-side gate is certificate authentication (§3.25).

**Extension procedure.** Add a `.Get<…>` / `.Post<…>` line here (§3.12 step 6), then the gateway
route. Note the ordering: the routes are registered in declaration order and Convey/ASP.NET Core
route matching handles specificity, so `customers/{customerId}` and `customers/{customerId}/state`
coexist safely.

**Failure modes.** A route template segment whose name does not match a command property binds
nothing and yields a default value — surfacing as `customer_not_found` rather than as a 400 for a
malformed request.

### 3.25 Certificate authentication and the `customers:read` ACL

**Definition.** The only server-side access control this service has. `customers-service` is the
**only service on the platform that declares a certificate ACL with named permissions** — a fact
worth stating precisely because it makes this section unique in the inventory.

**Registration** (`…Infrastructure/Extensions.cs:79,91`):

```csharp
.AddCertificateAuthentication()   // in AddInfrastructure
…
.UseCertificateAuthentication()   // in UseInfrastructure, before UseRabbitMq
```

Both are Convey `[convey]`.

**Configuration** (`…Api/appsettings.json:163-182`):

| Key | Value |
| --- | --- |
| `security.certificate.enabled` | `true` |
| `security.certificate.header` | `Certificate` |
| `security.certificate.skipRevocationCheck` | `false` |
| `security.certificate.allowedDomains` | `["pacco.io"]` |
| `security.certificate.allowSubdomains` | `true` |
| `security.certificate.allowedHosts` | `["localhost"]` |
| `security.certificate.acl["availability-service"].validIssuer` | `localhost` |
| `security.certificate.acl["availability-service"].permissions` | `["customers:read"]` |

**How the certificate arrives.** Not as a TLS client certificate on the connection, but as an HTTP
**header** named `Certificate` (`header: "Certificate"`). The caller side is visible in
`availability-service`: `CustomersServiceClient` attaches the certificate using
`securityOptions.Certificate.GetHeaderName()` before calling
`GET {customers-service}/customers/{id}/state`
(`hianshul100_Pacco.Services.Availability/…/Infrastructure/Services/Clients/CustomersServiceClient.cs`;
see `component-internals/availability-service.md` §3.25). The certificate itself is issued by Vault
PKI with common name `customers-service.pacco.io` for this service and the equivalent for callers
(§3.30) — which is why `allowedDomains: ["pacco.io"]` with `allowSubdomains: true` is the matching
rule.

**What `customers:read` actually gates.** The permission string appears **only** in this
configuration file. It does not appear anywhere in `src/` — no attribute, no policy, no handler
reads it. How Convey maps a permission to a route is
**`Unverifiable — Missing Source Evidence`**: Convey has no source in this workspace, and no route
in `…Api/Program.cs` declares a required permission. What can be stated from evidence:

- The ACL names exactly one client — `availability-service`.
- `pricing-service` also calls this service (`GET /customers/{customerId}`) and its
  `CustomersServiceClient` attaches **no certificate header**
  (`hianshul100_Pacco.Services.Pricing/…/Clients/CustomersServiceClient.cs`).

Those two facts are in tension. Either the ACL is advisory (permissions unenforced), or
`pricing-service` would be rejected wherever certificate authentication is on. The resolution is in
§3.31: **certificate authentication is disabled in both the `local` and the `docker` environment
profiles**, which are the only two environments defined in this repository. So in every runnable
configuration in this workspace, the ACL is inert and `pricing-service` works. See Q-2 in §8.

**Lifecycle.** Middleware, evaluated per HTTP request before the dispatcher. AMQP messages bypass it
entirely — **certificate authentication protects the HTTP surface only.** A caller with broker
access can invoke both commands without any certificate.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent |
| --- | --- | --- |
| Only certificates from `*.pacco.io` are accepted | `allowedDomains` + `allowSubdomains` | Convey `[convey]`; rejection status unverifiable |
| Revocation is checked | `skipRevocationCheck: false` | `[convey]` |
| Only `availability-service` is in the ACL | `acl` block | Configuration only |
| The ACL applies in production | **unknown** — no production profile exists in this repository | See §3.31 |

**Extension procedure.** To grant `pricing-service` access:

1. Add an `acl` entry keyed by its service name with the permissions it needs.
2. Ensure `pricing-service`'s HTTP client attaches the certificate header — today it does not, so
   the ACL entry alone changes nothing.
3. Provision a Vault PKI role for `pricing-service` with a `*.pacco.io` common name (§3.30).
4. Enable `security.certificate` in whichever environment profile should enforce it.

**What silently rejects the extension:** doing only step 1. The ACL entry is dead configuration
until the caller sends a certificate and the environment enables the feature.

**Failure modes.**

- The header-based transport means an intermediary that strips unknown headers (a proxy, Fabio)
  silently removes authentication. Whether Fabio forwards the `Certificate` header is
  `Unverifiable — Missing Source Evidence`.
- The ACL grants a *read* permission while the service's mutating routes carry no permission
  declaration at all — so if permissions were enforced, the write routes' required permission is
  undefined.

### 3.26 Inert JWT configuration

**Definition.** A complete, plausible-looking JWT configuration block that **no code in this service
reads**.

**Evidence** (`…Api/appsettings.json:76-84`):

```json
"jwt": {
  "certificate": { "location": "certs/localhost.cer" },
  "validIssuer": "pacco",
  "validateAudience": false,
  "validateIssuer": true,
  "validateLifetime": true
}
```

`appsettings.local.json:26-30` and `appsettings.docker.json:55-59` both blank the certificate
location, and `certs/localhost.cer` **does not exist in this repository**.

Against this: `…Infrastructure/Extensions.cs` calls neither `AddJwt()` nor `UseAuthentication()`;
`…Api/Program.cs` adds only `AddConvey().AddWebApi().AddApplication().AddInfrastructure()`. There is
no `[Authorize]` attribute anywhere (there are no controllers at all — §3.24).

**Why it matters.** A reader skimming configuration will conclude this service validates JWTs. It
does not. Token validation happens **once**, at `api-gateway` (`ntrada.yml` `auth: true`), and the
result is forwarded as an unsigned JSON header (§3.19-§3.20). The identical pattern is documented
for `availability-service` (`component-internals/availability-service.md` §1.2); Customers repeats
it. This is a platform-wide convention, not a per-service oversight — see
`patterns/index.md` for the candidate pattern covering gateway-terminated authentication.

**Extension procedure.** To make this service validate JWTs itself: add `AddJwt()` in
`AddInfrastructure`, `UseAuthentication()` in `UseInfrastructure` before `UseConvey()`, provision the
signing certificate, and decide per-route requirements. Note this would **double-validate** for
gateway traffic and **break** `availability-service`/`pricing-service` service-to-service calls,
which send certificates rather than tokens.

**Failure modes.** Misleading configuration. No runtime effect.

### 3.27 `MessageToLogTemplateMapper` — per-message log phrasing

**Definition.** A hand-maintained dictionary mapping message types to human-readable log lines,
consumed by Convey's handler-logging decorator
(`…Infrastructure/Logging/MessageToLogTemplateMapper.cs:9-41`, registered via
`AddHandlersLogging()` at `…Infrastructure/Extensions.cs:76`).

**The complete map** (`:11-34`):

| Message | `After` template |
| --- | --- |
| `CompleteCustomerRegistration` | `Completed a registration for the customer with id: {CustomerId}.` |
| `ChangeCustomerState` | `Changed a customer with id: {CustomerId} state to: {State}.` |
| `OrderCompleted` | `Order with id: {OrderId} for the customer with id: {CustomerId} has been completed.` |
| `SignedUp` | `Created a new customer with id: {UserId}.` |
| anything else | `null` → **no log line at all** (`:39`) |

**Only `After` is populated.** No `Before` and no `OnError` template is set on any entry, so the
decorator emits a line only on success `[convey]`.

Two consequences worth internalising:

1. **The `ChangeCustomerState` line lies on the silent-no-op path.** When the requested state equals
   the current one, the handler returns early (§3.4) without changing anything — but it returns
   *successfully*, so the decorator logs `Changed a customer with id: … state to: …`. The log claims
   a change that did not happen.
2. **Failures are invisible here.** With no `OnError` template, a handler that throws produces
   whatever Convey's generic exception logging emits, not a domain-phrased line. Combined with §3.16
   (silent rejections) and §3.15 (`LogTrace` publishing below the configured level), **the log is
   the only trace of several failure modes and it is a thin one.**

**Lifecycle.** `MessageTemplates` is an **expression-bodied property** (`=>`, line 12), so a **new
dictionary is allocated on every `Map` call**, i.e. once per handled message. Functionally harmless;
worth knowing if this service ever becomes throughput-sensitive.

**Extension procedure.** Add an entry keyed by the message type. Placeholders in braces are matched
against the message's property names by the logging decorator `[convey]`.

**What silently rejects the extension:** a placeholder that does not match a property name. The
template renders with the literal placeholder text rather than a value; nothing errors.

**Failure modes.** Templates that assert an outcome the handler may not have produced (point 1
above). Missing entries produce no log at all, so a new handler is silent by default.

### 3.28 Log redaction and path exclusion

**Definition.** The logging policy that keeps secrets out of logs and noise out of traces.

**Evidence** (`…Api/appsettings.json:32-68`):

- `logger.level: "information"` — so `LogTrace`/`LogDebug` calls, including
  `MessageBroker.cs:75`'s publish trace, are **not emitted** by default.
- `logger.excludePaths: ["/", "/ping", "/metrics"]` — health and metrics scrapes do not generate
  request logs. The same three paths are excluded from Jaeger tracing
  (`appsettings.json:74`), so the ping endpoint is invisible to both systems.
- `logger.excludeProperties` — a redaction list including `api_key`, `access_key`, `ApiKey`,
  `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`, `Password`, `Email`, `Login`,
  `Secret`, `Token` (`appsettings.json:35-49`; the list is matched on property name).
- Sinks: `console`, `file`, `elk`, `seq` blocks (`appsettings.json:50-68`), each independently
  toggled per environment (§3.31).
- `httpClient.requestMasking.enabled: true` with `maskTemplate: "*****"`
  (`appsettings.json:27-30`) — outbound HTTP request logging is masked.

**Why `Email` in the exclusion list is load-bearing here.** This service is the platform's holder of
customer email addresses (`CustomerDocument.Email`), and `SignedUpHandler` receives an email on
every sign-up. The redaction list is the mechanism that keeps it out of Seq/ELK.

**Invariants & enforcement.** Redaction is by **property name matching**, performed by Convey's
logging setup `[convey]`. A value logged as part of an interpolated string — e.g. inside an
exception `Message` — is **not** redacted. `InvalidRoleException`'s message content is therefore
worth checking before adding user data to exception messages.

**Extension procedure.** Add the property name to `excludeProperties` in `…Api/appsettings.json`.
Note the list mixes casing styles (`api_key` and `ApiKey`), implying the match is not
case-insensitive — `Unverifiable — Missing Source Evidence` (Convey source unavailable); the
defensive move is to add both casings.

**Failure modes.** Secrets embedded in exception messages or in log *templates* rather than
properties bypass redaction entirely. `logger.level: information` hides the publish trace, so the
absence of a publish is undetectable from logs.

### 3.29 Consul registration and Fabio addressing

**Definition.** How other services find this one.

**Evidence** (`…Api/appsettings.json:7-31`; `…Infrastructure/Extensions.cs:67-68`):

| Key | `appsettings.json` | `appsettings.docker.json` |
| --- | --- | --- |
| `consul.enabled` | `true` | `true` |
| `consul.url` | `http://localhost:8500` | `http://consul:8500` |
| `consul.service` | `customers-service` | `customers-service` |
| `consul.address` | `docker.for.win.localhost` | `customers-service` |
| `consul.port` | `5002` | `80` |
| `consul.pingEnabled` / `pingEndpoint` | `true` / `ping` | same |
| `consul.pingInterval` / `removeAfterInterval` | `3` / `3` (seconds) | same |
| `fabio.enabled` / `url` / `service` | `true` / `http://localhost:9999` / `customers-service` | `true` / `http://fabio:9999` / `customers-service` |
| `httpClient.type` | `fabio` | `fabio` |

The base `appsettings.json` `consul.address` value `docker.for.win.localhost` is a
**Docker-for-Windows-specific hostname** — it is the developer-machine default and is overridden in
Docker. It will not resolve on Linux or macOS hosts, so a developer running the service outside
Docker on those platforms registers an unreachable address in Consul. `appsettings.local.json:2-8`
disables Consul and Fabio entirely, which is the intended local workflow (`scripts/start.sh` sets
`ASPNETCORE_ENVIRONMENT=local`).

**`httpClient.services` is empty** (`appsettings.json:26`) — this service makes **no outbound HTTP
calls to other Pacco services.** `AddHttpClient()` is registered (`Extensions.cs:66`) and there is no
client class anywhere in `src/`. Customers is a pure sink for HTTP: it is called, it never calls.

**Lifecycle.** Registration at startup, deregistration after `removeAfterInterval` consecutive
failed pings (3 × 3 s ≈ 9 s of unreachability). The ping endpoint is `GET /ping`, supplied by Convey
`[convey]` — it is **not** in `…Api/Program.cs`, which is why it appears in `excludePaths` (§3.28)
without appearing in the route list (§3.24).

**Extension procedure.** To call another service over HTTP, add an entry under `httpClient.services`
mapping a logical name to a Consul/Fabio service name, inject `IHttpClient` `[convey]`, and follow
`availability-service`'s `CustomersServiceClient` shape
(`component-internals/availability-service.md` §3.25) — including attaching the certificate header
if the callee has an ACL.

**Failure modes.** A wrong `consul.address` registers an unreachable endpoint that Fabio then routes
traffic to; the failure appears in *callers*, not here. Deregistration after ~9 s means a brief GC
pause or a slow startup can drop the service from routing.

### 3.30 Vault — KV settings, PKI certificate, dynamic Mongo credentials

**Definition.** Three distinct Vault integrations, enabled by a single `UseVault()` call
(`…Api/Program.cs:43`).

**Evidence** (`…Api/appsettings.json:183-212`):

| Integration | Configuration | Effect |
| --- | --- | --- |
| **KV** | `kv.enabled: true`, `engineVersion: 2`, `mountPoint: kv`, `path: customers-service/settings` | Secrets fetched at startup are merged into `IConfiguration`, overriding `appsettings.json` `[convey]` |
| **PKI** | `pki.enabled: true`, `roleName: customers-service`, `commonName: customers-service.pacco.io` | Issues this service's own X.509 certificate — the identity it presents when calling others, and the identity `allowedDomains: ["pacco.io"]` (§3.25) is designed to match |
| **Dynamic DB lease** | `lease.mongo.type: database`, `roleName: customers-service`, `enabled: true`, `autoRenewal: true`, `templates.connectionString: mongodb://{{username}}:{{password}}@localhost:27017` | Vault mints short-lived Mongo credentials and the template interpolates them into the connection string, **overriding `mongo.connectionString`** |

Auth is `authType: "token"` with a token, a username and a password committed in
`…Api/appsettings.json:186-189`. **Per this repository's convention, credential material is cited by
path only and no values are reproduced here** (`patterns/index.md`). These are development
placeholders; the production values are expected to come from the environment. That expectation is
itself `Unverifiable — Missing Source Evidence`: there is no production configuration profile in
this repository (§3.31).

**Ordering matters.** `UseVault()` is applied to the **host builder** (`Program.cs:43`), i.e. it
runs during configuration building, before `ConfigureServices` consumes `mongo.connectionString`.
That ordering is what lets the dynamic lease template take effect.

**Lifecycle.** KV read once at startup — **a KV change requires a restart**. PKI certificate issued
at startup with a lease. Mongo credentials issued at startup and renewed automatically
(`autoRenewal: true`). What happens if renewal fails is **`Unverifiable — Missing Source Evidence`**
(no Convey source); the plausible failure is that the service keeps a connection open with expired
credentials until the next reconnect, at which point every database operation fails at once.

**Invariants & enforcement.** None in this repository's code — the whole integration is
configuration plus one `UseVault()` call.

**Extension procedure.** To add a secret: write it to `kv/customers-service/settings` under a key
matching the configuration path you want to override (Vault KV keys are merged into
`IConfiguration`), then read it through the normal options mechanism. To rotate the PKI role, change
`roleName`/`commonName` here **and** the `allowedDomains`/`acl` entries in every service that trusts
this one (§3.25).

**Failure modes.** Vault unreachable at startup with `enabled: true` → startup behaviour is
`Unverifiable — Missing Source Evidence`. Both `local` and `docker` disable Vault entirely
(§3.31), so no environment in this repository actually exercises any of it.

### 3.31 Environment layering — `appsettings.{local,docker,development}.json`

**Definition.** Which behaviours are on in which environment. This table resolves several apparent
contradictions elsewhere in the document.

| Setting | base `appsettings.json` | `local` | `docker` | `development` |
| --- | --- | --- | --- | --- |
| `consul.enabled` | `true` | **`false`** | `true` (url `http://consul:8500`, port `80`) | — |
| `fabio.enabled` | `true` | **`false`** | `true` (`http://fabio:9999`) | — |
| `httpClient.type` | `fabio` | **`""`** | `fabio` | — |
| `logger.level` | `information` | **`verbose`** | — | — |
| `logger.file` / `seq` | per base | both **`false`** | file `false`, **seq `true`** (`http://seq:5341`) | — |
| `logger.console` | per base | — | `true` | — |
| `logger.elk` | per base | — | `false` (`http://elk:9200`) | — |
| `jaeger.enabled` | `true` | **`false`** | `true` (`udpHost: jaeger`, `serviceName: customers`) | — |
| `metrics.enabled` / `prometheusEnabled` | `true` / `true` | **`false` / `false`** | `true` / `true` (`env: docker`) | — |
| `outbox.enabled` | `true` | **`false`** | — (inherits `true`) | — |
| **`security.certificate.enabled`** | **`true`** | **`false`** | **`false`** | — |
| `mongo.connectionString` | `mongodb://localhost:27017` | — | `mongodb://mongo:27017` | — |
| `rabbitMq.hostnames` | `["localhost"]` | — | `["rabbitmq"]` | — |
| `redis.connectionString` | per base | — | `redis` | — |
| `swagger.enabled` | per base | — | `true` (`routePrefix: docs`, `includeSecurity: true`) | — |
| `vault.enabled` (+ kv/pki/lease) | `true` | **all `false`** | **all `false`** (`url: http://vault:8200`) | — |
| `jwt.certificate.location` | `certs/localhost.cer` | `""` | `""` | — |

`appsettings.development.json` is **an empty JSON object** (`{}`) — a placeholder, not a profile.

**The four conclusions that matter:**

1. **The certificate ACL never runs.** `security.certificate.enabled` is `true` only in the base
   file, and **both** concrete environment profiles set it to `false`
   (`appsettings.local.json:38-42`, `appsettings.docker.json:83-87`). Since `Dockerfile:10` sets
   `ASPNETCORE_ENVIRONMENT docker` and `scripts/start.sh:2` sets `local`, **there is no way to run
   this service from this repository with certificate authentication on.** The ACL is aspirational
   configuration. This resolves the §3.25 tension about `pricing-service`.
2. **Vault never runs either** — disabled in both profiles. The `mongo.connectionString` in effect is
   always the static one.
3. **The outbox is off locally**, so local development exercises the direct-publish branch of
   `MessageBroker.cs:83-84` while Docker exercises the outbox branch — different failure semantics
   between the two environments a developer uses.
4. **`local` sets `logger.level: verbose`**, which is the only configuration in which
   `MessageBroker`'s publish trace (§3.15) is visible.

**Extension procedure.** To add a production profile, create `appsettings.production.json`, set
`ASPNETCORE_ENVIRONMENT=production` in the deployment, and — at minimum — re-enable
`security.certificate` and `vault`. Note that enabling the certificate ACL will **break
`pricing-service`'s calls** until that service attaches a certificate (§3.25).

**Failure modes.** Environment-specific behaviour differences that no test covers (§3.36). A
setting present only in the base file is, in practice, a setting that never applies.

### 3.32 Mongo collection naming and the `"customers"` literal

**Definition.** Where the collection name lives and what depends on it.

**Evidence.** One literal: `.AddMongoRepository<CustomerDocument, Guid>("customers")`
(`…Infrastructure/Extensions.cs:77`). The database name is separate:
`mongo.database: "customers-service"` (`…Api/appsettings.json:96`; same in `docker`).

So the full address of customer data is **database `customers-service`, collection `customers`** —
note the naming asymmetry (hyphenated database, plain-plural collection). The outbox and inbox live
in the **same database** as collections `outbox` and `inbox` (`appsettings.json:104-105`).

`mongo.seed: false` (`appsettings.json:97`) — Convey's seeding hook is off and **there is no
`IMongoDbSeeder` implementation in `src/`** anyway.

**Lifecycle.** MongoDB creates the collection lazily on first write. Nothing in this repository
creates it explicitly, and nothing creates indexes (§5.6).

**Invariants & enforcement.** The literal is compile-time; changing it silently points the service
at a different (empty) collection — the service starts cleanly and behaves as if every customer had
vanished.

**Extension procedure.** To make it configurable, bind it from `IConfiguration` before the
`AddMongoRepository` call. To rename it, you must copy the data — there is no migration runner
(§5.5).

**Failure modes.** As above: a rename without a data copy is a silent, total data loss from the
application's point of view.

### 3.33 Redis — configured with no consumer

**Definition.** A registered, configured, and entirely unused caching dependency.

**Evidence.** `.AddRedis()` (`…Infrastructure/Extensions.cs:73`) plus configuration
(`…Api/appsettings.json:150-153`: `connectionString`, `instance: "customers:"`;
`appsettings.docker.json:79-82`: `connectionString: "redis"`, same instance prefix).

Against this: grepping `src/` for `IDistributedCache`, `ICacheClient`, `IConnectionMultiplexer` or
any Redis type returns **nothing outside the registration**. No query handler caches
(§3.13 confirms every read hits Mongo). No service injects a cache.

`availability-service` has exactly the same dead registration
(`component-internals/availability-service.md` §3.35) — this is a platform-wide template artefact,
not a Customers-specific decision.

**Lifecycle.** A Redis connection is established at startup `[convey]` and never used. A Redis
outage therefore has no functional effect on this service, but may affect startup —
`Unverifiable — Missing Source Evidence`.

**Extension procedure.** The natural first use is caching `GetCustomerState`, which
`availability-service` calls on every reservation attempt. Key it on the customer id with the
`customers:` instance prefix, and **invalidate on `CustomerStateChanged`** — noting that the
publisher of that event is this service itself, so invalidation can be done inline in
`ChangeCustomerStateHandler` rather than by round-tripping through the broker. Beware the silent
no-op path (§3.4): it changes nothing, so it needs no invalidation, and it also emits no event —
so an event-driven invalidation strategy is safe here by accident.

**Failure modes.** A dependency that operations must provision, monitor and patch for zero
functional benefit; and a plausible-looking `instance: "customers:"` prefix that suggests caching
exists.

### 3.34 `IDateTimeProvider` — the only clock seam

**Definition.** The single abstraction over "now" (`…Application/Services/IDateTimeProvider.cs`,
implemented by `…Infrastructure/Services/DateTimeProvider.cs`, registered as a singleton at
`…Infrastructure/Extensions.cs:55`).

**Usage.** Exactly one call site: `SignedUpHandler.cs:39` — `_dateTimeProvider.Now` supplies
`Customer.CreatedAt`. Nothing else in the service reads the clock.

**Representation & storage.** `CreatedAt` is a `DateTime` on both the entity (`Customer.cs:18`) and
the document (`CustomerDocument.cs:16`), surfaced on both DTOs
(`Documents/Extensions.cs:30,42`).

**Whether it is UTC.** The property is named `Now`, which in the .NET idiom implies local time. The
implementation is one line in `DateTimeProvider.cs`; whether it returns `DateTime.UtcNow` or
`DateTime.Now` determines whether stored timestamps are comparable across hosts in different time
zones. Consult that file before relying on `CreatedAt` for ordering — and note that MongoDB stores
`DateTime` as UTC milliseconds, so a local-time value is silently shifted on round-trip.

**Lifecycle.** Singleton, stateless.

**Invariants & enforcement.** None. `CreatedAt` is set once at construction and never updated —
there is no `UpdatedAt` anywhere in this service, so **a customer document carries no evidence of
when it was last modified.**

**Extension procedure.** Add `UpdatedAt` by setting it in `AsDocument` from a clock injected into
the mapper (today `Extensions.cs` is a static class with no dependencies, so this requires
restructuring) — or, more simply, set it inside each `Customer` mutator from a value passed in by
the handler.

**Failure modes.** Timezone ambiguity as above; no modification audit trail; the seam exists but,
with no test project (§3.36), it is never actually substituted.

### 3.35 Metrics and tracing wiring

**Definition.** Observability registrations and what they actually produce.

**Evidence.**

| Concern | Registration | Activation | Configuration |
| --- | --- | --- | --- |
| App metrics / Prometheus | `.AddMetrics()` (`Extensions.cs:74`) | `UseMetrics()` (`:90`) | `appsettings.json:85-93`: `prometheusEnabled: true`, `influxEnabled: false`, `database: pacco`, `env: local`, `interval: 5` |
| Jaeger tracing | `.AddJaeger()` (`:75`) | `UseJaeger()` (`:87`) | `appsettings.json:68-75`: `serviceName: customers`, `udpHost`, `sampler: const`, `excludePaths: ["/","/ping","/metrics"]` |
| Jaeger over RabbitMQ | `AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())` (`:69`) | — | — |
| Swagger | `.AddWebApiSwaggerDocs()` (`:78`) | `UseSwaggerDocs()` (`:86`) | `appsettings.docker.json:88-96` (`routePrefix: docs`) |

The Jaeger RabbitMQ plugin is what makes the `span_context` header meaningful — it is the producer
side of the value `MessageBroker` reads back (§3.15 step 3), so a trace started by an HTTP request
continues into the event a handler publishes and into the consuming service.

`sampler: "const"` with the default sampling parameter means **every** request is traced
(`Unverifiable — Missing Source Evidence` for the parameter value, which is not present in the
configuration).

**No custom metrics.** Unlike `availability-service`, which ships a `CustomMetricsMiddleware` and a
`MetricsJob` background service (`component-internals/availability-service.md` §3.29),
`customers-service` has **no `Metrics/` directory and no `BackgroundService` of any kind**. The only
metrics are whatever AppMetrics emits by default `[convey]`: request counts, durations, and process
counters. **There is no metric for VIP promotions, state changes, or customers created.**

Prometheus scrapes it as job `customers-service` targeting host `customers-service`
(`hianshul100_Pacco/compose/prometheus/prometheus.yml:18-20`).

**Extension procedure.** To add a domain metric, inject `IMetricsRoot` `[convey]` into a handler, or
follow `availability-service`'s middleware/background-job pattern. To reduce trace volume, change
`sampler` from `const` to `probabilistic` with a rate.

**Failure modes.** No domain-level observability: the silent failures catalogued throughout this
document (§3.4 no-op, §3.14 unmapped event, §3.16 unmapped exception, §3.11 lost update) would each
be trivially detectable with a counter, and none has one.

### 3.36 Absence of a test suite

**Definition.** A hard negative finding, stated because it changes how every "invariant" in this
document should be read.

**Evidence.** `find . -name '*.csproj'` returns exactly four files, all under `src/`:
`Pacco.Services.Customers.{Api,Application,Core,Infrastructure}.csproj`. There is **no `tests/`
directory** and no test project in `Pacco.Services.Customers.sln`. `scripts/test.sh` exists but has
nothing to run against.

**Why it matters here specifically.** Compare `availability-service`, which ships five test projects
(Unit, Integration, EndToEnd, Performance, Shared —
`component-internals/availability-service.md` §3.36). Customers has none, and it is the service with:

- a hard-coded business threshold (`20`, §3.5),
- an enum persisted by ordinal (§3.10),
- three hand-written mapping functions with no compiler pressure (§3.9),
- two hand-written exception-mapping switches with silent fallbacks (§3.16, §3.18),
- and no optimistic concurrency (§3.11).

Every one of those is a change that compiles cleanly and fails silently at runtime. **Nothing in
this repository would catch any of them.**

**Extension procedure.** The highest-value first tests, in order:

1. `VipPolicy` — pure, dependency-free, pins the threshold and the `HashSet` semantics
   (`…Core/Services/VipPolicy.cs`).
2. `Customer` state transitions and `CompleteRegistration` guards — pure domain
   (`…Core/Entities/Customer.cs`).
3. A round-trip test over `AsDocument`/`AsEntity` — catches the dropped-field failure of §3.9.
4. `EventMapper.Map` exhaustiveness — assert that every `IDomainEvent` implementation in the Core
   assembly maps to a non-null integration event; this is the test that would prevent §3.14's
   silent-drop class of bug permanently.
5. `ExceptionToMessageMapper` — assert every exception type in `src/` produces a non-null message for
   at least one command; today this test would fail on three exceptions (§3.16).

**Failure modes.** Every claim in this document is derived from reading code, not from an executable
specification. Where behaviour and this document disagree, the code is authoritative — and nothing
enforces that they agree.

### 3.37 Deployment identity — ports, images, process names

**Definition.** How this component is named and reached in each runtime topology.

| Facet | Value | Evidence |
| --- | --- | --- |
| Container image | `devmentors/pacco.services.customers` | `hianshul100_Pacco/compose/services.yml:25` |
| Container name / DNS name | `customers-service` | `compose/services.yml:26` |
| Host port → container port | `5002 → 80` | `compose/services.yml:28-29` |
| Docker network | `pacco` | `compose/services.yml:30-31` |
| Restart policy | `unless-stopped` | `compose/services.yml:27` |
| In-container environment | `docker` | `Dockerfile:10` (`ENV ASPNETCORE_ENVIRONMENT docker`) |
| In-container URL binding | `http://*:80` | `Dockerfile:9` |
| Entry point | `dotnet Pacco.Services.Customers.Api.dll` | `Dockerfile:11` |
| Local dev environment | `local`, `dotnet run` from the Api project | `scripts/start.sh:2-4` |
| Consul-advertised port | `5002` (base) / `80` (docker) | `appsettings.json:12`; `appsettings.docker.json:11` |
| Prometheus job / target | `customers-service` / `customers-service` | `compose/prometheus/prometheus.yml:18-20` |
| CI image repository | `$DOCKER_USERNAME/pacco.services.customers`, tags `latest`(master) / `dev`(develop) + build number | `scripts/dockerize.sh:5-20` |
| Build | .NET Core SDK 3.1 multi-stage → aspnet 3.1 runtime | `Dockerfile:1-8` |

**The port asymmetry.** In the base profile the service listens on `5002` and advertises `5002`; in
Docker it listens on `80` inside the container, advertises `80` to Consul, and is published on the
host as `5002`. Anything reaching it **through Consul/Fabio** must use the container port; anything
reaching it from the developer's host uses `5002`.

**Lifecycle.** Built by Travis (`scripts/dockerize.sh` keys off `$TRAVIS_BRANCH`), pushed to Docker
Hub, started by compose. There is no Kubernetes manifest and no Helm chart in any clone.

**Extension procedure.** Changing the container port requires updating `Dockerfile:9`,
`appsettings.docker.json:11` (the Consul-advertised port) and `compose/services.yml:28-29` together;
changing only one silently breaks either routing or host access.

**Failure modes.** `restart: unless-stopped` with a ~9 s Consul deregistration window (§3.29) means a
crash-loop cycles the service in and out of routing rather than removing it cleanly.

---

## 4. Primary control flows

Each flow is traced from the true entry point to the last durable side effect, naming every file it
passes through.

### 4.1 Sign-up → customer creation (the implicit creation path)

**Trigger.** `identity-service` publishes `SignedUp` to exchange `identity`.

1. RabbitMQ routes the message to queue `customers-service/identity.signed_up`
   (§3.22), bound because `SignedUp` carries `[Message("identity")]`
   (`…Application/Events/External/SignedUp.cs:7`) and `Extensions.cs:95` subscribes it.
2. Convey deserializes into this service's local `SignedUp` class and resolves
   `IEventHandler<SignedUp>` `[convey]`.
3. `OutboxEventHandlerDecorator<SignedUp>` intercepts. `messageProperties.MessageId` **is** present
   (this is a real broker message), so the inbox key is the broker's id and redelivery within
   `expiry: 3600` seconds is skipped (§3.17).
4. `SignedUpHandler.HandleAsync` (`:26-41`):
   a. `@event.Role != "user"` → `InvalidRoleException` → **§3.16: unmapped → no rejected event → the
   failure is log-only.**
   b. `_customerRepository.GetAsync(@event.UserId)` → `CustomerMongoRepository.cs:19-24` →
   `IMongoRepository.GetAsync(o => o.Id == id)` → `document?.AsEntity()`. Non-null →
   `CustomerAlreadyCreatedException` → **also unmapped, also silent.**
   c. `new Customer(@event.UserId, @event.Email, _dateTimeProvider.Now)` (`Customer.cs:26-29`) →
   `State.Incomplete`, `IsVip = false`, empty name/address, empty completed-orders set.
   d. `AddAsync` → `AsDocument` (`Documents/Extensions.cs:12-23`) → insert into
   `customers-service.customers`.
5. **The handler returns.** No `IEventMapper`, no `IMessageBroker`, no publish.
6. `MessageToLogTemplateMapper` emits `Created a new customer with id: {UserId}.` (§3.27) — the only
   externally visible trace of the event.

**End state.** A customer exists, is `Incomplete`, and **no other service on the platform knows**.
`availability-service`, `orders-service` and `parcels-service` learn about this customer only after
flow 4.2.

### 4.2 `POST /customers` — complete registration (HTTP)

**Trigger.** A signed-in user submits their name and address. The gateway route is
`POST /customers` → `customers-service/customers`, `auth: true`, **binding
`customerId:@user_id`** (`ntrada.yml:162-170`) — so a user can only ever complete *their own*
registration; the service never checks this, the binding enforces it.

1. Gateway validates the JWT, builds the correlation context, forwards to
   `customers-service/customers` with the body plus the bound `customerId`. Note
   `payload: create_customer` / `schema: create_customer.schema` reference files that **do not exist
   in the gateway repository** (`component-internals/api-gateway.md` §3.10, Q-1), so **the body is
   unvalidated at the edge**.
2. `UseDispatcherEndpoints` matches `Post<CompleteCustomerRegistration>("customers")`
   (`…Api/Program.cs:38`); Convey binds the JSON body to the command `[convey]`.
3. `OutboxCommandHandlerDecorator<CompleteCustomerRegistration>` intercepts. There is no inbound
   broker message, so `_messageId = Guid.NewGuid()` — **the inbox de-duplicates nothing on this
   path** (§3.17).
4. `CompleteCustomerRegistrationHandler.HandleAsync` (`:26-44`):
   a. `GetAsync` → `null` → `CustomerNotFoundException` → HTTP **400** `customer_not_found` (§3.18).
   b. `State is State.Valid` → `CustomerAlreadyRegisteredException` → 400
   `customer_already_registered`.
   c. `customer.CompleteRegistration(FullName, Address)` (`Customer.cs:44-65`) — blank name → 400
   `invalid_customer_fullname`; blank address → 400 `invalid_customer_address`; state not
   `Incomplete` → 400 `cannot_change_customer_state`. On success: `FullName`, `Address` assigned,
   `State = State.Valid`, `CustomerRegistrationCompleted` buffered.
   d. `UpdateAsync` → whole-document replace by `_id` (§3.11).
   e. `MapAll(customer.Events).ToArray()` → `EventMapper.cs:19` → `Application.Events.CustomerCreated(id)`.
   f. `MessageBroker.PublishAsync` (§3.15) → outbox (`docker`) or direct publish (`local`) →
   routing key `customer_created` on exchange `customers`.
5. `afterDispatch` writes **201 Created** with `Location: customers/{customerId}`
   (`…Api/Program.cs:39`).
6. Downstream: `availability-service`, `orders-service` and `parcels-service` each consume
   `customer_created` on their own queue (§3.14).

**Failure asymmetry.** All five failures above are 400s with distinct `code` values, so the HTTP
caller is well informed. The same command over AMQP (flow 4.4) produces
`complete_customer_registration_rejected` for all five — correct here, because they all genuinely
belong to this command.

### 4.3 `PUT /customers/{customerId}/state/{state}` — change state (HTTP)

**Trigger.** An admin changes a customer's state. The gateway requires `claims: role: admin`
(`ntrada.yml:172-181`) and binds both `customerId` and `state` from the path.

1. Route matched at `…Api/Program.cs:40`; both properties bound from route values, **empty body**.
2. Outbox decorator: fresh GUID, no de-duplication.
3. `ChangeCustomerStateHandler.HandleAsync` (`:26-65`):
   a. `GetAsync` → `null` → `CustomerNotFoundException` → 400.
   b. `Enum.TryParse<State>(command.State, true, out var state)` fails → 
   `CannotChangeCustomerStateException(customer.Id, State.Unknown)` → 400
   `cannot_change_customer_state`.
   c. **`if (customer.State == state) return;`** → the handler completes successfully having done
   nothing. `afterDispatch` writes **204 No Content**; no event is published; the log line claims a
   change occurred (§3.27).
   d. The `switch` calls `SetValid` / `SetIncomplete` / `MarkAsSuspicious` / `Lock`; each funnels
   into `SetState` (`Customer.cs:75-80`) which buffers `CustomerStateChanged(this, previousState)`.
   `"unknown"` reaches `default:` → 400.
   e. `UpdateAsync`, `MapAll` → `EventMapper.cs:21-23` → `CustomerStateChanged(id, current, previous)`
   with both states lowercased → publish on routing key `customer_state_changed`.
4. **Nobody is listening.** No handler for `customer_state_changed` exists in any clone (§3.14).
   The event's only consumer is the operations UI's message catalogue.

### 4.4 AMQP command consumption — the same handlers, a different envelope

**Trigger.** The gateway's async profile (`ntrada-async.yml`) publishes to exchange `customers` with
routing key `complete_customer_registration` or `change_customer_state`, and returns **202 Accepted**
to the client immediately.

1. Message lands on `customers-service/customers.<routing_key>` (§3.22).
2. Convey deserializes to the command type and resolves the same `ICommandHandler<T>` as the HTTP
   path — **the handler code is identical; only the envelope differs.**
3. The outbox decorator now has a real `MessageId`, so **de-duplication is live on this path**.
4. `IAppContext` would be built from the broker's `message_context` header rather than the
   `Correlation-Context` HTTP header (`AppContextFactory.cs:21-28`) — though nothing consumes it
   (§3.19).
5. On success: identical persistence and publication as flows 4.2/4.3.
6. On failure: `ExceptionToMessageMapper` (§3.16) decides. For `CompleteCustomerRegistration` every
   possible failure maps correctly. **For `ChangeCustomerState`, only
   `CannotChangeCustomerStateException` maps correctly**; `CustomerNotFoundException` produces a
   `complete_customer_registration_rejected` — a rejection naming a command the caller never sent.
7. The caller's only feedback channel is `operations-service`, which it polls with the operation id
   returned in the 202.

### 4.5 `OrderCompleted` → VIP promotion

**Trigger.** `orders-service` publishes `OrderCompleted` to exchange `orders`.

1. Delivered to `customers-service/orders.order_completed`; inbox de-duplication is live.
2. `OrderCompletedHandler.HandleAsync` (`:27-42`):
   a. `GetAsync(@event.CustomerId)` → `null` → `CustomerNotFoundException` → mapped to
   **`CompleteCustomerRegistrationRejected`** (§3.16) — a rejection for a command, emitted in
   response to an event.
   b. `var isVip = customer.IsVip;` (kept only for the dead local at line 38).
   c. `AddCompletedOrder(@event.OrderId)` — `Guid.Empty` silently dropped; duplicate id silently
   deduped by the `HashSet` (§3.6).
   d. `VipPolicy.ApplyVipStatusIfEligible` — if not already VIP and `CompletedOrders.Count() >= 20`,
   `SetVip()` buffers `CustomerBecameVip` (§3.5).
   e. `UpdateAsync` — **always**, even when nothing changed.
   f. `MapAll` → `EventMapper.cs:20` → `customer_became_vip`, published — **to no consumer** (§3.14).
3. Log: `Order with id: {OrderId} for the customer with id: {CustomerId} has been completed.`

**The whole VIP feature, end to end, is observable only through
`GET /customers/{customerId}` — an admin-gated route.**

### 4.6 `GET /customers/{customerId}/state` — the read path `availability-service` depends on

**Trigger.** Either an admin through the gateway (`ntrada.yml:154-160`, `role: admin`) **or**
`availability-service` calling service-to-service with a Vault PKI certificate header
(§3.25).

1. Route matched at `…Api/Program.cs:37`; Convey binds `customerId` and dispatches `GetCustomerState`.
2. `GetCustomerStateHandler.cs:20-31` queries `IMongoRepository<CustomerDocument, Guid>` directly —
   **no aggregate, no domain rules** — and returns `null` for a missing customer, else
   `{ Id, State = document.State.ToString().ToLowerInvariant() }`.
3. `availability-service` interprets the result: a state other than the one it expects (or a null
   response) blocks the reservation
   (`component-internals/availability-service.md` §3.26).

This is the **only synchronous cross-service dependency on this component's data**, and it is a
single-field read. `pricing-service` additionally calls `GET /customers/{customerId}` (the full
details projection, §3.13) without a certificate.

### 4.7 Startup composition

`…Api/Program.cs:23-45`, in order:

1. `WebHost.CreateDefaultBuilder(args)` — loads `appsettings.json` then
   `appsettings.{ASPNETCORE_ENVIRONMENT}.json` (§3.31).
2. `.ConfigureServices(… AddConvey().AddWebApi().AddApplication().AddInfrastructure().Build())` —
   `AddApplication()` registers command/event handlers and the in-memory dispatchers;
   `AddInfrastructure()` (`…Infrastructure/Extensions.cs:50-81`) registers everything else, ending
   with `AddMongoRepository<CustomerDocument, Guid>("customers")`, `AddWebApiSwaggerDocs()`,
   `AddCertificateAuthentication()`, `AddSecurity()`.
3. `.Configure(app => app.UseInfrastructure().UseDispatcherEndpoints(…))` —
   `UseInfrastructure()` (`Extensions.cs:83-99`) orders the middleware:
   `UseErrorHandler` → `UseSwaggerDocs` → `UseJaeger` → `UseConvey` → `UsePublicContracts` →
   `UseMetrics` → `UseCertificateAuthentication` → `UseRabbitMq` → four `Subscribe*` calls.
   **`UseErrorHandler` is first**, which is why every exception — including one thrown by
   certificate authentication — is shaped by `ExceptionToResponseMapper` (§3.18).
4. `.UseLogging()` — Serilog configuration from the `logger` section (§3.28).
5. `.UseVault()` — configuration-layer secrets and the dynamic Mongo lease (§3.30); disabled in both
   environments.

**Subscription happens at `Configure` time, not `ConfigureServices` time**, which is why the queues
are declared only once the HTTP pipeline is built.

---

## 5. Persistence & schema evolution

### 5.1 What is stored, and where

| Store | Database | Collection | Written by | Read by |
| --- | --- | --- | --- | --- |
| MongoDB | `customers-service` | `customers` | `CustomerMongoRepository.{AddAsync,UpdateAsync}` | `CustomerMongoRepository.GetAsync`; all three query handlers directly |
| MongoDB | `customers-service` | `outbox` | Convey outbox `[convey]` | Convey outbox processor |
| MongoDB | `customers-service` | `inbox` | `Outbox*HandlerDecorator` via `_outbox.HandleAsync` | same |
| Redis | — | — | **nothing** (§3.33) | nothing |

Connection: `mongodb://localhost:27017` (base) / `mongodb://mongo:27017` (docker), or a
Vault-leased credentialed string where Vault is enabled — which is nowhere (§3.30, §3.31).

### 5.2 The `customers` document shape

```json
{
  "_id":            "<Guid>            — identity-service user id",
  "Email":          "<string>          — from SignedUp, never updated",
  "FullName":       "<string>          — empty until registration completes",
  "Address":        "<string>          — empty until registration completes",
  "IsVip":          "<bool>            — derived, never commanded",
  "State":          "<int>             — enum ordinal: 0 Unknown, 1 Valid, 2 Incomplete, 3 Suspicious, 4 Locked",
  "CreatedAt":      "<DateTime>        — set once at construction",
  "CompletedOrders":"[<Guid>, …]       — unbounded embedded array, distinct"
}
```

Field names are the C# property names — there is **no** `[BsonElement]` attribute and no
camel-casing convention registered in `src/`, so the driver's default (PascalCase, matching the
property) applies. `Unverifiable — Missing Source Evidence`: whether Convey's `AddMongo()` registers
a global convention pack that alters this cannot be determined without Convey's source.

There is **no `Version` field, no `UpdatedAt`, no soft-delete flag and no schema-version marker.**

### 5.3 The three mapping functions

Covered in §3.9. Restated as a rule: `AsDocument` and `AsEntity` must stay mutually exhaustive, and
nothing enforces it. `AsDto` (3 of 8 fields) and `AsDetailsDto` (8 of 8) define the two public
projections.

### 5.4 The enum ordinal — the schema decision with the widest blast radius

Covered in §3.10. Restated as a rule for anyone touching `State.cs`:

> **Append only. Never reorder. Never delete. Never renumber.**
> Every stored customer's state is an integer whose meaning is positional.

### 5.5 How to evolve the schema safely

**There is no migration framework in this repository, and none in any sibling service in this
workspace** — no `Migrations/` directory, no `IMongoDbSeeder` implementation, no startup script that
transforms documents. `mongo.seed: false` (`appsettings.json:97`) confirms even Convey's seeding
hook is off. Recorded platform-wide in `patterns/index.md`.

Consequently, evolution must be **additive and backward-compatible**:

| Change | Safe? | Procedure |
| --- | --- | --- |
| Add a nullable/defaultable field | **Yes** | Add to `CustomerDocument`, both mappers, and DTOs. Existing documents read the CLR default (`null`, `0`, `false`). |
| Add an enum member at the end | **Yes** | §3.4 extension procedure. |
| Rename a field | **No** | Old documents keep the old key; the new property reads its default. Requires an offline `$rename`. |
| Reorder/remove an enum member | **No** | Silent data corruption (§3.10). |
| Change a field's type | **No** | Deserialization fails at read time for untouched documents. |
| Rename the collection or database | **No** | Silent "all customers vanished" (§3.32). |
| Add `Version` for concurrency | **Yes, with care** | §3.11 extension procedure; missing field defaults to `0`. |
| Remove a field | **Tolerable** | Drop it from the document class and both mappers; the stored key is simply ignored by the driver and lingers forever. |

**The rollback consideration.** Because writes are whole-document replaces (§3.11), a deployment that
adds a field, runs, and is then rolled back will find that the **old code's `AsDocument` omits the
new field, so the next write of any customer deletes it.** Additive changes are forward-safe but not
rollback-safe.

### 5.6 Indexes

**No index is created by this repository.** There is no `CreateIndex` call, no index attribute, and
no initialization script anywhere in `src/`. MongoDB's automatic `_id` index is therefore the only
one that exists.

Consequences for each access path:

| Query | Predicate | Index used |
| --- | --- | --- |
| `GetAsync(o => o.Id == id)` (all lookups) | `_id` | **`_id`** — efficient |
| `GetCustomersHandler` `FindAsync(_ => true)` | none | full collection scan by definition (§3.13) |

Because every single-document access is by `_id`, the missing indexes cost nothing today. The first
query added on any other field (e.g. `Email`, or `State` for an admin filter) will be a collection
scan, and nothing in the repository will say so.

`Unverifiable — Missing Source Evidence`: whether Convey's outbox creates indexes on the `outbox` /
`inbox` collections cannot be determined without Convey's source; the `expiry: 3600` setting implies
a TTL mechanism, which would normally require a TTL index.

---

## 6. Surface → internals map

For each externally visible surface: what internal mechanism it actually drives, and whether it is
read-only, mutating, or absent.

### 6.1 HTTP endpoints

| Endpoint | Kind | Internal mechanism driven | Gateway gate |
| --- | --- | --- | --- |
| `GET /` | Read-only | Returns `AppOptions.Name` from an inline lambda (`Program.cs:34`) — no repository, no dispatcher | not exposed via `ntrada.yml` |
| `GET /customers` | **Read-only** | `GetCustomers` → `GetCustomersHandler.cs:24` → `FindAsync(_ => true)` → **unpaginated full-collection scan** → `AsDto` (3 of 8 fields) | `auth: true` + `claims: role: admin` (`ntrada.yml:135-138`) |
| `GET /customers/{customerId}` | **Read-only** | `GetCustomer` → `GetCustomerHandler.cs:22` → `_id` lookup → `AsDetailsDto` (all 8 fields, **including the full completed-order id array and the email**) | `role: admin` (`ntrada.yml:146-152`); also called by `pricing-service` without a certificate |
| `GET /customers/{customerId}/state` | **Read-only** | `GetCustomerState` → `GetCustomerStateHandler.cs:22-30` → `_id` lookup → `{Id, state}` lowercased | `role: admin` (`ntrada.yml:154-160`); also called by `availability-service` **with** a PKI certificate |
| `POST /customers` | **Mutating** | `CompleteCustomerRegistration` → `Customer.CompleteRegistration` → `State: Incomplete → Valid` → publishes **`customer_created`**. Despite the verb and the `201 Created`, **it creates nothing** (§3.24) | `auth: true`, `bind: customerId:@user_id`, `payload`/`schema` files **missing** (`ntrada.yml:162-170`) |
| `PUT /customers/{customerId}/state/{state}` | **Mutating** | `ChangeCustomerState` → `SetState` → publishes `customer_state_changed`; **silently no-ops** when already in the target state, still returning 204 | `role: admin`, binds both path values (`ntrada.yml:172-181`) |
| `GET /me` (gateway route) | **Read-only, gateway-only** | Rewrites to `customers-service/customers/@user_id` — i.e. it drives the **`GET /customers/{customerId}`** mechanism; the service has no `/me` concept (`ntrada.yml:140-144`) | `auth: true`, no admin claim |

### 6.2 Non-route HTTP surface (middleware, not `Program.cs`)

| Surface | Provided by | Mechanism |
| --- | --- | --- |
| `GET /ping` | Convey `[convey]` | Consul health check target (`consul.pingEndpoint: ping`); excluded from logs and traces (§3.28) |
| `GET /metrics` | AppMetrics/Prometheus `[convey]` | Scraped as job `customers-service` (`compose/prometheus/prometheus.yml:18-20`) |
| Swagger UI at `/docs` | `UseSwaggerDocs()` | Enabled in `docker` (`appsettings.docker.json:88-96`) |
| Public contracts listing | `UsePublicContracts<ContractAttribute>()` | Lists the two `[Contract]` commands (§3.23); route unverifiable |

### 6.3 AMQP surface — consumed

| Exchange | Routing key | Queue | Drives |
| --- | --- | --- | --- |
| `customers` | `complete_customer_registration` | `customers-service/customers.complete_customer_registration` | Same handler as `POST /customers`; **de-duplicated** by the inbox |
| `customers` | `change_customer_state` | `customers-service/customers.change_customer_state` | Same handler as the `PUT` route |
| `identity` | `signed_up` | `customers-service/identity.signed_up` | **The only customer-creation path** (§4.1); publishes nothing |
| `orders` | `order_completed` | `customers-service/orders.order_completed` | `AddCompletedOrder` + `VipPolicy` (§4.5) |

### 6.4 AMQP surface — published

| Routing key | Produced from | Payload | Consumers |
| --- | --- | --- | --- |
| `customer_created` | `CustomerRegistrationCompleted` (registration completion, **not** creation) | `Id` only | `availability-service`, `orders-service`, `parcels-service` |
| `customer_state_changed` | `CustomerStateChanged` | `Id`, `State`, `PreviousState` (both lowercase) | **none** |
| `customer_became_vip` | `CustomerBecameVip` | `Id` only | **none** |
| `complete_customer_registration_rejected` | 4 exception types, **command-agnostic** | `CustomerId`, `Reason`, `Code` | `operations-service` |
| `change_customer_state_rejected` | `CannotChangeCustomerStateException` **only when the inbound message was `ChangeCustomerState`** | `CustomerId`, `State`, `Reason`, `Code` | `operations-service` |

### 6.5 Outbound calls

**None.** `httpClient.services` is empty (`appsettings.json:26`) and there is no client class in
`src/`. This service calls no other service, synchronously or otherwise; its only egress is event
publication.

### 6.6 Absent surfaces

Surfaces a reader might reasonably expect, which **do not exist**:

| Expected | Reality |
| --- | --- |
| `DELETE /customers/{id}` | No delete endpoint, no repository delete, no soft-delete field (§1.2) |
| An endpoint to create a customer | Creation is event-driven only (§4.1) |
| An endpoint to set/clear VIP | VIP is derived only (§3.5) |
| Pagination on `GET /customers` | `GetCustomers` has no properties (§3.13) |
| A 404 for an unknown customer | Reads return `null`; writes return **400** (§3.18) |
| A 5xx for an internal error | Everything is 400 (§3.18) |
| Filtering customers by state or email | No such query exists; would be a collection scan (§5.6) |
| JWT validation | Configured but inert (§3.26) |
| Authorization inside the service | `IAppContext` built and unused (§3.19) |
| Caching | Redis registered and unused (§3.33) |
| Tests | None (§3.36) |
| Migrations | None (§5.5) |

---

## 7. Change/extension guide

### 7.1 Add a new command

Follow §3.12's nine steps. The three that fail silently if skipped: the `EventMapper` case, the
`ExceptionToMessageMapper` arm, and the `SubscribeCommand<T>()` call.

### 7.2 Add or change a business rule

Rules live in `…Core/Entities/Customer.cs` (invariants) and `…Core/Services/VipPolicy.cs` (the
one policy). Put the rule in `Core`, raise a domain event if it is externally visible, map the event
(§3.14), and map any new exception in **both** mappers (§3.16, §3.18). `Core` has no configuration
access, so a configurable rule needs the §3.5 restructuring.

### 7.3 Add a state to the state machine

Follow §3.4's five steps. The two traps: **append the enum member, never insert** (§3.10), and add
the `switch` arm in `ChangeCustomerStateHandler.cs:44-60` or the new state will report itself as
*invalid* rather than unimplemented.

### 7.4 Subscribe to a new external event

Follow §3.21's five steps. The silent trap is a wrong exchange name in `[Message("…")]` — the queue
binds to the wrong exchange and stays empty forever with no error.

### 7.5 Make the service safe under concurrency

Follow §3.11's four steps. Do all four; steps 1-2 without step 3 reproduce
`deliveries-service`'s dead version field.

### 7.6 Change the persistence shape

Follow §5.5. Additive changes only, and remember they are **not rollback-safe** (§5.5).

### 7.7 Introduce real authorization

Follow §3.19's four steps, then decide whether §3.18 should gain a 403 arm — today an unauthorized
caller would receive 400. Note that the gateway is currently the only enforcement point and that
`GET /me` (`ntrada.yml:140-144`) is the only non-admin read route.

### 7.8 Make failures observable

The highest-value changes, in order:

1. Add the missing `ExceptionToMessageMapper` arms for `InvalidRoleException` and
   `CustomerAlreadyCreatedException` (§3.16) — today the only two failures the creation path can
   produce are invisible.
2. Nest `CustomerNotFoundException` by message type so `ChangeCustomerState` produces
   `change_customer_state_rejected` (§3.16).
3. Add a counter for the silent no-op branch in `ChangeCustomerStateHandler.cs:39-42` and remove or
   act on the dead `vipApplied` local in `OrderCompletedHandler.cs:38` (§3.5).
4. Add an `EventMapper` exhaustiveness test (§3.36 step 4).
5. Add a 500 arm to `ExceptionToResponseMapper` so 5xx alerting can work (§3.18).

### 7.9 The maintenance contract

**Any later phase that changes this component's internals must update this document in the same
change.** In particular: a new command, event, state, exception, configuration key, mapping function,
or persistence field invalidates a specific section above — update §2's concept table and the
affected §3 subsection together, and re-check §6's surface map.

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

| # | Assumption | Basis | Impact if wrong |
| --- | --- | --- | --- |
| A-1 | Convey 0.4.x's `IMongoRepository.UpdateAsync(document)` replaces the whole document by `_id` | The repository passes no filter (`CustomerMongoRepository.cs:27`) and no other overload is used | §3.11's lost-update analysis would change if Convey performs a partial update |
| A-2 | The MongoDB driver serializes `State` as its integer ordinal | No `[BsonRepresentation]`, no serializer registration, no convention pack in `src/` | §3.10 and §5.4 would not apply; enum reordering would be safe |
| A-3 | `SubscribeCommand`/`SubscribeEvent` declare durable queues named per `queue.template` | The template is configured and the platform-wide convention matches (§3.22) | The topology table in §6.3 would be wrong |
| A-4 | Convey's error handler returns the mapped status for exceptions escaping the dispatcher | `UseErrorHandler()` is first in the pipeline (`Extensions.cs:85`) | §3.18's status-code table would be wrong |
| A-5 | `identity-service` emits `SignedUp.Role` as exactly `"user"` for customers | `SignedUpHandler.cs:28` is an ordinal comparison and customers demonstrably exist on the platform | Every sign-up would fail silently (§3.21) |
| A-6 | Deployments use `ASPNETCORE_ENVIRONMENT=docker` (container) or `local` (dev) | `Dockerfile:10`, `scripts/start.sh:2` | If some other environment name is used in production, §3.31's conclusions about the disabled ACL and Vault do not hold there |

### 8.2 Blockers

| # | Blocker | Effect on this document |
| --- | --- | --- |
| B-1 | **Convey 0.4.x has no source in this workspace** (NuGet only) | Every `[convey]` mechanism — dispatcher binding, Mongo repository semantics, outbox processing, certificate authentication, queue declaration, the null-result HTTP status, the contracts endpoint route — is described from its call sites and configuration, not from its implementation |
| B-2 | **No test suite in this repository** (§3.36) | No executable specification exists to confirm any behavioural claim; all claims are derived by reading code |
| B-3 | **No production configuration profile** | §3.25 and §3.30 describe capabilities that no environment in this workspace enables |
| B-4 | `operations-service` internals are out of this batch's scope | How a rejected event is correlated back to an async caller's operation id is asserted only from `messages.json`, not from that service's code |

### 8.3 Open questions

| # | Question | Why it matters | Where the answer would come from |
| --- | --- | --- | --- |
| Q-1 | Does Convey enforce `security.certificate.acl` permissions per route, or is the permission string purely advisory? | `customers:read` appears only in configuration and nowhere in `src/`; if it is enforced, the write routes have **no** declared permission (§3.25) | Convey 0.4.x source |
| Q-2 | Why does `pricing-service` call `GET /customers/{customerId}` without a certificate when only `availability-service` is in the ACL? | Either the ACL is inert (which §3.31 shows it currently is, in both environments) or `pricing-service` breaks the moment certificate authentication is enabled | A production profile, or a decision record — **none exists in any clone** (`patterns/index.md`) |
| Q-3 | Is `customer_state_changed` / `customer_became_vip` intended to have consumers? | Two of three published events have none (§3.14). Either they are speculative, or a consumer is missing | Product/architecture intent; no artefact in the workspace records it |
| Q-4 | Is the VIP threshold of `20` a business constant or a placeholder? | It is hard-coded in `Core` with no configuration seam (§3.5) | Business stakeholders; no requirement document exists in this workspace |
| Q-5 | Should `POST /customers` be renamed, given that it completes rather than creates? | The route name actively misleads (§3.24, §6.1), and `customer_created` fires from it | An API-versioning decision; changing it breaks `ntrada.yml` and every client |
| Q-6 | Is the `ChangeCustomerState` silent no-op (`ChangeCustomerStateHandler.cs:39-42`) intended? | It returns 204 and publishes nothing, so an async saga waiting on `customer_state_changed` hangs (§3.4) | Original author intent; no comment or test records it |
| Q-7 | What is the intended behaviour when `SignedUp` carries a non-`user` role? | Today: a silent, permanent gap between `identity-service` and this service (§3.21) | `identity-service`'s role model — out of this batch's scope |
| Q-8 | Does `DateTimeProvider.Now` return UTC? | Determines whether `CreatedAt` is comparable across hosts (§3.34) | `…Infrastructure/Services/DateTimeProvider.cs` — one line, but its semantics are a platform-wide convention question |

### 8.4 Cross-references

- `component-internals/api-gateway.md` — the upstream half of every HTTP and async contract,
  including the missing `create_customer` payload/schema files (its §3.10 / Q-1).
- `component-internals/availability-service.md` — the sibling that *does* have optimistic
  concurrency (§3.11 there), *does* consume `customer_created`, and *does* call this service's
  `GET /customers/{id}/state` with a PKI certificate (its §3.25-§3.26).
- `baselines/service-summaries.md` — the surface catalogue this document complements.
- `repo-summary/Pacco.Services.Customers.md` — repository-level inventory.
- `patterns/index.md` — the platform-wide candidate-pattern catalogue.

**Patterns this component instantiates** (each links to `patterns/<area>/<file>.md`):

| Pattern | Where it shows up here |
| --- | --- |
| [[inward-dependency-service-skeleton]] | Four-project layering; `Core` references nothing (§1, §3.5) |
| [[aggregate-buffered-domain-events]] | `AggregateRoot._events` + `MapAll(customer.Events)` (§3.2, §3.7) |
| [[dispatcher-bound-cqrs-endpoints]] | All six routes declared in `UseDispatcherEndpoints` (§3.24) |
| [[dual-mode-edge-write]] | Both commands over HTTP and AMQP with different failure semantics (§1.3, §4.4) |
| [[service-owned-topic-exchange-messaging]] | Exchange `customers`, queue template `customers-service/{{exchange}}.{{message}}` (§3.22) |
| [[declarative-message-manifest-subscription]] | `[Message("identity")]` / `[Message("orders")]` + Operations' `messages.json` (§3.21) |
| [[rejected-event-failure-contract]] | `ExceptionToMessageMapper` → `*_rejected` events — **instantiated with three defects** (§3.16) |
| [[transactional-outbox-handler-decorator]] | `OutboxCommandHandlerDecorator` / `OutboxEventHandlerDecorator` (§3.17) |
| [[database-per-service-with-document-mapping]] | DB `customers-service`, hand-written `AsEntity`/`AsDocument`/`AsDto` (§3.9, §5) |
| [[event-carried-reference-replica]] | `customer_created` carries only an id; consumers hold a bare reference (§3.14) |
| [[narrow-synchronous-point-read]] | `GET /customers/{id}/state` consumed by `availability-service` (§4.6) |
| [[edge-enforced-authentication-with-identity-binding]] | JWT terminated at the gateway; `bind: customerId:@user_id` (§3.26, §6.1) |
| [[transport-agnostic-caller-context]] | `IAppContext`/`IIdentityContext` — instantiated but **with no consumer** (§3.19) |
| [[vault-issued-dynamic-credentials-and-service-pki]] | KV, PKI `customers-service.pacco.io`, Mongo lease (§3.30) |
| [[correlation-and-span-propagation]] | `Correlation-Context` header, `span_context`, `Saga` forwarding (§3.15, §3.20) |
| [[structured-logging-with-property-redaction]] | `excludeProperties` incl. `Email`; `excludePaths` (§3.28) |
| [[registry-mediated-discovery-and-routing]] | Consul + Fabio (§3.29) |
| [[composable-per-concern-environment-stacks]] | `appsettings.{local,docker}.json` layering (§3.31) |
| [[independent-per-repository-release]] | `scripts/dockerize.sh`, own Dockerfile and image (§3.37) |
| [[framework-supplied-platform-conventions]] | Every `[convey]` mechanism in this document |
| [[prefix-partitioned-shared-cache]] | `redis.instance: "customers:"` — **declared, never used** (§3.33) |
| [[layered-service-test-suite]] | **Not instantiated** — this service has no tests (§3.36) |
| [[consumer-driven-contract-test-pair]] | **Not instantiated** — external event contracts are duplicated classes with no contract test (§3.21) |

