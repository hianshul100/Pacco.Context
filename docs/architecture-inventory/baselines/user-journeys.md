# Pacco — User Journeys & Cross-Capability Hand-offs (Discovery)

**Project Key:** Common Architecture
**Stage:** `architecture_discovery` — observed current-state journey landscape.
**No ADRs, no recommendations, no target state, no KG JSON, no test inventory.**
**Date of analysis:** 2026-09-04
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`

This document records the **sequenced navigation that is observable in the fourteen cloned Pacco
repositories** — what ordered paths a user or an automated actor is evidenced to take through the
platform, which capabilities each ordered step belongs to, where the owning capability changes
between two consecutive steps, and what data has to cross that change. It does **not** propose a
navigation model, an MFE boundary, a shell application, a composition strategy, or a target
information architecture.

**Inputs used**

- The thirteen cloned source repositories plus the artifact repository, which are the **source of
  truth** for every statement below. Where a prior document and the code disagree, the code wins
  and the disagreement is stated explicitly in §10.
- `docs/architecture-inventory/baselines/ui-inventory.md` — screen and route census. Its §1
  (exhaustive frontend asset sweep), §7.3 (routes and screens), §11.3 (routing model) and §12
  (capabilities mapped) are used as the **cross-reference for known screens and routes** rather than
  being re-derived. Screen properties are not restated here.
- `docs/architecture-inventory/baselines/capability-baseline.md` — capability identifiers `CAP-01`
  to `CAP-16` and their owning services. Every capability attribution in this file resolves there.
  Capability definitions are not restated here.
- `docs/architecture-inventory/baselines/api-inventory.md` — endpoint, event and hub contracts. Its
  §2 (41 synchronous gateway routes), §3 (asynchronous gateway routes), §4 (service HTTP
  endpoints), §5.2 (SignalR surface) and §9.6 (saga message flow) are the **cross-reference for
  every endpoint named in a journey step**. Request and response contracts are not restated here.
- `docs/architecture-inventory/repo-inventory.md` — repository inventory and per-repo file census.
- `.attachments/01_product_backlog_20260903_170135_37cf143b.xlsx` — backlog issue **12998**
  "Pacco - Discovery - Attempt-2", which fixes the fourteen-repository scope.
- **Catalogue retrieval outcome, recorded so it is auditable.** The knowledge catalogue was queried
  for documented user journeys, business workflows, personas and cross-service choreography. The
  graph holds **no nodes at all** under this workspace's tenant scope, and the prose fallback
  returned material about an unrelated product — a quick-service-restaurant order-management
  integration, with assumptions about "the upstream Order Management System", "an external event
  bus" and "a subset of pilot cafes (5–10 locations)". It names no Pacco service, capability,
  screen, route, persona or journey. **No documented Pacco journey, persona, or workflow was
  retrievable**, so every journey below is reconstructed from the cloned source. This mirrors the
  same mismatch already recorded in `ui-inventory.md` §13.4 and is carried as **Q4**.

**What counts as a journey in this document.** A journey requires **sequenced navigation evidence**
— an ordered set of steps where a specific artifact shows step *N* being followed by step *N+1*,
and where output from step *N* is what makes step *N+1* addressable. A list of endpoints, a list of
routes, or a catalogue of requests in arbitrary order is **not** a journey and is not recorded as
one. Where a source is a catalogue rather than a sequence, §1 says so explicitly and it is excluded
from §5.

**Evidence taxonomy used throughout.** *Observed* — read directly from a source, test, template or
config file, path and line cited. *Inferred* — a conclusion drawn from two or more observed facts,
labelled as such. *Assumption* — belief beyond what evidence shows, labelled `[assumption]` and
rolled into the ABQ section. *Unknown* / *not observed* — labelled as such, never filled with a
guess.

**Confidence labels used for journeys**, as required by the discovery method:

- **Confirmed** — the sequence is evidenced in a human-authored, executable scenario or test file
  that fixes the order of the steps.
- **Inferred** — the sequence is derived from adjacency (a link, a navigation call, or an
  entity-id dependency between two endpoints) and is not fixed by any tested or scripted order.
- **Partial** — some steps confirmed, some inferred.

**Cross-reference convention.** `A#` / `B#` / `Q#` refer to the *Assumptions, Blockers & Open
Questions* tables at the end of **this** file and nowhere else. `CAP-##` refers to
`capability-baseline.md` §1. `api-inventory.md §x` and `ui-inventory.md §x` refer to those files'
numbered sections.

## Table of contents

