# Component Internals — Index

Working models of *how each Pacco component is built inside*, one document per component. These are
not surface catalogues: the surface inventory lives in `../baselines/service-summaries.md` and
`../baselines/api-inventory.md`, and the reusable-approach catalogue lives in `../patterns/index.md`.
This index is the registry for the folder — what has been modelled, what has not, and the conventions
every model in it follows.

**Every model is derived from source.** Where documentation and code disagreed, the code was taken as
authoritative and the disagreement was recorded in the affected model rather than resolved silently.
Where a mechanism lives in a NuGet package that is not in the workspace (`Ntrada`, `Convey`,
`Chronicle`, `Grpc.*`), the claim is tagged (`[ntrada]`, `[convey]`, `[chronicle]`, `[grpc]`,
`[framework]`) and, when the package's exact semantics would change a conclusion, marked
**`Unverifiable — Missing Source Evidence`**.

## Contents

1. [Catalogue](#1-catalogue)
2. [Coverage accounting](#2-coverage-accounting)
3. [Document conventions](#3-document-conventions)
4. [Naming notes and known discrepancies](#4-naming-notes-and-known-discrepancies)
5. [How this folder relates to the rest of the inventory](#5-how-this-folder-relates-to-the-rest-of-the-inventory)

---

## 1. Catalogue

Every component modelled so far, in batch order. **Concepts** counts the `§3.n` entries in the
document; **ABQ** counts assumptions + blockers, and **Open Qs** counts the open questions carried in
§8 — both are useful as a measure of what remains unsettled, not as a quality score.

| # | Component | Model | Source repository | Scoped path | Batch | Concepts | ABQ | Open Qs |
|---|---|---|---|---|---|---|---|---|
| 1 | `api-gateway` | [api-gateway.md](api-gateway.md) | `hianshul100_Pacco.APIGateway` | `.` (deployable at `src/Pacco.APIGateway`) | 1 of 7 | 22 | 8 | 8 |
| 2 | `availability-service` | [availability-service.md](availability-service.md) | `hianshul100_Pacco.Services.Availability` | `.` | 1 of 7 | 36 | 10 | 13 |
| 3 | `customers-service` | [customers-service.md](customers-service.md) | `hianshul100_Pacco.Services.Customers` | `.` | 2 of 7 | 37 | 10 | 8 |
| 4 | `deliveries-service` | [deliveries-service.md](deliveries-service.md) | `hianshul100_Pacco.Services.Deliveries` | `.` | 2 of 7 | 46 | 11 | 9 |
| 5 | `identity-service` | [identity-service.md](identity-service.md) | `hianshul100_Pacco.Services.Identity` | `.` | 3 of 7 | 40 | 14 | 7 |
| 6 | `operations-service` | [operations-service.md](operations-service.md) | `hianshul100_Pacco.Services.Operations` | `src/Pacco.Services.Operations.Api` | 3 of 7 | 45 | 23 | 9 |
| 7 | `operations-grpc-client` | [operations-grpc-client.md](operations-grpc-client.md) | `hianshul100_Pacco.Services.Operations` | `src/Pacco.Services.Operations.GrpcClient` | 4 of 7 | 25 | 7 | 5 |
| 8 | `ordermaker-saga-service` | [ordermaker-saga-service.md](ordermaker-saga-service.md) | `hianshul100_Pacco.Services.OrderMaker` | `.` (single project `src/Pacco.Services.OrderMaker`) | 4 of 7 | 40 | 15 | 8 |
| 9 | `orders-service` | [orders-service.md](orders-service.md) | `hianshul100_Pacco.Services.Orders` | `.` | 5 of 7 | 46 | 14 | 10 |
| 10 | `parcels-service` | [parcels-service.md](parcels-service.md) | `hianshul100_Pacco.Services.Parcels` | `.` | 5 of 7 | 40 | 16 | 8 |
| 11 | `pricing-service` | [pricing-service.md](pricing-service.md) | `hianshul100_Pacco.Services.Pricing` | `.` | 6 of 7 | 36 | 12 | 8 |
| 12 | `vehicles-service` | [vehicles-service.md](vehicles-service.md) | `hianshul100_Pacco.Services.Vehicles` | `.` | 6 of 7 | 47 | 12 | 9 |
| 13 | `platform-infrastructure-orchestration` | [platform-infrastructure-orchestration.md](platform-infrastructure-orchestration.md) | `hianshul100_Pacco` | `.` (no `*.csproj` — composition only) | 7 of 7 | 45 | 17 | 7 |

All thirteen were inspected at base ref `feature/12998/aidlc`. Every source repository was cloned
read-only and was never modified; the only writable repository in the discovery workspace is
`hianshul100_Pacco.Context`, which holds this inventory.

## 2. Coverage accounting

`../baselines/service-summaries.md` §7.1 fixes the modelling target at **14 entries** — 12 executable
host projects plus the 2 repositories that contain no `*.csproj` (`Pacco`, `Pacco.Web`). Class
libraries (21) and test projects (7) are excluded there and are excluded here for the same reason:
they are compiled into, and modelled under, their owning host.

| State | Count | Entries |
|---|---|---|
| **Modelled** | **13 / 14** | the thirteen rows in §1 (batches 1–7) |
| **Not modelled** | **1 / 14** | `Pacco.Web` (empty clone — README only) |

1. **`Pacco`** was modelled in batch 7 as `platform-infrastructure-orchestration` (§1 row 13). It has
   no `*.csproj` at all (`service-summaries.md` §3), so its model is an environment and composition
   model rather than a code-internals model: the concepts are compose stacks, PM2 manifests, the
   shared network, the container-name/DNS contract, image and tagging policy, and the workspace
   assembly scripts. The component name differs from the repository name because the artifact models
   a platform capability, not a codebase — see §4.
2. **`Pacco.Web`** remains unmodelled. Batch 7's authoritative scope was a single component
   (`platform-infrastructure-orchestration`), so `Pacco.Web` was not in it. The reason it was
   deferred still holds: it is a single-line `README.md` at commit `b3bf026` with no `src/`, no
   `package.json` and no `Dockerfile`, so there are no internals to model and any model written for
   it would be **`Unverifiable — Missing Source Evidence`** by construction. Its surface is already
   catalogued in `../repo-summary/Pacco.Web.md`. Whether it is in scope for the platform at all is
   carried as Q-7 in `platform-infrastructure-orchestration.md` §8.3.

An absence from §1 means *not modelled* — it does not mean the component was judged out of scope.

## 3. Document conventions

Every model in this folder carries sections **1–8** in this fixed shape — titles vary slightly in one
document, see the variations below — plus a `Contents` list immediately after the header block:

| § | Section | What belongs in it |
|---|---|---|
| 1 | Purpose & boundary | What the component is responsible for, and what it deliberately is not |
| 2 | Core concepts | The exhaustive concept list, enumerated so §3 can be checked for completeness |
| 3 | Per concept | One `§3.n` entry per concept: mechanism, citation, consequences |
| 4 | Primary control flows | End-to-end traces through the component, step by step |
| 5 | Persistence & schema evolution | What is stored, in what shape, and what a schema change costs |
| 6 | Surface → internals map | Each published route/message/RPC mapped to the internals that serve it |
| 7 | Change/extension guide | How to make the changes a maintainer will actually be asked for |
| 8 | Assumptions, Blockers & Open Questions | `A-n` / `B-n` / `Q-n`, each stated so it can be falsified or answered |

Two variations are in use and both are accepted:

1. **Cross-references** appear as a top-level `## 9. Cross-references` in the batch-1 models
   (`api-gateway`, `availability-service`) and as a `### 8.4` subsection in batches 2–4, titled for
   what it holds in that document (`Cross-references`, `Related patterns`, `Explicitly unverifiable`,
   `Baseline reconciliation`). New models should prefer the `§8.4` form.
2. **Section 2/3 titles** are `Core concepts (exhaustive)` / `Per concept` in ten models and
   `Core concepts` / `Concept-by-concept model` in `ordermaker-saga-service.md`. The content contract
   is identical.
3. **Sections 5 and 6 in a component with no code.** `platform-infrastructure-orchestration.md` is the
   only model whose component publishes no route, message or RPC and persists no application data. It
   keeps both sections and reinterprets them within the same contract: §5 covers the state the
   component *owns* (network, volumes, container writable layers) and how the declared topology
   evolves; §6 maps operator and tooling entry points — compose files, PM2 manifests, scripts — to the
   internals they trigger, and adds a table of the contracts the component asserts on the other twelve
   repositories. Any future model of a code-free component should follow that shape.

Other conventions that hold across the folder:

1. Pattern cross-references use `[[pattern-file-name]]`, matching `../patterns/`.
2. Committed credential material is cited **by path only**, adopting the rule already stated in
   `../patterns/index.md`. Real secret values are not reproduced here: the one place that did — the
   80-character symmetric `issuerSigningKey` quoted in `operations-service.md` §3.38 — is redacted
   to its citation. Placeholder literals such as `vault.token: "secret"` and
   `logger.seq.apiKey: "secret"` are quoted verbatim, because the fact that they are unchanged
   defaults *is* the evidence.
3. Open questions that restate a baseline gap name the baseline item they restate (e.g.
   `service-summaries.md` G14/Q11), so the same question is not counted twice across artifacts.
4. A **maintenance contract** — a change to the component's internals must update its model in the
   same change — closes `customers-service.md`, `deliveries-service.md`,
   `operations-grpc-client.md`, `ordermaker-saga-service.md`, `orders-service.md` and
   `parcels-service.md`; the last two state it as an explicit `§7.9` with a per-area table. It
   applies to every model in this folder regardless of whether the individual document restates it;
   new models should state it explicitly.
5. **Enumerated points are written as numbered lists, not bullet lists**, so a reader and a reviewer
   can refer to an individual point by its number. This index and the batch-6 models
   (`pricing-service.md`, `vehicles-service.md`) follow the convention throughout; the models from
   batches 1–5 still carry bullet lists in places and are to be converted the next time each is
   revised. Tables remain the preferred form wherever the points share a common set of columns.

## 4. Naming notes and known discrepancies

| Note | Detail |
|---|---|
| `ordermaker-saga-service` vs `ordermaker-service` | The model is filed under `ordermaker-saga-service`, describing its role. The deployable name in `Pacco/compose/services.yml` is **`ordermaker-service`** (port `5015`), and it is absent from both PM2 manifests and from all four `ntrada*.yml` — see `service-summaries.md` §7.2 and gap **G2**. |
| `operations-grpc-client` is not a deployable | `Pacco.Services.Operations.GrpcClient` appears in no compose file and no PM2 manifest. It is modelled as a component, not as a service (`service-summaries.md` §7.2). |
| Two models, one repository | `operations-service.md` and `operations-grpc-client.md` both scope into `hianshul100_Pacco.Services.Operations`, at `src/…Operations.Api` and `src/…Operations.GrpcClient` respectively. They model the two halves of the same gRPC contract and cite each other rather than re-deriving it. |
| The `orders` ↔ `parcels` PACT pair is disabled on both sides | The consumer test in `hianshul100_Pacco.Services.Orders` and the provider test in `hianshul100_Pacco.Services.Parcels` are each excluded from their own `.sln`, and the shared file-based `pacts` directory both point at exists in neither repository. The contract is verified by nobody — see `orders-service.md` §3.45 and `parcels-service.md` §3.38. Reviving one end alone is worse than reviving neither. |
| `parcel_deleted` is published and consumed on different exchanges | `parcels-service` publishes it on exchange `parcels`; `orders-service` declares its matching external event as `[Message("deliveries")]`. The binding never matches, so the handler has never run (`orders-service.md` §3.33, `parcels-service.md` §3.19). Recorded here because it is invisible from either model alone. |
| `pricing-service` is not a bounded context | **Confirmed in batch 6.** It has no write model, no exchange, no Mongo database and no persistence of any kind — one query, one HTTP client, one arithmetic service (`pricing-service.md` §3.17). `service-summaries.md` §3 had already recorded zero `rabbitMq` occurrences in its `appsettings.json`. |
| The `vehicles` variant search cannot match a filtered query | `Vehicle`'s constructor ORs `Variants.Standard` into every vehicle, while `SearchVehiclesHandler` filters on `Variants == query.Variants` exactly. A vehicle created with `variants=2` is stored as `3` and is unfindable by `?variants=2`; searching by capacity alone forces `Variants == 0`, which no vehicle satisfies. Both return HTTP 200 with an empty page (`vehicles-service.md` §3.4, §3.18). Recorded here because the two halves live in different projects and neither is wrong on its own. |
| `platform-infrastructure-orchestration` vs repository `Pacco` | The batch-7 model is filed under the capability it describes, not under its repository name, because the repository contains no code and the component it represents is the platform's composition layer. It is the only model in this folder whose name does not derive from a deployable or a project. |
| `Pacco.APIGateway.Ocelot` is referenced by tooling that nothing else knows about | The retired Ocelot gateway appears in `Pacco.sln` as the 41st project reference and in the repository array of all five `scripts/git-*.sh`/`.ps1` files, but in neither the README clone list nor the discovery scope, and there is no such directory in the workspace. Every workspace-assembly run therefore fails on one repository — silently under `xargs -P 0`. See `platform-infrastructure-orchestration.md` §3.4 and Q-1. |
| The platform has **seven** compose stacks, not eight | `../repo-summary/Pacco.md` says "8 stack definitions" / "eight compose entry points" at lines 41, 94 and 272. `compose/` holds seven `*.yml` stacks plus three image build contexts (`prometheus/`, `host-prometheus/`, `rabbitmq/`). Corrected with evidence in `platform-infrastructure-orchestration.md` §8.4.3, alongside three other reconciliations against that summary. |
| Every service maps every failure to HTTP 400 | `ExceptionToResponseMapper` hard-codes `BadRequest` on all arms in each service modelled so far — `vehicles-service.md` §3.26, `pricing-service.md` §3.13, `customers-service.md` §3.18. There is no 404 and no 5xx anywhere in the platform's service surface. Changing it is a coordinated, cross-repository break, which is why it is recorded at the index rather than treated as a per-service defect. |

## 5. How this folder relates to the rest of the inventory

| Question | Where to look |
|---|---|
| What components exist, and what is each one for? | `../baselines/service-summaries.md` §2–§3 |
| What does the platform look like end to end? | `../baselines/architecture-baseline.md` |
| What routes, messages and RPCs are published? | `../baselines/api-inventory.md` |
| What reusable approach does a mechanism belong to? | `../patterns/index.md` |
| What does a repository contain? | `../repo-summary/` |
| **How is a given component built inside?** | **this folder** |

A model here **complements** the baselines rather than superseding them: the baselines catalogue the
surface, the models explain the machinery behind it. Where a model corrects a baseline, it says so
explicitly and names the section it corrects — no baseline is edited silently from this folder.

---

*Index of the `component-internals` artifact set. Any batch that adds, renames or removes a model in
this folder must update §1 and §2 of this index in the same change.*
