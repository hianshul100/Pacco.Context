# Component internals — `orders-service`

| | |
| --- | --- |
| **Component** | `orders-service` |
| **Source repository** | `hianshul100_Pacco.Services.Orders` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository) |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 5 of 7 |
| **Status** | New artifact — no prior `component-internals/orders-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` §2.2 and `repo-summary/Pacco.Services.Orders.md` remain valid and are **complemented**, not replaced: those catalogue the surface, this document models the internals. |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full
> (`src/Pacco.Services.Orders.{Core,Application,Infrastructure,Api}`) plus one test project,
> `tests/Pacco.Services.Orders.PactConsumerTests`, which is **not referenced by
> `Pacco.Services.Orders.sln`** and therefore never runs in the pipeline (§3.45). `Convey 0.4.*` —
> which supplies the CQRS dispatchers, the Mongo repository, the RabbitMQ client, the outbox, the
> HTTP client and the WebApi endpoint mapping — is a NuGet reference with **no source in this
> workspace**. Mechanisms owned by Convey are marked `[convey]` and, where their exact semantics
> change a conclusion, flagged `Unverifiable — Missing Source Evidence`. The upstream half of every
> inbound HTTP and AMQP contract is modelled in `component-internals/api-gateway.md`; the saga that
> drives most of this service's commands is modelled in
> `component-internals/ordermaker-saga-service.md`.

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

`orders-service` owns **one aggregate — `Order`** — and is the platform's *composition point*: it is
the only component that holds, in one document, which customer wants which parcels carried by which
vehicle on which date for what price. Everything else on the platform contributes one fragment of
that record and learns about the rest by event.

Three properties distinguish it from its siblings:

1. **It is the busiest contract surface on the platform.** Seven commands, nine integration events,
   eleven rejected-event types and seven external event subscriptions
   (`…Infrastructure/Extensions.cs:95-108`) — more than any other service in the workspace.
2. **Its state machine is driven almost entirely from outside.** Only two of the five statuses can
   be reached by a command a user can send through the gateway. `Delivering` and `Completed` are
   reachable *only* by consuming `deliveries` events, and `Approved` is reachable *only* by
   consuming an `availability` event (§3.4, §4.6).
3. **It is the only synchronous caller of three other services** — `parcels`, `pricing` and
   `vehicles` (`…Infrastructure/Services/Clients/`) — and each call is a single-resource read that
   gates a write.

| Responsibility | Where it lives |
| --- | --- |
| Model an order as id + customer + status + parcels + vehicle + delivery date + price + cancellation reason | `src/Pacco.Services.Orders.Core/Entities/Order.cs:9-27` |
| Enforce the five-state order lifecycle and emit a domain event on every transition | `…Core/Entities/Order.cs:106-150` |
| Enforce the price rules (`New` only, non-negative) | `…Core/Entities/Order.cs:60-73` (`SetTotalPrice`) |
| Enforce parcel-set uniqueness on an order | `…Core/Entities/Order.cs:85-93` (`AddParcel`) |
| Buffer domain events and expose them for publication | `…Core/Entities/AggregateRoot.cs:7-17` |
| Accept 7 commands over HTTP **and** the same 7 over AMQP | `…Api/Program.cs:33-42`; `…Infrastructure/Extensions.cs:95-101` |
| Answer 2 queries from a Mongo read model | `…Infrastructure/Mongo/Queries/Handlers/*.cs` |
| React to 7 external events from three other services | `…Application/Events/External/*.cs` |
| Read a parcel, a vehicle and a price synchronously over HTTP before assigning a vehicle | `…Infrastructure/Services/Clients/{Parcels,Vehicles,Pricing}ServiceClient.cs` |
| Translate domain events into integration events and publish them | `…Infrastructure/Services/{EventMapper,MessageBroker}.cs` |
| Translate exceptions into HTTP responses **and** into rejected events | `…Infrastructure/Exceptions/ExceptionTo{Response,Message}Mapper.cs` |
| Persist orders in MongoDB (`orders` collection, `orders-service` DB) and a customer-id replica in `customers` | `…Infrastructure/Mongo/Repositories/*.cs`; `…Infrastructure/Extensions.cs:80-81` |
| Publish reliably via a Mongo-backed transactional outbox, de-duplicate inbound messages via an inbox | `…Infrastructure/Decorators/Outbox*Decorator.cs`; `…Api/appsettings.json:104-112` |
| Register with Consul, address peers through Fabio, emit Jaeger spans, expose Prometheus metrics | `…Infrastructure/Extensions.cs:70-78` |
| Fetch its own settings, a PKI certificate and dynamic Mongo credentials from Vault | `…Api/Program.cs:44` (`UseVault`); `…Api/appsettings.json:168-196` |

### 1.2 What this component explicitly is **not**

- **Not an authenticator.** There is no JWT validation and no certificate authentication in this
  service. `AddSecurity()` is registered (`…Infrastructure/Extensions.cs:83`) but no
  `AddCertificateAuthentication` / `UseCertificateAuthentication` pair appears anywhere in `src/`,
  and no `[Authorize]`, `UseAuthentication` or `UseAuthorization` call exists. The `jwt` block in
  `…Api/appsettings.json:72-79` is **inert configuration no code in this service reads** (§3.41).
  Identity arrives as a JSON header the gateway manufactures and is **believed unconditionally**
  (§3.31, §3.32).
- **Not a reliable authorization engine.** Five handlers *do* run an ownership check, but every one
  of them is written as
  `if (identity.IsAuthenticated && identity.Id != order.CustomerId && !identity.IsAdmin)` — so an
  **unauthenticated** caller passes every check (§3.20). This is the single most consequential
  internal fact in this document.
- **Not the owner of parcels.** It stores a *denormalised copy* of four parcel fields
  (`…Core/Entities/Parcel.cs:7-10`) taken from a one-shot HTTP read at the moment the parcel is
  added (§3.6, §3.34). It never refreshes them.
- **Not the owner of customers.** `Core/Entities/Customer.cs` is an id and nothing else
  (`…Core/Entities/Customer.cs:5-13`); the local `customers` collection is an existence set fed once
  by `customer_created` (§3.7). This is `service-summaries.md` coupling **C1**.
- **Not a pricing engine.** The total price is whatever `pricing-service` returns
  (`…Application/Commands/Handlers/AssignVehicleToOrderHandler.cs:60-62`); this service only
  validates that it is non-negative and that the order is still `New`.
- **Not concurrency-safe.** `AggregateRoot.Version` exists (`…Core/Entities/AggregateRoot.cs:10`)
  but is **never incremented and never used in a filter**. Every write is a whole-document
  last-writer-wins replace (§3.17).
- **Not a saga coordinator.** It forwards a `Saga` header when publishing
  (`…Infrastructure/Extensions.cs:118-132`) but defines no saga. The saga lives in
  `ordermaker-saga-service`.
- **Not effectively tested.** The one test project in the repository is excluded from the solution
  file, so `scripts/test.sh` never executes it (§3.45).

### 1.3 The dual-transport boundary

All seven commands have **two entry points with different failure semantics** — except that two of
them have no HTTP route at all. This asymmetry governs how every failure in §3 is experienced:

| | HTTP path | AMQP path |
| --- | --- | --- |
| Entry | `UseDispatcherEndpoints` (`…Api/Program.cs:33-42`) | `SubscribeCommand<T>()` (`…Infrastructure/Extensions.cs:95-101`) |
| Commands reachable | 5 of 7 — **`ApproveOrder` and `CancelOrder` have no route** (§3.37) | all 7 |
| Identity source | `Correlation-Context` header (`…Infrastructure/Extensions.cs:113-116`) | broker message context (`…Infrastructure/Contexts/AppContextFactory.cs:19-33`) |
| De-duplication | **none effective** — a fresh `MessageId` GUID per request (§3.29) | inbox, keyed on the broker `MessageId` |
| Failure surfaced as | HTTP **400**, always (§3.30) | a **rejected event** published to the broker (§3.26) |
| Failure when unmapped | generic 400 `{code:"error"}` | **nothing at all** — silent drop |
| Caller learns outcome | synchronously | only by polling `operations-service` |

Four asymmetries specific to this service, each derived in §3.26:

- **`CannotChangeOrderStateException` is absent from `ExceptionToMessageMapper` entirely.** It is
  the exception raised by `Approve`, `Cancel`, `Complete` and `SetDelivering`
  (`…Core/Entities/Order.cs:110,122,134,145`) — that is, by *every* state transition in the
  aggregate. Over AMQP, an illegal transition therefore fails **silently**.
- `OrderHasNoParcelsException` — raised only by `AssignVehicleToOrder` — is mapped **under the
  `AddParcelToOrder` message type**, and produces an `AssignVehicleToOrderRejected` whose
  `VehicleId` slot is filled with `m.ParcelId`
  (`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:39-44`). The arm is unreachable *and*
  wrong.
- `DeleteParcelFromOrderRejected` **does not implement `IRejectedEvent`**
  (`…Application/Events/Rejected/DeleteParcelFromOrderRejected.cs:6`), unlike the other ten
  rejection types (§3.27).
- `CompleteOrderRejected` and `DeliveringOrderRejected` exist as types and are declared in
  `operations-service`'s `messages.json`, but **no code path constructs either of them** (§3.26).

### 1.4 Position in the platform

| Direction | Counterpart | Mechanism | Evidence |
| --- | --- | --- | --- |
| Inbound (sync) | `api-gateway` | HTTP, 7 routes, all `auth: true` | `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:290-354` |
| Inbound (async) | `api-gateway` async profile | AMQP to exchange `orders`, routing keys `create_order`, `delete_order`, `add_parcel_to_order`, `delete_parcel_from_order`, `assign_vehicle_to_order` | `…/ntrada-async.yml:336-412` |
| Inbound (async) | `ordermaker-saga-service` | commands `create_order`, `add_parcel_to_order`, `assign_vehicle_to_order`, `cancel_order` | `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs:63,74,103,141` |
| Inbound (async) | `customers-service` | event `customer_created` on exchange `customers` | `…Application/Events/External/CustomerCreated.cs:7` |
| Inbound (async) | `deliveries-service` | events `delivery_started`, `delivery_completed`, `delivery_failed` on exchange `deliveries` | `…Application/Events/External/Delivery*.cs:7` |
| Inbound (async) | `availability-service` | events `resource_reserved`, `resource_reservation_canceled` on exchange `availability` | `…Application/Events/External/Resource*.cs:7` |
| Inbound (async) | `parcels-service` — **broken** | event `parcel_deleted` declared as `[Message("deliveries")]` while the producer publishes it on exchange `parcels` | `…Application/Events/External/ParcelDeleted.cs:7` vs. `component-internals/parcels-service.md` §3.13 |
| Outbound (sync) | `parcels-service` | `GET /parcels/{parcelId}` | `…Infrastructure/Services/Clients/ParcelsServiceClient.cs:21` |
| Outbound (sync) | `vehicles-service` | `GET /vehicles/{vehicleId}` | `…Infrastructure/Services/Clients/VehiclesServiceClient.cs:21` |
| Outbound (sync) | `pricing-service` | `GET /pricing?customerId=…&orderPrice=…` | `…Infrastructure/Services/Clients/PricingServiceClient.cs:21` |
| Outbound (async) | `ordermaker-saga-service` | `vehicle_assigned_to_order` | `…OrderMaker/src/Pacco.Services.OrderMaker/Extensions.cs:62` |
| Outbound (async) | `parcels-service` | `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order` | `component-internals/parcels-service.md` §3.13 |
| Outbound (async) | `customers-service` | `order_completed` | `component-internals/customers-service.md` §3.21 |
| Outbound (async) | `operations-service` | every command, event and rejected event, by manifest | `…Operations/src/Pacco.Services.Operations.Api/messages.json:84-117` |

---

## 2. Core concepts (exhaustive)

