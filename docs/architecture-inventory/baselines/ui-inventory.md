# Pacco — UI & Frontend Inventory (Discovery)

**Project Key:** Common Architecture
**Stage:** `architecture_discovery` — observed current-state frontend surface.
**No ADRs, no recommendations, no target state, no KG JSON, no test inventory.**
**Date of analysis:** 2026-09-04
**Branch:** `arch-discovery-21758174-49b6-4af2-9774-025561defc90`
**Workspace base ref for all analysed clones:** `feature/12998/aidlc`

This document inventories every user-facing or embedded presentation asset that exists in the
fourteen cloned Pacco repositories. It answers *what UI exists, what it is built from, how it is
built and shipped, what it talks to, and how it is composed* — nothing else. It does **not** propose
a target frontend architecture, a framework, a composition model, or a modernization path.

**Inputs used**

- The thirteen cloned source repositories plus the artifact repository (all read-only for analysis),
  which are the **source of truth** for every statement below. Where a prior document and the code
  disagree, the code wins and the disagreement is stated explicitly in §13.
- `docs/architecture-inventory/baselines/api-inventory.md` — HTTP, gRPC, SignalR and event contract
  baseline. Its §2, §4 and §5.2 are used as the **cross-reference for API usage** in §8 rather than
  re-deriving endpoints from JS and template scanning. Endpoint contracts are not restated here.
- `docs/architecture-inventory/baselines/capability-baseline.md` — capability identifiers `CAP-01`
  to `CAP-16`, used in §11. Capability definitions are not restated here.
- `docs/architecture-inventory/repo-inventory.md` §2.3 column 14 ("Frontend stack") — the prior
  per-repository frontend scan, cross-checked against source in §13.
- `docs/architecture-inventory/architecture-views.md` — runtime and deployment topology.
- `.attachments/01_product_backlog_20260903_170135_37cf143b.xlsx` — backlog issue **12998**
  "Pacco - Discovery - Attempt-2", which fixes the fourteen-repository scope, including
  `Pacco.Web`.
- No external frontend-application catalogue, screen register, route register, UI-to-service binding
  graph, or design-system reference for Pacco was retrievable for this baseline. The catalogue
  reachable from this workspace holds frontend material for an unrelated product (a WebPT EMR
  scheduling SPA), which describes no Pacco entity and is therefore not used anywhere in this
  document. Every frontend statement below is reconstructed from the cloned source.

**Evidence taxonomy used throughout.** *Observed* — read directly from a source, template, or config
file, path cited, with names copied verbatim. *Inferred* — a conclusion drawn from two or more
observed facts, labelled as such. *Assumption* — belief beyond what evidence shows, labelled
`[assumption]` and rolled into the ABQ section. *Unknown* / *not observed* — labelled as such, never
filled with a guess.

**Confidence scale used in every table.**

- **High** — the claim is fully readable from committed files whose paths are cited, with no
  dependency on runtime behaviour or on a package outside this workspace.
- **Medium** — the claim is anchored in committed files, but at least one element depends on
  framework behaviour implemented in a NuGet or npm package that is not vendored here.
- **Low** — existence is observed but the property claimed is not statically determinable.

**Cross-reference convention.** `A#` / `B#` / `Q#` refer to the *Assumptions, Blockers & Open
Questions* tables at the end of **this** file and nowhere else. `CAP-##` refers to
`capability-baseline.md` §1. `api-inventory.md §x` refers to that file's numbered sections.

**Terminology fixed once here.** A **UI surface** is a distinct presentation boundary evidenced by
its own entry asset, routes, templates, or app shell. A **frontend platform** is the shared stack
that cuts across surfaces — a common build pipeline, global vendor bundles, a shared layout shell,
or a single deployment unit carrying multiple surfaces. This document reports what the evidence
supports for each; it does not assume that either must exist.

## Table of contents