1. [Navigation source search — exhaustive, with absence proof](#1--navigation-source-search--exhaustive-with-absence-proof)
2. [Step sequences and adjacency](#2--step-sequences-and-adjacency)
3. [Journey-to-capability mapping and hand-off points](#3--journey-to-capability-mapping-and-hand-off-points)
4. [Shared state at hand-off points](#4--shared-state-at-hand-off-points)
5. [Journey catalogue](#5--journey-catalogue)
6. [Cross-capability navigation summary](#6--cross-capability-navigation-summary)
7. [Shared state summary](#7--shared-state-summary)
8. [Evidence quality](#8--evidence-quality)
9. [Confidence assessment](#9--confidence-assessment)
10. [Conflicts between sources](#10--conflicts-between-sources)

- [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions) — mandatory final section

---

## 1 — Navigation source search — exhaustive, with absence proof

The search was executed across **all fourteen clones**, walking each full tree with `.git` excluded.
Four sweeps were run: a **filename** sweep for router and test-runner configuration, a **directory**
sweep for the conventional navigation and end-to-end roots, a **keyword** sweep for navigation
primitives across every file type, and a **scenario-artifact** sweep for ordered request scripts.
All four results are given in full, matched and unmatched.

### 1.1 — Filename sweep for router and navigation configuration

*Observed.* A workspace-wide `find` for `router*`, `routes*`, `routes.rb`, `urls.py`, `web.php`,
`*.feature`, `cypress.json`, `cypress.config.*`, `playwright.config.*`, `*.cy.*`, `wdio.conf.*`,
`protractor.conf.*` returned **zero files in all fourteen clones**.

This covers, and finds nothing for, every framework form the discovery method names:
`src/router/index.*`, `src/routes.*`, `app/router.*`, `routes/index.*`, Rails `routes.rb`, Laravel
`routes/web.php`, and Django `urls.py`. Confidence: **high** — this is an exhaustive filesystem
walk, not a sample.

### 1.2 — Directory sweep for navigation and end-to-end roots

*Observed.* A directory-name walk for `router`, `routers`, `routes`, `pages`, `e2e`, `cypress`,
`playwright`, `integration`, `middleware`, `guards`, `__tests__`, `features`, `specs` and
`navigation` returned **zero matches in all fourteen clones**. A separate walk for any directory
matching `*test*` returned eleven directories, all C# test projects, listed in §1.4.

Specifically absent everywhere: `cypress/e2e/`, `cypress/integration/`, `tests/e2e/`, `e2e/`,
`playwright/`, `__tests__/e2e/`, a Next.js or Nuxt `pages/` directory, and a Next.js App Router
`app/` directory. Confidence: **high**.

### 1.3 — Keyword sweep for navigation primitives

*Observed.* Every clone was searched across **all file types** (including the 4 088-line vendored
`signalr.js` bundle) for each navigation primitive the discovery method names. The complete result
set:

| Primitive searched | Files matched (excluding this repository's own documentation) |
|---|---|
| `createBrowserRouter`, `createHashRouter`, `<Route`, `<Routes` (React Router) | **none** |
| `createRouter`, `routes:` array, `router-link`, `$router`, `this.$router.push` (Vue Router) | **none** |
| `RouterModule.forRoot`, `Routes` array, `canActivate` (Angular) | **none** |
| `router.push(`, `navigate(`, `redirect(`, `redirect_to` | **none** |
| `<Link`, `<NavLink` | **none** |
| `beforeEach`, `requireAuth`, `redirectIfAuthenticated` (navigation guards) | **none** |
| `pushState`, `hashchange`, `window.location`, `location.href` | **none** |
| `UseDefaultFiles`, `MapFallbackToFile`, `UseSpa` (ASP.NET Core client-route fallbacks) | **none** |
| `Express` `app.get` / `app.post` / `app.use` | **none** — no JavaScript server exists; there is no `package.json` anywhere (`ui-inventory.md` §1.5) |
| `<a href=…>` in any markup | **one match, and it is not navigation**: `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/index.html:6` is a `<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/…">` — a Bootstrap CDN stylesheet. There is **no `<a>` element** in the file |
| `<form>`, `action=`, `method=` on a form | **none** — the one HTML document has no `<form>` (`index.html:9-22`, read in full) |

The four rows that report a match against `hianshul100_Pacco.Context` only
(`pushState`, `hashchange`, `MapFallbackToFile`, `UseDefaultFiles`, `UseSpa`) match **the prose of
`ui-inventory.md` describing their own absence** — they are this documentation set quoting the
search terms, not code. They are recorded here so the raw result is auditable rather than silently
filtered. Confidence: **high**.

### 1.4 — Test-suite sweep — what exists and whether it sequences navigation

*Observed.* Eleven test directories exist, all C#, all in three repositories. Each was opened and
read to determine whether it fixes an **ordered multi-step sequence** or exercises a single
interaction.

| Test project | Files | What it actually does | Sequenced journey? |
|---|---|---|---|
| `Pacco.Services.Availability/tests/…Tests.EndToEnd/Sync/AddResourceTests.cs` | 1 (3 facts) | Each fact issues **one** `POST resources` through an in-process `PaccoApplicationFactory` client and asserts status `201`, the `Location` header, or the Mongo document (`:20-59`) | **No** — one HTTP call per test |
| `…Tests.EndToEnd/Sync/GetResourceTests.cs` | 1 (2 facts) | `GET resources/{id}` asserting `404` when absent, and a DTO after a **direct Mongo insert** used as arrange, not as a navigated step (`:18-40, :48-61`) | **No** — one HTTP call per test |
| `…Tests.Integration/Async/AddResourceTests.cs` | 1 (1 fact) | Publishes `AddResource` to the `availability` exchange, subscribes for `ResourceAdded`, reads the document (`:16-34`). A two-hop **message** flow inside one capability | **No** — single capability, no navigation |
| `…Tests.Performance/PerformanceTests.cs` | 1 | One NBomber step, `GET http://localhost:5001/resources`, asserting ≥100 RPS (`:12-38`) | **No** — a load step, not a sequence |
| `…Tests.Unit/…` (2 files) | 2 | `ReserveResourceHandlerTests`, `CreateResourceTests` — in-process handler and entity tests | **No** |
| `…Tests.Shared/…` (5 files) | 5 | `PaccoApplicationFactory`, `MongoDbFixture`, `RabbitMqFixture`, `OptionsHelper` — fixtures only | **No** |
| `Pacco.Services.Orders/tests/…PactConsumerTests/PACT/ParcelsApiPactConsumerTests.cs` | 1 | Declares **one** Pact interaction: `GET /parcels/{parcelId}` between consumer `orders` and provider `parcels` (`:24-37`) | **No** — one interaction, no order |
| `Pacco.Services.Parcels/tests/…PactProviderTests/…` | 4 | The provider side of the same single interaction, plus Mongo fixtures | **No** |

**No Gherkin feature file, no Cypress spec, no Playwright spec, and no `*.spec.*` / `*.test.*` /
`*.cy.*` file exists anywhere in the workspace** (§1.1, §1.2). The e2e-labelled project that does
exist —`Pacco.Services.Availability.Tests.EndToEnd` — is end-to-end in the *in-process HTTP + real
Mongo* sense, not in the *user walks through several screens* sense. Confidence: **high**.

### 1.5 — Scenario-artifact sweep — the sources that **do** carry sequence

*Observed.* Twelve `.rest` files exist. They are not covered by the discovery method's named
patterns, and they are the only artifacts in the workspace that fix an order of steps for a human
actor, so they were located by a dedicated `*.rest` / `*.http` sweep and each was read in full.

| File | Steps | Ordered? | Verdict |
|---|---:|---|---|
| `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest` | 24 | **Yes.** Every step carries a prose comment fixing its place — `### At first, create an account` (`:3`), `### Authenticate and grab the access token` (`:13`), `### Get your order details which should now be marked as completed` (`:196`) — and eight steps consume a value captured from an earlier step's response (`:25, :58, :77, :111, :116, :172`) | **A journey.** The single strongest navigation-sequence artifact in the workspace |
| `hianshul100_Pacco.APIGateway/Pacco.rest` | 41 | **No.** Grouped by service under banner comments (`# ============== IDENTITY ====================`, `:17`) with every variable pre-seeded to `00000000-0000-0000-0000-000000000000` (`:3-14`). No step consumes another step's output | **A catalogue, not a journey.** Excluded from §5 per the stated rule |
| The ten per-service `.rest` files (`Identity`, `Availability`, `Vehicles`, `Operations`, `Customers`, `Orders`, `Parcels`, `Pricing`, `Deliveries`, `OrderMaker`) | 5–7 each | **No.** Each is a flat list against one service's own port with placeholder GUID variables — e.g. `Pacco.Services.Orders.rest:2-6` seeds `orderId`, `parcelId`, `customerId`, `vehicleId` all to zeros | **Catalogues, not journeys.** Excluded from §5. `Pacco.Services.Identity.rest` is the closest to a sequence (sign-up → sign-in → me → refresh → revoke, `:6-49`) but it hard-codes `@accessToken = secret` (`:2`) rather than capturing it, so no step depends on another |

`Pacco-sample-scenario.rest` is not an incidental file: `Pacco/README.md:63-67` points at it by name
as the answer to "What HTTP requests can be sent to the API?", and identifies it as compatible with
the VS Code REST Client plugin — i.e. it is the platform's own designated, executable walkthrough.

### 1.6 — Per-repository navigation-source record

*Observed.* Every repository was walked in full. This table is the absence proof the discovery
method requires: for each repository, what was searched and what was actually there.

| Repository | Router config | Nav guards / middleware | E2E / Gherkin | Nav-bearing templates | Ordered scenario artifact |
|---|---|---|---|---|---|
| `hianshul100_Pacco` | None — top-level dirs are `scripts`, `compose`, `assets` only; `assets/` holds 4 PNGs | None | None | None — no markup file of any kind | None. `README.md:63-67` **points at** the scenario file in `Pacco.APIGateway` |
| `hianshul100_Pacco.APIGateway` | **No client router.** Four declarative HTTP route tables exist — `ntrada.yml`, `ntrada-async.yml`, `ntrada.docker.yml`, `ntrada-async.docker.yml` — see §1.7 | None. Route-level auth only: `auth: true` and 5 × `claims: role: admin` | None | None | **`Pacco-sample-scenario.rest`** (§1.5) plus the `Pacco.rest` catalogue |
| `hianshul100_Pacco.Services.Availability` | None | None | `tests/…Tests.EndToEnd/` — 2 files, 5 single-call facts (§1.4) | None | `Pacco.Services.Availability.rest` — catalogue |
| `hianshul100_Pacco.Services.Customers` | None | None | None — repo has `src` and `scripts` only | None | `Pacco.Services.Customers.rest` — catalogue |
| `hianshul100_Pacco.Services.Deliveries` | None | None | None | None | `Pacco.Services.Deliveries.rest` — catalogue |
| `hianshul100_Pacco.Services.Identity` | None | None | None | None | `Pacco.Services.Identity.rest` — catalogue (§1.5) |
| `hianshul100_Pacco.Services.Operations` | **None.** The one screen is addressed by filesystem path under `UseStaticFiles()`; `UseDefaultFiles` is not called anywhere (`ui-inventory.md` §7.3, §11.3) | None | None | **`wwwroot/ui/index.html`** — the only markup in the workspace. It contains **no `<a>`, no `<form>`, no `action`** (§1.3); its only outbound reference is a SignalR hub URL in `js/app.js:7` | `Pacco.Services.Operations.rest` — 1 request |
| `hianshul100_Pacco.Services.OrderMaker` | None | None | None | None | `Pacco.Services.OrderMaker.rest` — 1 request |
| `hianshul100_Pacco.Services.Orders` | None | None | `tests/…PactConsumerTests/` — 1 interaction (§1.4) | None | `Pacco.Services.Orders.rest` — catalogue |
| `hianshul100_Pacco.Services.Parcels` | None | None | `tests/…PactProviderTests/` — 1 interaction (§1.4) | None | `Pacco.Services.Parcels.rest` — catalogue |
| `hianshul100_Pacco.Services.Pricing` | None | None | None | None | `Pacco.Services.Pricing.rest` — 1 request |
| `hianshul100_Pacco.Services.Vehicles` | None | None | None | None | `Pacco.Services.Vehicles.rest` — catalogue |
| `hianshul100_Pacco.Web` | **None — the clone contains one file.** A full listing returns the clone root and `README.md`, whose entire content is `# Pacco.Web`. No `src/`, no `public/`, no `pages/`, no manifest, no markup | None | None | None | None |
| `hianshul100_Pacco.Context` | None — `docs/` markdown only, and none expected | None | None | None | None |

### 1.7 — What the gateway route tables are, and what they are not

*Observed.* `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` declares ten modules — `home`
(`:68`), `availability` (`:76`), `customers` (`:130`), `deliveries` (`:190`), `identity` (`:238`),
`operations` (`:278`), `orders` (`:293`), `parcels` (`:357`), `pricing` (`:401`), `vehicles`
(`:416`) — each with a `path:` prefix and a list of `upstream:` / `method:` / `downstream:` route
entries. `api-inventory.md` §2 enumerates all 41 resulting routes and §3 does the same for the
asynchronous configuration.

This is a **server-side HTTP routing table for an API edge**. It is genuinely a router configuration
and is treated as one in §2, but two properties bound what can be concluded from it, and both are
observed rather than assumed:

1. **It contains no navigation edges.** No route references another route. There is no redirect, no
   `Location`-driven forward between routes, and no route whose declaration names a successor. The
   one non-proxy route, `GET /` (`:69-71`), uses `use: return_value` with
   `returnValue: Welcome to Pacco API!` — a literal string, not a page and not a link.
2. **It defines no screens.** Every one of the 41 routes returns JSON or an empty body
   (`api-inventory.md` §2). No route serves markup. The platform's only screen,
   `/ui/index.html` on `operations-service`, appears in **none** of the four `ntrada*.yml` files
   (`ui-inventory.md` §11.3), so it is not reachable through the edge at all.

**Consequence, stated plainly:** route adjacency in the sense the discovery method describes —
component *A* rendering a link to route *B* — **cannot exist on this platform**, because no route
renders a component. What can and does exist is **request-sequence adjacency**: step *N*'s response
supplies the identifier that makes step *N+1*'s URL addressable. §2 builds that map, and labels it
as what it is.

### 1.8 — Conclusion of the navigation-source search

*Observed.* The workspace contains **no client-side router, no navigation guard, no automated
browser e2e suite, and no navigation-bearing template** — proven by four exhaustive sweeps whose
complete result sets are given in §1.1–§1.4, not by sampling. It contains **one screen with zero
outbound navigation** and **one human-authored, ordered, twenty-four-step cross-capability scenario**
that the platform's own README designates as its walkthrough. Journeys are reconstructed from that
scenario, from the entity-id dependencies between gateway routes, from the Operations console's
connect flow, and from the one saga that sequences steps for a machine actor. Nothing is
reconstructed for repositories that have none of these.

---

## 2 — Step sequences and adjacency

### 2.1 — Route inventory with owner — the addressable surface

The 41 gateway routes and the one screen are already enumerated with file-path evidence in
`api-inventory.md` §2 and §3 and in `ui-inventory.md` §7.3. They are **not** restated route by
route here. What this document adds is the owner-per-step attribution and the ordering, which is
what a journey needs. The module-to-owner binding, *observed* from `ntrada.yml`:

| Gateway module (`path:`) | Declared at | Downstream owner | Capability |
|---|---|---|---|
| `home` (`/`) | `ntrada.yml:68-71` | none — `use: return_value`, literal `Welcome to Pacco API!` | CAP-02 |
| `availability` | `:76-118` | `availability-service` | CAP-04 |
| `customers` | `:130-181` | `customers-service` | CAP-03 |
| `deliveries` | `:190-226` | `deliveries-service` | CAP-09 |
| `identity` | `:238-266` | `identity-service` | CAP-01 |
| `operations` | `:278-283` | `operations-service` | CAP-11 |
| `orders` | `:293-343` | `orders-service` | CAP-07 |
| `parcels` | `:357-…` | `parcels-service` | CAP-06 |
| `pricing` | `:401-…` | `pricing-service` | CAP-08 |
| `vehicles` | `:416-455` | `vehicles-service` | CAP-05 |

Two addressable surfaces sit **outside** this table, and both matter for §5:

| Surface | Address | Owner | Capability | Evidence |
|---|---|---|---|---|
| Operations message console | `/ui/index.html` on the `operations-service` host — filesystem path, no gateway route | `operations-service` | CAP-11 | `ui-inventory.md` §7.3, §11.3 |
| `ordermaker-service` saga entry | `POST /orders` on port 5015 — **no gateway route in any of the four `ntrada*.yml` files** | `ordermaker-service` | CAP-10 | `Pacco.Services.OrderMaker.rest:1-12`; `api-inventory.md` §3 closing note; `capability-baseline.md` CAP-10 |

### 2.2 — Adjacency map — how one step makes the next one addressable

Client-side link adjacency does not exist (§1.7). The adjacency that **is** observable is
identifier-passing between ordered requests, and it is read directly off
`Pacco-sample-scenario.rest`, where each capture is written as an explicit variable assignment.
*Observed*, every row cited to its line:

| # | Captured at | Variable | Value source | Consumed by (step → line) |
|---|---|---|---|---|
| 1 | `:14` `# @name sign_in`, `:25` | `@accessToken` | `{{sign_in.response.body.$.accessToken}}` — the sign-in **response body** | Every subsequent step, as `Authorization: Bearer {{accessToken}}` — 20 occurrences, `:27` through `:198` |
| 2 | `:44` `# @name add_parcel`, `:58` | `@parcelId` | `{{add_parcel.response.headers.Resource-ID}}` — a **response header** the gateway generates | `:63` volume query, `:82` attach-to-order |
| 3 | `:67` `# @name create_order`, `:77` | `@orderId` | `{{create_order.response.headers.Resource-ID}}` | `:82`, `:90`, `:135`, `:164`, `:197` |
| 4 | `:94` `# @name add_vehicle`, `:111` | `@vehicleId` | `{{add_vehicle.response.headers.Resource-ID}}` | `:135` assign-vehicle-to-order |
| 5 | `:116` | `@resourceId` | **The same** `{{add_vehicle.response.headers.Resource-ID}}` — the vehicle id is reused verbatim as the availability resource id | `:119`, `:144`, `:153` |
| 6 | `:133` | `@deliveryDate` | Literal `2020-01-10`, authored not captured | `:135` order delivery date, `:144` reservation date, `:166` delivery date |
| 7 | `:158` `# @name start_delivery`, `:172` | `@deliveryId` | `{{start_delivery.response.headers.Resource-ID}}` | `:173`, `:178`, `:184`, `:189`, `:193` |

Row 5 is the load-bearing one and is stated as observed, not interpreted: **the scenario reserves
availability against the vehicle's own id.** There is no separate resource-creation identity. The
comment at `:115` says so in words — `### Add a vehicle as the available resource being able to
deliver the pacakge` (spelling verbatim).

A second adjacency mechanism exists and is *observed* but is **not** client-carried: the gateway
injects the caller's identity into downstream paths and payloads from the JWT `@user_id` claim —
`ntrada.yml:143` (`customers-service/customers/@user_id`), `:298`
(`orders-service/orders?customerId=@user_id`), `:362`
(`parcels-service/parcels?customerId=@user_id`), and the `bind: customerId:@user_id` blocks on the
write routes (`api-inventory.md` §3). So `GET /customers/me`, `GET /orders` and `GET /parcels` are
addressable **without** the client holding a customer id at all. That is recorded in §4 as
edge-injected session context, and it is why no `customerId` variable is ever captured in the
scenario file.

### 2.3 — Confirmed sequence — `Pacco-sample-scenario.rest`

*Observed.* The complete ordered step list, with owning capability per step. Line numbers are the
request line in `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest`. Every request goes to
`{{api}} = http://localhost:5000` (`:1`), i.e. through the gateway, so **CAP-02 mediates every
step** and is not repeated per row.

| Step | Line | Request | Owning capability | Authored intent (verbatim comment) |
|---:|---:|---|---|---|
| 1 | `:4` | `POST /identity/sign-up` | CAP-01 | `At first, create an account` |
| 2 | `:15` | `POST /identity/sign-in` | CAP-01 | `Authenticate and grab the access token` |
| 3 | `:26` | `GET /identity/me` | CAP-01 | `Get your user account details` |
| 4 | `:30` | `POST /customers` | CAP-03 | `Complete the customer registration process` |
| 5 | `:40` | `GET /customers/me` | CAP-03 | `Get your customer account details` |
| 6 | `:45` | `POST /parcels` | CAP-06 | `Add a parcel and grab its id` |
| 7 | `:59` | `GET /parcels` | CAP-06 | `Get your parcels` |
| 8 | `:63` | `GET /parcels/volume?parcelIds=["…"]` | CAP-06 | `Calculate the parcel volume to see whether it works as expected` |
| 9 | `:68` | `POST /orders` | CAP-07 | `Create a new order and grab its id` |
| 10 | `:78` | `GET /orders` | CAP-07 | `Get your orders` |
| 11 | `:82` | `POST /orders/{orderId}/parcels/{parcelId}` | CAP-07 | `Add a parcel to the order` |
| 12 | `:90` | `GET /orders/{orderId}` | CAP-07 | `Get your order details which should now contain a package` |
| 13 | `:95` | `POST /vehicles` | CAP-05 | `Add a new vehicle` |
| 14 | `:112` | `GET /vehicles?payloadCapacity=0&loadingCapacity=0&variants=1` | CAP-05 | `Get a newly added vehicle` |
| 15 | `:119` | `POST /availability/resources` | CAP-04 | `Add a vehicle as the available resource being able to deliver the pacakge` |
| 16 | `:129` | `GET /availability/resources?tags=…&matchAllTags=false` | CAP-04 | `Browse the resources filtered by tags if needed` |
| 17 | `:135` | `POST /orders/{orderId}/vehicles/{vehicleId}` | CAP-07 | `Assign a vehicle to your order and set the desired delivery date` |
| 18 | `:144` | `POST /availability/resources/{resourceId}/reservations/{deliveryDate}` | CAP-04 | `Make a reservation for the given date to deliver the package` |
| 19 | `:153` | `GET /availability/resources/{resourceId}` | CAP-04 | `Ensure that resource was reserved for the chosen date` |
| 20 | `:159` | `POST /deliveries` | CAP-09 | `Start the delivery and grab its id` |
| 21 | `:173` | `POST /deliveries/{deliveryId}/registrations` | CAP-09 | `Add some internal delivery transportation details` |
| 22 | `:184` | `POST /deliveries/{deliveryId}/complete` | CAP-09 | `Complete the delivery` |
| 23 | `:193` | `GET /deliveries/{deliveryId}` | CAP-09 | `Check the delivery details` |
| 24 | `:197` | `GET /orders/{orderId}` | CAP-07 | `Get your order details which should now be marked as completed` |

**Entry point:** step 1, `POST /identity/sign-up` (`:4`). It is the only step in the file that
requires no prior step and no token — `ntrada.yml:254-262` declares it with `auth: false`
(`api-inventory.md` §2.4). It is also the only route besides `sign-in` and `GET /` that is reachable
cold.

**Terminal point:** step 24, `GET /orders/{orderId}` (`:197`). Nothing follows it in the file and it
captures nothing. Its authored purpose is to observe the terminal state the whole sequence was
constructed to reach — the order marked completed.

**Steps that are observation, not progression.** Steps 3, 5, 7, 8, 10, 12, 14, 16, 19 and 23 are all
`GET`s whose results feed no later step. They are *observed* as authored checkpoints ("to see
whether it works as expected", `:62`; "which should now contain a package", `:89`). They are kept in
the sequence because the author fixed their position, but they are not adjacency edges.

### 2.4 — Other sequences observable in the workspace

| Sequence | Steps | Actor | Ordering evidence | Label |
|---|---|---|---|---|
| **Operations console connect** | (a) obtain a JWT out of band; (b) open `/ui/index.html`; (c) paste the token into `<input id="jwt">`; (d) click `#connect`; (e) `connection.start()`; (f) `connection.invoke('initializeAsync', $jwt.value)`; (g) receive `connected` or `disconnected` | Human | Fixed by code, not by a test: `app.js:11` binds `$connect.onclick`, `:12-16` guards the token, `:19` starts, `:21` invokes, `:26-32` handles the two outcomes | **Partial** — steps (c)–(g) are fixed in code; step (a) has no observable source in the workspace |
| **Async write → outcome** | (a) any of the 20 async write routes; (b) gateway publishes to RabbitMQ and returns a correlation id; (c) outcome learned by `GET /operations/{operationId}` **or** by SignalR `operation_pending` / `operation_completed` / `operation_rejected` | Human or client | `ntrada-async.yml` 20 × `use: rabbitmq` (`api-inventory.md` §3); `Operations.Api/Services/OperationsService.cs` keying on correlation id; `app.js:34-44` | **Inferred** — the pairing is designed and both halves exist, but no artifact in the workspace performs (a) then (c) in order. `Pacco-sample-scenario.rest` never calls `/operations/{id}` |
| **AI order-making saga** | `MakeOrder` → `CreateOrder` → *n* × `AddParcelToOrder` → wait for all `ParcelAddedToOrder` → `GetBestAsync()` vehicle → `GetBestAsync()` reservation date → `AssignVehicleToOrder` → `ReserveResource` → `OrderApproved` → `MakeOrderCompleted` → `CompleteAsync()` | **Machine** | `AIOrderMakingSaga.cs:56-132`, read in full; entry `POST /orders` on port 5015, `Pacco.Services.OrderMaker.rest:6-12` | **Confirmed as a workflow, but not a user journey** — see §5, J4 |
| **Availability resource lifecycle** | create → read → reserve → release → delete | Human | `Pacco.Services.Availability.rest:8-37` lists all five, and `Tests.EndToEnd` covers create and read individually (§1.4) | **Inferred** — the file is a catalogue (§1.5); the dependency create-before-reserve is real, but no artifact fixes the order |

### 2.5 — Entry and terminal points across the platform

*Observed.* Entry points are addresses reachable without a preceding step.

| Address | Why it is an entry point | Evidence |
|---|---|---|
| `POST /identity/sign-up` | `auth: false` — no token needed | `ntrada.yml:254-262`; `api-inventory.md` §2.4 |
| `POST /identity/sign-in` | `auth: false` | `ntrada.yml:263-266` |
| `GET /` | `use: return_value`, no auth block | `ntrada.yml:69-71` |
| `GET /operations/{operationId}` | **`auth: false`** — the one authenticated-looking read that is not authenticated. Reachable cold **if** the correlation id is known | `ntrada.yml:280-283`; `api-inventory.md` §2.5 |
| `/ui/index.html` on `operations-service` | A static file with no auth middleware in front of it; the token is only needed to *invoke the hub*, not to *load the page* | `ui-inventory.md` §7.3, §9 |
| `POST /orders` on `ordermaker-service:5015` | No gateway route, no auth block observed on the host; it is the saga's only trigger and has no upstream step | `Pacco.Services.OrderMaker.rest:6`; `AIOrderMakingSaga.cs:56`; `api-inventory.md` §4.9 |

Every other gateway route carries `auth: true` and is therefore **not** an entry point — including
`GET /availability/resources` (`ntrada.yml:78-82`), which `Pacco.rest:84` sends without an
`Authorization` header. That omission is a defect in the catalogue file, not an open route; it is
recorded as a source conflict in §10.

Terminal points — addresses from which no further step is evidenced:

| Address | Evidence |
|---|---|
| `GET /orders/{orderId}` as scenario step 24 | `Pacco-sample-scenario.rest:197` is the last line of the file |
| `/ui/index.html` after `connected` | `app.js` has no navigation, no link, no second view. The page's terminal state is an appending `<ul>` (`app.js:46-52`) |
| `GET /` | Returns a literal string with no link in it | `ntrada.yml:71` |
| Every `DELETE` route | No route or artifact is evidenced as following a delete | `api-inventory.md` §2 |

---

## 3 — Journey-to-capability mapping and hand-off points

Every step in §2.3 maps to a capability in `capability-baseline.md` §1 through its gateway module's
`downstream:` service (§2.1). **No step in any journey below is unmappable**, so the label
`capability unknown` is not used anywhere in this document — not because ownership was invented, but
because the gateway declares the owning service on every route it carries. Two addresses do sit
outside the gateway (`/ui/index.html`, `ordermaker-service:5015/orders`) and both are attributed
from their owning repository, which `capability-baseline.md` §2 binds to CAP-11 and CAP-10
respectively.

**CAP-02 is a mediator, not a step owner.** Every gateway-routed step passes through
`api-gateway`, which terminates the request, validates the JWT, and injects `@user_id`. Recording a
CAP-02 hand-off between every consecutive pair would say nothing. CAP-02 is therefore recorded once
per journey as the mediating capability, and hand-offs are counted between the *downstream owners*.

### 3.1 — Hand-off points in J1 (the confirmed scenario)

*Observed.* Nine points where the owning capability changes between consecutive steps.

| ID | Step transition | From capability (repo) | To capability (repo) | Route transition | Evidence |
|---|---|---|---|---|---|
| **H1** | 3 → 4 | CAP-01 Identity & Access Management (`Pacco.Services.Identity`) | CAP-03 Customer Profile & Lifecycle Management (`Pacco.Services.Customers`) | `GET /identity/me` → `POST /customers` | `Pacco-sample-scenario.rest:26 → :30` |
| **H2** | 5 → 6 | CAP-03 (`Pacco.Services.Customers`) | CAP-06 Parcel Catalogue & Volume Calculation (`Pacco.Services.Parcels`) | `GET /customers/me` → `POST /parcels` | `…rest:40 → :45` |
| **H3** | 8 → 9 | CAP-06 (`Pacco.Services.Parcels`) | CAP-07 Order Lifecycle Management (`Pacco.Services.Orders`) | `GET /parcels/volume` → `POST /orders` | `…rest:63 → :68` |
| **H4** | 12 → 13 | CAP-07 (`Pacco.Services.Orders`) | CAP-05 Vehicle Fleet Catalogue (`Pacco.Services.Vehicles`) | `GET /orders/{orderId}` → `POST /vehicles` | `…rest:90 → :95` |
| **H5** | 14 → 15 | CAP-05 (`Pacco.Services.Vehicles`) | CAP-04 Resource Availability & Reservation (`Pacco.Services.Availability`) | `GET /vehicles` → `POST /availability/resources` | `…rest:112 → :119`; the `@resourceId` assignment at `:116` is the edge itself |
| **H6** | 16 → 17 | CAP-04 (`Pacco.Services.Availability`) | CAP-07 (`Pacco.Services.Orders`) | `GET /availability/resources` → `POST /orders/{orderId}/vehicles/{vehicleId}` | `…rest:129 → :135` |
| **H7** | 17 → 18 | CAP-07 (`Pacco.Services.Orders`) | CAP-04 (`Pacco.Services.Availability`) | `POST /orders/{orderId}/vehicles/{vehicleId}` → `POST /availability/resources/{resourceId}/reservations/{deliveryDate}` | `…rest:135 → :144` |
| **H8** | 19 → 20 | CAP-04 (`Pacco.Services.Availability`) | CAP-09 Delivery Execution & Tracking (`Pacco.Services.Deliveries`) | `GET /availability/resources/{resourceId}` → `POST /deliveries` | `…rest:153 → :159` |
| **H9** | 23 → 24 | CAP-09 (`Pacco.Services.Deliveries`) | CAP-07 (`Pacco.Services.Orders`) | `GET /deliveries/{deliveryId}` → `GET /orders/{orderId}` | `…rest:193 → :197` |

**H6 and H7 are a single round trip, and the ordering is authored, not enforced.** *Observed*: the
scenario assigns the vehicle to the order (step 17) and only then reserves the resource (step 18).
Nothing in either handler requires that order —`ReserveResourceHandler.cs:35-55` checks the resource
exists and the customer state is valid, and never looks at the order. Recorded as observed sequence,
not as a constraint.

### 3.2 — Hand-offs the journey triggers but does not navigate

*Observed.* Three steps cause a capability change **inside** the platform, invisible to the caller.
They are hand-offs in the data sense and are listed because §4 has to account for the state they
move, but they are **not navigation** and are excluded from the §6 navigation edge table.

| Trigger step | From → To | Mechanism | Evidence |
|---|---|---|---|
| Step 4 `POST /customers` is only valid because step 1 created a customer record | CAP-01 → CAP-03, **asynchronously** | `identity-service` publishes `signed_up`; `customers-service` handles it and creates the customer | `Customers.Application/Events/External/Handlers/SignedUpHandler.cs`; `capability-baseline.md` CAP-03 |
| Step 17 `POST /orders/{orderId}/vehicles/{vehicleId}` | CAP-07 → CAP-05, then CAP-07 → CAP-08 | Two synchronous HTTP calls inside one handler: `_vehiclesServiceClient.GetAsync(command.VehicleId)` then `_pricingServiceClient.GetOrderPriceAsync(order.CustomerId, vehicle.PricePerService)` | `Orders.Application/Commands/Handlers/AssignVehicleToOrderHandler.cs:53-60`; `api-inventory.md` §6 rows 4 and 5 |
| Step 18 `POST /availability/resources/{resourceId}/reservations/{deliveryDate}` | CAP-04 → CAP-03 | `_customersServiceClient.GetStateAsync(command.CustomerId)`, rejecting the reservation unless `customerState.IsValid` | `Availability.Application/Commands/Handlers/ReserveResourceHandler.cs:41-50`; `api-inventory.md` §6 row 1 — the one call on the platform that carries a Vault PKI client certificate |

Step 11 `POST /orders/{orderId}/parcels/{parcelId}` likewise reaches CAP-06 from CAP-07
(`api-inventory.md` §6 row 3, `orders-service` → `parcels-service` `GET /parcels/{id}`), which is the
same interaction the Pact pair in §1.4 pins.

### 3.3 — Hand-offs in the other journeys

| ID | Journey | From → To | Transition | Evidence |
|---|---|---|---|---|
| **H10** | J2 Operations console | CAP-01 (`Pacco.Services.Identity`) → CAP-11 Operation Status Projection & Real-Time Notification (`Pacco.Services.Operations`) | Token issued by `POST /identity/sign-in` → token supplied to `connection.invoke('initializeAsync', …)` | `app.js:21`; `Operations.Api/Hubs/PaccoHub.cs` `InitializeAsync(string token)`; `api-inventory.md` §5.2. **The transition itself has no code path** — see §4.2 |
| **H11** | J3 Async write → outcome | CAP-02 Edge Routing & Access Enforcement (`Pacco.APIGateway`) → CAP-11 (`Pacco.Services.Operations`) | Any of the 20 `use: rabbitmq` routes → `GET /operations/{operationId}` or a SignalR push | `ntrada-async.yml` (`api-inventory.md` §3); `ntrada.yml:280-283`; `app.js:34-44` |
| **H12** | J4 Saga | CAP-10 Automated Order Orchestration (`Pacco.Services.OrderMaker`) → CAP-07 (`Pacco.Services.Orders`) | `MakeOrder` → publish `CreateOrder` | `AIOrderMakingSaga.cs:63-68` |
| **H13** | J4 Saga | CAP-07 → CAP-05 (`Pacco.Services.Vehicles`) | `ParcelAddedToOrder` (all parcels attached) → `_vehiclesServiceClient.GetBestAsync()` | `AIOrderMakingSaga.cs:87-95`; `api-inventory.md` §6 row 6 |
| **H14** | J4 Saga | CAP-05 → CAP-04 (`Pacco.Services.Availability`) | vehicle selected → `_resourceReservationsService.GetBestAsync(Data.VehicleId)` | `AIOrderMakingSaga.cs:97-101`; `api-inventory.md` §6 row 7 |
| **H15** | J4 Saga | CAP-04 → CAP-07 | `VehicleAssignedToOrder` → publish `ReserveResource`, then `OrderApproved` closes the saga | `AIOrderMakingSaga.cs:112-132` |

---

## 4 — Shared state at hand-off points

For each hand-off, what the receiving capability needs from the sending one, and by what mechanism.
Nothing is listed that is not visible in a cited file.

### 4.1 — J1 hand-offs

| ID | State carried | Mechanism | API calls triggered on entry (cross-ref `api-inventory.md`) | Evidence |
|---|---|---|---|---|
| **H1** CAP-01 → CAP-03 | (a) the bearer JWT; (b) the customer identity — **not** carried by the client | (a) `Authorization: Bearer {{accessToken}}` header; (b) **edge injection**: the gateway binds `customerId` from the `@user_id` claim into the `complete_customer_registration` payload, so `POST /customers` sends only `{ fullName, address }` | `POST /customers` → `customers-service` `CompleteCustomerRegistration` (§4.2 of api-inventory) | `Pacco-sample-scenario.rest:31-37`; `ntrada.yml:162-171`; `api-inventory.md` §2.2, §3 |
| **H2** CAP-03 → CAP-06 | Bearer JWT only; `customerId` again edge-injected into `add_parcel` | Header + `@user_id` binding | `POST /parcels` → `parcels-service` `AddParcel`; returns `Resource-ID` | `…rest:45-54`; `api-inventory.md` §2.7 |
| **H3** CAP-06 → CAP-07 | **`@parcelId`** — a CAP-06-owned entity id captured from a response header and then quoted into CAP-07's URL two steps later | Response header `Resource-ID` → REST-Client variable → URL path segment at `:82` | `POST /orders` → `orders-service` `CreateOrder`; returns `Resource-ID` = `orderId` | `…rest:58, :68, :82`; `api-inventory.md` §2.6, §2.7 |
| **H4** CAP-07 → CAP-05 | **None from CAP-07.** The vehicle is created independently; no order id, parcel id or order state crosses into `POST /vehicles`. Only the bearer token is shared | Header only | `POST /vehicles` → `vehicles-service` `AddVehicle` | `…rest:95-107` |
| **H5** CAP-05 → CAP-04 | **`@vehicleId`, aliased as `@resourceId`** — the vehicle's own id becomes the availability resource's id, plus the authored `@tags = ["vehicle", "armor"]` | Response header `Resource-ID` reused verbatim in a second variable (`:116`), then sent in the request **body** | `POST /availability/resources` → `availability-service` `AddResource` | `…rest:111, :116-126`; `api-inventory.md` §2.1 |
| **H6** CAP-04 → CAP-07 | **`@vehicleId`** and **`@orderId`** together, plus `@deliveryDate` | Both ids as URL path segments; `deliveryDate` in the body; `customerId` edge-injected | `POST /orders/{orderId}/vehicles/{vehicleId}` → `orders-service` `AssignVehicleToOrder`, which on entry calls `vehicles-service GET /vehicles/{id}` and `pricing-service GET /pricing?customerId=&orderPrice=` (`api-inventory.md` §6 rows 4, 5) | `…rest:135-141`; `AssignVehicleToOrderHandler.cs:53-60` |
| **H7** CAP-07 → CAP-04 | **`@resourceId`** and **`@deliveryDate`** — the same date the order was just given, so the two capabilities agree on it only because the caller passes the same variable to both | URL path segments (`:144`); `customerId` edge-injected from `@user_id`; `priority: 0` in the body | `POST /…/reservations/{dateTime}` → `availability-service` `ReserveResource`, which on entry calls `customers-service GET /customers/{id}/state` over a Vault PKI client certificate (`api-inventory.md` §6 row 1) | `…rest:133, :144-150`; `ReserveResourceHandler.cs:41-50` |
| **H8** CAP-04 → CAP-09 | **`@orderId`** and **`@deliveryDate`** — note the resource id does **not** cross. `deliveries-service` is never told which vehicle or resource is delivering | Request body `{ orderId, description, dateTime }` | `POST /deliveries` → `deliveries-service` `StartDelivery`; gateway generates `deliveryId` and returns it in `Resource-ID` | `…rest:159-167`; `api-inventory.md` §2.3 |
| **H9** CAP-09 → CAP-07 | **None carried by the caller.** The client re-reads the order with the `@orderId` it has held since step 9. The state that actually crossed CAP-09 → CAP-07 moved **asynchronously over the broker** as `delivery_completed`, before the read | Read-back of a previously held id; the real transfer is an event the client never sees | `GET /orders/{orderId}`; the state change is applied by `Orders.Application/Events/External/Handlers/DeliveryCompletedHandler.cs` | `…rest:197`; `capability-baseline.md` CAP-07, CAP-09 |

**Session context, once, for the whole journey.** *Observed*: after step 2 every subsequent request
carries `Authorization: Bearer {{accessToken}}` — 20 occurrences between `:27` and `:198`. There is
no cookie, no session id, and no server-side session on any route (`api-inventory.md` §7.2;
`ui-inventory.md` §9). The JWT is the only session context that crosses any hand-off, and it is
carried identically at all nine.

**No `localStorage`, `sessionStorage`, or client-side persistence crosses any hand-off anywhere in
the workspace.** *Observed*: a workspace-wide search returns zero occurrences of either API
(`ui-inventory.md` §11.5). The scenario file's variables are REST Client tooling state, held by the
editor, not by the platform — recorded as **A2**.

### 4.2 — J2 Operations console hand-off (H10) — the one hand-off with no mechanism

*Observed.* The receiving capability needs exactly one thing: a valid JWT whose `payload.Subject`
parses as a `Guid`, which `PaccoHub.InitializeAsync` uses to add the connection to
`subject.ToUserGroup()` (`Operations.Api/Hubs/PaccoHub.cs`).

What is **not** observable is any code path that delivers it. Enumerated, each checked:

| Candidate mechanism | Present? | Evidence |
|---|---|---|
| URL parameter or query string on `/ui/index.html` | **No** — the page reads no `location`, no query, no fragment | `app.js` in full; `window.location` returns zero matches (§1.3) |
| Session cookie | **No** — nothing reads or writes `document.cookie` | `app.js` in full |
| `localStorage` / `sessionStorage` | **No** | `ui-inventory.md` §11.5 |
| A link or redirect from a sign-in screen | **No** — no sign-in screen exists, and `POST /identity/sign-in` is called by no UI asset | `ui-inventory.md` §7.3, §8.2 |
| Server-injected value in the document | **No** — the file is static; there is no inline `<script>` body and no `data-*` attribute | `index.html:1-26`, read in full |
| **Manual entry into `<input type="text" id="jwt">`** | **Yes — the only one** | `index.html:14`; `app.js:12` reads `$jwt.value` |

**Finding: the CAP-01 → CAP-11 hand-off is performed by the human operator, outside the software.**
The token crosses the boundary by being pasted. This is recorded as an observed property of the
surface, not as a defect, and it is why J2 is labelled **partial** rather than confirmed: the
receiving half is fixed in code, the sending half has no source in this workspace. Carried as **Q1**.

The `Authorization` header is **not** used on this hand-off. The token travels as the string
argument of a hub method — `connection.invoke('initializeAsync', $jwt.value)` (`app.js:21`) — i.e.
in the message body, which is a different mechanism from every hand-off in §4.1.

### 4.3 — J3 and J4 hand-offs

| ID | State carried | Mechanism | Evidence |
|---|---|---|---|
| **H11** CAP-02 → CAP-11 | The **correlation id**. The gateway generates it (`generateRequestId`, `generateTraceId`) and exposes it as the `Request-ID` response header; `operations-service` keys its operation record on the same value under Redis `requests:{id}` with a 300-second sliding expiry | Response header on the write, then either a path segment on `GET /operations/{operationId}` or the `operation` payload of a SignalR push | `ntrada.yml` `generateRequestId` / `exposedHeaders`; `Operations.Api/Services/OperationsService.cs`; `Operations.Api/appsettings.json` `requests.expirySeconds: 300`; `api-inventory.md` §2.5, §5.2 |
| **H12** CAP-10 → CAP-07 | `OrderId`, `CustomerId` | Broker message `CreateOrder` on the `orders` exchange, with `messageContext: _accessor.CorrelationContext` and a `Saga: Pending` header | `AIOrderMakingSaga.cs:63-68` |
| **H13** CAP-07 → CAP-05 | **Nothing from the order.** `GetBestAsync()` calls `GET /vehicles` with no filter and takes `Items.FirstOrDefault()` | Synchronous HTTP, no parameters | `AIOrderMakingSaga.cs:93`; `api-inventory.md` §6 row 6 |
| **H14** CAP-05 → CAP-04 | `Data.VehicleId` — again the vehicle id **is** the resource id, matching H5 | Synchronous HTTP path segment `GET /resources/{resourceId}` | `AIOrderMakingSaga.cs:98`; `api-inventory.md` §6 row 7 |
| **H15** CAP-04 → CAP-07 | `VehicleId`, `CustomerId`, `ReservationDate`, `ReservationPriority` — all held in the saga's own `AIMakingOrderData` between steps | Saga state keyed on `OrderId` (`ResolveId`, `:45-54`), then a `ReserveResource` broker message | `AIOrderMakingSaga.cs:98-119` |

**J4 carries its shared state in a persistent saga aggregate, not in a URL.** That is the one place
on the platform where cross-capability journey state is held by a component rather than by the
caller. It is keyed entirely on `OrderId` and every one of the five message types resolves to that
key (`AIOrderMakingSaga.cs:45-54`).

### 4.4 — Hand-offs with no shared state

*Observed.* Two of the fifteen hand-offs carry nothing but the credential:

- **H4 (CAP-07 → CAP-05)** — creating a vehicle after reading an order shares no entity, no id and
  no form state. **No shared state observed at this hand-off — context boundary appears clean.**
- **H13 (CAP-10 → CAP-05)** — the saga asks for the whole fleet and takes the first row; no order,
  parcel, customer or capacity constraint crosses. **No shared state observed at this hand-off —
  context boundary appears clean.** Note that this is clean in the coupling sense and blunt in the
  behavioural sense: `api-inventory.md` §6 records that the "best" vehicle is simply the first one
  returned.

---

## 5 — Journey catalogue

Five journeys are evidenced in this workspace. One is confirmed, one is partial, three are inferred.
Nothing here is a re-labelled route list — each entry names the artifact that fixes the order of its
steps, or says plainly that no artifact does.

### J1 — Place and complete an order end to end

- **Actor:** a self-registering customer (`"role": "user"`, `Pacco-sample-scenario.rest:10`).
- **Entry point:** `POST /identity/sign-up` (`Pacco-sample-scenario.rest:4`) — reachable with no
  token (`ntrada.yml:254-262`).
- **Exit point:** `GET /orders/{orderId}` (`…rest:197`) — the order read back as completed.
- **Step sequence with owning capability:** 24 steps, listed in full with line numbers and the
  author's own comments in **§2.3**. In capability terms the sequence is:
  CAP-01 (1–3) → CAP-03 (4–5) → CAP-06 (6–8) → CAP-07 (9–12) → CAP-05 (13–14) → CAP-04 (15–16) →
  CAP-07 (17) → CAP-04 (18–19) → CAP-09 (20–23) → CAP-07 (24).
- **Capabilities crossed:** CAP-01, CAP-03, CAP-04, CAP-05, CAP-06, CAP-07, CAP-09 — seven owners,
  mediated throughout by CAP-02, and touching CAP-08 indirectly at step 17 (§3.2).
- **Hand-off points:** H1–H9 (**§3.1**), plus three internal hand-offs the caller never sees (§3.2).
- **Shared state per hand-off:** **§4.1**. In one line: a bearer JWT on every step from 2 onward,
  plus four entity ids (`@parcelId`, `@orderId`, `@vehicleId`/`@resourceId`, `@deliveryId`), each
  captured from a `Resource-ID` response header and replayed into a later URL, and one authored
  literal (`@deliveryDate = 2020-01-10`, `:133`).
- **Evidence type:** an ordered, dependency-linked request script with variable capture —
  `Pacco-sample-scenario.rest`, designated by `hianshul100_Pacco/README.md:63-67` as *the* answer to
  "What HTTP requests can be sent to the API?".
- **Confidence:** **Confirmed** for the sequence; see §9 for the residual uncertainty (the file is
  authored intent that no CI job executes).
- **Evidence paths:**
  `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest` (whole file);
  `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`;
  `hianshul100_Pacco/README.md:63-67`;
  `hianshul100_Pacco.Services.Orders/…/Handlers/AssignVehicleToOrderHandler.cs`;
  `hianshul100_Pacco.Services.Availability/…/Handlers/ReserveResourceHandler.cs`.

### J2 — Watch operation outcomes on the operations console

- **Actor:** a human holding a JWT, described by the surface as an operator; the page states no role
  and the hub applies none beyond parsing the token's subject.
- **Entry point:** `GET /ui/index.html` on `operations-service:5005` — a static page, loadable with
  no token (`ui-inventory.md` §7.3, §9).
- **Exit point:** none. The page's terminal state is an appending `<ul id="messages">`
  (`app.js:46-52`); there is no navigation away and no second view.
- **Step sequence with owning capability:**
  1. load `/ui/index.html` — CAP-11 (`index.html:1-26`);
  2. **obtain a JWT** — CAP-01, by a means with no code path in this workspace (§4.2);
  3. paste it into `<input id="jwt">` and press `#connect` — CAP-11 (`index.html:14, :16`);
  4. `connection.start()` to `http://localhost:5005/pacco`, then
     `connection.invoke('initializeAsync', $jwt.value)` — CAP-11 (`app.js:6-9, :19-22`);
  5. receive `connected`, then any of `operation_pending` / `operation_completed` /
     `operation_rejected` — CAP-11 (`app.js:26-44`).
- **Capabilities crossed:** CAP-01 → CAP-11. CAP-02 is **not** involved: the hub is addressed
  directly on port 5005, bypassing the gateway (`app.js:8`).
- **Hand-off points:** H10 only (§3.3).
- **Shared state per hand-off:** the JWT, carried as a **hub method argument** rather than an
  `Authorization` header, and placed there by a human paste. Full enumeration of the mechanisms
  checked and excluded is in **§4.2**.
- **Evidence type:** two source files read in full — the receiving half is fixed in code; the
  sending half is a text input.
- **Confidence:** **Partial** — steps 1 and 3–5 are Confirmed from source; step 2 is an observed
  requirement with no observed provider.
- **Evidence paths:**
  `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/index.html`;
  `…/wwwroot/ui/js/app.js`;
  `…/Pacco.Services.Operations.Api/Hubs/PaccoHub.cs`;
  `ui-inventory.md` §7.3, §8.2, §9, §11.5.

### J3 — Submit an asynchronous write and learn its outcome

- **Actor:** any authenticated caller — human or client program.
- **Entry point:** any of the 20 routes declared `use: rabbitmq` in `ntrada-async.yml`
  (`api-inventory.md` §3).
- **Exit point:** either `GET /operations/{operationId}` (`ntrada.yml:280-283`) or a SignalR
  `operation_completed` / `operation_rejected` push (`app.js:38-44`).
- **Step sequence with owning capability:**
  1. send the write — CAP-02 accepts it, publishes to RabbitMQ and returns `202` with a
     correlation id in the `Request-ID` response header;
  2. the owning service handles the message — CAP-03 to CAP-09 depending on the route;
  3. `operations-service` records the outcome against that correlation id — CAP-11;
  4. the caller reads `GET /operations/{operationId}` **or** receives the push — CAP-11.
- **Capabilities crossed:** CAP-02 → (one write owner) → CAP-11.
- **Hand-off points:** H11 (§3.3).
- **Shared state per hand-off:** the correlation id, and only the correlation id (§4.3). It is
  short-lived: the operation record sits in Redis under `requests:{id}` with a 300-second sliding
  expiry (`Operations.Api/appsettings.json`), so a caller that waits longer than five minutes and
  then polls learns nothing.
- **Evidence type:** both halves exist in configuration and source; **no artifact anywhere in the
  workspace performs step 1 and step 4 in order.** `Pacco-sample-scenario.rest` never calls
  `/operations/{id}`; neither does `Pacco.rest` in a chained way (its `{{operationId}}` is a zeroed
  GUID, §1.5).
- **Confidence:** **Inferred** — "inferred from configuration adjacency", not from an executed or
  authored sequence.
- **Evidence paths:**
  `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada-async.yml`;
  `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:278-292`;
  `hianshul100_Pacco.Services.Operations/…/Services/OperationsService.cs`;
  `…/Pacco.Services.Operations.Api/appsettings.json`;
  `api-inventory.md` §2.5, §3, §5.2.

### J4 — Automated order making (machine actor — listed for completeness, not a user journey)

- **Actor:** **a program, not a person.** The saga is triggered by a `MakeOrder` message and runs to
  completion with no human step. It is catalogued here because it crosses four capabilities and
  because omitting it would misrepresent the platform's cross-capability traffic — but it is
  explicitly **not** counted as a user journey in §6 or §9.
- **Entry point:** `POST /orders` on `ordermaker-service:5015`
  (`Pacco.Services.OrderMaker.rest:6-12`) — **no gateway route exists for this service**
  (`api-inventory.md` §4.9).
- **Exit point:** `MakeOrderCompleted` published, then `CompleteAsync()`
  (`AIOrderMakingSaga.cs:121-132`).
- **Step sequence with owning capability:** `MakeOrder` → publish `CreateOrder` (CAP-10 → CAP-07,
  `:56-69`) → on `OrderCreated`, publish `AddParcelToOrder` per parcel (CAP-07, `:71-82`) → on the
  last `ParcelAddedToOrder`, `GetBestAsync()` a vehicle (CAP-05, `:93`) → `GetBestAsync()` a
  reservation date (CAP-04, `:98`) → publish `AssignVehicleToOrder` (CAP-07, `:103-109`) → on
  `VehicleAssignedToOrder`, publish `ReserveResource` (CAP-04, `:112-119`) → on `OrderApproved`,
  publish `MakeOrderCompleted` and complete (CAP-07 → CAP-10, `:121-132`).
- **Capabilities crossed:** CAP-10, CAP-07, CAP-05, CAP-04.
- **Hand-off points:** H12–H15 (§3.3).
- **Shared state per hand-off:** held in the saga's own `AIMakingOrderData`, keyed on `OrderId` by
  `ResolveId` for all five message types (`:45-54`) — the only place on the platform where journey
  state lives in a component rather than in the caller (§4.3).
- **Evidence type:** a single source file read in full, whose control flow *is* the sequence.
- **Confidence:** **Confirmed as a workflow.** Its *reachability* is **Inferred** — no artifact in
  the workspace sends `MakeOrder`, and `docker-compose` exposure for port 5015 was not established
  here (carried as **Q2**).
- **Evidence paths:**
  `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/Sagas/AIOrderMakingSaga.cs`
  (lines 45–153);
  `hianshul100_Pacco.Services.OrderMaker/Pacco.Services.OrderMaker.rest`;
  `api-inventory.md` §4.9, §6 rows 6–7.

### J5 — Manage the lifecycle of an availability resource

- **Actor:** any authenticated caller; all six availability routes carry `auth: true` and none
  carries `role: admin` (`ntrada.yml:76-122`).
- **Entry point:** `POST /availability/resources` (`ntrada.yml:90-94`).
- **Exit point:** `DELETE /availability/resources/{resourceId}` (`ntrada.yml:115-121`).
- **Step sequence with owning capability:** create → read → reserve → cancel the reservation →
  delete the resource. Every step is CAP-04; the reservation step reaches into CAP-03 internally
  (§3.2).
- **Capabilities crossed:** **one** — CAP-04. This is a single-capability journey and contributes no
  row to the navigation summary in §6.
- **Hand-off points:** none between capabilities. **No shared state observed at any hand-off in this
  journey — there are no capability hand-offs to observe.**
- **Shared state per step:** `resourceId`, returned by the create as `Resource-ID` and used as a path
  segment by every later step.
- **Evidence type:** route adjacency plus a data dependency (a resource must exist before it can be
  reserved or deleted, `ReserveResourceHandler.cs:36-40`). `Pacco.Services.Availability.rest` lists
  all the steps but is a catalogue with zeroed GUIDs, so it does **not** fix the order (§1.5).
- **Confidence:** **Inferred** — inferred from router adjacency and handler preconditions. Steps 1
  and 2 in isolation are additionally covered by `Tests.EndToEnd` (§1.4), but no test runs the
  sequence.
- **Evidence paths:**
  `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:76-122`;
  `hianshul100_Pacco.Services.Availability/…/Handlers/ReserveResourceHandler.cs`;
  `hianshul100_Pacco.Services.Availability/tests/…Tests.EndToEnd/`;
  `api-inventory.md` §2.1.

### Deliberately not catalogued

*Observed.* Three candidates were considered and rejected, each with a reason:

| Candidate | Why it is not a journey |
|---|---|
| `Pacco.rest` (41 requests) | A catalogue. Every id variable is a zeroed GUID seeded at `:3-14`; no request consumes another's output; the banner comments group by **service**, not by sequence (§1.5) |
| The ten per-service `.rest` files | Same shape as above, one service each |
| The gateway route tables | Route tables. A list of routes is not a journey (§1.7); no route renders a component, so no link adjacency can exist |

---

## 6 — Cross-capability navigation summary

Edges where a **human-driven** journey moves from one capability to another. Machine-driven edges
(J4) are listed separately below, and J5 contributes nothing because it never leaves CAP-04.

| From Capability | To Capability | Journeys Using This Edge | Evidence |
|---|---|---|---|
| CAP-01 Identity & Access Management | CAP-03 Customer Profile & Lifecycle Management | J1 (H1) | `Pacco-sample-scenario.rest:26 → :30` |
| CAP-03 Customer Profile & Lifecycle Management | CAP-06 Parcel Catalogue & Volume Calculation | J1 (H2) | `…rest:40 → :45` |
| CAP-06 Parcel Catalogue & Volume Calculation | CAP-07 Order Lifecycle Management | J1 (H3) | `…rest:63 → :68`, with `@parcelId` captured at `:58` and spent at `:82` |
| CAP-07 Order Lifecycle Management | CAP-05 Vehicle Fleet Catalogue | J1 (H4) | `…rest:90 → :95` |
| CAP-05 Vehicle Fleet Catalogue | CAP-04 Resource Availability & Reservation | J1 (H5) | `…rest:112 → :119`; the alias `@resourceId = {{add_vehicle.response.headers.Resource-ID}}` at `:116` |
| CAP-04 Resource Availability & Reservation | CAP-07 Order Lifecycle Management | J1 (H6) | `…rest:129 → :135` |
| CAP-07 Order Lifecycle Management | CAP-04 Resource Availability & Reservation | J1 (H7) | `…rest:135 → :144` |
| CAP-04 Resource Availability & Reservation | CAP-09 Delivery Execution & Tracking | J1 (H8) | `…rest:153 → :159` |
| CAP-09 Delivery Execution & Tracking | CAP-07 Order Lifecycle Management | J1 (H9) | `…rest:193 → :197`; the real transfer is the `delivery_completed` event consumed by `Orders.Application/Events/External/Handlers/DeliveryCompletedHandler.cs` |
| CAP-01 Identity & Access Management | CAP-11 Operation Status Projection & Real-Time Notification | J2 (H10) | `app.js:21`; `PaccoHub.InitializeAsync`. **Edge exists in requirement only — no code path performs the transfer** (§4.2) |
| CAP-02 Edge Routing & Access Enforcement | CAP-11 Operation Status Projection & Real-Time Notification | J3 (H11) | `ntrada-async.yml` 20 × `use: rabbitmq`; `ntrada.yml:280-283`; `app.js:34-44`. **Inferred from configuration adjacency** |

**CAP-02 mediates every J1 and J3 edge** and is therefore not repeated as a source or target on the
nine J1 rows; all 24 J1 steps address `http://localhost:5000` (`Pacco-sample-scenario.rest:1`).

**Machine-driven edges (J4), listed for completeness and excluded from the count above:**

| From Capability | To Capability | Journeys Using This Edge | Evidence |
|---|---|---|---|
| CAP-10 Automated Order Orchestration | CAP-07 Order Lifecycle Management | J4 (H12) | `AIOrderMakingSaga.cs:63-68` |
| CAP-07 Order Lifecycle Management | CAP-05 Vehicle Fleet Catalogue | J4 (H13) | `AIOrderMakingSaga.cs:93`; `api-inventory.md` §6 row 6 |
| CAP-05 Vehicle Fleet Catalogue | CAP-04 Resource Availability & Reservation | J4 (H14) | `AIOrderMakingSaga.cs:98`; `api-inventory.md` §6 row 7 |
| CAP-04 Resource Availability & Reservation | CAP-07 Order Lifecycle Management | J4 (H15) | `AIOrderMakingSaga.cs:112-132` |

**Capabilities with no navigation edge in any evidenced journey:** CAP-08 Pricing & Discount
Calculation (reached only server-side, §3.2), CAP-12 through CAP-16. Their absence from this table is
a statement about the evidenced journeys, not about the capabilities — see §8 for which of them look
like genuine coverage gaps.

---

## 7 — Shared state summary

| Hand-off | State Carried | Mechanism | Evidence |
|---|---|---|---|
| H1 CAP-01 → CAP-03 | Bearer JWT; `customerId` | `Authorization` header; `customerId` bound from the `@user_id` claim by the gateway, never sent by the client | `Pacco-sample-scenario.rest:31-37`; `ntrada.yml:162-171` |
| H2 CAP-03 → CAP-06 | Bearer JWT; `customerId` | Header; `@user_id` binding | `…rest:45-54`; `ntrada.yml` parcels module `:357-400` |
| H3 CAP-06 → CAP-07 | `@parcelId` | `Resource-ID` response header → variable → URL path segment | `…rest:58, :82` |
| H4 CAP-07 → CAP-05 | **Nothing beyond the token** | Header only | `…rest:95-107` |
| H5 CAP-05 → CAP-04 | `@vehicleId`, aliased to `@resourceId`; `@tags` | `Resource-ID` header → second variable → request **body** | `…rest:111, :116-126` |
| H6 CAP-04 → CAP-07 | `@orderId`, `@vehicleId`, `@deliveryDate` | Two URL path segments plus a body field; `customerId` edge-injected | `…rest:135-141` |
| H7 CAP-07 → CAP-04 | `@resourceId`, `@deliveryDate` | Two URL path segments; `customerId` edge-injected via `bind: customerId:@user_id` | `…rest:144-150`; `ntrada.yml:101-104` |
| H8 CAP-04 → CAP-09 | `@orderId`, `@deliveryDate` | Request body. The resource id does **not** cross | `…rest:159-167` |
| H9 CAP-09 → CAP-07 | `@orderId` on the read; the delivery outcome itself | Client replays an id it already holds; the state transfer is the `delivery_completed` broker event | `…rest:184, :197`; `DeliveryCompletedHandler.cs` |
| H10 CAP-01 → CAP-11 | Bearer JWT | **Human paste into a text input**, then a SignalR method argument — not a header, not a URL, not a cookie, not storage | `index.html:14`; `app.js:12, :21` |
| H11 CAP-02 → CAP-11 | Correlation id | `Request-ID` response header on the write; Redis `requests:{id}`, 300 s sliding expiry; then a path segment or a push payload | `ntrada.yml` `generateRequestId`/`exposedHeaders`; `Operations.Api/appsettings.json` |
| H12 CAP-10 → CAP-07 | `OrderId`, `CustomerId` | `CreateOrder` broker message with the saga's correlation context | `AIOrderMakingSaga.cs:63-68` |
| H13 CAP-07 → CAP-05 | **Nothing** | Unparameterised `GET /vehicles`, `Items.FirstOrDefault()` | `AIOrderMakingSaga.cs:93` |
| H14 CAP-05 → CAP-04 | `VehicleId` used as `resourceId` | HTTP path segment | `AIOrderMakingSaga.cs:98` |
| H15 CAP-04 → CAP-07 | `VehicleId`, `CustomerId`, `ReservationDate`, `ReservationPriority` | Persistent saga state keyed on `OrderId`, then a `ReserveResource` message | `AIOrderMakingSaga.cs:45-54, :98-119` |

**Three mechanisms carry all shared state on this platform**, and no others were observed:

1. **The `Resource-ID` response header → URL path segment** loop. Nine of the fifteen hand-offs move
   an entity id this way. The header exists because the gateway declares
   `resourceId: { property: …, generate: true }` on the write route (`api-inventory.md` §2), so the
   id is minted at the edge before the owning service ever sees the command.
2. **Gateway claim injection** (`bind: …:@user_id`). Customer identity crosses four hand-offs
   without ever appearing in a request the caller writes.
3. **Broker messages and saga state**, for everything the caller cannot see (H9, H11, H12, H15).

**Not observed anywhere, having been looked for:** cookies, `localStorage`, `sessionStorage`,
server-side sessions, hidden form fields, and query-string state hand-off between capabilities. The
only query strings in the scenario are filter parameters consumed by the same capability that
receives them (`:63`, `:112`, `:129`).

---

## 8 — Evidence quality

### 8.1 — Confirmed (a named artifact fixes the order)

| Item | Artifact |
|---|---|
| J1's 24-step sequence, its entry and terminal points, and hand-offs H1–H9 | `Pacco-sample-scenario.rest`, with seven explicit variable captures (§2.2) |
| J1's shared state at every hand-off | The same file's `@name` / `{{…response…}}` lines, read directly |
| J2 steps 1 and 3–5 | `index.html` and `app.js`, both read in full |
| J4's internal step order and saga keying | `AIOrderMakingSaga.cs`, read in full |
| Route ownership for every step in every journey | `ntrada.yml` `downstream:` declarations |
| The three internal hand-offs in §3.2 | Handler source in `Orders`, `Availability`; `api-inventory.md` §6 |

### 8.2 — Inferred (derived, and labelled as derived)

| Item | Basis | Label used inline |
|---|---|---|
| J3 end to end | Configuration adjacency: async routes emit a correlation id, `operations-service` keys on it | "inferred from configuration adjacency" |
| J5 end to end | Router adjacency plus handler preconditions | "inferred from router adjacency" |
| J4's reachability from outside the platform | `.rest` file targets port 5015 directly; no gateway route | Inferred |
| The claim that J1's step order is *required* rather than *chosen* | Only the variable-capture dependencies are required; H6/H7's relative order is authored (§3.1) | Stated inline at §3.1 |

### 8.3 — Gaps

Each of the following is a capability or flow that the platform implements but that **no evidenced
journey exercises**. These are marked `[likely gap]` because the route or handler exists in code
while no ordered artifact reaches it — not because the topic went unexamined.

| `[likely gap]` | What exists in code | What no artifact does |
|---|---|---|
| Administrator customer administration | Five `role: admin` routes: `GET /customers`, `GET /customers/{id}`, `GET /customers/{id}/state`, `PUT /customers/{id}/state/{state}` (`ntrada.yml:138, :152, :160, :181`) and `GET /identity/users/{userId}` (`:246`) | No sequence signs in as an admin and uses them. `Pacco.rest:26` signs up an admin but its requests are unchained (§1.5) |
| Order cancellation | `DELETE /orders/{orderId}` and the saga's `CancelOrder` compensation (`AIOrderMakingSaga.cs:140-146`) | No artifact cancels an order after creating one |
| Delivery failure | `POST /deliveries/{deliveryId}/fail` (`api-inventory.md` §2.3) | J1 takes only the success path (`…rest:184` completes) |
| Reservation release | `DELETE /availability/resources/{resourceId}/reservations/{dateTime}` (`ntrada.yml:106-113`) | J1 reserves and never releases; J5's release step is inferred, not evidenced |
| Vehicle fleet administration | `PUT` and `DELETE /vehicles/{vehicleId}` (`ntrada.yml:416-455`) | J1 only creates and reads |
| Token renewal | `POST /identity/access-token/refresh` and `.../revoke` (`api-inventory.md` §2.4) | No sequence refreshes; J1 holds one token across 20 requests |
| Customer state promotion | `PUT /customers/{customerId}/state/{state}`, and `ReserveResourceHandler.cs:46-50` rejects reservations when the state is invalid | No artifact ever sets a customer state, so the rejection branch is unexercised by any evidenced journey |

### 8.4 — What was searched and found empty

Recorded in full in §1 rather than summarised here, because the hard requirement is proof and proof
is the list itself: §1.1 (filenames), §1.2 (directories), §1.3 (keywords, with the complete result
set), §1.4 (all 11 test projects), §1.5 (scenario artifacts), §1.6 (all 14 repositories). No client
router, no navigation guard, no e2e navigation suite and no navigation-bearing template exists in
this workspace, and §1 shows the paths that establish it.

---

## 9 — Confidence assessment

| Journey | Confidence | Key Uncertainty | Evidence Basis |
|---|---|---|---|
| **J1** Place and complete an order | **High** | The file is authored intent, not an executed test — nothing in CI runs it, so a step could have rotted against the current routes without anyone noticing. Every route it calls does exist in `ntrada.yml` today, which was checked | An ordered script with seven response-to-request variable captures; 24 steps each resolvable to a gateway route and an owning service |
| **J2** Operations console | **Medium** | How the operator gets a token. Six mechanisms were checked and only manual paste survives (§4.2). Also unverified: whether any deployment actually serves this page, since it is a `wwwroot` asset of a service | Two source files read in full, plus the hub method they call |
| **J3** Async write → outcome | **Low–Medium** | Whether any client performs both halves. The design is unambiguous; the usage is unevidenced. Also unknown whether the 300-second expiry is long enough for real callers | Gateway configuration plus the operations projection; no ordered artifact |
| **J4** Automated order making | **Medium** | Reachability, not behaviour. The saga's internal order is certain; who sends `MakeOrder`, and whether port 5015 is exposed in any deployment, is not established here | One source file read in full; a `.rest` file naming the entry port |
| **J5** Availability resource lifecycle | **Low** | Whether the five steps are ever performed as a sequence by anyone. The data dependency is real; the sequence is an inference from it | Router adjacency, handler preconditions, and two isolated end-to-end tests |

**Confidence in the absence findings (§1) is High.** They rest on exhaustive sweeps whose complete
result sets are published, including the four matches that are `Pacco.Context`'s own prose.

**Confidence in capability attribution is High** for all gateway-routed steps, because the route
declares its downstream service and `capability-baseline.md` §2 binds services to capabilities. It is
**Medium** for the two non-gateway surfaces (`/ui/index.html`, `ordermaker-service:5015`), which are
attributed from their owning repository rather than from a route table.

---

## 10 — Conflicts between sources

Source code is authoritative. Each conflict below states what the document claims, what the code
shows, and the file-path evidence. None has been silently reconciled.

### C1 — `capability-baseline.md` says vehicle mutation is admin-gated; the gateway does not gate it

- **What the doc claims:** `capability-baseline.md:480` — "CAP-05 | Admin-gated vehicle mutation at
  the edge | `ntrada.yml` vehicles module (`POST`/`PUT`/`DELETE /vehicles`)", at confidence
  *medium*, with the caveat "the `role: admin` blocks were counted, not individually re-attributed
  per route".
- **What the code shows:** all five routes in the vehicles module (`ntrada.yml:416-455`) carry
  `auth: true` and nothing else. All five `role: admin` blocks in the file sit in other modules —
  `customers` at `:138`, `:152`, `:160`, `:181` and `identity` at `:246`. The counts are identical in
  `ntrada-async.yml`, `ntrada.docker.yml` and `ntrada-async.docker.yml`.
- **Resolution:** the code wins. **Vehicle mutation is authenticated, not admin-gated.**
- **Why it matters here, specifically:** J1 signs up with `"role": "user"`
  (`Pacco-sample-scenario.rest:10`) and adds a vehicle at `:95`. If vehicles were admin-gated, the
  confirmed journey would be infeasible for its own actor. The journey is feasible **because** the
  gate the document describes does not exist. Carried as **B1**.

### C2 — `Pacco.rest` sends an availability read with no `Authorization` header; the route requires one

- **What the artifact shows:** `Pacco.rest:84` sends
  `GET {{api}}/availability/resources?tags=…` with no `Authorization` line, while its immediate
  neighbour at `:87-88` sends one.
- **What the code shows:** `ntrada.yml:78-82` declares that route `auth: true`.
- **Resolution:** the route configuration wins. The route is protected; the catalogue entry is
  simply incomplete. It is called out because it is the kind of discrepancy that reads as an
  unauthenticated endpoint if the `.rest` file is trusted over the gateway config. Carried as **Q3**.

### C3 — A web client repository exists but contains no application

- **What the repository layout implies:** `hianshul100_Pacco.Web` is one of the 14 clones in this
  workspace, named as the platform's web client.
- **What the code shows:** the repository contains one file, `README.md`, whose entire body is the
  string `# Pacco.Web`, on a single commit `b3bf026 Initial commit`. There is no `package.json`, no
  source directory, no build. `hianshul100_Pacco/README.md:33-44` lists eleven repositories to clone
  and **does not include `Pacco.Web`**.
- **Resolution:** **Future/Intended State (Not Implemented).** A web client is implied by the
  repository's existence and by nothing else. This is the reason the platform has no client router
  and no screen-to-screen journeys: the only rendered surface in the workspace is the operations
  console of J2 (`ui-inventory.md` §7).

### C4 — Prior-context retrieval returned material about a different product

- **What the retrieval returned:** the CAKE lookups recorded in *Inputs used* returned narrative
  about quick-service restaurant and cafe pilot operations — personas, workflows and journeys that
  correspond to no repository, route or capability in this workspace.
- **What the code shows:** every repository here is Pacco — a .NET Core parcel-delivery platform
  (`hianshul100_Pacco/README.md:6`).
- **Resolution:** **Unverifiable — Missing Source Evidence.** No claim from that material is used
  anywhere in this document, and none is reconciled against the code. It is disclosed so that a
  later reader who runs the same lookup and sees the same content knows it was seen and set aside.
  Carried as **Q4**.

### C5 — `Pacco-sample-scenario.rest` and `Pacco.rest` disagree about the actor's role

- **What the artifacts show:** the scenario signs up `"role": "user"` (`Pacco-sample-scenario.rest:10`);
  the catalogue signs up `"role": "admin"` (`Pacco.rest:26`).
- **Resolution:** not a defect — they are different artifacts with different purposes, and both
  roles are accepted by `POST /identity/sign-up`. It is recorded because the choice determines which
  routes each artifact can reach, and because it is the reason the five admin-gated routes appear in
  the catalogue but in no journey (§8.3).

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> **This section captures what could not be resolved from the available evidence.** Assumptions are
> stated so they can be checked. Blockers are things that stop later work until someone decides or
> supplies something. Open questions are things that need an answer but do not stop work today.
> Every item names who acts and when.

### Assumptions

| ID | Assumption | Why it was needed | How to check it | Risk if wrong |
|---|---|---|---|---|
| **A1** | An ordered request script whose later steps consume earlier steps' response values counts as evidence of a sequenced journey, in the same way an end-to-end test would | Without it this platform has no confirmed journey at all, because it has no client router and no navigation test | Read `Pacco-sample-scenario.rest` end to end and follow the seven variable captures listed in §2.2 | J1 would drop from Confirmed to Inferred, and the whole document would rest on route adjacency |
| **A2** | The `@parcelId`, `@orderId`, `@vehicleId` and `@deliveryId` variables are editor state held by the REST Client tool, not state the platform stores for the caller | It changes who is responsible for carrying journey state — here, the caller | Nothing in the platform reads these names; they exist only inside the `.rest` file | §4's account of shared state would be wrong about the direction of responsibility |
| **A3** | The 24 steps in the scenario file are meant to be run in file order, top to bottom | The file has no explicit "run in this order" statement; the order is implied by the captures and by the comments | `hianshul100_Pacco/README.md:63-67` presents the file as the walkthrough for the platform | Steps without a capture dependency (the ten observation `GET`s) might be re-orderable, which would not change any hand-off |
| **A4** | Each service repository owns exactly the capabilities `capability-baseline.md` §2 assigns to it, and a gateway route's `downstream:` service therefore names the owning capability | It is how every step in §2.3 was attributed without inventing ownership | Compare `ntrada.yml` `downstream:` values against `capability-baseline.md` §2 | Step-level capability attribution would shift, and with it the hand-off boundaries in §3 |
| **A5** | The operations console page at `/ui/index.html` is served in at least one running configuration | J2 is catalogued as a journey rather than as dead code | Check whether `operations-service` enables static file serving and whether port 5005 is published in the compose file | J2 would be an unreachable surface rather than a partial journey |

### Blockers

| ID | Blocker | What is blocked | Who acts | What is needed to clear it |
|---|---|---|---|---|
| **B1** | **[ACTION NOW]** `capability-baseline.md:480` states that vehicle mutation is admin-gated at the edge; `ntrada.yml:416-455` shows the vehicles routes carry only `auth: true`, and all five `role: admin` blocks belong to `customers` and `identity` (§10, C1) | Any later stage that reasons about who may do what — the baseline and this document currently disagree about the platform's access rules | The owner of `capability-baseline.md` | Correct the CAP-05 row in `capability-baseline.md` to say authenticated rather than admin-gated, or produce the configuration that gates it. The confidence caveat already on that row says the blocks were counted rather than attributed per route, so the correction is small |

### Open Questions

| ID | Question | Why it matters | Who answers | When it is needed |
|---|---|---|---|---|
| **Q1** | How does an operator obtain the JWT they paste into the operations console? Six mechanisms were checked and only manual paste survives (§4.2) | It is the only capability hand-off on the platform with no code path, and it is the boundary between CAP-01 and CAP-11 | The team that owns `Pacco.Services.Operations` | **[handled later by the target-architecture stage]** — before any decision is taken about how the operations surface is authenticated |
| **Q2** | Who sends `MakeOrder`, and is `ordermaker-service` port 5015 published in any deployment? | It decides whether J4 is a live path or an unreached component, which changes how much of the CAP-10 → CAP-07 → CAP-05 → CAP-04 traffic is real | The team that owns `Pacco.Services.OrderMaker` | **[handled later by the target-architecture stage]** — when cross-capability traffic is sized |
| **Q3** | Should `Pacco.rest:84` carry an `Authorization` header, given that `ntrada.yml:78-82` protects the route? (§10, C2) | A sample file that omits a required header teaches the wrong thing and can be mistaken for an open endpoint | The owner of `hianshul100_Pacco.APIGateway` | **[handled later by the migration-planning stage]** — whenever the sample files are next revised |
| **Q4** | The prior-context lookups returned material about restaurant and cafe operations, which matches nothing in this workspace (§10, C4). Is there prior context for **this** platform under a different project code? | If real Pacco context exists somewhere, journeys documented by the team could confirm or contradict the inferred journeys J3 and J5 | Whoever administers the shared context store for this engagement | **[handled later by the target-architecture stage]** — useful before, not required for, this baseline |
| **Q5** | Is J1's ordering of steps 17 and 18 — assign the vehicle, then reserve the resource — required, or incidental? `ReserveResourceHandler.cs:35-55` never inspects the order (§3.1) | If it is incidental, the H6/H7 round trip between CAP-07 and CAP-04 is not a real ordering constraint and the boundary is looser than it looks | The teams that own `Pacco.Services.Orders` and `Pacco.Services.Availability` | **[handled later by the target-architecture stage]** — before any conclusion is drawn about coupling between these two capabilities |
| **Q6** | Seven implemented flows are reached by no evidenced journey — admin customer administration, order cancellation, delivery failure, reservation release, vehicle administration, token refresh, customer state promotion (§8.3) | Each is either an unused feature or a journey nobody wrote down. The two call for opposite decisions | The product owner, with the service teams | **[handled later by the target-architecture stage]** — when scope is set |
| **Q7** | Is the 300-second sliding expiry on `requests:{id}` long enough for the callers J3 assumes? | If an operation outlives its record, the caller in J3 can never learn the outcome, and the CAP-02 → CAP-11 hand-off silently fails | The team that owns `Pacco.Services.Operations` | **[handled later by the target-architecture stage]** — before asynchronous write handling is redesigned |