Every distinct internal mechanism this service implements. The Owner column names the file that
*defines* the concept.

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | `Order` — the aggregate root | `…Core/Entities/Order.cs` | §3.1 |
| 2 | `AggregateRoot` — the event buffer, the never-called `ClearEvents`, the dead `Version` | `…Core/Entities/AggregateRoot.cs` | §3.2 |
| 3 | `AggregateId` — the id value object and its implicit conversions | `…Core/Entities/AggregateId.cs` | §3.3 |
| 4 | `OrderStatus` — the five-state machine and who can drive it | `…Core/Entities/OrderStatus.cs` | §3.4 |
| 5 | The three derived guards — `CanBeDeleted`, `CanAssignVehicle`, `HasParcels` | `…Core/Entities/Order.cs:19-21` | §3.5 |
| 6 | `Parcel` — the denormalised line item and its id-only equality | `…Core/Entities/Parcel.cs` | §3.6 |
| 7 | `Customer` — the id-only reference replica | `…Core/Entities/Customer.cs` | §3.7 |
| 8 | `DeleteParcel` — the event that is emitted without the mutation | `…Core/Entities/Order.cs:95-104` | §3.8 |
| 9 | `SetTotalPrice` — the only two price invariants | `…Core/Entities/Order.cs:60-73` | §3.9 |
| 10 | `SetVehicle` — the unvalidated setter | `…Core/Entities/Order.cs:75-78` | §3.10 |
| 11 | `SetDeliveryDate` — midnight truncation, and why the reservation lookup depends on it | `…Core/Entities/Order.cs:80-83` | §3.11 |
| 12 | Domain events — the three internal signals | `…Core/Events/*.cs` | §3.12 |
| 13 | Exception hierarchy and the fifteen error codes | `…Core/Exceptions`, `…Application/Exceptions` | §3.13 |
| 14 | `IOrderRepository` — six methods, declared in a file named `IParcelRepository.cs` | `…Core/Repositories/IParcelRepository.cs` | §3.14 |
| 15 | `OrderDocument`, the three mapping functions, and the unpersisted `CancellationReason` | `…Infrastructure/Mongo/Documents/*.cs` | §3.15 |
| 16 | `OrderStatus` persistence — an unresolved int-vs-string question | `Documents/OrderDocument.cs:13`; `Documents/Extensions.cs:39` | §3.16 |
| 17 | Absence of optimistic concurrency — last-writer-wins | `…Mongo/Repositories/OrderMongoRepository.cs:43` | §3.17 |
| 18 | The seven commands and the `[Contract]` gap on `AssignVehicleToOrder` | `…Application/Commands/*.cs` | §3.18 |
| 19 | `CreateOrder` — the command that generates its own id | `…Application/Commands/CreateOrder.cs:14` | §3.19 |
| 20 | The inline ownership guard and the unauthenticated bypass | five handlers in `…Application/Commands/Handlers/` | §3.20 |
| 21 | `AssignVehicleToOrderHandler` — the silent no-op and the unpublished domain events | `…Application/Commands/Handlers/AssignVehicleToOrderHandler.cs` | §3.21 |
| 22 | Queries and the read model — the unbounded scan and the silent empty list | `…Infrastructure/Mongo/Queries/Handlers/*.cs` | §3.22 |
| 23 | `EventMapper` — the status switch, and `null` as silent skip | `…Infrastructure/Services/EventMapper.cs` | §3.23 |
| 24 | The nine integration events and their consumers | `…Application/Events/*.cs` | §3.24 |
| 25 | `MessageBroker` — the publish pipeline | `…Infrastructure/Services/MessageBroker.cs` | §3.25 |
| 26 | Rejected events and `ExceptionToMessageMapper` — five defects | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` | §3.26 |
| 27 | `DeleteParcelFromOrderRejected` — the rejection that is not an `IRejectedEvent` | `…Application/Events/Rejected/DeleteParcelFromOrderRejected.cs` | §3.27 |
| 28 | `VehicleAssignedToOrder` — the hand-published event outside the mapper | `…Application/Events/VehicleAssignedToOrder.cs` | §3.28 |
| 29 | Transactional outbox and inbox | `…Infrastructure/Decorators/Outbox*.cs` | §3.29 |
| 30 | `ExceptionToResponseMapper` — everything is HTTP 400 | `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` | §3.30 |
| 31 | `IAppContext` / `IIdentityContext` — the caller-context abstraction | `…Infrastructure/Contexts/*.cs` | §3.31 |
| 32 | `Correlation-Context` header ingestion | `…Infrastructure/Extensions.cs:113-116` | §3.32 |
| 33 | External event subscription — seven events, one bound to the wrong exchange | `…Application/Events/External/**` | §3.33 |
| 34 | The three synchronous service clients | `…Infrastructure/Services/Clients/*.cs` | §3.34 |
| 35 | Queue naming and message conventions | `…Api/appsettings.json` (`rabbitMq` section) | §3.35 |
| 36 | `[Contract]` and `UsePublicContracts` | `…Application/ContractAttribute.cs`; `Extensions.cs:92` | §3.36 |
| 37 | Dispatcher-bound HTTP endpoints — and the two commands with no route | `…Api/Program.cs:33-42` | §3.37 |
| 38 | `MessageToLogTemplateMapper` — per-message log phrasing | `…Infrastructure/Logging/MessageToLogTemplateMapper.cs` | §3.38 |
| 39 | Log redaction and path exclusion | `…Api/appsettings.json` (`logger` section) | §3.39 |
| 40 | Consul registration and Fabio addressing | `…Api/appsettings.json` (`consul`, `fabio`, `httpClient`) | §3.40 |
| 41 | Inert JWT configuration | `…Api/appsettings.json:72-79` | §3.41 |
| 42 | Vault — KV settings, PKI certificate, dynamic Mongo credentials | `…Api/appsettings.json:168-196` | §3.42 |
| 43 | Environment layering — `appsettings.{local,docker,development}.json` | `…Api/appsettings.*.json` | §3.43 |
| 44 | Redis and `IDateTimeProvider` — one unused dependency, one clock seam | `Extensions.cs:76`; `…Infrastructure/Services/DateTimeProvider.cs` | §3.44 |
| 45 | The Pact consumer test that never runs | `tests/…PactConsumerTests`; `Pacco.Services.Orders.sln` | §3.45 |
| 46 | Deployment identity — port, image, environment names | `Dockerfile`, `scripts/`, `hianshul100_Pacco/compose` | §3.46 |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition**, **Representation & storage**,
**Lifecycle**, **Invariants & enforcement**, **Extension procedure**, **Failure modes**.

### 3.1 `Order` — the aggregate root

**Definition.** The single aggregate of this bounded context, declared
`public class Order : AggregateRoot` (`…Core/Entities/Order.cs:9`). It carries eight pieces of state:
`CustomerId`, `VehicleId`, `Status`, `CreatedAt`, `DeliveryDate`, `TotalPrice`, `CancellationReason`
and the parcel set (`…Core/Entities/Order.cs:11-18`). The parcel set is held in a private
`ISet<Parcel> _parcels = new HashSet<Parcel>()` (`:11`) and exposed as
`IEnumerable<Parcel> Parcels { get => _parcels; private set => _parcels = new HashSet<Parcel>(value); }`
(`:23-27`) — the setter is private, so the only ways into the set are the constructor and `AddParcel`.

**Representation & storage.** In memory, a POCO with an `AggregateId Id`. On the wire and on disk it
is flattened to `OrderDocument` with an embedded array of parcel sub-documents
(`…Infrastructure/Mongo/Documents/OrderDocument.cs`); the aggregate is *never* persisted directly.
There is no separate parcel collection in this service. **`CancellationReason` has no counterpart on
`OrderDocument` and is therefore never persisted** — see §3.15, which is the most surprising
persistence fact in this service.

**Lifecycle.** Two entry points, and the difference between them matters:

- The **public constructor** (`:29-50`) —
  `Order(AggregateId id, Guid customerId, OrderStatus status, DateTime createdAt, IEnumerable<Parcel> parcels = null, Guid? vehicleId = null, DateTime? deliveryDate = null, decimal totalPrice = 0)`
  — assigns the scalars, defaults `Parcels` to empty (`:37`), routes `vehicleId`/`deliveryDate`
  through `SetVehicle`/`SetDeliveryDate` when non-null (`:38-46`), and **always** sets
  `CancellationReason = string.Empty` (`:49`). It raises **no** domain event. This is the constructor
  the Mongo mapper uses to rehydrate (`…Mongo/Documents/Extensions.cs:9-12`).
- The **static factory** `Order.Create(AggregateId id, Guid customerId, OrderStatus status, DateTime createdAt)`
  (`:52-58`) — four parameters only, no parcels, no vehicle, no date, no price — calls the
  constructor and then buffers `new OrderStateChanged(order)` (`:55`). This is the only creation path
  that emits an event, and `CreateOrderHandler` is its only caller.

Because rehydration uses the constructor rather than the factory, a loaded order arrives with an
**empty** event buffer — the aggregate does not re-announce itself on every read. Thereafter it is
mutated only by the eight public methods on it. It is destroyed by `DeleteOrderHandler` calling
`_orderRepository.DeleteAsync`
(`…Application/Commands/Handlers/DeleteOrderHandler.cs:43`) — a hard delete, with no tombstone.

**Invariants & enforcement.**

| Invariant | Enforced by | Loud or silent? |
| --- | --- | --- |
| Id is not `Guid.Empty` | `AggregateId` ctor (`…Core/Entities/AggregateId.cs:17-20`) | **Loud** — `InvalidAggregateIdException` |
| Price may change only while `New` | `SetTotalPrice` (`Order.cs:62-65`) | **Loud** — `CannotChangeOrderPriceException` |
| Price is non-negative | `SetTotalPrice` (`Order.cs:67-70`) | **Loud** — `InvalidOrderPriceException` |
| A parcel appears at most once | `AddParcel` (`Order.cs:87-90`) | **Loud** — `ParcelAlreadyAddedToOrderException` |
| Status transitions follow the machine in §3.4 | `Approve`/`Cancel`/`Complete`/`SetDelivering` | **Loud** in-process, **silent** over AMQP (§3.26) |
| An order carries at least one parcel before a vehicle is assigned | **not** on the aggregate — in `AssignVehicleToOrderHandler.cs:44-47` | Loud |
| A vehicle is assigned only while `New` or `Canceled` | **not** on the aggregate — in `AssignVehicleToOrderHandler.cs:49-52` | **Silent** — the handler `return`s (§3.21) |
| A deleted parcel leaves the parcel set | **nowhere** — `DeleteParcel` never removes it (§3.8) | **Silent** — the invariant does not hold |

Two invariants a reader would expect and that are **absent**: nothing constrains `CustomerId` to a
customer this service knows about at the aggregate level (the check is in the handler,
`CreateOrderHandler.cs:31-35`), and nothing constrains `VehicleId` at all (§3.10).

**Extension procedure.** To add a field: add the property with a private setter to `Order.cs`, add
the parameter to the **constructor** (and to `Order.Create` only if it must be settable at creation),
add the property to `OrderDocument.cs`, extend `AsEntity`/`AsDocument`/`AsDto` in
`…Mongo/Documents/Extensions.cs`, and decide whether the field belongs in `OrderDto`. Missing any of
the four mapping touch-points is silent — `CancellationReason` is the living proof (§3.15). Existing
documents will read the field back as the CLR default (§5.2). To add a behaviour: add a method that
validates, mutates and calls `AddEvent(...)`, then add a `case` to `EventMapper.Map` — **omitting the
`EventMapper` arm is the default failure mode**, and it fails silently (§3.23).

**Failure modes.**

- `CancellationReason` survives only in memory and in the published `order_canceled` event; it is
  gone from the aggregate the moment it is reloaded (§3.15).
- `CreatedAt` is set once in the constructor from whatever the caller passes; `CreateOrderHandler`
  passes `_dateTimeProvider.Now` (`CreateOrderHandler.cs:37`), so it is UTC — but no invariant
  enforces that, and the constructor is public, so any future call site can supply anything.
- Because `AggregateRoot.ClearEvents()` is never called (§3.2), an `Order` instance reused across two
  operations would re-publish its earlier events. In practice every handler loads a fresh instance per
  message, so this is latent rather than active.

### 3.2 `AggregateRoot` — the event buffer, the never-called `ClearEvents`, the dead `Version`

**Definition.** A 19-line abstract base class (`…Core/Entities/AggregateRoot.cs`) providing three
things: an identity (`AggregateId Id`, `:9`), a version (`int Version`, `:10`) and a buffer of
domain events (`:7-8`, `:12-15`).

**Representation & storage.** The buffer is
`private readonly IList<IDomainEvent> _events = new List<IDomainEvent>()` (`:7`), exposed as
`public IEnumerable<IDomainEvent> Events => _events` (`:8`). It is **in-memory only** — no field of
`OrderDocument` corresponds to it, and `Version` is likewise absent from the document
(`…Infrastructure/Mongo/Documents/OrderDocument.cs`). Both die with the instance.

**Lifecycle.** `AddEvent(IDomainEvent @event)` (`:12-15`) appends unconditionally. `ClearEvents()`
(`:17`) empties the list. A repository-wide search finds **no call site for `ClearEvents()` and no
assignment to `Version`** anywhere in `src/` — the two mechanisms are declared and never used.

**Invariants & enforcement.** None. `AddEvent` performs no null check and no de-duplication. This
matters for `Order.Cancel` → `Approve` → `Cancel` within one message: three `OrderStateChanged`
events accumulate and all three are mapped and published, because nothing clears the buffer between
mutations. No handler in this service performs more than one state mutation per message today, so
the effect is latent.

**Extension procedure.** If a concurrency check is ever needed, `Version` is the intended seam:
increment it in `AddEvent`, add `Version` to `OrderDocument`, and change
`OrderMongoRepository.UpdateAsync` to a conditional replace on `(Id, Version)`. All four steps are
required — adding the field alone changes nothing, because the repository replaces the whole document
(§3.17).

**Failure modes.**

- **The `Version` field is a trap for a reader**: its presence suggests optimistic concurrency that
  does not exist. Do not assume any write is guarded.
- The event buffer is drained by *convention*, not by contract: each handler that wants to publish
  calls `_eventMapper.MapAll(order.Events)` explicitly. `AssignVehicleToOrderHandler` does **not**
  (§3.21), and nothing detects the omission — the events are simply discarded when the instance goes
  out of scope. This is the silent-loss channel that §3.23 and §3.21 both feed into.

### 3.3 `AggregateId` — the id value object

**Definition.** A `Guid` wrapper implementing `IEquatable<AggregateId>`
(`…Core/Entities/AggregateId.cs:5`), used as the identity type of `AggregateRoot`.

**Representation & storage.** `public Guid Value { get; }` (`:7`). It never reaches the datastore as
a wrapper — `OrderDocument.Id` is a plain `Guid` (`…Mongo/Documents/OrderDocument.cs:8`) and the
implicit operators do the conversion at the mapping boundary.

**Lifecycle.** Three constructors: `AggregateId()` → `new Guid()` … in fact
`AggregateId() : this(Guid.NewGuid())` (`:9-11`), `AggregateId(string value)` parsing a string
(`:13`), and `AggregateId(Guid value)` (`:15-23`) which is the only one that validates. Implicit
conversions `AggregateId → Guid` and `Guid → AggregateId` (`:43-47`) mean call sites read as if the
wrapper were not there; this is why `Order.Id` can be compared to a raw `Guid` throughout the
handlers.

**Invariants & enforcement.** `if (value == Guid.Empty) throw new InvalidAggregateIdException(value)`
(`:17-20`) — **loud**. But note the reach of the implicit operator: any `Guid.Empty` flowing into a
position typed `AggregateId` throws. `CreateOrder`'s constructor pre-empts this by substituting a
fresh GUID when the caller sends `Guid.Empty` (§3.19), so the exception is effectively unreachable on
the create path.

**Extension procedure.** Nothing here should change. If a stricter id format is ever wanted (e.g.
version-4 only), the check belongs at `:17`, and `InvalidAggregateIdException` must then be added to
`ExceptionToMessageMapper` — it is **absent** today (§3.26), so the failure would be silent on AMQP.

**Failure modes.** `InvalidAggregateIdException` maps to HTTP 400 `invalid_aggregate_id` via the
generic domain-exception arm (§3.30) but to **nothing** on the AMQP path. A malformed id delivered
over the broker produces no rejected event and no client-visible signal.

### 3.4 `OrderStatus` — the five-state machine and who can drive it

**Definition.** `public enum OrderStatus { New, Approved, Delivering, Completed, Canceled }`
(`…Core/Entities/OrderStatus.cs`) — ordinals 0–4. Whether the *ordinal* or the *member name*
reaches disk is not settled by anything in this repository; §3.16 sets out the evidence on both
sides and the consequence for schema changes.

**Representation & storage.** Persisted as an `OrderStatus` field on `OrderDocument`
(`…Mongo/Documents/OrderDocument.cs:13`), which the MongoDB C# driver serialises as an `int` by
default `[framework]`. Transported to clients as a lowercase string via
`Status.ToString().ToLowerInvariant()` (`…Mongo/Documents/Extensions.cs:39`).

**Lifecycle.** Four transition methods, each of the same shape — check, assign, buffer event:

| Method | Legal from | Sets | Also does | Evidence |
| --- | --- | --- | --- | --- |
| `Approve()` | `New`, `Canceled` | `Approved` | clears `CancellationReason` to `string.Empty` (`:114`) | `Order.cs:106-116` |
| `Cancel(reason)` | anything except `Completed` and `Canceled` | `Canceled` | `CancellationReason = reason ?? string.Empty` (`:126`) | `Order.cs:118-128` |
| `SetDelivering()` | `Approved` only | `Delivering` | — | `Order.cs:141-150` |
| `Complete()` | `Delivering` only | `Completed` | — | `Order.cs:130-139` |

Each ends with `AddEvent(new OrderStateChanged(this))` (`:115`, `:127`, `:138`, `:149`), so the
*same* domain event type signals all four transitions; `EventMapper` then re-reads `order.Status` to
decide which integration event to emit (§3.23).

**Who can actually drive each transition** — this is the part that is not visible from the route list:

| Target status | Reachable by | Evidence |
| --- | --- | --- |
| `New` | `CreateOrder`, over HTTP or AMQP | `CreateOrderHandler.cs:37` |
| `Approved` | **only** the `resource_reserved` external event | `…Events/External/Handlers/ResourceReservedHandler.cs:31` |
| `Delivering` | **only** the `delivery_started` external event | `…Events/External/Handlers/DeliveryStartedHandler.cs:29` |
| `Completed` | **only** the `delivery_completed` external event | `…Events/External/Handlers/DeliveryCompletedHandler.cs:29` |
| `Canceled` | `CancelOrder` (AMQP only — no HTTP route, §3.37), `delivery_failed`, `resource_reservation_canceled` | `CancelOrderHandler.cs:41`; `DeliveryFailedHandler.cs:29`; `ResourceReservationCanceledHandler.cs:32` |

The `ApproveOrder` command exists (`…Application/Commands/ApproveOrder.cs`), is subscribed
(`…Infrastructure/Extensions.cs:96`) and is declared in the operations manifest, but **no producer in
this workspace publishes it**: it has no gateway route in either `ntrada.yml` or `ntrada-async.yml`,
and the saga never publishes it. See §3.37 and **Q-4**.

**Invariants & enforcement.** All four guards are **loud in-process** — they throw
`CannotChangeOrderStateException(Id, Status)`. On the HTTP path that becomes a 400
`cannot_change_order_state`. On the AMQP path it becomes **nothing**: `CannotChangeOrderStateException`
appears nowhere in `ExceptionToMessageMapper.cs`, so every arm falls through to `_ => null` and
`MessageBroker` drops null events (`MessageBroker.cs:67-72`). Since three of the five transitions are
*only* reachable over AMQP, **the majority of state-machine violations on this service are silent**.

**Extension procedure.** Adding a status is a five-file change:

1. Add the member to `OrderStatus.cs` — **append it, never insert**, because the ordinal may be the
   stored value (§3.16, §5.3).
2. Add a transition method to `Order.cs` in the shape of `:106-116`.
3. Add an arm to the `switch` in `EventMapper.Map` (`…Services/EventMapper.cs:22-33`) — the fallback
   is `null` and therefore silent (§3.23).
4. Add the integration event type under `…Application/Events/` with `[Contract]`.
5. Add the event name to `…Operations…/messages.json` under `orders-service`, or
   `operations-service` will not correlate it (`component-internals/operations-service.md` §3.9).

Steps 3–5 are each independently silent when skipped. Note also that `CanBeDeleted` and
`CanAssignVehicle` (§3.5) enumerate statuses positively, so a new status is excluded from both by
default.

**Failure modes.**

- `Approve()` from `Canceled` is legal (`:110`) and clears the cancellation reason (`:113`) — an
  order can be resurrected, and the audit trail of *why* it was cancelled is destroyed in place.
- `Cancel` after `Complete` is refused, but `Cancel` from `Delivering` is allowed (`:122` excludes
  only `Completed` and `Canceled`) — the order is cancelled while a delivery is in flight, and
  nothing in this service notifies `deliveries-service`. The published `order_canceled` event is
  consumed by `parcels-service` only (`component-internals/parcels-service.md` §3.13); no delivery
  handler subscribes to it (**Q-5**).
- Re-cancelling an already-cancelled order throws rather than being idempotent — which means
  `resource_reservation_canceled` arriving twice fails the second time, silently, on AMQP.

### 3.5 The three derived guards

**Definition.** Three computed properties on the aggregate (`…Core/Entities/Order.cs:19-21`):

```
CanBeDeleted     => Status == OrderStatus.New
CanAssignVehicle => Status == OrderStatus.New || Status == OrderStatus.Canceled
HasParcels       => Parcels.Any()
```

**Representation & storage.** Computed, never persisted, never exposed on `OrderDto`
(`…Application/DTO/OrderDto.cs`) — a client cannot ask whether an order is deletable; it can only
try.

**Lifecycle.** Evaluated at the moment a handler consults them. `CanBeDeleted` is read by
`DeleteOrderHandler.cs:39`; `CanAssignVehicle` and `HasParcels` by
`AssignVehicleToOrderHandler.cs:44` and `:49`.

**Invariants & enforcement.** These are **handler-enforced, not aggregate-enforced** — the aggregate
will happily let you call `SetVehicle` on a `Completed` order (§3.10). The guard exists only on the
one code path that remembers to consult it. Enforcement is **loud** for `CanBeDeleted`
(`CannotDeleteOrderException`) and `HasParcels` (`OrderHasNoParcelsException`) but **silent** for
`CanAssignVehicle` (§3.21).

**Extension procedure.** If a new status should permit deletion or vehicle assignment, edit these
expressions — they are positive enumerations, so new statuses are excluded by default. If you want
the guard to hold regardless of call path, move it inside `SetVehicle`, which is the real fix.

**Failure modes.** `HasParcels` reads `Parcels.Any()`, and because `DeleteParcel` never removes from
`_parcels` (§3.8), an order whose only parcel was "deleted" still reports `HasParcels == true`. The
`OrderHasNoParcelsException` guard in `AssignVehicleToOrderHandler` therefore passes for an order with
no live parcels.

### 3.6 `Parcel` — the denormalised line item

**Definition.** A local entity (`…Core/Entities/Parcel.cs`) with four fields — `Guid Id`,
`string Name`, `string Variant`, `string Size` (`:7-10`). Note that `Variant` and `Size` are
**strings here**, while `parcels-service` models them as enums
(`component-internals/parcels-service.md` §3.2): the type is degraded at the boundary and never
validated on this side.

**Representation & storage.** Embedded in `OrderDocument` as a nested
`public class Parcel { Guid Id; string Name; string Variant; string Size; }`
(`…Mongo/Documents/OrderDocument.cs:19-25`) — an array inside the order document, not a collection.
There is no parcel repository in this service.

**Lifecycle.** Constructed in exactly one place:
`new Parcel(parcel.Id, parcel.Name, parcel.Variant, parcel.Size)` from the DTO returned by the
synchronous `GET /parcels/{id}` call (`…Application/Commands/Handlers/AddParcelToOrderHandler.cs:44`).
The values are a **point-in-time snapshot** and are never refreshed: if the parcel is renamed or
resized in `parcels-service`, this copy silently diverges. It is removed only when the order document
is replaced with a version lacking it — which, given §3.8, essentially never happens.

**Invariants & enforcement.** `Equals` and `GetHashCode` are overridden to compare **`Id` only**
(`…Core/Entities/Parcel.cs:20-36`). Combined with `_parcels` being a `HashSet<Parcel>`, this is what
makes `AddParcel`'s duplicate detection work (`Order.cs:87`). No validation of `Name`, `Variant` or
`Size` occurs on this side — a parcel with an empty name or a `Size` string that matches no
`parcels-service` enum member is accepted without complaint.

**Extension procedure.** Adding a field means changing four places: `…Core/Entities/Parcel.cs`, the
nested class in `OrderDocument.cs`, `AsDocument`/`AsDto` in `…Mongo/Documents/Extensions.cs`, and the
construction site in `AddParcelToOrderHandler.cs:44` — plus confirming that
`ParcelsServiceClient`'s `ParcelDto` (`…Application/DTO/ParcelDto.cs`) actually carries the value.
Do **not** add the new field to `Equals`: identity is deliberately id-only, and widening it would
break duplicate detection when the upstream parcel changes.

**Failure modes.**

- Stale snapshot, described above — this is `service-summaries.md` coupling **C2**.
- Because equality is id-only, `AddParcel` of the *same* parcel with *changed* attributes is rejected
  as a duplicate rather than treated as a refresh.
- `Variant`/`Size` as free strings means a downstream consumer of `OrderDto.Parcels` cannot rely on
  them parsing into any enum.

### 3.7 `Customer` — the id-only reference replica

**Definition.** `public class Customer { public Guid Id { get; } }` and nothing else
(`…Core/Entities/Customer.cs:5-13`). This is the whole of what `orders-service` knows about a
customer.

**Representation & storage.** A separate Mongo collection, registered as
`AddMongoRepository<CustomerDocument, Guid>("customers")` (`…Infrastructure/Extensions.cs:80`)
`[convey]`. `CustomerDocument` likewise carries only `Id` (`…Mongo/Documents/CustomerDocument.cs`).

**Lifecycle.** Written once, by `CustomerCreatedHandler` on the `customer_created` event
(`…Application/Events/External/Handlers/CustomerCreatedHandler.cs:19-24`); read once per order
creation, by `CreateOrderHandler.cs:31-35` calling `ICustomerRepository.ExistsAsync`. Never updated,
never deleted — there is no `customer_deleted` subscription.

**Invariants & enforcement.** `CustomerCreatedHandler` throws `CustomerAlreadyAddedException` when the
id is already present (`:20-23`) — the handler is **not idempotent**. Because
`CustomerAlreadyAddedException` is **absent from `ExceptionToMessageMapper`** (§3.26), a redelivered
`customer_created` fails silently and the message is consumed. That is benign here (the replica is
already correct), but the mechanism is the same one that hides real failures elsewhere.

**Extension procedure.** If order handling ever needs a customer attribute (name, VIP status,
address), the change is: add the field to `Customer` and `CustomerDocument`, extend the customer
mappers (`…Mongo/Documents/Extensions.cs:52-59`), subscribe to whichever `customers` event carries
updates (only `customer_created` is subscribed today), and add a backfill for the customers already
replicated — nothing in this service can reconstruct historical customers, since the replica holds no
source of truth. See §5.4.

**Failure modes.**

- **Ordering dependency.** `CreateOrder` fails with `CustomerNotFoundException` if
  `customer_created` has not yet been consumed. Over HTTP this is a 400 `customer_not_found`; over
  AMQP it becomes a `CreateOrderRejected` (`ExceptionToMessageMapper.cs:31`) — one of the few
  external-dependency failures that *is* reported on both transports.
- The replica has no repair path: a `customer_created` event lost while this service was down leaves
  that customer permanently unable to create orders here. This is `service-summaries.md` gap **G12**.
- The check is existence-only, so a customer deleted or suspended upstream still passes.

### 3.8 `DeleteParcel` — the event that is emitted without the mutation

**Definition.** `Order.DeleteParcel(Guid parcelId)` (`…Core/Entities/Order.cs:95-104`) is the
aggregate method behind the `DELETE /orders/{orderId}/parcels/{parcelId}` route. Its body is:

1. `var parcel = _parcels.SingleOrDefault(p => p.Id == parcelId);` (`:97`)
2. `if (parcel is null) throw new OrderParcelNotFoundException(parcelId, Id);` (`:98-101`)
3. `AddEvent(new ParcelDeleted(this, parcel));` (`:103`)

**There is no step that removes `parcel` from `_parcels`.** The method looks up the parcel, confirms
it exists, and announces its deletion — but the aggregate is left unchanged. This is the single
clearest correctness defect in the write model and it deserves to be read carefully before any change
is made near it.

**Representation & storage.** The consequence is concrete: `DeleteParcelFromOrderHandler` calls
`_orderRepository.UpdateAsync(order)` (`…Application/Commands/Handlers/DeleteParcelFromOrderHandler.cs:47`),
which replaces the stored document with one that **still contains the parcel**
(`…Mongo/Documents/Extensions.cs:14-31` maps `order.Parcels` verbatim). The write succeeds; the
document is unchanged in substance.

**Lifecycle.** Every downstream effect proceeds as though the deletion happened. The buffered
`ParcelDeleted` domain event is mapped by `EventMapper` to the integration event
`ParcelDeletedFromOrder(orderId, parcelId)` (`…Services/EventMapper.cs:39`), published
(`DeleteParcelFromOrderHandler.cs:49`), and consumed by `parcels-service`, which **does** clear the
parcel's `OrderId` (`component-internals/parcels-service.md` §3.13). The two services therefore end
in permanently inconsistent states: `parcels-service` believes the parcel is free, `orders-service`
believes it is still on the order.

**Invariants & enforcement.** The invariant "the parcel set reflects the parcels on the order" is
**not enforced anywhere** and does not hold. The failure is **silent** in the strongest sense: no
exception, HTTP 200/202, a published event, and a stored document that contradicts it. A second
`DELETE` of the same parcel succeeds again and publishes the event again, because `SingleOrDefault`
still finds it.

**Extension procedure.** The fix is one line — insert `_parcels.Remove(parcel);` between `:101` and
`:103`. Before making it, note what else changes:

- `HasParcels` (§3.5) begins returning `false` for orders whose parcels were all deleted, so
  `AssignVehicleToOrderHandler.cs:44-47` starts throwing `OrderHasNoParcelsException` for cases that
  previously passed.
- Repeat deletes begin throwing `OrderParcelNotFoundException` instead of succeeding, changing the
  HTTP status for a class of request that currently returns success.
- `GetContainingParcelAsync` (§3.14) begins failing to find orders it previously found, which changes
  `ParcelDeletedHandler`'s behaviour (§3.33).
- Existing stored documents are not repaired by the code change; a migration is needed to reconcile
  them against `parcels-service` (§5.5).

**Failure modes.** Beyond the above: `SingleOrDefault` throws `InvalidOperationException` if two
parcels shared an id — impossible via `AddParcel` because of the `HashSet` and id-only equality
(§3.6), and impossible via rehydration for the same reason, since the constructor also funnels the
list into a `HashSet` (`Order.cs:26`). It remains reachable only if the stored document is edited
outside this service. An `InvalidOperationException` is not a domain exception, so it maps to the
generic HTTP 400 `{code:"error"}` (§3.30) and to nothing at all on AMQP.

### 3.9 `SetTotalPrice` — the only two price invariants

**Definition.** `public void SetTotalPrice(decimal totalPrice)` (`…Core/Entities/Order.cs:60-73`).

**Representation & storage.** `decimal TotalPrice` (`Order.cs:15`) → `decimal TotalPrice` on
`OrderDocument` (`…Mongo/Documents/OrderDocument.cs:14`). The MongoDB driver stores `decimal` as
`Decimal128` `[framework]`; no `[BsonRepresentation]` attribute overrides that anywhere in
`…Mongo/Documents/`.

**Lifecycle.** Initialised to `0` by the constructor's default argument
(`Order.cs:31`, assigned at `:48`) — `Order.Create` does not take a price at all and set exactly once thereafter, in `AssignVehicleToOrderHandler.cs:62` from
`OrderPricingDto.OrderDiscountPrice` returned by `pricing-service`.

**Invariants & enforcement.**

- `if (Status != OrderStatus.New) throw new CannotChangeOrderPriceException(Id);` (`:62-65`) —
  **loud in-process**, and **silent on AMQP**, because `CannotChangeOrderPriceException` is absent
  from `ExceptionToMessageMapper` (§3.26).
- `if (totalPrice < 0) throw new InvalidOrderPriceException(Id, totalPrice);` (`:67-70`) — same
  asymmetry.

Note the interaction with `CanAssignVehicle` (§3.5): that guard permits assignment while `Canceled`,
but `SetTotalPrice` permits a price change only while `New`. `AssignVehicleToOrderHandler` calls
`SetVehicle`, then `SetTotalPrice`, then `SetDeliveryDate` (`:63-65`) — so assigning a vehicle to a
**cancelled** order passes the handler guard and then throws `CannotChangeOrderPriceException` from
inside `SetTotalPrice`, after the synchronous vehicle and pricing calls have already been made and
after `SetVehicle` has already mutated the in-memory aggregate. On AMQP that exception vanishes; the
`UpdateAsync` at `:66` is never reached, so the mutation is discarded — but the caller is told
nothing.

**Extension procedure.** To allow re-pricing in other statuses, widen the check at `:62`. To add a
maximum price or a currency, add the field and the check here; then add the new exception to
**both** mappers (`ExceptionToResponseMapper` handles it automatically via the domain-exception arm;
`ExceptionToMessageMapper` requires an explicit case, §3.26).

**Failure modes.** `TotalPrice` has no relationship to the parcels on the order — adding a parcel
after pricing does not invalidate the price, and nothing recalculates it. Any re-pricing after the
order leaves `New` is impossible without a code change.

### 3.10 `SetVehicle` — the unvalidated setter

**Definition.** `public void SetVehicle(Guid vehicleId) { VehicleId = vehicleId; }`
(`…Core/Entities/Order.cs:75-78`). Four lines, no guard.

**Representation & storage.** `Guid? VehicleId` (`Order.cs:16`) → `Guid? VehicleId` on
`OrderDocument` (`…Mongo/Documents/OrderDocument.cs:15`).

**Lifecycle.** Called from the private constructor when a non-null `vehicleId` is supplied
(`Order.cs:38-41`) and from `AssignVehicleToOrderHandler.cs:61`. Never cleared — cancelling an order
leaves `VehicleId` set, which is what allows `ResourceReservationCanceledHandler` to find the order
again by `(vehicleId, deliveryDate)` (§3.14).

**Invariants & enforcement.** **None on the aggregate.** No status check, no `Guid.Empty` check, no
verification that the vehicle exists. The existence check lives in the handler, which calls
`_vehiclesServiceClient.GetAsync(command.VehicleId)` and throws `VehicleNotFoundException` on null
(`AssignVehicleToOrderHandler.cs:54-58`); the status check lives in the handler too, and is silent
(§3.21). Calling `SetVehicle` from any future code path inherits **no** protection.

**Extension procedure.** If vehicle assignment ever needs to be safe by construction, move the
`CanAssignVehicle` check into this method and throw `CannotChangeOrderStateException`. That converts
`AssignVehicleToOrderHandler`'s silent `return` into a loud domain exception — which on the AMQP path
is still silent until `CannotChangeOrderStateException` is added to `ExceptionToMessageMapper`.

**Failure modes.** `Guid.Empty` is an accepted vehicle id at the aggregate level; only the upstream
HTTP lookup rejects it in practice (`vehicles-service` would return 404). A direct call — for example
from a future handler or a test — would store an empty vehicle reference silently.

### 3.11 `SetDeliveryDate` — midnight truncation

**Definition.**
`public void SetDeliveryDate(DateTime deliveryDate) { DeliveryDate = deliveryDate.Date; }`
(`…Core/Entities/Order.cs:80-83`). The `.Date` call is load-bearing.

**Representation & storage.** `DateTime? DeliveryDate` (`Order.cs:17`) → `DateTime? DeliveryDate`
(`…Mongo/Documents/OrderDocument.cs:16`), stored by the driver as a BSON UTC datetime at midnight
`[framework]`.

**Lifecycle.** Set from the constructor when supplied (`Order.cs:43-46`) and from
`AssignVehicleToOrderHandler.cs:63`, which passes `command.DeliveryDate` straight through from the
inbound command. Never cleared.

**Invariants & enforcement.** The only "invariant" is the truncation itself, applied silently — a
caller who sends `2026-09-08T14:30:00Z` gets `2026-09-08T00:00:00` stored and is not told. There is
no check that the date is in the future, none that it is within any window, and — critically — **no
normalisation of `DateTimeKind`**. `.Date` preserves `Kind`, so a `Local` input yields a `Local`
midnight while an `Unspecified` input yields an `Unspecified` midnight; what reaches Mongo depends on
how the driver converts each `[framework]`, and no code in this repository pins it.

**Why this matters more than it looks.** The truncation is the reason the reservation-correlation
lookup works at all. `IOrderRepository.GetAsync(Guid vehicleId, DateTime deliveryDate)` is
implemented as
`GetAsync(o => o.VehicleId == vehicleId && o.DeliveryDate == deliveryDate.Date)`
(`…Mongo/Repositories/OrderMongoRepository.cs:29-30`) — an **exact equality match on a `DateTime`**.
It only ever matches because both sides are truncated to midnight. The inbound side is
`ResourceReservedHandler.cs:26` passing `@event.DateTime` from the `availability` event; if
`availability-service` ever emits a reservation at a non-midnight instant, or with a different
`DateTimeKind`, every correlation lookup in this service fails and every affected order silently
stops advancing to `Approved`. See **Q-6** and `component-internals/availability-service.md` §3.12.

**Extension procedure.** If time-of-day delivery windows are ever needed, `.Date` must be removed
*and* `OrderMongoRepository.GetAsync(vehicleId, deliveryDate)` must change from equality to a
day-range filter in the same change; changing either alone breaks the correlation. Store the date as
a `DateTimeOffset` or as an explicit UTC date string if `Kind` ambiguity is to be closed.

**Failure modes.** Exact-equality date matching against a nullable, kind-ambiguous, driver-converted
`DateTime` is fragile in the ordinary case and undiagnosable in the failing case: the handler that
misses throws `OrderForReservedVehicleNotFoundException`
(`ResourceReservedHandler.cs:27-30`) which — being an application exception with no
`ExceptionToMessageMapper` arm for `ResourceReserved` (`ExceptionToMessageMapper.cs:80-82`) — is
dropped silently.

### 3.12 Domain events — the three internal signals

**Definition.** Three types implementing the empty marker `IDomainEvent`
(`…Core/IDomainEvent.cs`):

| Type | Payload | Raised by |
| --- | --- | --- |
| `OrderStateChanged` | the whole `Order` | `Order.Create` and all four transition methods |
| `ParcelAdded` | the `Order` **and** the `Parcel` | `Order.AddParcel` (`:92`) |
| `ParcelDeleted` | the `Order` and the `Parcel` | `Order.DeleteParcel` (`:103`) |

(`…Core/Events/OrderStateChanged.cs`, `ParcelAdded.cs`, `ParcelDeleted.cs`.)

**Representation & storage.** In-memory only, held in `AggregateRoot._events` (§3.2). Each event
holds a **live reference to the aggregate**, not a snapshot — so `EventMapper` reading
`@event.Order.Status` (`…Services/EventMapper.cs:24`) reads the status *as of mapping time*, not as
of the moment the event was raised. With one mutation per message this is indistinguishable; with
two it would silently mis-map the first event.

**Lifecycle.** Buffered by `AddEvent`, read by a handler calling `order.Events`, mapped by
`IEventMapper.MapAll`, published by `IMessageBroker.PublishAsync`, then discarded with the instance —
never cleared (§3.2), never persisted.

**Invariants & enforcement.** None. `IDomainEvent` declares no members
(`…Core/IDomainEvent.cs`), so nothing forces an event to carry an aggregate id, a timestamp or a
version. Nothing enforces that a domain event has a corresponding `EventMapper` arm.

**Extension procedure.** Add the type under `…Core/Events/`, raise it with `AddEvent`, **and add the
mapping arm in `EventMapper.Map`**. Skipping the third step compiles, runs, and drops the event
(§3.23).

**Failure modes.** The name `ParcelDeleted` is used for three different types in this
codebase — the domain event here, the *external* integration event
`…Application/Events/External/ParcelDeleted.cs` (§3.33), and `parcels-service`'s own published
`ParcelDeleted`. They are distinguished only by namespace. When editing subscription or mapping code,
check the `using` block: this is exactly the shape of confusion that produced the wrong-exchange
subscription described in §3.33.

### 3.13 Exception hierarchy and the fifteen error codes

**Definition.** Two parallel hierarchies:

- `DomainException` (`…Core/Exceptions/DomainException.cs`) — abstract, with `abstract string Code`.
- `AppException` (`…Application/Exceptions/AppException.cs`) — same shape, one layer out.

**Representation & storage.** Each concrete exception hard-codes its `Code` string. The full set:

| Layer | Exception | Code | Thrown by |
| --- | --- | --- | --- |
| Core | `CannotChangeOrderPriceException` | `cannot_change_order_price` | `Order.SetTotalPrice:64` |
| Core | `CannotChangeOrderStateException` | `cannot_change_order_state` | all four transitions |
| Core | `CannotDeleteOrderException` | `cannot_delete_order` | `DeleteOrderHandler:41` |
| Core | `CustomerNotFoundException` | `customer_not_found` | `CreateOrderHandler:33` |
| Core | `InvalidAggregateIdException` | `invalid_aggregate_id` | `AggregateId.cs:19` |
| Core | `InvalidOrderPriceException` | `invalid_order_price` | `Order.SetTotalPrice:69` |
| Core | `OrderParcelNotFoundException` | `order_parcel_not_found` | `Order.DeleteParcel:100` |
| Core | `ParcelAlreadyAddedToOrderException` | `parcel_already_added_to_order` | `Order.AddParcel:89` |
| App | `CustomerAlreadyAddedException` | `customer_already_added` | `CustomerCreatedHandler:21` |
| App | `OrderForReservedVehicleNotFoundException` | `order_for_reserved_vehicle_not_found` | `Resource*Handler` |
| App | `OrderHasNoParcelsException` | `order_has_no_parcels` | `AssignVehicleToOrderHandler:46` |
| App | `OrderNotFoundException` | `order_not_found` | 6 command handlers + 3 delivery event handlers |
| App | `ParcelNotFoundException` | `parcel_not_found` | `AddParcelToOrderHandler:41` |
| App | `UnauthorizedOrderAccessException` | `unauthorized_order_access` | six handlers (§3.20) |
| App | `VehicleNotFoundException` | `vehicle_not_found` | `AssignVehicleToOrderHandler:57` |

**Lifecycle.** Thrown in Core or Application; caught by Convey's exception middleware on the HTTP path
`[convey]` and routed to `ExceptionToResponseMapper` (§3.30), or by the RabbitMQ subscriber on the
AMQP path `[convey]` and routed to `ExceptionToMessageMapper` (§3.26).

**Invariants & enforcement.** Every code is written twice in effect — once as the literal `Code`
property, once as the message name in the rejected event. Nothing checks that they agree, and nothing
checks that a code appears in `operations-service`'s manifest. `ExceptionToResponseMapper` has a
fallback that derives a code from the type name
(`…Infrastructure/Exceptions/ExceptionToResponseMapper.cs`, `GetCode`) — but only for exceptions that
do not already carry one.

**Extension procedure.** Derive from `DomainException` or `AppException` depending on layer; supply a
snake-case `Code`; then decide the AMQP behaviour explicitly by adding a case to
`ExceptionToMessageMapper` — the default is silence.

**Failure modes.** Of the fifteen codes, **five have no `ExceptionToMessageMapper` arm at all**:
`cannot_change_order_state`, `cannot_change_order_price`, `invalid_order_price`,
`invalid_aggregate_id`, `customer_already_added` (§3.26). Every one of them is silent on AMQP.

### 3.14 `IOrderRepository` — six methods, declared in a file named `IParcelRepository.cs`

**Definition.** `…Core/Repositories/IParcelRepository.cs` declares
`public interface IOrderRepository` — **the file name does not match the type it contains**. There is
no `IParcelRepository` type in this service. A maintainer searching by file name will not find the
order repository; a maintainer searching by type name will find it in an unexpected file.

**Representation & storage.** Six methods:

| Method | Implementation | Note |
| --- | --- | --- |
| `Task<Order> GetAsync(Guid id)` | `OrderMongoRepository.cs:19-23` | `document?.AsEntity()` |
| `Task<Order> GetAsync(Guid vehicleId, DateTime deliveryDate)` | `:25-32` | exact-equality date match (§3.11) |
| `Task<Order> GetContainingParcelAsync(Guid parcelId)` | `:34-41` | `p.Parcels.Any(pc => pc.Id == parcelId)` |
| `Task AddAsync(Order order)` | `:43` | `_repository.AddAsync(order.AsDocument())` |
| `Task UpdateAsync(Order order)` | `:45` | whole-document replace (§3.17) |
| `Task DeleteAsync(Guid id)` | `:47` | hard delete |

The implementation delegates to Convey's `IMongoRepository<OrderDocument, Guid>` `[convey]`
(`…Mongo/Repositories/OrderMongoRepository.cs:12-17`).

**Lifecycle.** Registered by `AddMongoRepository<OrderDocument, Guid>("orders")`
(`…Infrastructure/Extensions.cs:81`) plus
`services.AddTransient<IOrderRepository, OrderMongoRepository>()` (`:59`).

**Invariants & enforcement.** All three read methods return `null` on a miss; **every caller is
responsible for the null check**, and they do not all behave the same way. Compare:

- `DeleteOrderHandler.cs:34-37` → throws `OrderNotFoundException` (loud).
- `ParcelDeletedHandler.cs:25-29` → **`return`s silently** (§3.33).
- `ResourceReservedHandler.cs:27-30` → throws `OrderForReservedVehicleNotFoundException`, which is
  then dropped by the message mapper (§3.26) — loud in the log, silent to the caller.

**Extension procedure.** Add the method to the interface, implement it in `OrderMongoRepository`, and
**consider the index**: `GetAsync(vehicleId, deliveryDate)` and `GetContainingParcelAsync` are both
unindexed scans — no `CreateIndex` call, no index attribute and no migration script exists anywhere
in this repository. See §5.6.

**Failure modes.**

- `GetContainingParcelAsync` uses `Any` over the embedded array and returns **one** order. A parcel
  is expected to be on at most one order, but nothing in this service enforces that — and given §3.8,
  a parcel that has been "deleted" from order A can still be added to order B, at which point two
  documents contain it and the method returns an arbitrary one.
- Renaming `IParcelRepository.cs` to `IOrderRepository.cs` is a safe, mechanical improvement; it is
  called out here so it is not mistaken for an intentional indirection.

### 3.15 `OrderDocument` and the three mapping functions

**Definition.** `OrderDocument : IIdentifiable<Guid>` (`…Mongo/Documents/OrderDocument.cs:8`) is the
persistence shape. It has **eight** properties — `Id`, `CustomerId`, `VehicleId`, `Status`,
`CreatedAt`, `DeliveryDate`, `TotalPrice`, `IEnumerable<Parcel> Parcels` (`:10-17`) — where `Parcel`
is the nested class at `:19-25`. Three functions in `…Mongo/Documents/Extensions.cs` move between
shapes:

| Function | Direction | Notable behaviour |
| --- | --- | --- |
| `AsEntity()` | document → `Order` | `:9-12` — calls the **public constructor**, projecting each `OrderDocument.Parcel` into a `Core.Entities.Parcel`; raises no domain event |
| `AsDocument()` | `Order` → document | `:14-31` — field-by-field object initialiser |
| `AsDto()` | document → `OrderDto` | `:33-50` — `Status.ToString().ToLowerInvariant()` at `:39` |

**Representation & storage.** Collection `orders` in database `orders-service`
(`…Infrastructure/Extensions.cs:81`; the database name is configured in the `mongo` section of
`…Api/appsettings.json`). Customers use the parallel pair at `Extensions.cs:52-59`.

**Lifecycle.** `AsEntity` on every repository read, `AsDocument` on every write, `AsDto` on every
query.

**Invariants & enforcement.** The mapping layer enforces nothing, and it is **not exhaustive**:

> **`CancellationReason` is never persisted.** The aggregate has it (`Order.cs:18`), `Order.Cancel`
> sets it (`Order.cs:126`), and `EventMapper` reads it into the published `OrderCanceled` event
> (`…Services/EventMapper.cs:33`). But `OrderDocument` has **no such property** (`:10-17`),
> `AsDocument` does not write it (`:14-31`), `AsDto` does not read it (`:33-50`), and `OrderDto` does
> not carry it (`…Application/DTO/OrderDto.cs:10-17`). The reason exists only inside the object that
> set it, for the duration of that one message.

The consequences are exact:

- Cancelling an order stores the status but discards the reason. Reloading the order runs
  `Order.cs:49`, which sets `CancellationReason = string.Empty` — so the value is not merely absent,
  it is **replaced by the empty string** on every read.
- `GET /orders/{orderId}` can never tell a client why an order was cancelled. The only place the
  reason ever appears is the `order_canceled` integration event, and its sole subscriber is
  `parcels-service`, which ignores the field (`component-internals/parcels-service.md` §3.13). The
  saga's `"Because I'm saga"` reason (§3.37) and the `resource_reservation_canceled` handler's
  `"Reservation canceled at: {DateTime}"` (§3.33) are both written and immediately lost.
- `Order.Approve()`'s clearing of `CancellationReason` (`Order.cs:114`) is therefore a no-op against
  stored state — there was nothing stored to clear.

`AsDocument` is likewise a hand-written initialiser with no compiler check against `OrderDocument`'s
property set: adding a property to the document class and forgetting the assignment compiles cleanly
and silently stores the default. That is precisely how `CancellationReason` came to be missing.

**Extension procedure.** A field added to the aggregate must be threaded through **four** places —
`OrderDocument`, `AsDocument`, `AsEntity`, and (`AsDto` + `OrderDto`) if it is client-visible. Use
`CancellationReason` as the checklist: whatever change you make, verify it round-trips by reading the
order back, not by inspecting the write. See §5.2 for existing documents.

**Failure modes.** Silent field loss on persistence, described above, recorded as **B-2**. Note also
that `OrderDto` has a second, unused constructor taking an `Order`
(`…Application/DTO/OrderDto.cs:23-33`) that duplicates `AsDto`'s logic; the two could diverge, and a
maintainer editing one will not be prompted to edit the other.

### 3.16 `OrderStatus` persistence — an unresolved int-vs-string question

**Definition.** The same value has three representations: `OrderStatus.Approved` in memory, *something*
on disk, and the string `"approved"` in the API response. The transport form is settled —
`Status.ToString().ToLowerInvariant()` (`…Mongo/Documents/Extensions.cs:39`), derived from the enum
**member name**. The stored form is not.

**Representation & storage.** Two lines of evidence point in opposite directions, and neither is
conclusive from this repository alone:

| Evidence | Points to | Strength |
| --- | --- | --- |
| Nothing in `…Mongo/Documents/` applies `[BsonRepresentation(BsonType.String)]`, and the MongoDB C# driver's *unconfigured* default for an enum is its underlying integer `[framework]` | **integer ordinal** | weak — it only holds if no convention is registered |
| Two independent copies of Convey's Mongo initializer in this workspace register `new EnumRepresentationConvention(BsonType.String)` into a convention pack named **`convey_conventions`**, applied to all types (`hianshul100_Pacco.Services.Parcels/tests/…/Fixtures/MongoDbFixtureInitializer.cs:46,54`; `hianshul100_Pacco.Services.Availability/tests/…Tests.Shared/Fixtures/MongoDbFixtureInitializer.cs:46,54`) | **member name string** | strong — the pack name and the `Convey.Persistence.MongoDB` imports identify these as stand-ins for what `AddMongo` does at startup |

Both fixtures are test-side copies; **Convey's own initializer is not in this workspace**, so this is
inference, not proof. On balance the second is more likely — a global convention registered by
`AddMongo` `[convey]` overrides the driver default for every document type, including
`OrderDocument`. Marked **`Unverifiable — Missing Source Evidence`** and carried as **Q-10**; one
`db.orders.findOne()` against any deployed environment settles it.

**Lifecycle.** Write: enum → (ordinal *or* member name). Read: stored value → enum → lowercase string.

**Invariants & enforcement.** None either way. Under integer storage, a value matching no member
deserialises into an out-of-range enum without error and `ToString()` yields the number — `"7"`
rather than a status name. Under string storage, an unrecognised string throws on deserialisation
`[framework]`, which surfaces as a 400 from `ExceptionToResponseMapper`'s catch-all (§3.27) rather
than as a corrupted read. No code validates the range in either case.

**Extension procedure.** Because the storage form is unresolved, **treat both reordering and renaming
as unsafe**, and append only:

- **Appending** a member is safe under either representation.
- **Reordering or inserting** is catastrophic if storage is ordinal and harmless if it is string.
- **Renaming** is catastrophic if storage is string, and if storage is integer it still silently
  changes the API response value.

Appending is the only operation that is safe without first answering **Q-10**. Note the American
spelling `canceled`: it is the enum member name and therefore the wire value regardless.

**Failure modes.** A maintainer who reads only `OrderDocument.cs` will conclude "integer, so never
reorder"; a maintainer who reads only the Convey convention will conclude "string, so never rename".
Each is half right, and acting on either half alone risks the other failure. §5.3 states the
migration.

### 3.17 Absence of optimistic concurrency

**Definition.** `OrderMongoRepository.UpdateAsync(Order order)` is
`_repository.UpdateAsync(order.AsDocument())` (`…Mongo/Repositories/OrderMongoRepository.cs:45`) —
a whole-document replace keyed on `Id` alone `[convey]`.

**Representation & storage.** No version field on `OrderDocument`, no `ETag`, no `_v`. `Version` on
`AggregateRoot` is never written (§3.2).

**Lifecycle.** Every mutating handler follows read → mutate → replace, with no compare-and-swap.

**Invariants & enforcement.** None — **last writer wins, whole document**. The concrete loss window:
`AssignVehicleToOrderHandler` reads the order, then makes **two sequential outbound HTTP calls**
(`:54` vehicles, `:60` pricing) before writing (`:66`). Any `AddParcelToOrder` that commits inside
that window is erased when the vehicle handler's replace lands, because its document snapshot predates
the parcel. The parcel's `parcel_added_to_order` event was already published, so `parcels-service`
believes the parcel is attached to an order that no longer lists it.

**Extension procedure.** See §3.2 — four coordinated changes are required, and every concurrent
writer must then handle the retry.

**Failure modes.** Silent, whole-document lost updates, with the widest window on the path that has
the most external latency. There is no telemetry that would reveal it: both writes succeed, both
publish, and the log shows two successful operations.

### 3.18 The seven commands

**Definition.** `…Application/Commands/` holds seven records implementing `ICommand` `[convey]`:

| Command | Fields | `[Contract]` | HTTP route | AMQP |
| --- | --- | --- | --- | --- |
| `CreateOrder` | `OrderId`, `CustomerId` | yes | `POST orders` | yes |
| `DeleteOrder` | `OrderId` | yes | `DELETE orders/{orderId}` | yes |
| `AddParcelToOrder` | `OrderId`, `ParcelId` | yes | `POST orders/{orderId}/parcels/{parcelId}` | yes |
| `DeleteParcelFromOrder` | `OrderId`, `ParcelId` | yes | `DELETE orders/{orderId}/parcels/{parcelId}` | yes |
| `AssignVehicleToOrder` | `OrderId`, `VehicleId`, `DeliveryDate` | **no** | `POST orders/{orderId}/vehicles/{vehicleId}` | yes |
| `ApproveOrder` | `OrderId` | yes | **none** | yes |
| `CancelOrder` | `OrderId`, `Reason` | yes | **none** | yes |

(`…Application/Commands/*.cs`; routes from `…Api/Program.cs:33-42`; subscriptions from
`…Infrastructure/Extensions.cs:95-101`.)

**Representation & storage.** Immutable classes with `get`-only properties and a constructor. Not
persisted, except as outbox/inbox rows when the outbox is enabled (§3.29).

**Lifecycle.** Deserialised by the WebApi endpoint binder or by the RabbitMQ subscriber `[convey]`,
dispatched through `ICommandDispatcher` to the single matching `ICommandHandler<T>`, then discarded.

**Invariants & enforcement.** No validation attributes anywhere and no validation middleware
registered in `…Infrastructure/Extensions.cs:50-84`. A command with every field defaulted is dispatched
and reaches its handler; whatever guard exists there is the only one. `AssignVehicleToOrder` with a
default `DeliveryDate` (`0001-01-01`) is accepted and stored.

**Extension procedure.** New command → class with `[Contract]`, handler, `SubscribeCommand<T>()` in
`UseInfrastructure`, route in `Program.cs`, log template in `MessageToLogTemplateMapper` (§3.38),
`ExceptionToMessageMapper` arm plus a rejected event (§3.26), and a `messages.json` entry in
`operations-service`. Six of those seven steps fail silently when skipped.

**Failure modes.** `AssignVehicleToOrder` missing `[Contract]` means it is **excluded from the
published-contracts document** served by `UsePublicContracts<ContractAttribute>`
(`…Infrastructure/Extensions.cs:92`) — the one command that carries a business-critical
`DeliveryDate` is the one absent from the machine-readable contract surface (§3.36).

### 3.19 `CreateOrder` — the command that generates its own id

**Definition.** `…Application/Commands/CreateOrder.cs:14` reads
`OrderId = orderId == Guid.Empty ? Guid.NewGuid() : orderId;` — the constructor substitutes a fresh
GUID when the caller supplies none.

**Representation & storage.** The generated id becomes the aggregate id and hence the document `_id`.

**Lifecycle.** Two id-generating mechanisms are in play and they must not be confused:

1. The **gateway** generates the id when the async route fires, via `resourceId: generate: true` on
   the `POST orders` route (`hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada-async.yml`,
   orders block) `[ntrada]`. That id is what the client is told to poll.
2. **This constructor** generates one only if the arriving command still has `Guid.Empty`.

For the async gateway path, (1) always wins and (2) never fires. For a direct AMQP publisher that
omits the id — such as a test — (2) fires, and **the id it invents is never communicated back**:
`CreateOrderHandler` publishes `OrderCreated(command.OrderId)` (`CreateOrderHandler.cs:41`), which
carries the id, but the original publisher has no correlation to match it to.

**Invariants & enforcement.** The substitution *prevents* `InvalidAggregateIdException` from ever
firing on this path (§3.3) — a `Guid.Empty` is silently replaced rather than rejected. There is **no
idempotency check**: re-sending `CreateOrder` with the same non-empty `OrderId` calls
`_orderRepository.AddAsync` again, and Convey's Mongo repository does not upsert `[convey]`. Whether
that raises a duplicate-key error or silently overwrites is
**`Unverifiable — Missing Source Evidence`** (Convey's `AddAsync` body is not in this workspace); the
inbox de-duplication (§3.29) is what protects the AMQP path, and it does not protect the HTTP path.

**Extension procedure.** If client-supplied ids should be rejected rather than replaced, remove the
ternary at `:14` and let `AggregateId` throw — then add `InvalidAggregateIdException` to
`ExceptionToMessageMapper`, or the rejection is silent.

**Failure modes.** Silent id substitution on a create is the kind of behaviour that makes a failed
create look like a successful one to a caller that expected to choose the id.

### 3.20 The inline ownership guard and the unauthenticated bypass

**Definition.** Six of the seven command handlers contain the same condition, duplicated rather than
shared:

```
if (identity.IsAuthenticated && identity.Id != order.CustomerId && !identity.IsAdmin)
{
    throw new UnauthorizedOrderAccessException(command.OrderId, identity.Id);
}
```

| Handler | Form | Line |
| --- | --- | --- |
| `AddParcelToOrderHandler` | private `ValidateAccessOrFail(Order)` method, called at `:38` | `:51-58` |
| `ApproveOrderHandler` | inline | `:34-38` |
| `AssignVehicleToOrderHandler` | inline | `:38-42` |
| `CancelOrderHandler` | inline | `:34-38` |
| `DeleteOrderHandler` | inline | `:33-37` |
| `DeleteParcelFromOrderHandler` | inline; throws with `order.Id` rather than `command.OrderId` | `:34-38` |
| `CreateOrderHandler` | **none** — correctly, since no order exists yet | — |

(all under `…Application/Commands/Handlers/`.) Exactly one handler extracted the check into a method;
the other five inlined it. Any fix must therefore be applied six times, or the extraction generalised
first.

**Representation & storage.** `identity` is `_appContext.Identity`
(`…Infrastructure/Contexts/AppContext.cs`), derived from the correlation context (§3.31).

**Lifecycle.** Evaluated after the order is loaded and before any mutation.

**Invariants & enforcement.** Read the first conjunct carefully: **`identity.IsAuthenticated &&`**.
When `IsAuthenticated` is `false` the whole condition is `false` and the guard passes. So:

| Caller | `IsAuthenticated` | Outcome |
| --- | --- | --- |
| Authenticated owner | true | passes (ids match) |
| Authenticated non-owner | true | **blocked** |
| Authenticated admin | true | passes |
| **Unauthenticated** | **false** | **passes — full access to any order** |

`IsAuthenticated` is `!string.IsNullOrWhiteSpace(Id)` where `Id` is the `id` field of the correlation
context (`…Infrastructure/Contexts/IdentityContext.cs`). A message published directly to the `orders`
exchange with no correlation context — or with one whose `user.id` is blank — is therefore treated as
a privileged caller. This is not a theoretical path: **`ApproveOrder` and `CancelOrder` are
AMQP-only** (§3.37), and `ordermaker-saga-service` publishes `CancelOrder` (§1.4). The gateway is the
only thing standing between the internet and this exchange, and it is a network boundary rather than a
cryptographic one — nothing in this service verifies who published a message.

The failure is **loud when it triggers** (`UnauthorizedOrderAccessException` → HTTP 400
`unauthorized_order_access`, or `AssignVehicleToOrderRejected` / `DeleteOrderRejected` /
`DeleteParcelFromOrderRejected` on AMQP per `ExceptionToMessageMapper.cs:47-57`) and **silent when it
is bypassed** — an unauthenticated caller sees an ordinary success.

**Extension procedure.** The correct fix is to invert the sense: refuse unless the caller is
authenticated *and* (owner or admin) — that is, replace the condition with
`if (!identity.IsAuthenticated || (identity.Id != order.CustomerId && !identity.IsAdmin))`. Two things
must be checked before doing so: the saga publishes `CancelOrder` with whatever correlation context it
forwards (`ordermaker-saga-service` §3.14), and the async gateway profile publishes commands with a
context built from the JWT — if either omits `user.id`, tightening the guard breaks them. Because the
condition is duplicated six times — five of them inline — extract it to a shared method in the same
change so the next handler inherits the fix. See **B-1**.

**Failure modes.** Beyond the bypass: the guard compares `identity.Id` to `order.CustomerId`, and
`identity.Id` is `Guid.Empty` when the header value is not a parseable GUID
(`IdentityContext.cs:26`) — so a malformed id is *authenticated* (non-blank string) but matches no
order, producing a hard block rather than a bypass. The two malformed-input paths fail in opposite
directions.

### 3.21 `AssignVehicleToOrderHandler` — the silent no-op and the unpublished domain events

**Definition.** The most complex handler in the service
(`…Application/Commands/Handlers/AssignVehicleToOrderHandler.cs`), and the only one that makes
synchronous calls to two other services.

**Lifecycle.** In order:

| Step | Line | Failure behaviour |
| --- | --- | --- |
| Load order | `:32` | `OrderNotFoundException` if null (`:33-36`) |
| Ownership guard, written inline rather than via `ValidateAccessOrFail` | `:38-42` | `UnauthorizedOrderAccessException`, subject to §3.20 |
| `if (!order.HasParcels) throw new OrderHasNoParcelsException(...)` | `:44-47` | loud; mapped **wrongly** on AMQP (§3.26) |
| `if (!order.CanAssignVehicle) return;` | `:49-52` | **silent success — no exception, no event, no write** |
| `GET /vehicles/{vehicleId}` | `:54` | `VehicleNotFoundException` if null (`:55-58`) |
| `GET /pricing?customerId&orderPrice` | `:60` | **no null check** — see below |
| `SetVehicle` / `SetTotalPrice` / `SetDeliveryDate` | `:61-63` | `SetTotalPrice` throws if status ≠ `New` (§3.9) |
| `UpdateAsync` | `:64` | whole-document replace (§3.17) |
| `PublishAsync(new VehicleAssignedToOrder(command.OrderId, command.VehicleId))` | `:65` | the **only** event published |

**Invariants & enforcement.** Three defects, in descending order of consequence:

1. **The `CanAssignVehicle` guard returns instead of throwing** (`:49-52`). Assigning a vehicle to an
   order that is `Approved`, `Delivering` or `Completed` produces: HTTP 200/202 with no body change,
   no rejected event, no log line beyond the ordinary handled-command entry, and no state change.
   To the caller — including the saga, which awaits `vehicle_assigned_to_order`
   (`ordermaker-saga-service` §3.11) — this is indistinguishable from success, except that the event
   never arrives and the saga stalls with no timeout. This is the most likely production symptom of
   this service.
2. **The handler never maps the aggregate's buffered domain events.** It publishes exactly one
   hand-constructed event and ignores `order.Events` entirely — unlike every other mutating handler.
   As it happens `SetVehicle`/`SetTotalPrice`/`SetDeliveryDate` raise no domain events, so nothing is
   lost *today*; but the aggregate still carries the `OrderStateChanged` that `AsEntity` buffered on
   load (§3.15), and any future event added to these setters will be dropped here silently.
3. **`_pricingServiceClient.GetOrderPriceAsync` is not null-checked** (`:60-62`) while the vehicle
   call immediately above it is. If `pricing-service` is unavailable or returns an empty body, the
   next line dereferences the DTO and raises `NullReferenceException` — not a domain exception, so
   HTTP 400 `{code:"error"}` (§3.30) and **nothing at all** on AMQP.

**Extension procedure.** To make the silent no-op loud, replace the `return` at `:51` with
`throw new CannotChangeOrderStateException(order.Id, order.Status);` **and** add that exception to
`ExceptionToMessageMapper` under `AssignVehicleToOrder` — otherwise the AMQP path merely trades a
silent success for a silent failure. Add a null check after `:60` in the same change.

**Failure modes.** Summarised: a stalled saga with no error anywhere, and an unguarded dereference on
the one call that has no fallback.

### 3.22 Queries and the read model

**Definition.** Two queries, each with a handler that reads `IMongoRepository<OrderDocument, Guid>`
directly — the read side does not go through `IOrderRepository`.

**`GetOrder`** (`…Infrastructure/Mongo/Queries/Handlers/GetOrderHandler.cs`) — one field, `OrderId`;
`GetAsync(p => p.Id == query.OrderId)` then `document?.AsDto()`. Returns `null` for a missing order,
which Convey's endpoint mapping renders as HTTP 404 `[convey]`. **No ownership check** — any caller
who knows an order id reads that order in full, including `CustomerId`, price and the parcel list.

**`GetOrders`** (`…GetOrdersHandler.cs`) — fields `CustomerId` and `Status`. Three behaviours worth
knowing:

- `Collection.AsQueryable()` (`:29`) with `Where` applied only when `CustomerId` has a value; **a
  query with no `CustomerId` scans the entire collection** and returns every order in the system.
- When the caller is authenticated and `query.CustomerId` does not match `identity.Id` and the caller
  is not admin, the handler **`return Enumerable.Empty<OrderDto>()`** (`:33-36`) — a silent empty
  list, not a 403. Indistinguishable from "this customer has no orders".
- **No paging, no limit, no sort.** `service-summaries.md` gap **G12** notes the same shape elsewhere.

**Representation & storage.** `OrderDto` (`…Application/DTO/OrderDto.cs`) carries `Id`, `CustomerId`,
`Status` (lowercased), `TotalPrice`, `VehicleId`, `DeliveryDate`, `CreatedAt` and
`IEnumerable<ParcelDto> Parcels` — **eight fields, with no cancellation reason** (§3.15).

**Invariants & enforcement.** Read authorization is **partial and inconsistent**: enforced by silence
on `GetOrders`, absent entirely on `GetOrder`. Both share the unauthenticated-bypass shape of §3.20 —
`GetOrdersHandler`'s check is also gated on `identity.IsAuthenticated`.

**Extension procedure.** Adding a filter means adding the field to the query class and a `Where` to
the handler. Add paging by taking `Page`/`Results` and using Convey's paged extensions `[convey]`;
this is the highest-value change in the read model. Adding an ownership check to `GetOrderHandler`
requires injecting `IAppContext`, which it does not currently take.

**Failure modes.** Unbounded scan under load; silent empty results that look like data loss to a
client; and an unauthenticated `GET /orders` returning the entire order book — mitigated today only
by the gateway's `auth: true` on that route (`ntrada.yml`, orders block).

### 3.23 `EventMapper` — the status switch, and `null` as silent skip

**Definition.** `…Infrastructure/Services/EventMapper.cs` (45 lines) implements Convey's
`IEventMapper` `[convey]`. `MapAll(IEnumerable<IDomainEvent> events) => events.Select(Map)`
(`:14`) — note it does **not** filter nulls; that happens later, in `MessageBroker` (§3.25).

**Lifecycle.** `Map(IDomainEvent @event)` is a type switch (`:16-44`):

| Domain event | Produces |
| --- | --- |
| `OrderStateChanged` with `Status == New` | `OrderCreated(Id)` |
| … `Approved` | `OrderApproved(Id)` |
| … `Delivering` | `OrderDelivering(Id)` |
| … `Completed` | `OrderCompleted(Id, CustomerId)` |
| … `Canceled` | `OrderCanceled(Id, CancellationReason)` |
| `ParcelAdded` | `ParcelAddedToOrder(orderId, parcelId)` |
| `ParcelDeleted` | `ParcelDeletedFromOrder(orderId, parcelId)` |
| anything else | **`return null;`** (`:43`) |

`OrderCompleted` is the only mapped event that carries a second field, and it exists because
`customers-service` counts completed orders per customer
(`component-internals/customers-service.md` §3.21).

**Invariants & enforcement.** The `null` fallback at `:43` is the service's principal silent-loss
channel. A domain event with no arm — or an `OrderStateChanged` for a status added without a
corresponding arm — is mapped to `null`, skipped by `MessageBroker.cs:67-72`, and disappears with no
log line and no metric. Compilation succeeds; tests (if any ran, §3.45) would not catch it.

**Extension procedure.** One arm per new domain event or status. Consider replacing `:43` with a
warning log or a throw — a throw would surface the omission at the first occurrence, which on the AMQP
path means a rejected/dead-lettered message rather than an invisible gap.

**Failure modes.** Silent event loss, described above. Note also that the switch reads
`@event.Order.Status` *at mapping time* (§3.12) — with more than one transition per message, the
first event maps to the last status.

### 3.24 The nine integration events and their consumers

**Definition.** `…Application/Events/` holds nine published event types:

| Event | Fields | `[Contract]` | Consumed by |
| --- | --- | --- | --- |
| `OrderCreated` | `OrderId` | yes | no subscriber in workspace |
| `OrderApproved` | `OrderId` | yes | no subscriber in workspace |
| `OrderDelivering` | `OrderId` | yes | no subscriber in workspace |
| `OrderCompleted` | `OrderId`, `CustomerId` | yes | `customers-service` |
| `OrderCanceled` | `OrderId`, `Reason` | yes | `parcels-service` |
| `OrderDeleted` | `OrderId` | yes | `parcels-service` |
| `ParcelAddedToOrder` | `OrderId`, `ParcelId` | yes | `parcels-service` |
| `ParcelDeletedFromOrder` | `OrderId`, `ParcelId` | yes | `parcels-service` |
| `VehicleAssignedToOrder` | `OrderId`, `VehicleId` | **no** | `ordermaker-saga-service` |

(`…Application/Events/*.cs`; consumers verified by searching every clone in the workspace for a
matching `SubscribeEvent<>`.)

**Representation & storage.** Immutable classes implementing `IEvent` `[convey]`. Published to the
topic exchange `orders` with snake-case routing keys derived from the type name `[convey]`; the
exchange name and casing convention are set in the `rabbitMq` section of `…Api/appsettings.json`
(§3.35).

**Lifecycle.** Constructed by `EventMapper` (eight of nine) or by hand in the handler
(`VehicleAssignedToOrder`, §3.28), then handed to `IMessageBroker.PublishAsync`.

**Invariants & enforcement.** Nothing ties this list to `operations-service`'s manifest. The manifest
lists nine events for `orders-service`
(`…Operations/src/Pacco.Services.Operations.Api/messages.json`, `orders-service` block) and they
agree today — but the agreement is maintained by hand across two repositories.

**Extension procedure.** New event → class with `[Contract]`, `EventMapper` arm, `messages.json`
entry, log template (§3.38). Adding a **field** to an existing event is the riskier change:
consumers deserialise by name, so a removed or renamed field silently arrives as `default` at every
subscriber (§5.7).

**Failure modes.** Three events (`order_created`, `order_approved`, `order_delivering`) have no
subscriber anywhere in the workspace — they exist for `operations-service` correlation and for
external consumers not present here. `VehicleAssignedToOrder` lacking `[Contract]` excludes the
saga's trigger event from the published contract document (§3.36), which is the same omission as its
command (§3.18).

### 3.25 `MessageBroker` — the publish pipeline

**Definition.** `…Infrastructure/Services/MessageBroker.cs` (87 lines) is the single publication
seam; every handler depends on `IMessageBroker`, never on the RabbitMQ client directly.

**Lifecycle.** `PublishAsync(params IEvent[] events)` → `PublishAsync(IEnumerable<IEvent> events)`,
which:

1. Returns immediately when `events` is `null` (`:38-41`).
2. Builds a correlation id from `_correlationContextAccessor.CorrelationContext` or the HTTP
   `_httpContextAccessor` (`:43-55`) `[convey]`.
3. Resolves a span context: the message-properties header if present, otherwise
   `_tracer.ActiveSpan?.Context.ToString()` (`:57-61`) — so a publish outside any span carries an
   empty `span_context` and the trace is broken at that hop.
4. Collects headers to forward via `_headersToForward` — which returns **only the `Saga` header**
   (`…Infrastructure/Extensions.cs:118-132`).
5. **Skips null events**: `foreach (var @event in events) { if (@event is null) continue; ... }`
   (`:67-72`). This is where `EventMapper`'s nulls (§3.23) vanish, with no log.
6. If `_outbox.Enabled`, calls `_outbox.SendAsync(...)` per event; otherwise publishes directly
   (`:76-81`) `[convey]`.

**Invariants & enforcement.** Nothing guarantees an event is publishable — no schema check, no
`[Contract]` check. The null skip is unconditional and unlogged.

**Extension procedure.** To forward more headers (a tenant id, a request id), extend
`GetHeadersToForward` at `…Infrastructure/Extensions.cs:118-132`; the list is a hard-coded single
entry today. To make dropped events visible, add a log line at `:69`.

**Failure modes.** Silent null skip (the terminal end of §3.23's channel); broken traces when
publishing outside a span; and — since the outbox is per-message rather than per-transaction — a
handler that publishes two events can have the first stored and the second lost if the process dies
between them (§3.29).

### 3.26 Rejected events and `ExceptionToMessageMapper`

**Definition.** `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs` (84 lines) implements
Convey's `IExceptionToMessageMapper` `[convey]`: `Map(Exception exception, object message)` returns
the event to publish when a subscribed command or event throws. Its structure is a switch on the
exception type, each arm containing a nested switch on the inbound message type.

**Lifecycle.** Called by the RabbitMQ subscriber on an unhandled exception `[convey]`. A non-null
result is published to the `orders` exchange; **a null result means the exception is swallowed and
the message is acknowledged**.

**Invariants & enforcement.** This file is the AMQP failure contract, and it has five distinct
defects. They are listed with their exact evidence because each one changes how a maintainer should
reason about a failure report.

1. **Five exceptions have no arm at all** — `CannotChangeOrderStateException`,
   `CannotChangeOrderPriceException`, `InvalidOrderPriceException`, `InvalidAggregateIdException`,
   `CustomerAlreadyAddedException`. The first of these is raised by every state transition in the
   aggregate (§3.4), and three of the five transitions are AMQP-only. **Most state-machine violations
   on this service produce no rejected event.**
2. **`OrderHasNoParcelsException` is mapped under the wrong message type and builds the wrong event.**
   `:39-44` reads, in effect: on `OrderHasNoParcelsException`, if the inbound message is
   `AddParcelToOrder m`, produce `AssignVehicleToOrderRejected(m.OrderId, m.ParcelId, …)`. But
   `OrderHasNoParcelsException` is thrown **only** by `AssignVehicleToOrderHandler.cs:46`, never by
   `AddParcelToOrderHandler`. The arm is therefore unreachable, *and* it would place a parcel id in
   the `VehicleId` slot if it were reached. The real case — `OrderHasNoParcelsException` during
   `AssignVehicleToOrder` — falls through to `_ => null` at `:43` and is silent.
3. **`CompleteOrderRejected` and `DeliveringOrderRejected` are never constructed.** Both types exist
   under `…Application/Events/Rejected/` and both are declared in `operations-service`'s manifest, but
   no line in this mapper or anywhere else in `src/` instantiates either. The failures they were
   written for — `delivery_completed` / `delivery_started` arriving for an unknown order — throw
   `OrderNotFoundException`, whose arms (`:47-57`) cover only `DeleteOrder`,
   `DeleteParcelFromOrder`, `AddParcelToOrder`, `AssignVehicleToOrder` and `CancelOrder`. Inbound
   *events* are not in the switch, so they hit `_ => null`.
4. **Seven `_ => null` fallbacks** (`:36`, `:43`, `:50`, `:57`, `:73`, `:80`, `:82`) — every
   unmatched (exception, message) pair is silent. `MessageBroker.cs:67-72` then discards the null
   without logging, so the only trace is whatever the framework's error log emitted `[convey]`.
5. **`OrderForReservedVehicleNotFoundException`** — thrown by both `ResourceReservedHandler` and
   `ResourceReservationCanceledHandler` — has an arm producing `OrderForReservedVehicleNotFound`, but
   the nested message switch does not cover the inbound event types, so it too resolves to `null`
   (`:80-82`).

**Extension procedure.** For a new exception, add an outer arm; for a new command, add an inner arm to
**every** exception that command's handler can throw — the mapper is a matrix, and the diagonal is
what is filled in today. After editing, cross-check `…Operations…/messages.json` so
`operations-service` recognises the rejected event, and check that the rejected type implements
`IRejectedEvent` (§3.27).

**Failure modes.** The dominant one, stated plainly: **on the AMQP path, most failures in this service
are invisible**. A client that submits a command through the async gateway profile and polls
`operations-service` will wait forever rather than receive a rejection — there is no timeout anywhere
in that chain (`component-internals/api-gateway.md` §3.17). See **B-3**.

### 3.27 `DeleteParcelFromOrderRejected` — the rejection that is not an `IRejectedEvent`

**Definition.** Ten of the eleven types in `…Application/Events/Rejected/` are declared
`: IRejectedEvent`. `DeleteParcelFromOrderRejected` is declared `: IEvent` only
(`…Application/Events/Rejected/DeleteParcelFromOrderRejected.cs:6`), while carrying the same
`Reason`/`Code` shape as its siblings.

**Representation & storage.** Identical on the wire — the same fields, the same routing key
`delete_parcel_from_order_rejected`.

**Lifecycle.** Produced by `ExceptionToMessageMapper` on `OrderNotFoundException`,
`UnauthorizedOrderAccessException` and `OrderParcelNotFoundException` during `DeleteParcelFromOrder`.

**Invariants & enforcement.** `IRejectedEvent` is the marker Convey's outbox/operations tooling uses to
recognise a rejection `[convey]`. What breaks when the marker is missing is
**`Unverifiable — Missing Source Evidence`** — Convey's source is not in this workspace, and
`operations-service` correlates on the message *name* from `messages.json` rather than on the
interface (`component-internals/operations-service.md` §3.9), so the observable effect may be nil.
The inconsistency is nonetheless real and one-of-eleven.

**Extension procedure.** Add `, IRejectedEvent` to the declaration. Before doing so, confirm against
`operations-service` that nothing keys off the current shape.

**Failure modes.** Any future tooling that filters rejections by interface will silently omit this one
type — which happens to be the rejection for the operation that is already the most broken (§3.8).

### 3.28 `VehicleAssignedToOrder` — the hand-published event

**Definition.** The one integration event this service publishes **without** going through
`EventMapper`: `AssignVehicleToOrderHandler.cs:65` constructs and publishes it directly from the
command's fields.

**Representation & storage.** `OrderId`, `VehicleId`. **No `[Contract]` attribute** (§3.36).

**Lifecycle.** Published after `UpdateAsync`, on the outbox when enabled. Consumed by
`ordermaker-saga-service` (`…OrderMaker/src/Pacco.Services.OrderMaker/Extensions.cs:62`), where it
advances the saga to the reservation step.

**Invariants & enforcement.** Because it is built from `command.OrderId` and `command.VehicleId`
rather than from the aggregate, it is published even if the aggregate's state disagrees — though in
practice the preceding lines have already written those exact values. The event carries **neither the
delivery date nor the price**, so a consumer that needs them must call back.

**Extension procedure.** Prefer routing this through `EventMapper` like every other event: raise a
domain event from `SetVehicle` and add a mapper arm. That would also close the asymmetry noted in
§3.21: this is the one mutating handler whose published output is unrelated to the aggregate's event
buffer, so it is the one handler where the buffer/mapper discipline does not apply and a future
maintainer's assumptions will not hold.

**Failure modes.** The event is the saga's sole trigger for its next step, and the handler that
publishes it can return silently without publishing (§3.21). The combination — one trigger, one silent
path that skips it, no timeout downstream — is the platform's most consequential stall.

### 3.29 Transactional outbox and inbox

**Definition.** Two `[Decorator]` classes, `OutboxCommandHandlerDecorator<T>` and
`OutboxEventHandlerDecorator<T>` (`…Infrastructure/Decorators/`), wrap every command and event
handler `[convey]`.

**Representation & storage.** Two Mongo collections in the same database, named in the `outbox`
section of `…Api/appsettings.json` — an inbox collection for de-duplication and an outbox collection
for pending publications. The section also selects sequential processing, sets an expiry, and
**disables Mongo transactions** — meaning the "transactional" outbox is not transactional here: the
state write and the outbox insert are separate operations, so a crash between them loses the
publication. This is `service-summaries.md` gap **G12**'s sibling and applies identically across
Pacco services.

**Lifecycle.** The decorator reads
`messagePropertiesAccessor.MessageProperties?.MessageId`, falling back to
`Guid.NewGuid().ToString("N")` when absent (`Outbox*HandlerDecorator.cs`), then calls
`_outbox.HandleAsync(_messageId, () => _handler.HandleAsync(command))` when the outbox is enabled and
the inner handler directly otherwise.

**Invariants & enforcement.** De-duplication is keyed on the inbound `MessageId`. On the AMQP path
that id comes from the broker and repeats on redelivery, so the inbox suppresses the duplicate. **On
the HTTP path there is no inbound `MessageId`, so a fresh GUID is minted per request and every retry
is a distinct key** — HTTP is not idempotent at all. Since `CreateOrder` has no idempotency check of
its own (§3.19), a retried `POST /orders` creates a second order.

**Extension procedure.** The outbox is on by default and disabled in the local profile
(`…Api/appsettings.local.json`, `outbox.enabled`) — so a developer running locally exercises the
direct-publish path and will not reproduce outbox behaviour. To make HTTP idempotent, take an
idempotency key from a request header and pass it as the message id.

**Failure modes.** Non-transactional outbox (lost publication on crash); ineffective HTTP
de-duplication; and expiry-based cleanup that will silently re-admit a duplicate whose inbox entry has
aged out.

### 3.30 `ExceptionToResponseMapper` — everything is HTTP 400

**Definition.** `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` (46 lines) implements
Convey's `IExceptionToResponseMapper` `[convey]`.

**Lifecycle.** `Map(Exception exception)` switches on `DomainException` and `AppException`, returning
`new ExceptionResponse(new { code = GetCode(ex), reason = ex.Message }, HttpStatusCode.BadRequest)`;
anything else returns `new { code = "error", reason = "There was an error." }` — **also 400**.

**Invariants & enforcement.** **Every** failure is HTTP 400. Not found is 400. Unauthorized is 400.
A `NullReferenceException` from the unguarded pricing call (§3.21) is 400. A client cannot distinguish
"you sent something invalid" from "we broke" by status code — only by parsing `code`, and the generic
arm deliberately gives no detail. The generic arm is the right choice for not leaking internals; the
status code is not.

`GetCode` caches derived codes in a `ConcurrentDictionary` and, for exceptions without an explicit
`Code`, derives one from the type name via `Name.Underscore().Replace("_exception", "")`.

**Extension procedure.** To return meaningful statuses, add arms mapping specific exception types to
404/403/409 before the generic fallback. Coordinate with the gateway, which forwards downstream status
codes unchanged (`component-internals/api-gateway.md` §3.11), and with `Pacco.Web` clients that may
already branch on 400.

**Failure modes.** Monitoring cannot separate client errors from server errors by status; alerting on
5xx will never fire for this service.

### 3.31 `IAppContext` / `IIdentityContext`

**Definition.** Four small classes in `…Infrastructure/Contexts/` — `AppContext`, `IdentityContext`,
`CorrelationContext` and `AppContextFactory` — giving handlers a transport-agnostic view of the
caller.

**Representation & storage.** `AppContext` holds `RequestId`, `UserId`, `TraceId`, `IpAddress`,
`UserAgent` and an `IIdentityContext Identity`. `IdentityContext` exposes `Id`, `Role`, `Claims`,
`IsAuthenticated` and `IsAdmin`.

**Lifecycle.** `AppContextFactory.Create()` prefers `ICorrelationContextAccessor` — the AMQP path —
deserialising the correlation context from JSON; failing that it reads the HTTP `Correlation-Context`
header (§3.32); failing that it returns `AppContext.Empty`.

**Invariants & enforcement.** Two conversions decide authorization outcomes:

- `Id = Guid.TryParse(id, out var userId) ? userId : Guid.Empty;` (`IdentityContext.cs:26`) — a
  malformed id becomes `Guid.Empty` **silently**.
- `IsAdmin = Role.Equals("admin", StringComparison.InvariantCultureIgnoreCase);` — the admin role is
  a hard-coded string; a role of `"administrator"` or `"Admins"` is not admin.

`IsAuthenticated` is a **non-empty-string test on the id**, not a signature check. Nothing in this
service validates a token. The entire identity chain is: the gateway validates a JWT, writes a JSON
header, and this service believes it.

**Extension procedure.** Adding a claim means extending `IdentityContext` and confirming the gateway
actually populates it (`api-gateway.md` §3.9). Do not add authorization logic that assumes
`IsAuthenticated` implies verification.

**Failure modes.** Header spoofing is possible for anything that can reach the service directly,
bypassing the gateway; `Guid.Empty` from a malformed id silently fails every ownership comparison;
and the `AppContext.Empty` fallback is exactly the unauthenticated state that §3.20's guard lets
through.

### 3.32 `Correlation-Context` header ingestion

**Definition.** `…Infrastructure/Extensions.cs:113-116` — `GetCorrelationContext(IHttpContextAccessor)`
reads the `Correlation-Context` request header and returns its value, or an empty string.

**Lifecycle.** Wired into the Convey correlation context accessor at startup `[convey]`; consumed by
`AppContextFactory`.

**Invariants & enforcement.** No validation, no signature, no size limit. Malformed JSON yields an
empty context — which is the unauthenticated state (§3.20).

**Extension procedure.** Changing the header name requires the same change at the gateway
(`api-gateway.md` §3.9) and in every service that reads it — all seven use the same literal.

**Failure modes.** This header is the platform's authorization channel and it is plaintext,
unauthenticated and trusted. Its trustworthiness rests entirely on network isolation. See **B-1** and
`service-summaries.md` gap **G14**.

### 3.33 External event subscription — seven events, one bound to the wrong exchange

**Definition.** Seven `SubscribeEvent<T>()` calls (`…Infrastructure/Extensions.cs:102-108`), each
resolving its exchange from a `[Message("<exchange>")]` attribute on the event class.

| Event | Declared exchange | Handler behaviour on a miss |
| --- | --- | --- |
| `CustomerCreated` | `customers` | throws `CustomerAlreadyAddedException` on duplicate (silent on AMQP) |
| `DeliveryStarted` | `deliveries` | throws `OrderNotFoundException` → **`_ => null`**, silent |
| `DeliveryCompleted` | `deliveries` | same |
| `DeliveryFailed` | `deliveries` | same |
| `ResourceReserved` | `availability` | throws `OrderForReservedVehicleNotFoundException` → silent |
| `ResourceReservationCanceled` | `availability` | same; cancels with reason `"Reservation canceled at: {DateTime}"` (`ResourceReservationCanceledHandler.cs:32`) |
| **`ParcelDeleted`** | **`deliveries`** | `GetContainingParcelAsync` then **`return`s silently** on null |

**The `ParcelDeleted` subscription is bound to the wrong exchange.**
`…Application/Events/External/ParcelDeleted.cs:7` declares `[Message("deliveries")]`, but the producer
of `parcel_deleted` is `parcels-service`, which publishes on the exchange `parcels`
(`component-internals/parcels-service.md` §3.13). The queue
`orders-service/deliveries.parcel_deleted` is bound to an exchange on which that routing key is never
published, so **the handler has never executed and cannot execute**. Its purpose — removing a parcel
from any order that contains it when the parcel is deleted upstream — is therefore not happening. Note
the compounding: even if the binding were corrected, `Order.DeleteParcel` does not remove the parcel
(§3.8), so the handler would still leave the order unchanged. Two independent defects on the same
path.

**Lifecycle.** Queues are declared from the template in the `rabbitMq` section of
`…Api/appsettings.json` (§3.35), producing names of the form `orders-service/<exchange>.<message>`.

**Invariants & enforcement.** Nothing validates that a declared exchange has a producer. A wrong
`[Message]` value produces a bound, empty, permanently idle queue — visible only by inspecting the
broker.

**Extension procedure.** Add the event class with the **producer's** exchange name, add the handler,
add `SubscribeEvent<T>()`, add a log template (§3.38), and decide the failure behaviour explicitly —
throwing gets you a `_ => null` silent drop unless you also add an `ExceptionToMessageMapper` arm.
After deploying, confirm the queue is receiving; an idle queue is the symptom of this class of bug.

**Failure modes.** Beyond the mis-binding: five of the seven handlers throw on a miss and every one of
those throws is swallowed (§3.26), and the sixth (`ParcelDeletedHandler.cs:25-29`) returns silently by
construction. **No inbound event failure in this service is reported to anyone.**

### 3.34 The three synchronous service clients

**Definition.** Three thin wrappers over Convey's `IHttpClient` `[convey]`
(`…Infrastructure/Services/Clients/`):

| Client | Call | Returns |
| --- | --- | --- |
| `ParcelsServiceClient` | `GET {parcels}/parcels/{id}` | `ParcelDto` |
| `VehiclesServiceClient` | `GET {vehicles}/vehicles/{id}` | `VehicleDto` (`Id`, `PricePerService`) |
| `PricingServiceClient` | `GET {pricing}/pricing?customerId={…}&orderPrice={…}` | `OrderPricingDto` |

Base URLs come from the `httpClient.services` map in `…Api/appsettings.json`, keyed `parcels`,
`pricing`, `vehicles`. In the default and docker profiles those are **logical names resolved through
Fabio/Consul**; in the local profile they are explicit `localhost` ports (§3.40, §3.43).

**Lifecycle.** `ParcelsServiceClient` is called by `AddParcelToOrderHandler:38`; the other two by
`AssignVehicleToOrderHandler:54` and `:60`.

**Invariants & enforcement.** Each client returns `null` on a non-success response `[convey]`. Two of
the three call sites null-check; the pricing call does not (§3.21). **No timeout, no retry and no
circuit breaker is configured** — `…Infrastructure/Extensions.cs:50-84` registers
`AddHttpClient()` with no Polly policy, and the `httpClient` section sets no timeout. A slow
`pricing-service` therefore blocks a RabbitMQ consumer thread indefinitely. This is
`service-summaries.md` gap **G12** / coupling **C6**.

**Extension procedure.** Add a client by writing the interface + implementation, registering it in
`Extensions.cs:52-63`, and adding the service key to `httpClient.services` in **all four**
appsettings profiles — a missing key yields a null base URL and a request to a relative path, which
fails at runtime rather than at startup.

**Failure modes.** Unbounded blocking on the AMQP consumer thread; a `NullReferenceException` on the
pricing path; and two sequential synchronous calls inside a read-modify-write window that has no
concurrency control (§3.17). `AssignVehicleToOrder` is thus the single most fragile operation in the
service: three defects (§3.17, §3.21, §3.34) intersect on it.

### 3.35 Queue naming and message conventions

**Definition.** The `rabbitMq` section of `…Api/appsettings.json` fixes the messaging conventions:
a topic exchange named `orders` (durable, auto-created), snake-case casing for both exchanges and
messages, a queue-name template of the form `orders-service/{{exchange}}.{{message}}`, a
`message_context` context header, and a `span_context` header for tracing.

**Representation & storage.** The template yields per-service, per-exchange, per-message queues —
`orders-service/customers.customer_created`, `orders-service/availability.resource_reserved`, and so
on. Every service in the platform follows this template, so a queue name identifies both consumer and
producer at a glance; an idle queue is therefore diagnosable by name (§3.33).

**Lifecycle.** Queues and bindings are declared at startup by the Convey RabbitMQ client from the
`SubscribeCommand`/`SubscribeEvent` calls `[convey]`.

**Invariants & enforcement.** Casing is convention-driven: the routing key is the snake-cased type
name. **Renaming a C# class silently renames the routing key**, breaking every subscriber, with no
compile-time signal — this is the single most dangerous refactor in the service.

**Extension procedure.** Do not rename message classes. If a rename is unavoidable, deploy a
transitional subscriber for the old name, migrate producers, then remove it — and update
`…Operations…/messages.json` in the same change.

**Failure modes.** Silent rename breakage; and no dead-letter configuration appears in the section, so
a message whose handler throws is acknowledged and gone (§3.26) rather than parked.

### 3.36 `[Contract]` and `UsePublicContracts`

**Definition.** `…Application/ContractAttribute.cs` is a marker; `…Infrastructure/Extensions.cs:92`
calls `UsePublicContracts<ContractAttribute>()` `[convey]`, which exposes the annotated types as a
machine-readable contract document.

**Lifecycle.** Reflection over the loaded assemblies at startup `[convey]`.

**Invariants & enforcement.** Purely by convention — nothing fails when the attribute is missing.
**Two types are missing it**: the command `AssignVehicleToOrder` (§3.18) and the event
`VehicleAssignedToOrder` (§3.28), plus three rejected events — `AssignVehicleToOrderRejected`,
`OrderForDeliveryNotFound` and `OrderForReservedVehicleNotFound`. Every one of the five sits on the
vehicle-assignment path, which suggests they were added in one later change that skipped the
convention.

**Extension procedure.** Add `[Contract]` to every new command, event and rejected event. Adding it to
the five above changes the published contract document, so confirm no consumer asserts on its current
content first.

**Failure modes.** A consumer generating clients from the contract document silently omits the vehicle
assignment operation entirely.

### 3.37 Dispatcher-bound HTTP endpoints — and the two commands with no route

**Definition.** `…Api/Program.cs:33-42` maps eight endpoints inside `UseDispatcherEndpoints`
`[convey]`:

| Method | Path | Bound to |
| --- | --- | --- |
| GET | `""` | a static service-name response |
| GET | `orders/{orderId}` | `GetOrder` query |
| GET | `orders` | `GetOrders` query |
| POST | `orders` | `CreateOrder`, responding `Created($"orders/{cmd.OrderId}")` |
| DELETE | `orders/{orderId}` | `DeleteOrder` |
| POST | `orders/{orderId}/parcels/{parcelId}` | `AddParcelToOrder` |
| DELETE | `orders/{orderId}/parcels/{parcelId}` | `DeleteParcelFromOrder` |
| POST | `orders/{orderId}/vehicles/{vehicleId}` | `AssignVehicleToOrder` |

**Lifecycle.** Convey binds route values and the request body onto the command, resolves the handler
and dispatches `[convey]`. Only `POST orders` returns a `201` with a location header; the other
mutating routes return the framework default.

**Invariants & enforcement.** **`ApproveOrder` and `CancelOrder` have no HTTP route.** They are
subscribed on AMQP (`…Infrastructure/Extensions.cs:96-97`) and declared in
`…Operations…/messages.json`, but they appear in **neither** `ntrada.yml` **nor** `ntrada-async.yml`
— so no external caller can invoke either through the platform's front door. Their only reachable
producer is `ordermaker-saga-service`, which publishes `CancelOrder` with the literal reason
`"Because I'm saga"` (`…OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs:141`) and
**never publishes `ApproveOrder` at all**. `ApproveOrder` therefore has **no producer anywhere in this
workspace**: approval happens exclusively as a side effect of `resource_reserved` (§3.4). See
**Q-4**.

**Extension procedure.** Adding a route means one line in `Program.cs` *and* a corresponding entry in
both gateway profiles — the gateway is an allow-list, so an unlisted route is unreachable even when
the service exposes it (`api-gateway.md` §3.5).

**Failure modes.** The route table and the subscription list are maintained independently and have
already diverged by two commands; nothing detects the divergence.

### 3.38 `MessageToLogTemplateMapper`

**Definition.** `…Infrastructure/Logging/MessageToLogTemplateMapper.cs` maps each message type to a
`HandlerLogTemplate` with `After` and, in some cases, `OnError` phrasing `[convey]`. Fifteen
`typeof(...)` entries are registered.

**Lifecycle.** Consulted by Convey's logging decorator around each handler `[convey]`; `UseLogging()`
is enabled at `…Api/Program.cs:43`.

**Invariants & enforcement.** A message type absent from the dictionary logs nothing beyond the
framework default. The dictionary is hand-maintained and covers commands and inbound events, not
outbound events.

**Extension procedure.** Add an entry per new message. Interpolate only ids — the templates embed
`{OrderId}`-style placeholders, and adding a customer or user field here would route personal data
into the log stream, which §3.39's redaction list does not cover.

**Failure modes.** Silent absence of logging for unmapped messages; no compile-time link between the
dictionary and the handler set.

### 3.39 Log redaction and path exclusion

**Definition.** The `logger` section of `…Api/appsettings.json` configures Serilog-style sinks with a
level, an application name, and two filtering lists: excluded HTTP paths (the health/root route) and
excluded log properties.

**Representation & storage.** Console and file sinks are enabled by default; a Seq sink is configured
with a placeholder API key literal and is disabled. The placeholder is called out here because an
unchanged default is itself the finding — it is not a real credential.

**Invariants & enforcement.** The excluded-property list is a static allow-of-omission; it is not
content-aware. Nothing prevents a new log template (§3.38) from emitting an unlisted sensitive field.

**Extension procedure.** Add the property name to the exclusion list when introducing a field that
should not be logged, in the **same change** as the template that would emit it.

**Failure modes.** Redaction by enumeration fails open — the default is to log.

### 3.40 Consul registration and Fabio addressing

**Definition.** Three cooperating sections in `…Api/appsettings.json`: `consul` (service id
`orders-service`, the port it advertises, and a ping/health path), `fabio` (the load-balancer address
used as the HTTP client's base) and `httpClient.services` (§3.34).

**Lifecycle.** Registration happens at startup and deregistration at shutdown `[convey]`. Outbound
calls address `http://fabio/<service>/...`, so the logical names in `httpClient.services` are resolved
by Fabio against Consul's registry rather than by DNS.

**Invariants & enforcement.** A service that fails to register is simply unreachable by logical name;
there is no startup assertion. The advertised port must match the container's actual port — the
`Dockerfile` exposes 80 and the docker profile aligns them (§3.43, §3.46).

**Extension procedure.** Both `consul` and `fabio` are disabled in the local profile, so a developer
must set explicit URLs in `httpClient.services` — which the local profile does.

**Failure modes.** A stale Consul registration routes traffic to a dead instance until the health check
expires; nothing in this service detects that its own registration lapsed.

### 3.41 Inert JWT configuration

**Definition.** `…Api/appsettings.json:72-79` carries a complete `jwt` block — a certificate location,
an issuer, and validation switches.

**Invariants & enforcement.** **No code in `src/` reads it.** There is no
`AddJwt()`, no `UseAuthentication()`, no `[Authorize]` and no `AddCertificateAuthentication()`
anywhere in the service; `AddSecurity()` (`…Infrastructure/Extensions.cs:83`) registers
encryption/hashing helpers `[convey]`, not authentication. The block is inherited from the solution
template.

**Extension procedure.** If this service is ever to validate tokens itself — which is the correct fix
for §3.31's trust problem — this block becomes live and the identity chain changes from
"trust the header" to "verify the token". That is a cross-cutting change: the gateway would keep
forwarding the header, and the two mechanisms must agree on the claim names.

**Failure modes.** A reader auditing configuration reasonably concludes this service authenticates
requests. It does not. Recorded as **A-3**.

### 3.42 Vault — settings, PKI and dynamic Mongo credentials

**Definition.** The `vault` section of `…Api/appsettings.json` enables three features: a KV path from
which settings are loaded at startup, a PKI role that issues a certificate, and a database lease that
issues short-lived MongoDB credentials for the `orders-service` role. `…Api/Program.cs:44` calls
`UseVault()` `[convey]`.

**Lifecycle.** At startup, before configuration binding: settings are pulled, the certificate is
issued, and Mongo credentials are leased and renewed by a background task `[convey]`.

**Invariants & enforcement.** The token used to authenticate to Vault is the unchanged template
placeholder in the default profile. Vault is **disabled** in the local and docker profiles, so the
static Mongo connection string in those profiles is what is actually used during development.

**Extension procedure.** Adding a secret means adding a key under the service's KV path and reading it
through the ordinary configuration binding — Vault-sourced values are merged into `IConfiguration`, so
no code change is needed beyond the settings class.

**Failure modes.** A lease that fails to renew invalidates the Mongo credentials mid-flight; nothing in
this service handles that beyond whatever Convey does `[convey]`. If Vault is unreachable at startup,
the service starts with un-overridden defaults — that is, with the placeholder configuration —
**`Unverifiable — Missing Source Evidence`** as to whether `UseVault()` fails fast.

### 3.43 Environment layering

**Definition.** Four profiles: `appsettings.json` (default), `.local.json`, `.docker.json` and
`.development.json` (empty).

| Profile | What changes |
| --- | --- |
| default | full stack: Consul, Fabio, Jaeger, metrics, outbox, Vault, Redis all enabled |
| `.local` | service discovery, tracing, metrics and Vault disabled; `httpClient.services` points at explicit localhost ports for parcels/pricing/vehicles; Mongo local |
| `.docker` | logical docker hostnames for Mongo, RabbitMQ, Consul, Fabio, Jaeger; Vault disabled |
| `.development` | `{}` — an empty object, so `ASPNETCORE_ENVIRONMENT=development` yields the **default** profile, i.e. the full production stack |

**Invariants & enforcement.** The empty `development` profile is a trap: a developer who sets the
conventional ASP.NET environment name gets the full external-dependency stack, not the local one. The
intended local name is `local`.

**Extension procedure.** A new configuration key must be added to the default profile at minimum;
profiles override rather than merge-with-defaults per key `[framework]`, so a key added only to
`.docker` is absent locally.

**Failure modes.** Environment-name confusion (also present in `parcels-service`, where the test
profile and the script's environment name disagree — `component-internals/parcels-service.md` §3.29).

### 3.44 Redis and `IDateTimeProvider`

**Definition.** Two small registrations with very different significance.

`AddRedis()` (`…Infrastructure/Extensions.cs:76`) `[convey]` registers a distributed cache with an
instance-name prefix set in the `redis` section of `…Api/appsettings.json`. **No code in `src/` reads
or writes the cache** — no `IDistributedCache` injection appears anywhere. It is registered
infrastructure with no consumer, present because the platform template includes it.

`DateTimeProvider` (`…Infrastructure/Services/DateTimeProvider.cs`) implements
`IDateTimeProvider.Now => DateTime.UtcNow`. It is the service's only clock seam and is used by
`CreateOrderHandler:37`.

**Extension procedure.** If caching is introduced, the prefix already partitions this service's keys
from its siblings' in a shared Redis instance. For time, inject `IDateTimeProvider` rather than calling
`DateTime.UtcNow` — several places (notably `SetDeliveryDate`, §3.11) take time from the caller
instead, which is why date handling there is kind-ambiguous.

**Failure modes.** An unused dependency still fails the service at startup if Redis is unreachable and
the health check covers it — **`Unverifiable — Missing Source Evidence`** as to whether Convey's Redis
registration is lazy.

### 3.45 The Pact consumer test that never runs

**Definition.** `tests/Pacco.Services.Orders.PactConsumerTests` defines the consumer half of a
Pactify contract with `parcels-service` — `orders` as consumer, `parcels` as provider, exchanged as a
file under a shared `pacts` directory rather than through a broker.

**Invariants & enforcement.** **`Pacco.Services.Orders.sln` references only the four `src` projects.**
The test project is not in the solution, so `dotnet build`/`dotnet test` over the solution — which is
what `scripts/test.sh` runs — never compiles or executes it. The pact file is therefore never
regenerated, and the provider-side verification in `parcels-service` (which reads the same directory,
and is *also* excluded from its own solution — `component-internals/parcels-service.md` §3.31) checks
a file that nothing produces. **The consumer-driven contract test pair is inert on both sides.**

**Extension procedure.** `dotnet sln add` on both repositories, then reconcile: the provider test seeds
a parcel document without `CustomerId` or `Description`
(`…Parcels/tests/…/PACT/ParcelsApiPactProviderTests.cs`), so the pact may not match current DTOs.
Expect the first run to fail.

**Failure modes.** A contract test that cannot fail provides no protection while appearing in the
repository as though it does. This is `service-summaries.md` gap **G12**'s testing counterpart and is
recorded as **Q-8**.

### 3.46 Deployment identity

**Definition.** `Dockerfile` builds and publishes the API project, sets `ASPNETCORE_ENVIRONMENT` to
`docker` and exposes port 80. `scripts/dockerize.sh` tags the image under the account's
`pacco.services.orders` name. `.travis.yml` runs build → test → dockerize on `master` and `develop`.

**Invariants & enforcement.** The Consul-advertised port in the default profile and the container port
must agree; they do in the docker profile. The test stage runs `scripts/test.sh`, which — per §3.45 —
executes no tests, so the pipeline's test gate is vacuous.

**Extension procedure.** Adding a test project requires both `dotnet sln add` and confirming
`scripts/test.sh`'s environment name matches an existing profile.

**Failure modes.** A green pipeline that has verified nothing beyond compilation.

---

## 4. Primary control flows

Seven traces. Each runs entry point → functions → datastore and side effects, and names the point at
which a failure becomes invisible.

### 4.1 Create an order

**Entry (sync):** `POST /orders` → gateway route with `bind: customerId:@user_id` and
`resourceId: generate: true` (`ntrada.yml`, orders block) `[ntrada]` → `…Api/Program.cs:37`.
**Entry (async):** gateway publishes `create_order` to exchange `orders` →
`…Infrastructure/Extensions.cs:95`.

1. Outbox decorator wraps the handler; message id from the broker, or a fresh GUID on HTTP (§3.29).
2. `CreateOrderHandler.HandleAsync` (`:29-40`).
3. `_customerRepository.ExistsAsync(command.CustomerId)` (`:31`) → Mongo `customers` collection.
   Miss → `CustomerNotFoundException` (`:33`) → HTTP 400 `customer_not_found`, or
   `CreateOrderRejected` on AMQP (`ExceptionToMessageMapper.cs:31`). **This is one of the few failures
   reported on both transports.**
4. `Order.Create(command.OrderId, command.CustomerId, OrderStatus.New, _dateTimeProvider.Now)`
   (`:36`) → buffers `OrderStateChanged`.
5. `_orderRepository.AddAsync(order)` (`:37`) → insert into `orders`.
6. `_eventMapper.MapAll(order.Events)` (`:38`) → `OrderCreated(orderId)` (§3.23).
7. `_messageBroker.PublishAsync(events.ToArray())` (`:39`) → outbox row, then exchange `orders`,
   routing key `order_created`.

**Side effects:** one `orders` document, one published event. **No subscriber consumes
`order_created`** in this workspace (§3.24) — `operations-service` correlates it for the client's
poll. **Invisible failure point:** none on this path; it is the best-behaved flow in the service.

### 4.2 Add a parcel to an order

**Entry:** `POST /orders/{orderId}/parcels/{parcelId}` (`Program.cs:39`) or `add_parcel_to_order`
(`Extensions.cs:98`).

1. `AddParcelToOrderHandler.HandleAsync` (`:30-49`).
2. `_orderRepository.GetAsync(command.OrderId)` (`:32`) → miss → `OrderNotFoundException`.
3. `ValidateAccessOrFail(order)` (`:38`, defined `:51-58`) — **passes for unauthenticated callers**
   (§3.20).
4. **Synchronous HTTP** `_parcelsServiceClient.GetAsync(command.ParcelId)` (`:39`) →
   `GET {parcels}/parcels/{id}`. Null → `ParcelNotFoundException` (`:42`). **No timeout** (§3.34).
5. `order.AddParcel(new Parcel(parcel.Id, parcel.Name, parcel.Variant, parcel.Size))` (`:45`) →
   duplicate → `ParcelAlreadyAddedToOrderException`; otherwise buffers `ParcelAdded`.
6. `_orderRepository.UpdateAsync(order)` (`:46`) → whole-document replace (§3.17).
7. `MapAll` → `ParcelAddedToOrder(orderId, parcelId)` → publish (`:47-48`).

**Side effects:** the order document gains an embedded parcel snapshot; `parcels-service` consumes
`parcel_added_to_order` and stamps `OrderId` onto its parcel
(`component-internals/parcels-service.md` §3.13). **Invisible failure point:** step 6's replace can
erase a concurrent write (§3.17), and the published event has already gone out either way.

### 4.3 Delete a parcel from an order — the flow that does not do what it says

**Entry:** `DELETE /orders/{orderId}/parcels/{parcelId}` (`Program.cs:40`) or
`delete_parcel_from_order` (`Extensions.cs:99`).

1. `DeleteParcelFromOrderHandler.HandleAsync` — load, ownership guard (`:34-38`).
2. `order.DeleteParcel(command.ParcelId)` → finds the parcel, throws
   `OrderParcelNotFoundException` if absent, **buffers `ParcelDeleted` and returns without removing
   it** (`Order.cs:95-104`, §3.8).
3. `UpdateAsync` → the document is rewritten **unchanged in substance**.
4. `MapAll` → `ParcelDeletedFromOrder(orderId, parcelId)` → publish.
5. `parcels-service` consumes it and clears its parcel's `OrderId`
   (`component-internals/parcels-service.md` §3.13).

**Net effect:** HTTP success, an event, a downstream state change — and **no local state change**. The
two services now disagree permanently. Repeating the call repeats the event. This is the highest-value
defect in this document; §3.8 states the one-line fix and its four knock-on effects.

### 4.4 Assign a vehicle — the flow that can succeed silently

**Entry:** `POST /orders/{orderId}/vehicles/{vehicleId}` (`Program.cs:41`) or
`assign_vehicle_to_order` (`Extensions.cs:100`), typically published by the saga
(`…OrderMaker/…/AIOrderMakingSaga.cs:103` region).

1. Load (`:32`), ownership guard (`:38-42`).
2. `!order.HasParcels` → `OrderHasNoParcelsException` (`:44-47`) — mapped **wrongly** on AMQP
   (§3.26 defect 2), so silent there.
3. **`!order.CanAssignVehicle` → `return;`** (`:49-52`) — silent success. The saga waits forever
   (§3.21).
4. `GET {vehicles}/vehicles/{id}` (`:54`) → null → `VehicleNotFoundException`.
5. `GET {pricing}/pricing?customerId&orderPrice` (`:60`) → **unchecked**; null → `NullReferenceException`
   → generic 400 / silence.
6. `SetVehicle`, `SetTotalPrice`, `SetDeliveryDate` (`:61-63`) — `SetTotalPrice` throws for any status
   but `New`, so the `Canceled` branch of `CanAssignVehicle` always fails here (§3.9).
7. `UpdateAsync` (`:64`), then `PublishAsync(new VehicleAssignedToOrder(...))` (`:65`) — the buffered
   domain events are **not** mapped (§3.21).

**Side effects:** vehicle, price and delivery date stored; `vehicle_assigned_to_order` published;
`ordermaker-saga-service` advances to reservation. **Invisible failure points:** steps 2, 3 and 5 —
three of the seven steps fail without telling anyone on the AMQP path that the saga actually uses.

### 4.5 Delete an order

**Entry:** `DELETE /orders/{orderId}` (`Program.cs:38`) or `delete_order` (`Extensions.cs:96` region).

1. Load, ownership guard (`DeleteOrderHandler.cs:33-37`).
2. `if (!order.CanBeDeleted) throw new CannotDeleteOrderException(order.Id);` — only a `New` order is
   deletable (§3.5).
3. `_orderRepository.DeleteAsync(command.OrderId)` — **hard delete**, no tombstone, no soft-delete
   flag.
4. `PublishAsync(new OrderDeleted(command.OrderId))` — constructed by hand, like
   `VehicleAssignedToOrder`, not via `EventMapper`.

**Side effects:** the document is gone; `parcels-service` releases every parcel it believes belongs to
that order — but **only one of them**, because it looks the order up with a single-result
`GetByOrderAsync` (`component-internals/parcels-service.md` §3.9). An order with three parcels leaves
two permanently marked as belonging to a deleted order.

### 4.6 Reservation approves an order — the correlation flow

**Entry:** `resource_reserved` on exchange `availability` (`Extensions.cs:102-108` region).

1. `ResourceReservedHandler.HandleAsync`.
2. `_orderRepository.GetAsync(@event.ResourceId, @event.DateTime)` → the **two-field lookup**:
   `o.VehicleId == vehicleId && o.DeliveryDate == deliveryDate.Date`
   (`…Mongo/Repositories/OrderMongoRepository.cs:29-30`). This is an unindexed exact-equality match on
   a nullable `DateTime` (§3.11, §3.14).
3. Miss → `OrderForReservedVehicleNotFoundException` → `ExceptionToMessageMapper` resolves it to
   `null` for inbound events (§3.26 defect 5) → **silent**, message acknowledged.
4. Hit → `order.Approve()` → `New`/`Canceled` only, else `CannotChangeOrderStateException` → also
   **silent** (§3.4).
5. `UpdateAsync`, `MapAll` → `OrderApproved(orderId)` → publish.

`resource_reservation_canceled` follows the identical shape but calls
`order.Cancel($"Reservation canceled at: {@event.DateTime}")` — a reason that is published in
`order_canceled` and then **discarded, never stored** (§3.15).

**Invisible failure points:** steps 3 and 4 — the only two ways this flow can fail, and both are
silent. An order that stops advancing to `Approved` produces no error anywhere; the only diagnostic is
comparing the `availability` reservation against the order's `VehicleId`/`DeliveryDate` by hand.

### 4.7 Delivery drives the order to completion

**Entry:** `delivery_started` / `delivery_completed` / `delivery_failed` on exchange `deliveries`.

1. `Delivery{Started,Completed,Failed}Handler` → `_orderRepository.GetAsync(@event.OrderId)`.
2. Miss → `OrderNotFoundException` → no mapper arm for inbound events → **silent** (§3.26 defect 3;
   this is exactly the case `DeliveringOrderRejected` and `CompleteOrderRejected` were written for and
   are never used).
3. `order.SetDelivering()` (from `Approved` only), `order.Complete()` (from `Delivering` only), or
   `order.Cancel(...)`. Wrong-state → `CannotChangeOrderStateException` → **silent**.
4. `UpdateAsync`, `MapAll` → `OrderDelivering` / `OrderCompleted(orderId, customerId)` /
   `OrderCanceled(orderId, reason)` → publish.

**Side effects:** `customers-service` consumes `order_completed` and increments that customer's
completed-order count, which feeds its VIP promotion
(`component-internals/customers-service.md` §3.21). Every other event on this path has no subscriber.

**Invisible failure points:** both step 2 and step 3. Since `Delivering` and `Completed` are reachable
*only* through this flow, **every failure to advance an order to completion is silent**.

---

## 5. Persistence & schema evolution

### 5.1 What is stored

| Collection | Document | Key | Written by | Read by |
| --- | --- | --- | --- | --- |
| `orders` | `OrderDocument` + embedded `Parcel[]` | `Guid Id` | `OrderMongoRepository.{AddAsync,UpdateAsync,DeleteAsync}` | `OrderMongoRepository.GetAsync` ×3, `GetOrderHandler`, `GetOrdersHandler` |
| `customers` | `CustomerDocument` (id only) | `Guid Id` | `CustomerCreatedHandler` | `CreateOrderHandler` |
| inbox | Convey outbox schema `[convey]` | message id | outbox decorators | outbox processor |
| outbox | Convey outbox schema `[convey]` | generated | `MessageBroker` | outbox processor |

Database name and collection names come from the `mongo` and `outbox` sections of
`…Api/appsettings.json`; the repository registrations are at `…Infrastructure/Extensions.cs:80-81`.
Database-per-service holds: no other component in the workspace connects to `orders-service`.

### 5.2 Adding a field

Schemaless storage means the change is mechanically easy and semantically dangerous. Add the property
to `Order`, to `OrderDocument`, to `AsDocument`, to `AsEntity`, and — if client-visible — to `AsDto`
and `OrderDto`. **No compiler check links these five edits** (§3.15). Documents written before the
change deserialise the new field to its CLR default: `null` for a reference type,
`0`/`Guid.Empty`/`default(DateTime)` for a value type. If the default is not a valid domain value,
backfill before deploying code that reads it — there is no migration mechanism, no migration folder
and no schema-version field anywhere in this repository, so the backfill is a one-off script written
by hand.

### 5.3 Changing `OrderStatus`

**Establish the stored form first** (§3.16, **Q-10**) — `db.orders.findOne({}, {status: 1})` on any
deployed environment. Until that is done, only *appending* a member is safe. With the answer in hand:

| Change | If stored as ordinal | If stored as member name |
| --- | --- | --- |
| **Append** | safe | safe |
| **Reorder / insert** | silently relabels every stored order — `Approved` (1) becoming `OnHold` (1) turns every approved order into an on-hold order, with no error at any layer | harmless on disk |
| **Rename** | stored data untouched; the API's lowercase status string changes — a silent break for any consumer matching on `"new"`, `"approved"`, `"delivering"`, `"completed"`, `"canceled"` | **both** the stored value and the API string change; existing documents fail to deserialise |

Whichever representation is in force, pin it explicitly before making a structural change: add
`[BsonRepresentation(BsonType.String)]` (or `BsonType.Int32`) to `OrderDocument.Status` so the
document type no longer depends on a globally registered convention, migrate every stored document
to that form, and only then reorder or rename. Note the American spelling `canceled` is the wire
value.

### 5.4 Evolving the customer replica

The `customers` collection holds ids only (§3.7) and is fed exclusively by `customer_created`. Adding
a field means (a) extending `Customer`/`CustomerDocument`/the mappers, (b) subscribing to whatever
event carries updates, and (c) **backfilling** — this service cannot reconstruct the replica, because
`customers-service` publishes no snapshot or replay event
(`component-internals/customers-service.md` §3.20). Practically, a backfill means a one-off read
against `customers-service`'s API.

### 5.5 Repairing the parcel-set divergence

If §3.8's missing `_parcels.Remove(parcel)` is fixed, stored documents are **not** repaired by the
deployment. Every order that has ever had a parcel "deleted" still contains it. Reconciliation
requires reading each order's embedded parcels and checking each parcel's `OrderId` in
`parcels-service` — a parcel whose `OrderId` is null or points elsewhere should be dropped from the
order. There is no event this service could replay to do this automatically. Sequence the fix as:
reconcile first, then deploy, because the code change alters `HasParcels` and therefore the behaviour
of `AssignVehicleToOrder` (§3.8).

### 5.6 Indexes

**No index is created anywhere in this repository** — no `CreateIndex` call, no index attribute, no
initialisation script. Three query shapes run unindexed:

| Query | Shape | Cost |
| --- | --- | --- |
| `GetAsync(id)` | `_id` | fine — the default index |
| `GetAsync(vehicleId, deliveryDate)` | two-field equality | full scan per reservation event |
| `GetContainingParcelAsync(parcelId)` | `$elemMatch` over an embedded array | full scan per parcel deletion |
| `GetOrders` with `CustomerId` | single-field equality | full scan per query |
| `GetOrders` without `CustomerId` | none | full scan **returning every order** |

Adding `{CustomerId: 1}`, `{VehicleId: 1, DeliveryDate: 1}` and `{"Parcels.Id": 1}` is the highest-value
persistence change available and requires no schema change. Recorded as **Q-7**.

### 5.7 Message-contract evolution

Messages are a schema too, and they are the more fragile one:

- **Renaming a message class renames the routing key** (§3.35) and silently breaks every subscriber.
- **Adding a field** to a published event is backward-compatible: older consumers ignore it.
- **Removing or renaming a field** is not: consumers deserialise it to `default` with no error. The
  `OrderCanceled.Reason` field is the illustrative case — it is already effectively vestigial, since
  the value is never persisted (§3.15) and its only subscriber ignores it.
- Any change here must be mirrored in `…Operations…/messages.json`, which is maintained by hand in a
  different repository (`component-internals/operations-service.md` §3.9).

---

## 6. Surface → internals map

Read-only, mutating, or absent — for every published surface.

### 6.1 HTTP routes

| Route | Kind | Internals reached | Notes |
| --- | --- | --- | --- |
| `GET /` | read-only | none — a static response (`Program.cs:34`) | health/identification |
| `GET /orders/{orderId}` | read-only | `GetOrderHandler` → `orders` collection → `AsDto` | **no ownership check** (§3.22) |
| `GET /orders` | read-only | `GetOrdersHandler` → `AsQueryable` scan | unbounded; silent empty list on identity mismatch |
| `POST /orders` | mutating | §4.1 | 201 + `Location: orders/{id}` |
| `DELETE /orders/{orderId}` | mutating | §4.5 | hard delete, `New` only |
| `POST /orders/{orderId}/parcels/{parcelId}` | mutating | §4.2 | synchronous call to `parcels-service` |
| `DELETE /orders/{orderId}/parcels/{parcelId}` | mutating | §4.3 | **publishes without mutating** |
| `POST /orders/{orderId}/vehicles/{vehicleId}` | mutating | §4.4 | two synchronous calls; can no-op silently |
| approve an order | **absent** | — | no route in `Program.cs` and none in either gateway profile (§3.37) |
| cancel an order | **absent** | — | same; AMQP-only, saga-driven |

### 6.2 Inbound AMQP — commands

| Routing key | Kind | Internals | Rejected event on failure |
| --- | --- | --- | --- |
| `create_order` | mutating | §4.1 | `CreateOrderRejected` |
| `delete_order` | mutating | §4.5 | `DeleteOrderRejected` |
| `add_parcel_to_order` | mutating | §4.2 | `AddParcelToOrderRejected` |
| `delete_parcel_from_order` | mutating | §4.3 | `DeleteParcelFromOrderRejected` (not an `IRejectedEvent`, §3.27) |
| `assign_vehicle_to_order` | mutating | §4.4 | `AssignVehicleToOrderRejected` — but **not** for `OrderHasNoParcelsException` (§3.26) |
| `approve_order` | mutating | `ApproveOrderHandler` → `Order.Approve()` | **none** — no mapper arm; **no producer in the workspace** |
| `cancel_order` | mutating | `CancelOrderHandler` → `Order.Cancel(reason)` | `CancelOrderRejected` for some exceptions only |

### 6.3 Inbound AMQP — events

| Routing key | Exchange | Kind | Internals | Failure signal |
| --- | --- | --- | --- | --- |
| `customer_created` | `customers` | mutating | writes the id replica | none (duplicate throws, silently) |
| `delivery_started` | `deliveries` | mutating | `SetDelivering` | **none** |
| `delivery_completed` | `deliveries` | mutating | `Complete` | **none** |
| `delivery_failed` | `deliveries` | mutating | `Cancel` | **none** |
| `resource_reserved` | `availability` | mutating | correlate → `Approve` | **none** |
| `resource_reservation_canceled` | `availability` | mutating | correlate → `Cancel` | **none** |
| `parcel_deleted` | `deliveries` | **absent in practice** | handler exists but the binding is wrong (§3.33) | never fires |

### 6.4 Outbound

| Direction | Surface | Kind |
| --- | --- | --- |
| Published events | the nine in §3.24 | mutating for consumers |
| Rejected events | the eleven in `…Application/Events/Rejected/`, two of which are never constructed | informational |
| `GET {parcels}/parcels/{id}` | read-only | blocking, no timeout |
| `GET {vehicles}/vehicles/{id}` | read-only | blocking, no timeout |
| `GET {pricing}/pricing?…` | read-only | blocking, no timeout, **unchecked result** |

### 6.5 Operational surface

Swagger (path configured in the `swagger` section of `…Api/appsettings.json`), the public-contracts
document (§3.36), Prometheus metrics, Jaeger spans and the Consul health endpoint. All read-only.
None is exposed through the gateway — the gateway's orders block routes only the eight application
routes above.

---

## 7. Change/extension guide

### 7.1 Fix the parcel deletion defect

Add `_parcels.Remove(parcel);` at `…Core/Entities/Order.cs:102`. Then, in the same change:
reconcile stored documents (§5.5); re-test `AssignVehicleToOrder` for orders whose parcels were all
deleted, which now throw `OrderHasNoParcelsException`; and accept that a repeated delete now returns
an error rather than success — coordinate with `Pacco.Web` if it retries.

### 7.2 Make AMQP failures visible

Three edits to `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs`, in priority order:

1. Add a `CannotChangeOrderStateException` arm covering `ApproveOrder`, `CancelOrder`,
   `AssignVehicleToOrder` and the three delivery events. This is the single highest-value change in
   the service — it converts the majority of silent failures into rejected events.
2. Add inbound **event** types to the nested switches for `OrderNotFoundException` and
   `OrderForReservedVehicleNotFoundException`, wiring up `DeliveringOrderRejected`,
   `CompleteOrderRejected` and `OrderForDeliveryNotFound`, which already exist and are already
   declared in `messages.json`.
3. Fix the `OrderHasNoParcelsException` arm: move it under `AssignVehicleToOrder` and construct
   `AssignVehicleToOrderRejected` with the vehicle id, not the parcel id (§3.26).

Add a warning log at `MessageBroker.cs:69` in the same change so future null drops are at least
visible.

### 7.3 Close the authorization bypass

Invert the six duplicated conditions (§3.20) to
`if (!identity.IsAuthenticated || (identity.Id != order.CustomerId && !identity.IsAdmin))`, extracting
them into one shared method first. **Before deploying**, verify that every internal publisher supplies
a correlation context with a `user.id` — specifically the saga's `CancelOrder`
(`…OrderMaker/…/AIOrderMakingSaga.cs:141`) and the gateway's async profile. If either does not,
tightening the guard breaks them; the alternative is a service-to-service identity rather than a
user identity. See **B-1** and **Q-2**.

### 7.4 Add a command

1. Class under `…Application/Commands/` with `[Contract]`.
2. `ICommandHandler<T>` under `Handlers/`, with the ownership guard if it touches an existing order.
3. `SubscribeCommand<T>()` in `…Infrastructure/Extensions.cs:95-101`.
4. Route in `…Api/Program.cs:33-42` **and** in both `ntrada.yml` and `ntrada-async.yml`.
5. A rejected-event type implementing `IRejectedEvent`, plus arms in `ExceptionToMessageMapper` for
   every exception the handler can throw.
6. A `MessageToLogTemplateMapper` entry.
7. A `messages.json` entry in `hianshul100_Pacco.Services.Operations`.

Steps 3–7 all fail silently when skipped. Steps 4 and 7 cross repository boundaries.

### 7.5 Add or change an order status

Follow §3.4's five-step procedure and §5.3's storage rules. Check `CanBeDeleted` and
`CanAssignVehicle` (§3.5), which enumerate statuses positively and therefore exclude anything new.

### 7.6 Add a synchronous dependency

Prefer not to — `AssignVehicleToOrder` already demonstrates the cost (§3.34). If unavoidable: create
the client in `…Infrastructure/Services/Clients/`, register it in `Extensions.cs:52-63`, add the
service key to **all four** appsettings profiles, **null-check the result**, and add a timeout policy
— none exists today, and a slow dependency blocks a RabbitMQ consumer thread indefinitely.

### 7.7 Add an index

See §5.6. Convey's Mongo registration offers no index hook that is visible from this repository, so an
index is created either by a startup task added to `…Infrastructure/Extensions.cs:50-84` or by an
operational script. Whichever is chosen, record it — the absence of any index artifact is currently
indistinguishable from a deliberate decision.

### 7.8 Make the tests actually run

`dotnet sln add tests/Pacco.Services.Orders.PactConsumerTests` (§3.45), then coordinate with
`parcels-service`, whose provider test is excluded from its own solution and seeds a parcel document
missing `CustomerId` and `Description` (`component-internals/parcels-service.md` §3.31). Expect the
first run to fail; that failure is the point.

### 7.9 Maintenance contract

**Any later phase that changes this component's internals must update this model in the same change.**
Concretely: a new command, event, status, collection or dependency adds a `§3.n` entry and a §6 row; a
change to a failure path updates the loud/silent statement in the affected concept; a fix to any
defect recorded here must strike the defect rather than leave it standing. If a `§3.n` entry is
removed, renumber and update §2 so the concept table and §3 stay in one-to-one correspondence, and
update the concept count in `component-internals/index.md` §1.

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

- **A-1.** `Convey 0.4.*` behaves as its usage implies: `AddMongoRepository` binds a collection by
  name; `SubscribeCommand`/`SubscribeEvent` declare a queue per `(service, exchange, message)`;
  `UseDispatcherEndpoints` binds route values and the body onto the command; a null from
  `IExceptionToMessageMapper` means "acknowledge and drop". None of this is verifiable in this
  workspace — the package has no source here.
- **A-2.** `decimal` is serialised as `Decimal128`, per the driver default, since no
  `[BsonRepresentation]` attribute overrides it (`…Mongo/Documents/OrderDocument.cs`). The parallel
  assumption about `OrderStatus` is **not** made: §3.16 shows the evidence is contradictory and
  carries it as **Q-10** instead.
- **A-3.** The `jwt` configuration block is inert. Asserted from the absence of any reader in `src/`;
  it would be falsified by a Convey extension that reads the section implicitly under `AddSecurity()`
  — see **Q-1**.
- **A-4.** The gateway is the only externally reachable path to this service, so the
  unauthenticated-bypass in §3.20 is currently mitigated by network topology. Asserted from the
  compose topology in `hianshul100_Pacco`, not from any control in this repository.
- **A-5.** `ntrada.yml` (sync) and `ntrada-async.yml` (async) are the two profiles actually deployed;
  the other gateway profiles in that repository are variants of the same two
  (`component-internals/api-gateway.md` §3.3).
- **A-6.** Base ref for every citation is `feature/12998/aidlc` in each read-only clone.

### 8.2 Blockers

- **B-1. The ownership guard is inert for unauthenticated callers.** Six handlers gate on
  `identity.IsAuthenticated &&` (§3.20). Any caller that reaches the service without a correlation
  context has unrestricted access to every order. Fixing it requires knowing what every internal
  publisher puts in the correlation context, which cannot be settled from this repository alone
  (**Q-2**).
- **B-2. `CancellationReason` is never persisted** (§3.15). The field exists on the aggregate, is set
  on cancellation and is published — but has no column, no mapping and no DTO field. Any feature that
  needs to show *why* an order was cancelled needs a schema change first.
- **B-3. Most AMQP failures are silent** (§3.26). Five exceptions have no mapper arm — including the
  one raised by every state transition — and seven `_ => null` fallbacks swallow the rest. A client
  using the async gateway profile has no way to learn that a command failed, and there is no timeout
  anywhere in the chain.
- **B-4. `DeleteParcel` does not delete** (§3.8, §4.3). Local state and `parcels-service` diverge
  permanently on every use of that route.
- **B-5. `parcel_deleted` is subscribed on the wrong exchange** (§3.33). The handler has never run.
  Correcting the binding alone does not fix the behaviour, because **B-4** sits behind it.
- **B-6. No concurrency control** (§3.17). Whole-document last-writer-wins, with the widest loss
  window on the path that makes two blocking HTTP calls.
- **B-7. No test executes.** The one test project is outside the solution (§3.45), so every change to
  this service is unverified by CI.
- **B-8. No timeout on any outbound HTTP call** (§3.34). A slow `pricing-service` blocks a consumer
  thread with no bound.

### 8.3 Open questions

- **Q-1.** Does `AddSecurity()` or any other Convey extension implicitly consume the `jwt`
  configuration block? Answerable only from Convey's source, which is not in this workspace.
  Restates `service-summaries.md` **Q2**.
- **Q-2.** What correlation context does the async gateway profile attach when it publishes a command,
  and what does `ordermaker-saga-service` forward? This decides whether §7.3's fix is safe. Restates
  `service-summaries.md` **Q5**.
- **Q-3.** Was the `OrderHasNoParcelsException` → `AssignVehicleToOrderRejected` arm under
  `AddParcelToOrder` (§3.26) a copy-paste slip, or does some caller rely on the current shape? Nothing
  in this workspace consumes `assign_vehicle_to_order_rejected`.
- **Q-4.** What is `ApproveOrder` for? It has a class, a handler, a subscription and a `messages.json`
  entry, but **no producer anywhere in this workspace** and no gateway route in either profile
  (§3.37). Either an intended manual-approval flow was never wired up, or the command is dead.
- **Q-5.** Should `deliveries-service` react to `order_canceled`? An order can be cancelled while
  `Delivering` (§3.4) and nothing tells the delivery side. Restates `service-summaries.md` **Q10**.
- **Q-6.** Does `availability-service` always emit reservation `DateTime` values truncated to
  midnight, and with what `DateTimeKind`? The correlation lookup in §3.11 is an exact-equality match
  and fails silently if not.
- **Q-7.** Are indexes created out-of-band (an ops runbook, a Mongo init script outside these repos)?
  Nothing in this repository creates one (§5.6), and five query shapes scan.
- **Q-8.** Is the file-based PACT exchange between `orders` and `parcels` still intended? Both halves
  are excluded from their solutions (§3.45), so the pact file has no producer and no verifier.
- **Q-9.** Why do five vehicle-assignment-path types lack `[Contract]` (§3.36) while every other
  message has it? A deliberate exclusion, or a change that skipped the convention?
- **Q-10.** Is `OrderStatus` stored as its integer ordinal or as its member name? The document type
  applies no `[BsonRepresentation]`, which implies the ordinal, but two copies of Convey's Mongo
  initializer in this workspace register a global `EnumRepresentationConvention(BsonType.String)`
  (§3.16). The two answers imply *opposite* prohibitions — never reorder, versus never rename — so
  the question blocks any structural change to the enum. Settled by one `db.orders.findOne()`
  against a deployed environment.

### 8.4 Cross-references

| Topic | Where |
| --- | --- |
| The upstream half of every route and async publish | `component-internals/api-gateway.md` |
| The saga that drives `CreateOrder`, `AddParcelToOrder`, `AssignVehicleToOrder`, `CancelOrder` | `component-internals/ordermaker-saga-service.md` |
| The consumer of `order_completed` and producer of `customer_created` | `component-internals/customers-service.md` |
| The producer of `delivery_*` events | `component-internals/deliveries-service.md` |
| The producer of `resource_reserved` / `resource_reservation_canceled` | `component-internals/availability-service.md` |
| The consumer of `order_canceled`, `order_deleted`, `parcel_added_to_order`, `parcel_deleted_from_order`; the other half of the PACT pair | `component-internals/parcels-service.md` |
| The `messages.json` manifest and the client-facing operation poll | `component-internals/operations-service.md` |
| Surface catalogue this model complements | `baselines/service-summaries.md` §2.2 |
| Route and message inventory | `baselines/api-inventory.md` |

**Patterns this component instantiates.**

| Pattern | How it appears here |
| --- | --- |
| [[inward-dependency-service-skeleton]] | `Core` → `Application` → `Infrastructure` → `Api`, four projects |
| [[aggregate-buffered-domain-events]] | `AggregateRoot._events` + `EventMapper` (§3.2, §3.23) — with `ClearEvents` never called |
| [[dispatcher-bound-cqrs-endpoints]] | `UseDispatcherEndpoints` (§3.37) |
| [[dual-mode-edge-write]] | the same seven commands over HTTP and AMQP (§1.3) |
| [[service-owned-topic-exchange-messaging]] | exchange `orders`, snake-case conventions (§3.35) |
| [[declarative-message-manifest-subscription]] | `[Message("<exchange>")]` on external events (§3.33) |
| [[rejected-event-failure-contract]] | eleven rejected events, five defects (§3.26, §3.27) |
| [[transactional-outbox-handler-decorator]] | `Outbox{Command,Event}HandlerDecorator` (§3.29) |
| [[database-per-service-with-document-mapping]] | `orders-service` DB, `AsEntity`/`AsDocument`/`AsDto` (§3.15) |
| [[event-carried-reference-replica]] | the id-only `customers` replica (§3.7) |
| [[narrow-synchronous-point-read]] | the three service clients (§3.34) |
| [[edge-enforced-authentication-with-identity-binding]] | `bind: customerId:@user_id` at the gateway; no verification here (§3.31) |
| [[transport-agnostic-caller-context]] | `IAppContext`/`AppContextFactory` (§3.31) |
| [[correlation-and-span-propagation]] | `Correlation-Context`, `span_context`, `Saga` forwarding (§3.25, §3.32) |
| [[structured-logging-with-property-redaction]] | `MessageToLogTemplateMapper` + the logger exclusion lists (§3.38, §3.39) |
| [[registry-mediated-discovery-and-routing]] | Consul + Fabio + `httpClient.services` (§3.40) |
| [[vault-issued-dynamic-credentials-and-service-pki]] | KV settings, PKI role, Mongo lease (§3.42) |
| [[composable-per-concern-environment-stacks]] | four appsettings profiles (§3.43) |
| [[prefix-partitioned-shared-cache]] | the Redis instance prefix — registered, unused (§3.44) |
| [[framework-supplied-platform-conventions]] | every `[convey]` mechanism in this document |
| [[independent-per-repository-release]] | Travis → Docker Hub per repository (§3.46) |
| [[consumer-driven-contract-test-pair]] | the Pactify consumer half — inert (§3.45) |
| [[layered-service-test-suite]] | **not** instantiated: no unit or integration test project exists here |

---

*Component-internals model for `orders-service`, batch 5 of 7. Derived from
`hianshul100_Pacco.Services.Orders` at `feature/12998/aidlc`, inspected read-only. Registered in
[component-internals/index.md](index.md) §1.*