1. [Frontend asset location — exhaustive search and absence proof](#1--frontend-asset-location--exhaustive-search-and-absence-proof)
2. [UI surfaces vs frontend platform](#2--ui-surfaces-vs-frontend-platform)
3. [Rendering model](#3--rendering-model)
4. [Framework and version, with usage classification](#4--framework-and-version-with-usage-classification)
5. [Build pipeline, with build/runtime distinction](#5--build-pipeline-with-buildruntime-distinction)
6. [MFE and composition-pattern scan](#6--mfe-and-composition-pattern-scan)
7. [Module structure, routes and screens](#7--module-structure-routes-and-screens)
8. [API usage](#8--api-usage)
9. [Auth and session handling](#9--auth-and-session-handling)
10. [CSS approach, design system and component catalogue](#10--css-approach-design-system-and-component-catalogue)
11. [Ownership, deployment boundary, routing model, asset loading, state management](#11--ownership-deployment-boundary-routing-model-asset-loading-state-management)
12. [Capabilities mapped](#12--capabilities-mapped)
13. [Conflicts between sources](#13--conflicts-between-sources)
14. [UI architecture and evolution signals](#14--ui-architecture-and-evolution-signals)

- [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions) — mandatory final section

---

## 1 — Frontend asset location — exhaustive search and absence proof

The search was run across **all fourteen clones**, walking the full tree of each (not only repo
roots), with `.git` excluded. Three searches were executed: a **directory-name** search for the
conventional frontend roots, an **extension** search for presentation assets and view templates, and
a **manifest** search for Node/JS tooling.

### 1.1 — Extension search — the complete result

The extension sweep covered `*.html`, `*.htm`, `*.js`, `*.jsx`, `*.ts`, `*.tsx`, `*.vue`, `*.css`,
`*.scss`, `*.less`, `*.cshtml`, `*.razor`, `*.hbs`, `*.ejs`, `*.erb`, `*.html.twig`, `*.phtml`,
`*.blade.php` across every clone. *Observed.* It returned **exactly three files**, all in one
repository:

| # | Path | Bytes | Kind |
|---|------|-------|------|
| 1 | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/index.html` | 835 | Entry document |
| 2 | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/js/app.js` | 1 691 | Application script |
| 3 | `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/js/signalr.js` | 180 968 | Vendored library bundle |

**No other file with any of those extensions exists anywhere in the workspace.** Confidence: **high**
— the sweep is an exhaustive filesystem walk, not a sample.

### 1.2 — Directory search — the complete result

*Observed.* A directory-name walk for `public`, `src`, `src/frontend`, `src/client`, `src/ui`,
`resources`, `resources/js`, `resources/css`, `resources/views`, `resources/assets`, `static`,
`assets`, `web`, `wwwroot`, `webroot`, `views`, `ui`, `client`, `frontend`, `components`, `shared`,
`lib`, `design-system`, and `.storybook` returned:

| Repository | Directories matched | What they actually contain |
|---|---|---|
| `hianshul100_Pacco` | `assets/` | Four PNG architecture diagrams only — `clean_architecture.png`, `pacco_logo.png`, `infrastructure.png`, `pacco_overview.png`. No markup, no script, no stylesheet |
| `hianshul100_Pacco.Services.Operations` | `src/`, `src/Pacco.Services.Operations.Api/wwwroot/`, `.../wwwroot/ui/` | **The only frontend assets in the workspace** — the three files in §1.1 |
| `hianshul100_Pacco.APIGateway` and the ten other service repos | `src/` only | C# projects. No `wwwroot/`, `public/`, `static/`, `assets/`, `views/`, `ui/`, `client/`, `frontend/`, `components/`, `design-system/`, or `.storybook/` under any of them |
| `hianshul100_Pacco.Context` | `docs/` | Markdown discovery artifacts only |
| `hianshul100_Pacco.Web` | **none** | See §1.4 |

*Observed.* Top-level directories per repository, verbatim: `Pacco` → `scripts compose assets`;
`Pacco.APIGateway` → `src scripts`; `Pacco.Context` → `docs`; `Availability`, `Orders`, `Parcels` →
`tests src scripts`; `Customers`, `Deliveries`, `Identity`, `Operations`, `OrderMaker`, `Pricing`,
`Vehicles` → `src scripts`; `Pacco.Web` → none.

### 1.3 — Per-repository absence record

*Observed.* Every repository was walked in full. `files` counts all tracked files excluding `.git`;
`fe assets` counts files matching the §1.1 extension set.

| Repository | files | fe assets | Frontend outcome |
|---|---:|---:|---|
| `hianshul100_Pacco` | 29 | 0 | **None.** `assets/` holds 4 PNGs; `compose/` holds YAML and a RabbitMQ `Dockerfile`; `scripts/` holds shell. No manifest, no markup |
| `hianshul100_Pacco.APIGateway` | 30 | 0 | **None.** `src/Pacco.APIGateway/` holds `*.cs`, four `ntrada*.yml`, `appsettings*.json`, `*.rest`. No static root of any kind |
| `hianshul100_Pacco.Services.Availability` | 125 | 0 | **None.** Under `src/**` only `certs/` and `Properties/` are non-code; `tests/` is C# |
| `hianshul100_Pacco.Services.Customers` | 91 | 0 | **None.** `src/**` is C#, `appsettings*.json`, `certs/` |
| `hianshul100_Pacco.Services.Deliveries` | 85 | 0 | **None.** `src/**` is C#, `appsettings*.json`, `certs/` |
| `hianshul100_Pacco.Services.Identity` | 91 | 0 | **None.** `src/**` is C#, `appsettings*.json`, `certs/` |
| `hianshul100_Pacco.Services.Operations` | 57 | **3** | **The sole repo-owned UI surface** — §2.1 |
| `hianshul100_Pacco.Services.OrderMaker` | 49 | 0 | **None.** `src/**` holds C#, `certs/`, `Properties/` |
| `hianshul100_Pacco.Services.Orders` | 143 | 0 | **None.** `src/**` and `tests/**` are C# and Pact fixtures |
| `hianshul100_Pacco.Services.Parcels` | 101 | 0 | **None.** `src/**` and `tests/**` are C# and Pact fixtures |
| `hianshul100_Pacco.Services.Pricing` | 39 | 0 | **None.** `src/**` holds C#, `certs/`, `Properties/`, `.idea/` |
| `hianshul100_Pacco.Services.Vehicles` | 69 | 0 | **None.** `src/**` is C#, `appsettings*.json`, `certs/` |
| `hianshul100_Pacco.Web` | **1** | 0 | **Empty repository** — §1.4 |
| `hianshul100_Pacco.Context` | — | 0 | Documentation repository; markdown only, no frontend assets and none expected |

### 1.4 — `Pacco.Web` — the named-but-empty repository

*Observed.* A full listing of the `hianshul100_Pacco.Web` clone returns exactly two entries besides
`.git`: the clone root and `README.md`. The file's entire content is the single line `# Pacco.Web`.
There is no `package.json`, no `src/`, no `public/`, no markup, no script, no stylesheet, no
project file, and no configuration.

This repository carries the name most likely to hold a web client, and it is on the discovery scope
list fixed by backlog issue 12998, so its emptiness is a **material finding, not an omission**. It is
recorded as **Unverifiable — Missing Source Evidence** for the question "does a Pacco web client
exist outside this workspace", and carried as **B1**. Confidence in the observation itself: **high**.

### 1.5 — Node and JS tooling manifest search

*Observed.* A workspace-wide search for `package.json`, `package-lock.json`, `yarn.lock`,
`pnpm-lock.yaml`, `tsconfig.json`, `angular.json`, `next.config.*`, `vite.config.*`,
`webpack.config.*`, `rollup.config.*`, `nuxt.config.*`, `esbuild.config.*`, `Gruntfile.js`,
`Gulpfile.js`/`gulpfile.js`, `.babelrc`, `babel.config.*`, `tailwind.config.*`,
`postcss.config.*`, `*.stories.*`, `.storybook/`, and `node_modules/` returned **zero matches in
all fourteen clones**. Confidence: **high**.

Per the guardrail, this absence is **not** treated as proof of no frontend — §1.1 already
established that a frontend exists. It is treated as what it is: evidence that the frontend that
does exist is authored and shipped without Node tooling (§5).

### 1.6 — Conclusion of Step 1

*Observed.* The workspace contains frontend assets. They are **three files in one directory of one
repository**, totalling 183 KB, of which 181 KB (98.9 %) is a single vendored third-party library
bundle. Twelve of the thirteen non-documentation repositories contain no presentation asset of any
kind. Analysis proceeds on the surface that exists; nothing is fabricated for the twelve that have
none.

---

## 2 — UI surfaces vs frontend platform

Two presentation surfaces are evidenced in the workspace, and they are **not** variants of one
another: one is authored in this repository, the other is generated at runtime by a package. A third
category — vendor operational consoles — is named and excluded, with the reason stated. **No shared
frontend platform is observed** (§2.5).

### 2.1 — Surface 1: the Operations SignalR message console *(repo-owned)*

| Property | Value | Evidence | Conf. |
|---|---|---|---|
| Surface name in the product | `Pacoo SignalR messages` (spelling verbatim, including the `Pacoo` typo) | `wwwroot/ui/index.html:5` `<title>`, `:12` `<h3>` | High |
| Surface type | **Developer / diagnostic console.** Its whole function is to paste a JWT, open a hub connection, and watch raw operation messages scroll. It exposes no domain workflow — no order, parcel, vehicle, delivery, customer, or reservation action is reachable from it | `index.html` (one text input, one button, one `<ul>`); `app.js` (five hub listeners, no domain call) | High |
| Owning repository | `hianshul100_Pacco.Services.Operations` | §1.1 | High |
| Root path in repo | `src/Pacco.Services.Operations.Api/wwwroot/ui/` | §1.1 | High |
| Entry asset | `wwwroot/ui/index.html` | Observed | High |
| Audience | **Not stated anywhere.** No README, comment, or config names its intended user. That it requires a raw JWT to be pasted by hand into a plain text box (§9) is *inferred* evidence of an operator/developer audience, not an end-customer one | `index.html:14`; absence of any doc in `Pacco.Services.Operations/` | Medium (inferred) |
| Related capability | **CAP-11** only | §12 | High |

### 2.2 — Surface 2: Swagger API documentation UI *(framework-generated, no assets in repo)*

*Observed.* Five components enable an interactive Swagger UI, and **none of them ships a single
frontend file into this workspace** — the markup, JS and CSS are emitted at runtime from inside the
`Convey.WebApi.Swagger` / `Convey.Docs.Swagger` NuGet packages, which are not vendored here.

| Host | Route prefix | Enabling evidence | Conf. |
|---|---|---|---|
| `api-gateway` | `/docs` | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:50-56` — `swagger: { name: Pacco, reDocEnabled: false, title: Pacco API, version: v1, routePrefix: docs, includeSecurity: true }`; identical block in `ntrada-async.yml`, `ntrada.docker.yml`, `ntrada-async.docker.yml` | High |
| `identity-service` | `/docs` | `Identity.Infrastructure/Extensions.cs:87` `.AddWebApiSwaggerDocs()`, `:94` `.UseSwaggerDocs()`; `Identity.Api/appsettings.json:160` `swagger` block | High |
| `availability-service` | `/docs` | `Availability.Infrastructure/Extensions.cs:91,99`; `Availability.Api/appsettings.json:157` | High |
| `vehicles-service` | `/docs` | `Vehicles.Infrastructure/Extensions.cs:73,80`; `Vehicles.Api/appsettings.json:155` | High |
| `operations-service` | `/docs` | `Operations.Api/Infrastructure/Extensions.cs:77,84`; `Operations.Api/appsettings.json:155` — `enabled: true`, `reDocEnabled: false`, `routePrefix: docs`, `includeSecurity: true` | High |

`ordermaker-service` imports `Convey.Docs.Swagger` (`OrderMaker/Extensions.cs:6`) but its
`Program.cs` calls `UseApp()` rather than `UseInfrastructure()`, so whether the docs middleware is
mounted on that host is **not observed** from this workspace. Confidence: **low** for
`ordermaker-service` specifically, **high** for the other five.

**Classification.** Surface 2 is a **real, reachable UI surface** and is inventoried as such, but it
is **not a frontend asset of this codebase**: nothing about its markup, styling, component set, or
bundle can be read from any file here, and it changes when the package version changes, not when
this repository changes. Every property of it below is therefore recorded as **not observed**
rather than guessed (Q1).

### 2.3 — Excluded: vendor operational consoles

*Observed.* `Pacco/compose/grafana-seq-jaeger-prometheus.yml` publishes Grafana on `3000:3000`
(`image: grafana/grafana`), Jaeger on `16686` among others (`image: jaegertracing/all-in-one`), and
Seq on `5341:80` (`image: datalust/seq`). `Pacco/compose/consul-fabio-vault.yml` and
`compose/rabbitmq/Dockerfile` bring further vendor consoles.

These are browser UIs that a Pacco operator will use, and naming them is honest. They are
**excluded from the frontend inventory** because they are unmodified third-party container images:
no Pacco repository contributes markup, script, style, theme, dashboard JSON, or configuration to
their presentation layer. Recording them as Pacco UI surfaces would attribute frontend assets to
this platform that it does not own. Confidence in both the observation and the exclusion: **high**.

### 2.4 — Cross-surface relationship

| Relationship dimension | Finding | Evidence | Conf. |
|---|---|---|---|
| Shared layout shell | **None.** Surface 1 has its own complete `<html>` document with no layout inheritance, no partial, no include, no template engine. Surface 2's layout lives inside a NuGet package | `index.html` (a whole document in 26 lines); no `*.cshtml`, `*.razor`, or any template file exists (§1.1) | High |
| Shared assets | **None.** Surface 1's only external asset is a Bootstrap CDN URL (§10). Surface 2 loads whatever the package emits. No file is referenced by both | `index.html:6,23,24` | High |
| Shared build output | **None.** There is no build (§5) | §1.5, §5 | High |
| Shared deployment | **Partial and incidental.** Surfaces 1 and 2 are both carried inside the `operations-service` process image, so they ship together — but only because both happen to live in that one host, not through any frontend packaging relationship. Surface 2 additionally ships inside five *other* service images | `Pacco.Services.Operations/Dockerfile`; §2.2 table | High |
| Classification | **Operationally coupled, not shared.** Separate entry points and separate origins of content, with a shared release only where they co-reside in one backend deployable | Rows above | High |

### 2.5 — Frontend platform: not observed

*Observed.* A frontend platform would require at least one of: a common build pipeline, a global
vendor bundle loaded by more than one surface, a shared layout shell, or a deployment unit that
carries multiple surfaces as its purpose. **None exists.** There is no build pipeline (§5), no
bundle shared across surfaces (§2.4), no layout shell (§2.4), and no deployment unit whose purpose
is to carry UI (§11.2).

**Finding: single repo-owned UI surface; no frontend platform.** Confidence: **high**.

---

## 3 — Rendering model

**Finding: static client-side-scripted HTML, served as a plain static file from a backend service.
Not a SPA. Not server-rendered in the templating sense. Not MFE.** Confidence: **high**.

Each candidate label was tested against the evidence rather than assumed:

| Candidate label | Verdict | Evidence for the verdict |
|---|---|---|
| **SPA** | **Rejected.** | No client-side router, no route table, no history/`pushState`/`hashchange` use, no application mount point, no component tree, no framework runtime. `app.js` is a 53-line IIFE that queries three fixed element ids and attaches handlers. Per the classification rule, the presence of a JS library is not sufficient — a composition or routing mechanism must be shown, and none is. `app.js` in full |
| **Server-rendered (templated)** | **Rejected.** | `index.html` is a static file with no template syntax, no directives, no server-side interpolation, and no `json_encode` / `@json` / `JSON.stringify` server handoff. No view engine is configured on any host — no `*.cshtml`, `*.razor`, `*.erb`, `*.twig`, `*.phtml`, `*.blade.php` or `*.hbs` exists in the workspace (§1.1). `operations-service` mounts `UseStaticFiles()` and no view middleware |
| **Hybrid** | **Rejected.** | Requires both of the above to be partly present. Neither is |
| **MFE** | **Rejected.** | §6 — no composition mechanism found |
| **No UI** | **Rejected.** | §1.1 — three assets exist |
| **Static HTML + client-side scripting** | **Accepted.** | The document is committed as-is, delivered byte-for-byte by `UseStaticFiles()`, and all behaviour after load is imperative DOM manipulation driven by a persistent SignalR connection |

**Server-to-client state handoff: none observed.** *Observed.* No inline `<script>` block exists in
`index.html`; the only two script tags are external `src` references (`:23`, `:24`). No `data-*`
attribute carries server state. No embedded JSON island exists. Every byte the client renders after
load arrives over the SignalR websocket, not through the HTML document. Confidence: **high**.

**Inline initialization / bootstrapping.** *Observed.* All initialization is inside `app.js` itself:
an IIFE (`app.js:2`) under `'use strict'` (`:1`) that builds the hub connection eagerly at parse
time (`:6-9`) but does **not** connect until the user clicks (`:11-24`). There is no separate
bootstrap file and no framework bootstrap sequence. Confidence: **high**.

**Feature-flag or permission-gated asset loading: none.** *Observed.* Both script tags and the one
stylesheet link are unconditional. This is consistent with the platform-wide finding in
`capability-baseline.md` §3 that no feature-flag system exists in any repository. Confidence: **high**.

---

## 4 — Framework and version, with usage classification

**No JS framework is used.** *Observed.* No `package.json` exists to declare dependencies (§1.5), so
the fallback method was applied: each library file was read directly for a version header.

### 4.1 — Libraries detected, with role and usage classification

| Library | Version | Approx. release year | Where declared / evidenced | Role | Usage classification | Conf. |
|---|---|---|---|---|---|---|
| **`@aspnet/signalr` JS client** | **1.1.0** | 2019 | `wwwroot/ui/js/signalr.js` — the string `VERSION = "1.1.0"` in the bundle; the package name `@aspnet/signalr` appears 20 times in the bundled module paths. The pre-`@microsoft/signalr` package name pins the era | Real-time transport client | **Actively used by Surface 1.** `app.js:6` calls `new signalR.HubConnectionBuilder()` against the global the UMD wrapper installs (`root["signalR"] = factory()`, `signalr.js:8`), and `index.html:23` loads it before `app.js` | High |
| **`es6-promise` (auto polyfill)** | **4.2.2+97478eb6** | 2017 | `signalr.js:175` — `* @version   v4.2.2+97478eb6`, an inlined vendored module inside the SignalR bundle | Promise polyfill for older browsers | **Globally shared dependency of Surface 1, transitively.** Not referenced by any Pacco-authored file; it is pulled in because it is compiled into `signalr.js`, and it self-installs via `Promise$2.polyfill()` (`signalr.js:~1352`) | High |
| **Bootstrap CSS** | **4.0.0** | 2018 | `index.html:6` — `https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0/css/bootstrap.min.css` with `integrity="sha384-Gn5384xqQ1aoWXA+058RXPxPg6fy4IWvTNh0E263XmFcJlSAwiGgFAW/dAiS6JXm"` and `crossorigin="anonymous"` | Visual framework / theme | **Actively used by Surface 1, CSS only.** Its class names drive the entire layout — `container`, `row`, `col-lg-12`, `form-group`, `form-control`, `btn btn-primary`, `list-group`, `list-group-item`, `list-group-item-${type}`. **Bootstrap's JavaScript, jQuery and Popper.js are not loaded at all**, so no Bootstrap JS component (modal, dropdown, tooltip, collapse) is available | High |

### 4.2 — Frameworks explicitly searched for and not found

*Observed.* React, Angular, AngularJS, Vue, Backbone, Ext JS, Svelte, Ember, jQuery, Knockout,
Alpine, Lit, Stencil, Preact, htmx, and Blazor: **none present**, in any form — no vendor file, no
CDN reference, no package declaration, no import, no NuGet package reference. The extension sweep
(§1.1) returned no `*.jsx`, `*.tsx`, `*.vue`, or `*.razor` file anywhere, and the manifest sweep
(§1.5) returned no `angular.json`, `next.config.*`, `vite.config.*`, `nuxt.config.*`, or
`webpack.config.*`. Confidence: **high**.

### 4.3 — Statement of the framework finding

**Vanilla JS / no framework detected.** The JS that is present is:

- **`app.js` (53 lines, Pacco-authored).** ES5-plus-template-literal script under `'use strict'`,
  wrapped in an IIFE. It uses `document.getElementById` three times, one `onclick` assignment, five
  `connection.on(...)` subscriptions, one `connection.invoke(...)`, one `alert()`, and one helper
  that appends to `innerHTML`. No module system (`import`, `export`, `require`, `define`) is used by
  it. No transpilation target, no polyfill authored here, no build step.
- **`signalr.js` (4 088 lines, third-party).** A **pre-built** webpack UMD bundle — its first ten
  lines are the standard `webpackUniversalModuleDefinition` wrapper, and it ends with
  `//# sourceMappingURL=signalr.js.map` (`:4088`). It carries WebSockets, ServerSentEvents and
  LongPolling transports and the `HubConnectionBuilder` API. Neither referenced source map
  (`signalr.js.map`, `es6-promise.auto.map`) is committed, so the bundle is not debuggable from this
  repository.

**SPA classification: not SPA** — see §3 for the evidence each rejected label rests on.

---

## 5 — Build pipeline, with build/runtime distinction

**No build pipeline detected. JS and CSS are served as plain static assets.** Confidence: **high**.

### 5.1 — Frontend build tooling: detected vs active vs dormant

| Category | Finding | Evidence |
|---|---|---|
| Build tooling **detected in repository** | **None.** Zero `package.json`, `webpack.config.*`, `vite.config.*`, `rollup.config.*`, `esbuild.config.*`, `Gruntfile.js`, `Gulpfile.js`, `.babelrc`, `babel.config.*`, `tsconfig.json`, `tailwind.config.*`, `postcss.config.*`, or `.storybook/` across all fourteen clones | §1.5 |
| Build tooling **actively used by the analyzed UI** | **None.** Nothing transforms, bundles, minifies, transpiles, hashes, or fingerprints `index.html` or `app.js`. What is committed is what the browser receives | §1.5; `Operations.Api/Infrastructure/Extensions.cs:88` `.UseStaticFiles()` |
| **Legacy or dormant** tooling | **None.** There is no orphan config and no superseded pipeline to find | §1.5 |
| **Imported build output** | **One artifact.** `signalr.js` is a webpack bundle, but it was produced by the SignalR project's own release pipeline and **vendored in pre-built**. No webpack exists here to have produced it. It is build *output*, not evidence of a build *pipeline* in this repository | `signalr.js:1-11` (webpack UMD preamble); §1.5 (no webpack config, no `node_modules`) |

### 5.2 — CI/CD and container configs — checked for frontend build steps

*Observed.* Every CI and container manifest in the workspace was read for a frontend build command
(`npm ci`, `npm run build`, `yarn build`, `dotnet publish` of a JS project, an asset task):

| Config checked | Frontend build step? | Evidence |
|---|---|---|
| `Pacco.Services.Operations/Dockerfile` | **No.** `dotnet publish src/Pacco.Services.Operations.Api -c release -o out` and nothing else. The `wwwroot/` tree is carried into `out` by the .NET SDK's default web-content convention, unmodified | File read in full (11 lines) |
| `Pacco.Services.Operations/scripts/build.sh` | **No.** Entire content: `dotnet build -c release` | Observed |
| `Pacco.Services.Operations/scripts/dockerize.sh` | **No.** Travis-driven `docker build` / `docker push` to `$DOCKER_USERNAME/pacco.services.operations` | Observed |
| `Pacco.Services.Operations/scripts/start.sh` | **No.** `export ASPNETCORE_ENVIRONMENT=local` then `dotnet run` | Observed |
| `.travis.yml` in every repository | **No.** Each chains `scripts/build.sh` → `scripts/dockerize.sh`; none installs Node or runs a JS task | Observed across all service repos |
| `Pacco/compose/*.yml` (six stacks) | **No.** Image pulls and port mappings only | Observed |
| GitHub Actions / Jenkinsfile / CircleCI / buildspec | **Not present.** No `.github/workflows/`, `Jenkinsfile`, `.circleci/config.yml`, or `buildspec.yml` in any clone | Observed |

### 5.3 — How the assets reach the runtime

*Observed, high confidence.* `Pacco.Services.Operations.Api.csproj` declares
`<Project Sdk="Microsoft.NET.Sdk.Web">` and contains an explicit
`<ItemGroup><Folder Include="wwwroot\ui\js" /></ItemGroup>`. That `Folder` item is an IDE
directory-visibility hint, **not** a packaging instruction; the files are published because the Web
SDK treats `wwwroot/**` as static web assets by default. `dotnet publish` copies them verbatim into
the image, and `Operations.Api/Infrastructure/Extensions.cs:88` mounts `.UseStaticFiles()` in the
middleware chain to serve them.

**`operations-service` is the only host in the platform that serves static files.** A workspace-wide
search for `UseStaticFiles`, `UseDefaultFiles`, `UseFileServer`, `WebRootPath`, `UseSpa`,
`MapFallbackToFile`, and `wwwroot` across `*.cs`, `*.csproj`, `*.json` and every `Dockerfile`
returned exactly two hits, both in that repository: the `csproj` `Folder` item and the
`UseStaticFiles()` call. Confidence: **high**.

---

## 6 — MFE and composition-pattern scan

**No micro-frontend architecture. No composition mechanism of any kind.** Confidence: **high**.

### 6.1 — Search executed

*Observed.* Every clone was searched (case-insensitive where meaningful, `.git` excluded, **all**
file types including the vendored bundle) for: `federation`, `mfe`, `microfrontend`,
`micro-frontend`, `__webpack_share_scopes__`, `customElements.define`, `single-spa`,
`loadRemoteModule`, `registerApplication`, `defineCustomElement`, `ModuleFederationPlugin`,
`remoteEntry`, `remotes:`, `exposes:`, `shared:`, `System.import`, `importmap`, `import-map`,
`web component`, `shadowRoot`, `attachShadow`.

### 6.2 — Complete match list

| Match | File and line | What the evidence actually suggests | Substantive? |
|---|---|---|---|
| `federation` | `hianshul100_Pacco/docker-images.txt:357` — `# external systems (federation, remote storage, Alertmanager).` | A comment in Prometheus operational notes describing **Prometheus federation**, a metrics-scraping topology between Prometheus servers. It concerns monitoring data flow. It is in a plain-text ops file, in a repository with no frontend assets at all (§1.3), and has no relationship to JavaScript, bundling, or UI composition | **No** — unrelated homonym |

**That is the entire result set.** Every other search term returned **zero matches in every file of
every clone**, including inside the 4 088-line `signalr.js` bundle, which was searched separately
and explicitly and matched none of them.

### 6.3 — Why no composition mechanism is concluded

Per the stated rule, an MFE conclusion requires a **concrete composition mechanism** in the
repository. Each accepted mechanism was tested and each is absent:

| Mechanism | Present? | Evidence |
|---|---|---|
| Module Federation config (`ModuleFederationPlugin`, `remotes:`, `exposes:`, `shared:`) | **No** | Zero matches; and no webpack config exists to hold one (§1.5) |
| `single-spa` registration (`registerApplication`) | **No** | Zero matches |
| Custom element registration (`customElements.define`, `defineCustomElement`, `attachShadow`) | **No** | Zero matches |
| Import maps with remote entries | **No** | Zero matches; `index.html` has no `<script type="importmap">` |
| `remoteEntry` / runtime remote module loading | **No** | Zero matches; `app.js` performs no dynamic `import()`, no script injection, no `fetch` of a module |
| Runtime script injection tied to UI composition | **No** | `app.js` creates no `<script>` element and mutates no `src` attribute |

The single `federation` hit is precisely the kind of weak keyword match the rule warns against, and
it is recorded above rather than suppressed, so the reasoning is auditable.

---

## 7 — Module structure, routes and screens

Step 5's template analysis is run **per surface**. Surface 2 (Swagger) has no analysable template in
this workspace (§2.2), so its rows are **not observed** rather than guessed.

### 7.1 — Surface 1 — entry-document analysis

*Evidence base for the whole subsection:*
`hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/index.html`
(26 lines, read in full) and `.../wwwroot/ui/js/app.js` (53 lines, read in full).

| Step 5 dimension | Finding for Surface 1 | Evidence | Conf. |
|---|---|---|---|
| **CSS files loaded** | Exactly one, remote: Bootstrap 4.0.0 from `maxcdn.bootstrapcdn.com`, with an SRI `integrity` hash and `crossorigin="anonymous"`. **No local stylesheet exists** | `index.html:6`; §1.1 (no `*.css` in workspace) | High |
| **JS files loaded, and in what order** | Two, both local, both at the end of `<body>`, both without `defer` or `async`, so they execute in document order: **(1)** `js/signalr.js`, **(2)** `js/app.js`. The order is load-bearing — `app.js:6` dereferences the `signalR` global that `signalr.js` installs, so reversing the tags breaks the page | `index.html:23-24`; `signalr.js:8`; `app.js:6` | High |
| **Server-side data serialised into inline JS** | **None.** No `<script>` block with a body exists; both script tags are `src`-only. No `json_encode` / `@json` / `JSON.stringify` server-side interpolation is possible because the file is static, not templated | `index.html` in full | High |
| **Feature-flag evaluations gating asset loading** | **None.** All three asset references are unconditional. Consistent with the platform-wide absence of any flag system recorded in `capability-baseline.md` §3 | `index.html:6,23,24` | High |
| **Layout inheritance / parent layout** | **None.** `index.html` is a complete standalone document — `<!doctype html>` through `</html>` in 26 lines. It extends nothing and is extended by nothing. No layout template exists in the workspace | `index.html:1,25-26`; §1.1 | High |
| **Partials / includes / shared fragments** | **None.** No include, no partial, no template engine, no fragment file | `index.html` in full; §1.1 | High |
| **Global vs domain-specific scripts** | **The distinction does not apply — there is no common layout to carry a global script.** Both scripts are page-scoped by construction: `signalr.js` is the vendor library and `app.js` is the page's own logic. Neither is shared with any other page, because no other page exists | `index.html:23-24`; §1.1 | High |
| **Inline initialization / bootstrapping** | All in `app.js`: an IIFE resolves three element ids at parse time (`:3-5`) and builds — but does not start — the hub connection (`:6-9`). Connection start is deferred to the click handler (`:19`) | `app.js:2-24` | High |
| **Server-to-client state handoff** | **None at page load.** No hydration, no embedded JSON, no `data-*` attribute, no server-set global. All state arrives after connect, over the websocket, as SignalR message arguments | `index.html` in full; `app.js:34-44` | High |
| **Feature-flag or permission-gated asset loading** | **Not observed.** No such gate exists on this surface | As above | High |
| **Forms and actions implying backend endpoints** | **None.** There is **no `<form>` element**, no `action` attribute, no `method`, and no named route. The one `<input id="jwt">` and one `<button id="connect">` are bound only by JS `onclick`; nothing submits. The surface therefore implies **no HTTP endpoint at all** — its entire backend contract is the SignalR hub (§8) | `index.html:13-17`; `app.js:11` | High |
| **Observable asset loading for this domain** | Template-driven inclusion of two page-scoped local scripts plus one CDN stylesheet. No bundle reference, no dynamic injection, no lazy load. Fully detailed in §11.4 | `index.html:6,23,24` | High |

### 7.2 — Surface 1 — module structure

*Observed.* There is **no module system** — no `import`/`export`, no `require`, no AMD `define`, no
`<script type="module">` in the Pacco-authored code. "Module" below means a distinct unit of the
delivered asset graph, tied to its path.

| Unit | Path (repo-relative) | Purpose | Authored here? | Conf. |
|---|---|---|---|---|
| Entry document | `src/Pacco.Services.Operations.Api/wwwroot/ui/index.html` | Declares the single screen's markup: a JWT text input, a Connect button, and an empty `<ul id="messages">` that all output is appended into. Loads the stylesheet and the two scripts | Yes | High |
| Page controller | `src/Pacco.Services.Operations.Api/wwwroot/ui/js/app.js` | The whole of the surface's behaviour: element lookup, hub-connection construction, click-to-connect with a whitespace/empty JWT guard, five inbound message subscriptions, and one DOM append helper | Yes | High |
| Transport library | `src/Pacco.Services.Operations.Api/wwwroot/ui/js/signalr.js` | `@aspnet/signalr` 1.1.0 client — `HubConnectionBuilder`, WebSockets / ServerSentEvents / LongPolling transports, plus an inlined `es6-promise` 4.2.2 polyfill. Installs the `signalR` global via a UMD wrapper | No — vendored pre-built | High |

*Observed.* `app.js` internal structure, in order: `'use strict'` (`:1`); IIFE open (`:2`); three
`getElementById` bindings — `$jwt`, `$connect`, `$messages` (`:3-5`); `HubConnectionBuilder` with
`.withUrl(...)`, `.configureLogging(signalR.LogLevel.Information)`, `.build()` (`:6-9`); the
`$connect.onclick` handler with its guard `if (!jwt || /\s/g.test(jwt))` → `alert('Invalid JWT.')`
(`:11-24`); five `connection.on` subscriptions (`:26-44`); the `appendMessage(message, type, data)`
helper (`:46-52`).

### 7.3 — Routes and screens

| Surface | Screen | How it is addressed | Route owner | Evidence | Conf. |
|---|---|---|---|---|---|
| Surface 1 | The one and only screen, titled `Pacoo SignalR messages` | Static file path served by `UseStaticFiles()` from the web root — i.e. `/ui/index.html` on the `operations-service` host | ASP.NET Core static-file middleware, from the on-disk path | `index.html:5,12`; `Operations.Api/Infrastructure/Extensions.cs:88` | High |
| Surface 1 | Directory-style URL `/ui/` | **Does not serve the page.** `UseDefaultFiles()` is **not** called anywhere in the workspace (§5.3), and `UseStaticFiles()` alone does not map a directory request to `index.html`. The explicit filename is required | Absence of `UseDefaultFiles` across all clones | High |
| Surface 1 | Any other screen | **None exist.** One HTML file, one `<h3>`, no navigation element, no `<a href>`, no router | `index.html` in full | High |
| Surface 2 | Swagger UI | `/docs` on six hosts (§2.2) | `Convey.WebApi.Swagger` package (not vendored) | §2.2 | High for the prefix; **not observed** for screens within it |

**Screen-to-capability count.** One repo-owned screen exists on this platform, and it covers one
capability (CAP-11). **Fifteen of the sixteen capabilities in `capability-baseline.md` have no
user interface anywhere in the workspace** (§12).

**No screen exists for the eleven business capabilities.** *Observed.* There is no sign-in screen,
no order screen, no parcel screen, no vehicle screen, no delivery screen, no customer screen, no
reservation screen, and no admin screen. This is stated as a positive observation about the asset
census, not as a judgement.

---

## 8 — API usage

Endpoint contracts are **not** re-derived here; they are cross-referenced to
`api-inventory.md`. The finding is narrow and precise.

### 8.1 — What Surface 1 actually calls

| Call | Kind | Target | Contract cross-reference | Evidence | Conf. |
|---|---|---|---|---|---|
| `new signalR.HubConnectionBuilder().withUrl('http://localhost:5005/pacco')` | SignalR hub connection | Hub `/pacco` on `operations-service` port 5005 | `api-inventory.md` §5.2 (SignalR hub surface); §4.5 (`operations-service`, port 5005) | `app.js:6-9` | High |
| `connection.invoke('initializeAsync', $jwt.value)` | Hub method invocation, client → server | `PaccoHub.InitializeAsync(string token)` | `api-inventory.md` §5.2 — the one client-callable hub method | `app.js:21`; `Operations.Api/Hubs/PaccoHub.cs` `public async Task InitializeAsync(string token)` | High |
| `connection.on('connected', …)` | Server → client message | `Clients.Client(Context.ConnectionId).SendAsync("connected")` | `api-inventory.md` §5.2 | `app.js:26`; `PaccoHub.cs` `ConnectAsync()` | High |
| `connection.on('disconnected', …)` | Server → client message | `SendAsync("disconnected")` | `api-inventory.md` §5.2 | `app.js:30`; `PaccoHub.cs` `DisconnectAsync()` | High |
| `connection.on('operation_pending', …)` | Server → client message | Emitted by `operations-service` on a `Pending` transition | `api-inventory.md` §5.2; `capability-baseline.md` CAP-11 | `app.js:34` | High |
| `connection.on('operation_completed', …)` | Server → client message | Emitted on a `Completed` transition | as above | `app.js:38` | High |
| `connection.on('operation_rejected', …)` | Server → client message | Emitted on a `Rejected` transition | as above | `app.js:42` | High |

The client subscribes to **five** of the five server-emitted messages catalogued in
`api-inventory.md` §5.2, and invokes the one client-callable method. The SignalR surface is
therefore **fully consumed** by this UI. Confidence: **high**.

### 8.2 — What Surface 1 does not call

*Observed, high confidence.* `app.js` and `index.html` contain **no `fetch`, no `XMLHttpRequest`, no
`$.ajax`, no `<form>`, no `<a href>` to any API, and no URL other than the hub URL**. Consequently,
of the surfaces catalogued in `api-inventory.md`:

| API surface (per `api-inventory.md`) | Count | Called by any UI asset in the workspace? |
|---|---:|---|
| `api-gateway` routes, synchronous config (§2) | 41 | **None** |
| `api-gateway` routes, asynchronous config (§3) | 41 | **None** |
| Service HTTP endpoints across the ten services (§4) | 35 business + 2 root | **None** |
| gRPC methods (§5.1) | 2 | **None** — the gRPC consumer is `Pacco.Services.Operations.GrpcClient`, a .NET console app, not a browser asset |
| SignalR hub method + server messages (§5.2) | 1 + 5 | **All six** — §8.1 |

**`GET /operations/{operationId}` is notably not called.** *Observed.* `api-inventory.md` §2.5 and
§4.5 document that endpoint as the poll-for-outcome path that pairs with the gateway's asynchronous
mode, where 20 write routes return only a correlation id (`api-inventory.md` §3). This UI never
polls it — it takes the push path exclusively. The two designed consumption models therefore have
**one browser client for the push half and no browser client for the poll half**. Recorded as an
observation, not a defect.

**Consequence for the platform's edge contract.** Every one of the 41 gateway routes, including the
whole sign-in and sign-up flow (`api-inventory.md` §2.4), has **no browser client in this
workspace**. The gateway's CORS configuration — `allowedOrigins: ['*']` with `allowCredentials:
true`, `allowedMethods: [post, put, delete]`, and `exposedHeaders: [Request-ID, Resource-ID,
Trace-ID, Total-Count]` (*observed*, `ntrada.yml:27-41`) — is a browser-facing configuration with no
observable browser consumer. Whether an unseen client exists is **unknown** and is carried as B1.

### 8.3 — Hub endpoint addressing

*Observed.* The hub URL in `app.js:7` is the hard-coded absolute literal
`'http://localhost:5005/pacco'`. It is not relative, not templated, not configured, and not injected.
Three consequences follow directly and are stated as evidence, not opinion:

1. The page works only when the browser resolves `localhost:5005` to the same
   `operations-service` the page came from — that is, the local development topology in
   `Pacco/services.yml`, where `operations-service` binds host port 5005 (`api-inventory.md` §4).
2. Under the Compose topology, `operations-service` listens on container port 80
   (`ENV ASPNETCORE_URLS http://*:80`, `Pacco.Services.Operations/Dockerfile:9`), so the hard-coded
   URL does not follow the container. Whether the published host mapping still lands on 5005 in
   every stack is **unknown** from the UI asset alone.
3. There is **no gateway route to `/ui` or `/pacco`** in any of the four `ntrada*.yml` files
   (*observed*, `api-inventory.md` §2 and §3 enumerate all 41 routes and none is either path), so
   the page is not reachable through the edge and its hub connection does not traverse the edge.
   `operations-service` declares **no `cors` section** in `appsettings.json` — a workspace-wide
   search for a `"cors"` key in every `appsettings*.json` returned zero files — which is consistent
   with the page and the hub being same-origin.

Confidence: **high** for the literal and for the absence of gateway routes and CORS config;
**medium** for the port-mapping consequence in point 2, which depends on the deployed stack.

---

## 9 — Auth and session handling

**Finding: manual bearer-token entry, passed as a hub method argument. No cookie, no session, no
token storage, no sign-in flow, no refresh.** Confidence: **high**.

| Dimension | Finding | Evidence | Conf. |
|---|---|---|---|
| How the token reaches the client | **It does not.** The user obtains a JWT out of band and **types or pastes it into a plain `<input type="text">`**. There is no sign-in screen, and `POST /identity/sign-in` (`api-inventory.md` §2.4) is never called from any UI asset | `index.html:14` `<input type="text" class="form-control" id="jwt" placeholder="JWT">`; §8.2 | High |
| How the token reaches the server | As the single string argument of the hub method: `connection.invoke('initializeAsync', $jwt.value)` — **in the message body, not in an `Authorization` header and not in the connection URL** | `app.js:21` | High |
| Client-side validation before send | One guard only: `if (!jwt || /\s/g.test(jwt)) { alert('Invalid JWT.'); return; }` — rejects empty or whitespace-containing input. It does not parse, decode, or check expiry | `app.js:12-16` | High |
| Server-side verification | `PaccoHub.InitializeAsync` calls `_jwtHandler.GetTokenPayload(token)`; a null payload or any thrown exception sends `disconnected`. On success it parses `payload.Subject` as a `Guid`, adds the connection to `subject.ToUserGroup()`, and sends `connected` | `Operations.Api/Hubs/PaccoHub.cs`, read in full | High |
| Per-user message scoping | Server-side, by SignalR group keyed on the JWT subject — so a client receives only its own operation transitions. The client does no filtering | `PaccoHub.cs` `Groups.AddToGroupAsync(Context.ConnectionId, group)` | High |
| Cookie / session | **None.** No cookie is read or written by any UI asset, and `api-inventory.md` §2 records that no route in either gateway configuration uses a session cookie or an API key anywhere on the platform | `app.js` in full; `api-inventory.md` §2 auth-column semantics | High |
| Token storage | **None.** No `localStorage`, `sessionStorage`, `document.cookie`, or in-memory persistence beyond the live DOM input value. Reloading the page loses the token | `app.js` in full | High |
| Token refresh | **None.** The client never calls `POST /refresh-tokens/use`, which `api-inventory.md` §4.4 records as having no gateway route in either configuration. With `jwt.expiryMinutes: 60` (`api-inventory.md` §4.4), a connection outliving the token has no observable renewal path in this UI | `app.js` in full; `api-inventory.md` §4.4 | High |
| Reconnection on token expiry | **Not observed.** `HubConnectionBuilder` is used without `withAutomaticReconnect()`, and no `onclose` handler is registered. What happens when the token expires mid-connection is **unknown** — it depends on `@aspnet/signalr` 1.1.0 behaviour and on server-side hub authorization, and the connection was authorized once at `initializeAsync` time rather than per message | `app.js:6-9`; `PaccoHub.cs` | Low |
| Auth context handoff from server to page | **None.** The static document carries no identity, no claims, no user object, and no anti-forgery token — it cannot, being a static file (§3) | `index.html` in full | High |

---

## 10 — CSS approach, design system and component catalogue

### 10.1 — CSS approach

**Finding: a single remote third-party CSS framework, consumed as a CDN theme. No authored CSS of
any kind.** Confidence: **high**.

| Candidate approach | Present? | Evidence |
|---|---|---|
| Plain CSS authored here | **No** | Zero `*.css` files in the workspace (§1.1) |
| SCSS / Sass / Less | **No** | Zero `*.scss`, `*.less` files; no compiler in any pipeline (§5) |
| CSS Modules | **No** | Requires a bundler; none exists (§5) |
| CSS-in-JS | **No** | `app.js` sets no `style` property, injects no `<style>`, and imports nothing |
| Tailwind | **No** | No `tailwind.config.*`, no `postcss.config.*`, no utility-class pattern in the markup (§1.5) |
| **Framework theme (CDN)** | **Yes — the only approach used** | `index.html:6` — Bootstrap 4.0.0 from `maxcdn.bootstrapcdn.com`, SRI-pinned |
| Inline `style=` attributes | **No** | No `style` attribute appears in `index.html` |
| Dynamic class construction | **Yes, one instance** | `app.js:51` builds `list-group-item-${type}` from the `type` argument, where callers pass `"primary"`, `"danger"`, `"light"`, or `"success"` (`:27,31,35,39,43`) — i.e. Bootstrap contextual-variant classes are selected at runtime |

**No custom theme, override, brand stylesheet, or CSS variable file exists.** The surface's entire
visual identity is stock Bootstrap 4.0.0 defaults.

### 10.2 — Shared component library

**None detected.** Confidence: **high** — proven by exhaustive search, not assumed.

*Observed.* Searched, per repository, across all fourteen clones:

| What was searched | Where | Result |
|---|---|---|
| Directories named `components/`, `ui/`, `shared/`, `lib/`, `design-system/` | At each repo root **and** recursively under every `src/` tree | Exactly one match: `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/` — which is not a component library but the single surface's asset root (§1.2). No repository contains `components/`, `shared/`, `lib/`, or `design-system/` |
| `package.json` dependencies or workspace packages named `*design-system*`, `*ui-kit*`, `*component-lib*`, `*component-library*` | All clones | **No `package.json` exists in any clone** (§1.5), so there is no dependency list and no workspace definition to hold such a package |
| Storybook config — `.storybook/`, `storybook.config.*`, `*.stories.*` | All clones, full tree walk | Zero matches |
| Reusable UI units in the code that does exist | `index.html`, `app.js` | **None.** The only repeated visual unit is the `<li class="list-group-item …">` string built inside `appendMessage` (`app.js:51`) — an inline template literal in one function in one file, with no parameterised component boundary, no registration, and no reuse from any second call site outside that function |

**Components identified: none.** There is no component to classify as globally shared,
surface-scoped, or domain-specific, and therefore no cross-domain import path to cite as evidence of
reuse. Recording a component here would require inventing one.

### 10.3 — Design tokens

**None detected.** Confidence: **high** — proven by exhaustive search.

*Observed.* Searched across all fourteen clones:

| Token artefact type | Filenames / patterns searched | Result |
|---|---|---|
| CSS custom properties | `variables.css`, `tokens.css`, any `*.css`, and the declaration patterns `--color-*`, `--spacing-*`, `--font-*`, `--radius-*`, `--size-*` | **Zero `*.css` files exist in the workspace** (§1.1), so no custom-property declaration can exist. The one stylesheet in use is remote (§10.1) |
| SCSS / Less variables | `_variables.scss`, `variables.less`, `theme.scss`, any `*.scss` / `*.less` | Zero files (§1.1) |
| JS / TS tokens | `tokens.ts`, `tokens.js`, `theme.ts`, `theme.js`, `design-tokens.*` | Zero files. The only `*.js` files in the workspace are the three listed in §1.1, none of which declares a token map |
| JSON tokens | `tokens.json`, `design-tokens.json` | Zero files |

The colour, spacing, typography and radius values the surface renders with are Bootstrap 4.0.0's
compiled defaults, fetched from a CDN. **No token is defined, overridden, or themed by Pacco.**

### 10.4 — Design-system summary

| Dimension | Finding | Conf. |
|---|---|---|
| Shared component library | **None detected** — paths searched listed in §10.2 | High |
| Components identified | **None** — §10.2 | High |
| Design tokens | **None detected** — paths and patterns searched listed in §10.3 | High |
| Storybook / component docs | **Not observed** — no `.storybook/`, no `*.stories.*` anywhere (§10.2) | High |
| Shared vs surface-specific split | **Not applicable.** One repo-owned surface exists; nothing is shared *between* surfaces because there is nothing to share (§2.4) | High |
| Packaging | **Not observed.** Not a standalone npm package, not a monorepo workspace, not co-located as a library. The three assets are loose files under one service's web root | High |
| Third-party design system in use | Bootstrap 4.0.0, CSS only, unmodified, CDN-hosted, SRI-pinned | High |

---

## 11 — Ownership, deployment boundary, routing model, asset loading, state management

### 11.1 — Frontend asset ownership

| Classification | Applies? | Evidence | Conf. |
|---|---|---|---|
| **Capability-scoped** | **Yes — this is the accurate label.** All three assets live inside the `operations-service` repository, under that service's own web root, and serve exactly one capability (CAP-11). No other capability's code references them | §1.1; §12 | High |
| Globally shared | **No.** Nothing outside `operations-service` loads, imports, or references these files. `operations-service` is the only host that serves static files at all (§5.3) | §5.3 | High |
| Cross-domain shared | **No.** The assets know about operation state and nothing else — no order, parcel, vehicle, delivery, or customer concept appears in them | `app.js`, `index.html` in full | High |
| Tenant-scoped | **No.** No tenant concept exists in the assets or anywhere in the platform's configuration | `app.js`, `index.html` in full | High |
| Layout-scoped | **No.** No layout exists (§7.1) | §7.1 | High |
| **Shared asset coupling** | **None observed.** There is no vendor bundle, layout shell, or stylesheet shared across surfaces or capabilities, so no coupling of that kind exists to document | §2.4 | High |

### 11.2 — Frontend deployment boundary

| Classification | Verdict | Evidence | Conf. |
|---|---|---|---|
| **Embedded within server-rendered runtime packaging** | **Yes — this is the accurate label.** The assets are published by `dotnet publish` into the `operations-service` output and baked into the `pacco.services.operations` container image. They ship when, and only when, that backend service ships | `Pacco.Services.Operations/Dockerfile` (`dotnet publish src/Pacco.Services.Operations.Api -c release -o out`, then `COPY --from=build /app/out .`); `Pacco.Services.Operations.Api.csproj` (`Microsoft.NET.Sdk.Web`); `Operations.Api/Infrastructure/Extensions.cs:88` | High |
| **Bundled with backend deployment** | **Yes — the same fact stated from the release side.** A change to `index.html` or `app.js` requires a rebuild and redeploy of the whole `operations-service` process. `scripts/dockerize.sh` builds and pushes one image; there is no separate frontend artifact to publish | `Pacco.Services.Operations/scripts/dockerize.sh`; `.travis.yml` | High |
| Independently deployable | **No.** There is no separate build, no separate artifact, no separate image, no separate pipeline stage, and no separate version. The frontend has no release identity of its own | §5.2; §5.3 | High |
| CDN-served | **Partially, for one third-party dependency only.** Bootstrap 4.0.0 is fetched from `maxcdn.bootstrapcdn.com` at page load. **No Pacco-authored asset is CDN-served**, and no asset-CDN configuration exists in any repository | `index.html:6`; absence of CDN config across all clones | High |
| Statically hosted | **No.** No object store, static-site host, or dedicated static server is configured anywhere. Serving is done in-process by ASP.NET Core static-file middleware | §5.3; `Pacco/compose/*.yml` | High |

**Deployment-boundary consequence, observed:** the UI's availability is exactly the availability of
`operations-service`. If that process is down, the page cannot be fetched *and* the hub cannot be
reached — there is no separate static host that would keep the shell serving.

### 11.3 — Routing model

| Dimension | Finding | Evidence | Conf. |
|---|---|---|---|
| **Route ownership** | The **backend host** owns the only UI route. The frontend owns no route | `Operations.Api/Infrastructure/Extensions.cs:88` | High |
| **Routing mechanism** | **Filesystem path mapping** by ASP.NET Core static-file middleware — the URL is the on-disk path under the web root. There is no route table, no route config, no route attribute, and no rewrite | `.UseStaticFiles()`; `wwwroot/ui/index.html` | High |
| **Server-side vs client-side** | **Entirely server-side.** No client-side router exists: no `pushState`, no `hashchange`, no `location` mutation, no route library, no route definition of any kind in `app.js` | `app.js` in full | High |
| **Centralized vs distributed** | **Distributed, and outside the edge.** Platform HTTP routing is centralized in the gateway's four `ntrada*.yml` files (`api-inventory.md` §2, §3), but the UI route is **not among the 41 routes** in either configuration — it exists only on the `operations-service` host. The one UI route is therefore governed by a different mechanism, in a different place, from every API route on the platform | `api-inventory.md` §2, §3; `ntrada*.yml` | High |
| **Route-to-capability mapping** | `/ui/index.html` on `operations-service` → **CAP-11**. That is the complete mapping | §12 | High |
| Deep linking / route parameters | **None.** One static path, no parameters, no query handling, no fragment handling | `app.js`, `index.html` in full | High |

### 11.4 — Frontend asset loading strategy

**Finding: template-driven inclusion of page-scoped local assets, mixed with one CDN-based
third-party stylesheet.** Confidence: **high**.

| Strategy | Present? | Where, and evidence |
|---|---|---|
| **Template-driven inclusion** | **Yes — the primary mode.** The entry document itself declares every asset, in fixed source order, with no bundler or asset-pipeline indirection: `<link>` in `<head>` (`index.html:6`), then `<script src="js/signalr.js">` and `<script src="js/app.js">` at the end of `<body>` (`:23-24`). Both script tags are render-blocking — no `defer`, no `async`. The relative `src` paths resolve against `/ui/`, so the surface is position-dependent on its own directory | High |
| **CDN-based loading** | **Yes, for one asset.** Bootstrap 4.0.0 from an absolute `https://maxcdn.bootstrapcdn.com` URL, SRI-pinned with `integrity` and `crossorigin="anonymous"` (`index.html:6`). This introduces a third-party runtime dependency on page load — if that host is unreachable, the page renders unstyled | High |
| **Page-scoped assets** | **Yes, trivially.** Both scripts load only on this one view. There is no layout to promote them to a global position, and no second view to scope them away from | High |
| **Globally bundled assets** | **No.** There is no vendor bundle and no common bundle produced or loaded by this platform. `signalr.js` is a bundle, but a **vendored third-party artifact** loaded by one page — not a platform-global bundle (§5.1) | High |
| **Lazy-loaded assets** | **No.** No dynamic `import()`, no deferred chunk, no conditional module load, no `IntersectionObserver`, no route-based split | High |
| **Runtime dynamic loading** | **No.** `app.js` creates no `<script>` element, sets no `src`, and fetches no remote module or loader manifest | High |
| **Mixed approaches** | **Yes — this is the honest summary.** Two local page-scoped scripts included directly by the document, plus one remote CDN stylesheet. Two loading modes, three assets, one page | High |

### 11.5 — Frontend state management

**No explicit frontend state management pattern detected.** Confidence: **high**.

*Observed.* Searched for and not found in any UI asset: Redux, Vuex, Pinia, NgRx, MobX, Zustand,
Recoil, a custom store or reducer, an event bus or pub/sub abstraction authored here, any global
`window.*` state assignment, `localStorage` / `sessionStorage` use, an observable, or a reactive
binding.

What exists instead, *observed*:

| Mechanism | Detail | Evidence |
|---|---|---|
| **The DOM is the store** | Message history is not held in any variable. `appendMessage` concatenates onto `$messages.innerHTML` with `+=`, so the accumulated `<ul>` markup **is** the application state. Nothing can read it back as data | `app.js:46-52` |
| **Three module-scope element references** | `$jwt`, `$messages`, `$connect` — element handles, not state | `app.js:3-5` |
| **One module-scope connection object** | The SignalR `connection`; its transport, retry and buffering state are internal to `@aspnet/signalr` 1.1.0 and are **not observable** from this repository | `app.js:6-9` |
| **The token lives only in the input's value** | Read fresh from `$jwt.value` at click time (`:12`) and again at invoke time (`:21`). It is never copied into application state | `app.js:12,21` |
| **Server-session-bound state** | The authoritative state is server-side: operation records in Redis under `requests:{id}` with a 300-second sliding expiry, and the client's SignalR group membership keyed on the JWT subject. The browser holds a projection of pushed messages only, with no reconciliation, no replay, and no catch-up on reconnect | `api-inventory.md` §2.5; `Operations.Api/Hubs/PaccoHub.cs` |
| **Inline hydrated state** | **None** (§3, §7.1) | `index.html` in full |

---

## 12 — Capabilities mapped

Mapping is to `capability-baseline.md` §1 and is made **only** where a UI asset is observably tied to
the capability.

| Capability | UI coverage in the workspace | Evidence | Conf. |
|---|---|---|---|
| **CAP-11 — Operation Status Projection & Real-Time Notification** | **Covered by Surface 1**, partially: the *push* half only. The console renders `operation_pending` / `operation_completed` / `operation_rejected` transitions live, and does **not** exercise the poll half (`GET /operations/{operationId}`, §8.2) | `app.js:34-44`; `capability-baseline.md` CAP-11 evidence list, which itself cites `wwwroot/ui/index.html` and `wwwroot/ui/js/app.js` | High |
| CAP-01 Identity & Access Management | **No UI.** No sign-in, sign-up, or user screen exists. The JWT is obtained out of band (§9) | §7.3; §8.2 | High |
| CAP-02 Edge Routing & Access Enforcement | **No UI**, and no UI asset routes through the gateway at all (§8.3) | §8.3 | High |
| CAP-03 Customer Profile & Lifecycle | **No UI** | §7.3 | High |
| CAP-04 Resource Availability & Reservation | **No UI** | §7.3 | High |
| CAP-05 Vehicle Fleet Catalogue | **No UI** | §7.3 | High |
| CAP-06 Parcel Catalogue & Volume Calculation | **No UI** | §7.3 | High |
| CAP-07 Order Lifecycle Management | **No UI** | §7.3 | High |
| CAP-08 Order Pricing & Discounting | **No UI** | §7.3 | High |
| CAP-09 Delivery Execution & Tracking | **No UI** | §7.3 | High |
| CAP-10 Automated Order Orchestration | **No UI** | §7.3 | High |
| CAP-12 Asynchronous Messaging & Event Distribution | **No UI.** Indirectly *visible* through Surface 1, since every operation transition it renders originates from a broker message, but no UI asset addresses the broker | `app.js:34-44` | High |
| CAP-13 Service Discovery & Load Balancing | **No UI.** The Consul console is vendor-supplied (§2.3) | §2.3 | High |
| CAP-14 Platform Observability | **No repo-owned UI.** Grafana, Jaeger and Seq consoles are vendor-supplied and excluded (§2.3) | §2.3 | High |
| CAP-15 Secrets & Service-Identity Management | **No UI.** The Vault console is vendor-supplied | §2.3 | High |
| CAP-16 Environment & Deployment Definition | **No UI** | §7.3 | High |

**Summary: 1 of 16 capabilities has a repo-owned user interface, and that one is covered
partially.** Confidence: **high**.

---

## 13 — Conflicts between sources

Source code is the authority throughout. Each item below states what a document claims, what the
code shows, and the resolution.

### 13.1 — Prior discovery documents: agreement, verified not assumed

`repo-inventory.md` §2.3 column 14 ("Frontend stack") was re-verified against the clones for this
baseline. **It agrees with the code on every repository**, and its `Pacco.Services.Operations` row
independently records the same three assets, the same absence of `package.json`, the same absence of
MFE, and the same Bootstrap-from-CDN fact found here. Likewise, `capability-baseline.md` CAP-11's
evidence list already cites `wwwroot/ui/index.html` and `wwwroot/ui/js/app.js`. **No conflict.**
Recorded explicitly because a verified agreement is evidence, not a silence.

### 13.2 — `Pacco/README.md` and `scripts/git-clone.sh` vs the discovery scope

- **The documents claim:** the platform consists of eleven repositories to clone.
  `Pacco/README.md:34-44` lists exactly eleven — APIGateway, Availability, Customers, Deliveries,
  Identity, Operations, OrderMaker, Orders, Parcels, Pricing, Vehicles.
  `Pacco/scripts/git-clone.sh:2` lists twelve: the same eleven plus `Pacco.APIGateway.Ocelot`.
- **The code and workspace show:** neither list contains `Pacco.Web`, yet `Pacco.Web` is in the
  fourteen-repository scope fixed by backlog issue 12998 and is cloned into this workspace. Its
  clone contains one file, `README.md`, whose whole content is `# Pacco.Web` (§1.4). Separately,
  `Pacco.APIGateway.Ocelot` — which the clone script *does* list — is **not** in this workspace at
  all.
- **Resolution:** the code wins. The platform as built has no web-client repository. `Pacco.Web` is
  labelled **Unverifiable — Missing Source Evidence**: this workspace cannot determine whether it is
  an abandoned placeholder or a pointer to a client held elsewhere. Carried as **B1**.
  `Pacco.APIGateway.Ocelot` carries no frontend implication and is noted only so the discrepancy is
  not silently dropped.

### 13.3 — `Pacco/README.md`'s product framing vs the observable UI

- **The document claims:** `Pacco/README.md` frames Pacco as a platform for "exclusive parcel
  delivery" built around "limited resources availability" — a customer-facing product framing, cited
  in `capability-baseline.md` CAP-04.
- **The code shows:** no customer-facing presentation asset exists. The only repo-owned UI is a
  diagnostic console for operation messages (§2.1), and no screen exists for any of the eleven
  business capabilities (§7.3, §12).
- **Resolution:** the code wins. Any customer-facing UI implied by that framing is
  **Future/Intended State (Not Implemented)** as far as this workspace can show — the README is a
  product description, not a claim about a delivered UI, and no repository contains such a UI.
  Carried as **Q3**.

### 13.4 — External catalogue content vs this platform

- **The catalogue returns:** the tenant corpus reachable from this workspace answers frontend and UI
  questions with material describing a **WebPT EMR scheduling SPA** and a micro-frontend
  modernization initiative for it.
- **The code shows:** no Pacco repository contains an EMR, a scheduler, an SPA, or any MFE
  mechanism (§6). The returned material names no Pacco service, capability, screen, route, or
  component.
- **Resolution:** the material describes a **different product** and is not evidence about Pacco. It
  is **not used** anywhere in this document, and specifically it is **not** treated as evidence that
  Pacco has an SPA or an MFE architecture. No Pacco frontend application node, screen, route,
  UI-to-service binding, or design-system reference was retrievable. Recorded so the retrieval
  outcome is auditable rather than invisible. Carried as **Q4**.

### 13.5 — `csproj` `Folder` item vs actual packaging behaviour

- **A plausible reading of the repo** is that
  `<ItemGroup><Folder Include="wwwroot\ui\js" /></ItemGroup>` in
  `Pacco.Services.Operations.Api.csproj` is what packages the UI.
- **The code shows:** a bare `Folder` item is an IDE directory-visibility hint and creates no
  content items. Publishing happens because the project declares
  `<Project Sdk="Microsoft.NET.Sdk.Web">`, whose default convention treats `wwwroot/**` as static
  web assets. The same `csproj` shows the contrasting *explicit* form for the one directory that
  genuinely needs it: `<Content Include="certs\**" CopyToPublishDirectory="Always" />`.
- **Resolution:** stated correctly in §5.3. Noted here because it is the kind of detail that would
  otherwise be mis-cited as an explicit frontend packaging step.

---

## 14 — UI architecture and evolution signals

Observed constraints only. This section describes frontend architectural pressure that the evidence
shows; it proposes no technology, no target architecture, and no modernization approach.

1. **No independently deployable UI boundary was observed.** The frontend has no build, no artifact,
   no version, and no pipeline stage of its own; it ships inside a backend service image and is
   released by that service's release (§5, §11.2). Frontend change and backend change are the same
   change event.

2. **UI availability is identical to one backend service's availability.** The shell and the live
   data channel are served by the same process on the same origin, so there is no configuration in
   which the page is reachable and the hub is not (§11.2).

3. **Rendering ownership sits entirely in the browser, but delivery ownership sits entirely in the
   backend.** The server contributes no markup, no state, and no template — it only serves bytes —
   while owning the route, the packaging, and the release (§3, §11.3). The two halves of the
   rendering boundary are governed by different mechanisms with no contract between them.

4. **The UI-to-backend binding is a hard-coded absolute URL, not configuration.** `app.js:7` pins
   `http://localhost:5005/pacco`. Changing environment requires editing and redeploying the asset;
   there is no environment variable, config file, or server-injected value in the path (§8.3). This
   is a source-level environment coupling, not a deployment-level one.

5. **The one UI route is outside the platform's centralized routing.** All 41 API routes are
   declared in the gateway's `ntrada*.yml`; the UI route exists only as a filesystem path on one
   service host and appears in no gateway config (§11.3). UI reachability and API reachability are
   governed by different mechanisms in different places.

6. **Frontend capability isolation is total, and so is the coverage gap.** One surface covers one
   capability, and fifteen capabilities have no UI (§12). There is no shared frontend surface across
   capabilities to decouple, and equally no existing UI foundation for capabilities that lack one.

7. **No frontend modularity exists to build on or to untangle.** No module system, no components, no
   design tokens, no shared library, no bundler (§4, §5, §10). The absence is symmetrical: nothing
   constrains change, and nothing supports it either.

8. **A third-party runtime dependency is on the page-load critical path.** Bootstrap 4.0.0 loads
   from `maxcdn.bootstrapcdn.com` at render time (§11.4), so the surface's appearance depends on an
   external host's availability and on that pinned version remaining published there.

9. **Two vendored assets are frozen at 2017–2019 versions with no upgrade mechanism.**
   `@aspnet/signalr` 1.1.0 and its inlined `es6-promise` 4.2.2 are committed as pre-built bytes with
   no manifest, no lockfile, and no dependency tooling (§4, §5). There is no declared version to
   bump and no pipeline that would rebuild them; neither referenced source map is committed, so the
   bundle is also not debuggable from this repository.

10. **Authentication state is manual and non-persistent, and the client has no renewal path.** The
    token is pasted by hand, held only in an input's value, never stored, and never refreshed, while
    the platform issues 60-minute access tokens and exposes no gateway route to the refresh endpoint
    (§9). Session continuity is not a property this surface has.

11. **The delivered state model is append-only DOM with no reconciliation.** Message history exists
    solely as concatenated markup (`innerHTML +=`), which cannot be read back as data, filtered,
    replayed, or reconciled after a reconnect; server-side operation records additionally expire on
    a 300-second sliding window (§11.5). Any UI behaviour that depends on history is constrained by
    both facts.

12. **Server-controlled message payloads are interpolated into markup.** `appendMessage` builds an
    `<li>` by string-concatenating `JSON.stringify(data)` and the caller-supplied `type` into
    `innerHTML` (`app.js:46-52`). The rendering path performs no escaping, so payload content is
    parsed as markup. Recorded as an observed characteristic of the rendering model.

13. **The push and poll consumption models have unequal client coverage.** The platform designs an
    asynchronous edge in which writes return a correlation id and outcomes are learned either by
    polling `GET /operations/{operationId}` or by SignalR push. Only the push path has a browser
    client; the poll path has none (§8.2).

14. **A browser-facing edge configuration exists with no observable browser consumer.** The gateway
    sets `allowedOrigins: ['*']` with `allowCredentials: true` and exposes correlation headers, yet
    no UI asset in the workspace calls any gateway route (§8.2). Whichever client that configuration
    was written for is not present here.

15. **Cross-surface relationships are minimal and incidental.** Surface 1 and Surface 2 share a
    release only where they co-reside in `operations-service`; they share no layout, asset, build
    output, or component (§2.4). Surface 2's entire content originates from a NuGet package, so its
    presentation cannot be inspected, versioned, or changed from this codebase (§2.2).

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The three files under `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/wwwroot/ui/` are the complete set of frontend assets for the Pacco platform | Proven exhaustively for the fourteen clones in this workspace (§1). The assumption is only that the workspace is the whole platform — that no UI is served from a repository, artifact store, or CDN we were not given | Every "no UI" statement in §7.3 and §12 would be wrong, and the platform would have an unexamined browser client consuming the gateway's `*`-origin CORS surface | Ask the platform owner to confirm the repository list is complete; check the Docker Hub `devmentors/pacco.*` image list for an image not built from any clone here |
| A2 | Surface 1's audience is developers or operators rather than end customers | Inferred from two observed facts: the user must paste a raw JWT by hand, and the screen exposes no domain action at all (§2.1, §9). No document states an audience | Access expectations for the page would be misjudged. It is served by a host with no gateway route and no CORS config, which is consistent with an internal-only audience but does not prove one | Ask whoever built `operations-service` who the page is for; check whether the `operations-service` port is exposed outside the private network in the deployed stack |
| A3 | `signalr.js` is the unmodified `@aspnet/signalr` 1.1.0 release bundle | Its embedded `VERSION = "1.1.0"`, package paths, and webpack UMD preamble all match that release (§4.1). It was not diffed byte-for-byte against the published artifact, because no lockfile or checksum is committed to diff against | A locally patched copy would carry behaviour no version bump would reproduce, and nothing records that it was patched | Diff the committed file against the published `@aspnet/signalr@1.1.0` `dist/browser/signalr.js` |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** The `Pacco.Web` repository is on the discovery scope list but contains only a one-line `README.md` — no code at all. We cannot tell from here whether a real Pacco web client exists in a repository we were not given, or whether this is an abandoned placeholder. The platform's own `README.md` and `scripts/git-clone.sh` do not list `Pacco.Web` at all (§13.2) | Completing the UI picture. If a real web client exists, this inventory is missing its main subject, and the gateway's 41 routes and `*`-origin CORS surface have a consumer nobody has examined | Platform owner | Someone must state plainly whether a Pacco web client exists. If it does, give us the repository and re-run this inventory against it. If it does not, drop `Pacco.Web` from the scope list so it stops reading as a gap | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** The Swagger `/docs` UI is enabled on six hosts including the public gateway, with `includeSecurity: true`. Its markup and scripts come from the `Convey.WebApi.Swagger` package, which is not in this workspace, so nobody here can see what it renders or which version it pins. Is it meant to be on in production? | It is a reachable browser surface on the platform's public edge whose content cannot be inspected or version-pinned from this codebase. Its presence is a deliberate config choice (`enabled: true`) that no document explains | Treat it as intentional for development. Confirm before any production exposure, since it advertises the full API surface at the edge | Platform owner / security reviewer |
| Q2 | **[ACTION NOW]** `app.js:7` hard-codes `http://localhost:5005/pacco`, but under Compose the service listens on container port 80. Does the page actually work in any deployed environment, or only when the services are run directly on a developer's host? | If it works only on a developer's machine, the one UI the platform has is a local-only tool, and treating it as a deployed surface downstream would be wrong | Likely local-development only: nothing rewrites the URL at build or serve time, and no gateway route exposes the page (§8.3) | Whoever maintains `operations-service` |
| Q3 | **[handled later by architecture_evolution_validation]** `Pacco/README.md` describes a customer-facing parcel-delivery product, but no customer-facing screen exists anywhere. Was a customer UI ever built, or was the platform always backend-only? | It decides whether "no UI for eleven business capabilities" is a gap to close or simply the platform's intended shape. Downstream scoping of any frontend work turns on this | The README is a product description, not a claim about delivered software. On the evidence here the platform is backend-only, and any customer UI is Future/Intended State (Not Implemented) | Product owner, with the architecture stage recording the verdict |
| Q4 | **[ACTION NOW]** The knowledge catalogue answers Pacco frontend questions with material about a WebPT EMR scheduling SPA and its micro-frontend programme — a different product entirely. Is Pacco frontend knowledge missing from the catalogue, or is the catalogue scoped to a different project than this workspace? | If a later stage queries the same catalogue and does not notice the mismatch, it may attribute an SPA and an MFE architecture to Pacco, which the code flatly contradicts (§6, §13.4) | The tenant catalogue holds no Pacco frontend content. Until it does, frontend statements about Pacco must come from the repositories only | Whoever owns the catalogue's tenant scoping |
| Q5 | **[handled later by architecture_evolution_validation]** `@aspnet/signalr` 1.1.0 (2019) and `es6-promise` 4.2.2 (2017) are committed as pre-built bytes with no manifest, lockfile, or tooling. How should these vendored assets be tracked for currency? | There is nothing to bump and no pipeline that would rebuild them, so a security advisory against either has no mechanism to act on. Both sit on the runtime critical path (§14.9) | None proposed — recording the constraint is this stage's job; choosing a handling approach is not | Architecture stage, with the `operations-service` maintainer |
| Q6 | **[ACTION NOW]** `appendMessage` interpolates `JSON.stringify(data)` and a caller-supplied `type` into `innerHTML` with no escaping, so server-pushed operation payloads are parsed as markup (§14.12). Is that payload content trusted end to end? | Operation payloads originate from broker messages published by eight services and from user-supplied command fields, so what reaches this rendering path is not necessarily operator-authored | Confirm whether the page's audience and network exposure make this acceptable, or whether the rendering path needs attention. This stage records the observation only | `operations-service` maintainer / security reviewer |
| Q7 | **[ACTION NOW]** The gateway sets `allowedOrigins: ['*']` with `allowCredentials: true` and exposes correlation headers, yet no UI asset in this workspace calls any of its 41 routes (§8.2). Which browser client was that written for? | A permissive browser-facing CORS policy with no identified consumer is either serving a client nobody has seen (which reopens B1) or is configured for a client that does not exist | Most likely written speculatively for a web client that was never built — consistent with the empty `Pacco.Web`. Resolving B1 resolves this too | Platform owner |
