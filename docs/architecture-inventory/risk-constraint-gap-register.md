# Pacco — Risk, Constraint & Gap Register

| Field | Value |
|-------|-------|
| Project | Pacco |
| Document role | The platform's single living queue of open architecture risks, binding constraints and evidenced gaps. Items are opened here, resolved here, and pruned from here |
| First authored | 2026-09-22, `architecture_evolution_generation`, work item 13652 |
| Base ref of all cited source | `feature/13652/aidlc` |
| Scope of the current FMEA | The architecture authored for work item 13652 — `ADR-021` and the design it records. Pre-existing platform constraints are carried in §3 and are not re-scored here unless `ADR-021` changes their exposure |

## How this register works

This file is **not** a per-ticket artifact and it is never copied per work item. It is one shared
queue with a lifecycle:

1. **Open** — an item is added with an id that is never reused.
2. **Resolve** — when an item is genuinely closed, its row is rewritten in place with
   `**RESOLVED <date> by <stage or person>**` and a sentence saying what closed it. It stays visible
   for one revision so a reader can see the disposition.
3. **Prune** — a resolved item is deleted at the next revision that touches this file. A deleted item
   leaves no id gap behind in the numbering.

Every row states, in plain words, **what it is · who must act · what breaks if it is ignored · and
whether it is needed now or later**, and carries **[ACTION NOW]** or **[handled later by \<stage\>]**.

## Contents

1. [Architecture Risk Assessment — FMEA](#1-architecture-risk-assessment--fmea)
2. [Risk detail](#2-risk-detail)
3. [Binding constraints](#3-binding-constraints)
4. [Evidenced gaps](#4-evidenced-gaps)
5. [Resolved, awaiting prune](#5-resolved-awaiting-prune)

---

## 1. Architecture Risk Assessment — FMEA

Scoring is the standard FMEA scale, 1 to 10 on each axis, `RPN = S × O × D`.

- **S — Severity.** How bad the effect is if the failure happens.
- **O — Occurrence.** How likely the failure mode is, given the design as recorded.
- **D — Detection.** How hard it is to notice **before** it causes harm. A failure that breaks
  loudly and immediately scores **low**. A failure that is silent, or that only shows up in
  production, scores **high**.

`RPN` ranks attention, it does not decide acceptance. `R-03` is the clearest example: it carries the
highest RPN in this register and is **deliberately accepted**, because removing it was placed out of
scope by an explicit reviewer decision.

| id | title | category | S | O | D | RPN | mitigation type | owner stage | residual |
|----|-------|----------|---|---|---|-----|-----------------|-------------|----------|
| `R-03` | Client-side logout leaves the issued token valid at the edge | security | 7 | 10 | 8 | **560** | accept | Platform owner — reopen only as a separate edge decision | **High — accepted** |
| `R-13` | The JWT signing key is committed in the repository, and the browser client makes the edge it protects end-user facing | security | 9 | 5 | 7 | **315** | transfer | Platform owner | High |
| `R-02` | The gateway returns downstream exception messages to the browser | security | 6 | 7 | 6 | **252** | mitigate — client-side | `DO1` implementation | Low once mitigated |
| `R-09` | No frontend standard exists, so the first client sets platform conventions by accident | governance | 4 | 9 | 7 | **252** | mitigate — write the minimum set | Platform architect | Medium |
| `R-12` | No repository has an owner, so the gateway's public-contract change has no reviewer | governance | 6 | 8 | 5 | **240** | mitigate — name owners | Platform owner | Medium |
| `R-08` | Credentials or tokens leak into browser storage, logs or URLs | security | 9 | 4 | 6 | **216** | mitigate — client-side | `DO1` implementation | Low once mitigated |
| `R-10` | The refresh token cannot be redeemed at the edge, so sessions end abruptly at expiry | usability | 5 | 9 | 4 | **180** | accept for now, decide later | Platform owner | Medium |
| `R-05` | The exact CORS origin is applied to a gateway configuration file the environment does not load | operability | 7 | 5 | 4 | **140** | mitigate — record the mapping | Platform owner | Low once recorded |
| `R-06` | An unknown or unsupported role renders the admin landing message | correctness | 9 | 3 | 5 | **135** | mitigate — closed-vocabulary check | `DO2` implementation | Low once mitigated |
| `R-01` | The exact `Pacco.Web` local origin is not fixed, so no browser call succeeds | operability | 8 | 6 | 2 | **96** | mitigate — fix the value | Platform owner with the `Pacco.Web` implementer | Low |
| `R-07` | The landing page is reachable without an authenticated session | security | 6 | 4 | 4 | **96** | mitigate — client-side guard | `DO2` implementation | Medium — see detail |
| `R-14` | The gateway library's actual CORS behaviour is unverified, because its source is not in the workspace | operability | 7 | 4 | 3 | **84** | mitigate — observe it | Platform owner | Low |
| `R-04` | The edge's allowed-method list omits `get`, blocking the first browser read surface | operability | 5 | 4 | 3 | **60** | defer | Platform architect | Low |
| `R-11` | A duplicate sign-in submission is accepted while a request is in flight | correctness | 4 | 5 | 3 | **60** | mitigate — client-side | `DO1` implementation | Low once mitigated |

---

## 2. Risk detail

Each entry names the failure mode, its effects, its causes, the mitigation, the ADR that governs it,
and what must be verified before the risk can be called closed.

### `R-03` — Client-side logout leaves the issued token valid at the edge · **[ACTION NOW — as an acceptance, not a fix]**

- **What it is.** Logout clears the browser's copy of the access token. It does not invalidate the
  token. Anyone holding a copy — a browser extension, a proxy log, a screenshot of a network tab, a
  shared machine — can keep using it against the gateway and every domain service until it expires.
- **Failure mode.** A token survives the session that created it.
- **Effects.** A user who believes they have logged out has not, from the platform's point of view.
  On a shared machine this is a real access-control failure, not a theoretical one.
- **Causes.** `ADR-007` obligation 4: only `identity-service` consults the revocation store, and its
  three token-management routes have **no gateway route in any of the four configurations**. There
  is no mechanism at the edge to revoke against.
- **Affected.** `Pacco.Web` (CAP-17), `api-gateway` (CAP-02), every authenticated route.
- **Mitigation · type `accept`.** Accepted by explicit reviewer decision: `DO2` holds the
  authentication and authorization backend out of scope, and exposing revocation at the edge would
  turn a login feature into a gateway architecture change. The mitigation is honesty — `ADR-021` §5
  rule 5 and §6.2 item 1 state the limitation in the record rather than implying it is solved.
- **ADR.** `ADR-021` §5 rule 5. Reversing it requires a **new** ADR that changes `ADR-007`'s
  revocation position — not an amendment to `ADR-021`.
- **Who must act.** Platform owner, if and when token revocation at the edge becomes in scope. **No
  action is required to ship `DO1` and `DO2`.**
- **Must verify.** That the limitation is visible to whoever accepts it: capture a token, log out,
  replay it against an authenticated route, and confirm it still succeeds. The test is there to keep
  the residual risk measured, not to pass.
- **Residual: High, accepted.**

### `R-13` — The committed signing key, now protecting an end-user-facing edge · **[ACTION NOW]**

- **What it is.** A symmetric `issuerSigningKey` is committed in all four `ntrada*.yml` files and in
  `Pacco.Services.Operations.Api/appsettings.json`. Anyone with repository access can mint a token
  the gateway accepts. This predates `ADR-021` — what changes is the exposure: the edge it protects
  stops being machine-only and starts carrying end-user sessions.
- **Failure mode.** A forged token is accepted as an authenticated user.
- **Effects.** Complete bypass of authentication and of every claim gate, for any route.
- **Causes.** The key was committed rather than injected. `ADR-006` obligation 4 already records
  that it must leave source control.
- **Affected.** `api-gateway` (CAP-02), `identity-service` (CAP-01), every service behind the edge.
- **Mitigation · type `transfer`.** Move the key to the platform's existing secret mechanism (Vault
  is already in the stack for database credentials) and rotate it. This is `ADR-006`'s obligation,
  not a new one, and `ADR-021` does not discharge it.
- **ADR.** `ADR-006` obligation 4. `ADR-021` §6 does not change it.
- **Who must act.** Platform owner. Not a blocker for `DO1` or `DO2` in local development, and a
  blocker for any non-local environment.
- **Must verify.** That no key material remains in any `ntrada*.yml` or `appsettings.json`, and that
  the gateway still validates tokens after the key is injected rather than committed.
- **Residual: High** until the key is moved.

### `R-02` — The gateway returns downstream exception messages to the browser · **[ACTION NOW]**

- **What it is.** `extensions.customErrors.includeExceptionMessage` is `true` in all four gateway
  configurations (`ntrada.yml:24-25` and the three siblings), so when a downstream service throws,
  the exception message travels to the caller. `DO1` targets **zero raw backend exceptions or stack
  traces shown to users**, so this is directly load-bearing on an acceptance target.
- **Failure mode.** A backend exception message is rendered to an end user.
- **Effects.** `DO1`'s target is missed. Exception text can disclose internal structure — service
  names, validation internals, and on a database or serialization error, field names.
- **Causes.** An edge configuration value set for machine callers, unchanged now that the caller is
  a browser.
- **Affected.** `Pacco.Web` Login screen (CAP-17), `api-gateway` (CAP-02).
- **Mitigation · type `mitigate`, client-side.** `Pacco.Web` maps every non-success response to a
  fixed non-technical message and **never renders a response body verbatim**. This is the mitigation
  `ADR-021` §8 `N2` requires. Turning `includeExceptionMessage` off at the edge would be a broader
  change affecting every existing machine caller, and is **not** proposed here.
- **ADR.** `ADR-021` §8 `N2`. The edge setting itself is governed by `ADR-004`.
- **Who must act.** The `DO1` implementer, in the Login screen's error handling.
- **Must verify.** Force invalid credentials, a stopped `identity-service`, and a malformed request.
  Assert the fixed message renders in all three and that the raw body appears nowhere in the DOM.
- **Residual: Low** once the client-side mapping exists. The edge setting remains as it is.

### `R-09` — No frontend standard exists, so the first client sets conventions by accident · **[ACTION NOW]**

- **What it is.** The platform has no frontend standard of any kind: no state-ownership rule, no
  accessibility level, no error-presentation rule, no client logging or redaction rule, no dependency
  or lockfile policy. `ADR-021` §7.1 records this rather than importing an outside convention.
- **Failure mode.** Whatever the Login screen happens to do becomes the platform's convention.
- **Effects.** The second and third surfaces inherit accidents. Accessibility in particular is very
  expensive to retrofit and nearly free to build in.
- **Causes.** No client ever existed, so no standard was ever needed.
- **Affected.** CAP-17 and every future browser surface.
- **Mitigation · type `mitigate`.** Write the minimum set **as the client is built**, not after:
  accessibility level, error-presentation rule, no-credential-logging rule, dependency policy.
- **ADR.** `ADR-021` §7.1 and `Q4`, FA6 for the related design-system question.
- **Who must act.** Platform architect, with the `DO1` implementer.
- **Must verify.** That the four rules above exist in writing somewhere the platform can cite before
  a second surface is started.
- **Residual: Medium** until written.

### `R-12` — The gateway's public-contract change has no reviewer · **[ACTION NOW]**

- **What it is.** `ADR-004` §2 obligation 1 makes the gateway configuration a reviewed architectural
  artifact: a change to it changes the platform's public contract. The CORS change `ADR-021` requires
  is exactly such a change. **No repository in the workspace has a `CODEOWNERS`, a contributing guide
  or any team metadata**, so there is no named reviewer to satisfy that obligation.
- **Failure mode.** A public-contract change ships unreviewed, and `ADR-021` stays `Proposed` forever.
- **Effects.** The governing obligation is unenforceable in practice. Every ADR in the corpus —
  all twenty-one — is blocked from moving past `Proposed` for the same reason.
- **Causes.** No ownership metadata has ever existed. Carried as `adr-candidates.md` B4 since the
  ADR corpus was written.
- **Affected.** `api-gateway` (CAP-02), `Pacco.Web` (CAP-17), the whole ADR corpus.
- **Mitigation · type `mitigate`.** Name an owner for `Pacco.APIGateway` and for `Pacco.Web`. Two
  names unblock this.
- **ADR.** `ADR-004` §2 obligation 1, `ADR-021` B3, `adr-candidates.md` B4.
- **Who must act.** Platform owner. Needed **before** `ADR-021` can be approved.
- **Must verify.** That a named person or team is recorded against both repositories.
- **Residual: Medium.**

### `R-08` — Credentials or tokens leak into browser storage, logs or URLs · **[handled later by `DO1` implementation]**

- **What it is.** `DO1` targets zero password values logged or displayed and zero credentials
  hard-coded in the client. Browsers make all three leak paths easy: a console log, a token in
  `localStorage`, a credential in a query string that lands in history and in any intermediary's logs.
- **Failure mode.** A password or token is written somewhere it outlives the request.
- **Effects.** Credential disclosure. Combined with `R-03`, a leaked token is usable until it expires
  with no way to revoke it.
- **Causes.** No client logging or redaction rule exists (`R-09`).
- **Affected.** `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** The password is read from the form, sent once in the request
  body, and never persisted, logged, or placed in a URL. No credential is compiled into the client.
- **ADR.** `ADR-021` §8 `N1`.
- **Who must act.** The `DO1` implementer.
- **Must verify.** Sign in with a known password and inspect the console, the network tab, browser
  storage and the built bundle for the literal value. Grep the bundle for any credential literal.
- **Residual: Low** once verified.

### `R-10` — The refresh token cannot be redeemed at the edge · **[ACTION NOW — as a decision, not a fix]**

- **What it is.** Sign-in returns a `refreshToken` in `AuthDto`, but `POST refresh-tokens/use` has no
  gateway route in any of the four configurations. The client receives a token it has no way to use.
- **Failure mode.** The session ends the moment the access token expires, mid-task, with no renewal.
- **Effects.** Acceptable for two screens. Not acceptable for a working application, and the longer
  it goes unstated the more likely a later surface assumes renewal exists.
- **Causes.** The token-management routes were never exposed at the edge (GAP-21).
- **Affected.** `Pacco.Web` (CAP-17), `api-gateway` (CAP-02), `identity-service` (CAP-01).
- **Mitigation · type `accept` now, `decide` later.** `DO1` and `DO2` do not need renewal. What is
  needed now is a **stated position** — either route the refresh endpoint in a later record, or state
  that Pacco sessions end at access-token expiry by design.
- **ADR.** `ADR-021` §6.3 item 3 and `Q1`.
- **Who must act.** Platform owner — a decision, before the third browser surface, not before `DO1`.
- **Must verify.** That the position is written down, whichever way it goes.
- **Residual: Medium** until stated.

### `R-05` — The origin is applied to a configuration file the environment does not load · **[ACTION NOW]**

- **What it is.** Four `ntrada*.yml` files exist with identical CORS blocks, selected by the
  `NTRADA_CONFIG` environment variable. Nobody has recorded which file each environment loads
  (`ADR-004` B2). Under a wildcard this was invisible. Under an exact origin it decides whether the
  browser works.
- **Failure mode.** The exact origin lands in a file the running gateway is not reading.
- **Effects.** All browser calls are refused cross-origin. It fails loudly and early, which is why
  `D` is low, but it is easy to misdiagnose as a client bug.
- **Causes.** Four configurations, no recorded environment mapping.
- **Affected.** `api-gateway` (CAP-02), `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** Apply the change to **all four** files — they are identical today
  and `ADR-021` §5 rule 4 keeps them identical — and record the environment-to-file mapping.
- **ADR.** `ADR-004` §2 obligation 2, `ADR-021` FA2 and B2.
- **Who must act.** Platform owner. Compose loads `ntrada-async.docker.yml`
  (`compose/services.yml:4-13`), which settles the local case and not the others.
- **Must verify.** Diff the CORS block across all four files after the change, and confirm the
  running gateway's loaded configuration.
- **Residual: Low** once recorded.

### `R-06` — An unknown role renders the admin landing message · **[handled later by `DO2` implementation]**

- **What it is.** `DO2` targets zero unknown-role or normal users shown `Welcome to Admin Area`. The
  role vocabulary is closed and lower-cased — `Role.cs` defines exactly `user` and `admin` and
  validates case-insensitively — so anything else is unknown.
- **Failure mode.** A default-to-admin branch, a case-sensitivity slip, or a role inferred from an
  email address or username shows the admin message to a non-admin.
- **Effects.** A user is told they are in an admin area. The gateway's claim gates still refuse the
  admin routes, so this misleads rather than escalates — which is why `O` is low and `S` is high.
- **Causes.** Role comparison written permissively, or a fallback branch that treats unknown as admin.
- **Affected.** `Pacco.Web` landing page (CAP-17).
- **Mitigation · type `mitigate`.** Compare against the closed vocabulary explicitly. Unknown renders
  the non-admin message. Never infer a role from an identifier.
- **ADR.** `ADR-021` §8 `N4` and §5.2.
- **Who must act.** The `DO2` implementer.
- **Must verify.** Sign in as `admin`, as `user`, and with a token carrying an unrecognised role.
  Assert the message in each case, and repeat after a reload and a logout/login cycle.
- **Residual: Low** once verified.

### `R-01` — The exact `Pacco.Web` local origin is not fixed · **[ACTION NOW]**

- **What it is.** `ADR-021` §5 rule 4 replaces the wildcard with `Pacco.Web`'s exact local origin.
  That origin — scheme, host and port — has not been chosen. `http://localhost:3000` is the intent's
  example, not a committed value.
- **Failure mode.** The allow-list names an origin the client does not serve on.
- **Effects.** Every credentialed browser call is refused. `DO1` cannot complete a sign-in round trip.
- **Causes.** The client's local runtime has not been chosen yet.
- **Affected.** `api-gateway` (CAP-02), `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** Fix the port when the client's local runtime is chosen, and apply
  it to all four files in one change together with `R-05`.
- **ADR.** `ADR-021` §5 rule 4, FA1, B1.
- **Who must act.** Platform owner with the `Pacco.Web` implementer, before `DO1` is exercised end to
  end.
- **Must verify.** A credentialed `POST /identity/sign-in` preflight from the allowed origin returns
  that origin in `Access-Control-Allow-Origin`, and one from a different origin does not.
- **Residual: Low** — it fails loud, immediately, at the browser.

### `R-07` — The landing page is reachable without an authenticated session · **[handled later by `DO2` implementation]**

- **What it is.** `DO2` targets zero unauthenticated requests reaching the landing page. The guard is
  a client-side route guard, because the landing page is a client-rendered screen with no server
  route behind it.
- **Failure mode.** A direct request, a reload, or a back-navigation after logout renders the landing
  screen without a session.
- **Effects.** The screen shows. **It shows nothing privileged** — `DO2` introduces no dashboard
  widgets and no business functionality, and any real data would come from a gateway call the missing
  token would refuse. The harm is a broken guarantee, not a data disclosure.
- **Causes.** A guard applied on navigation but not on direct load or on reload.
- **Affected.** `Pacco.Web` (CAP-17).
- **Mitigation · type `mitigate`.** Guard on every entry into the route — navigation, direct load,
  reload and back-navigation — and redirect to Login. An expired token takes the same path with a
  session-expired message.
- **ADR.** `ADR-021` §8 `N3`.
- **Who must act.** The `DO2` implementer.
- **Must verify.** Request the landing route with no session, with a cleared session after logout,
  and with an expired token. Assert the redirect in all three and that it survives a reload.
- **Residual: Medium** by nature: a client-side guard is a UX guarantee, not a security boundary. The
  security boundary stays at the edge, where `ADR-006` puts it. Any future landing screen that shows
  real data must get that data through an authenticated gateway call, which the edge enforces
  independently of this guard.

### `R-14` — The gateway library's actual CORS behaviour is unverified · **[ACTION NOW]**

- **What it is.** Ntrada is a NuGet reference; its source is not in the workspace. That
  `extensions.cors.allowedOrigins` is an exact-match allow-list which echoes the matched origin with
  `Access-Control-Allow-Credentials: true` is read from the key names, not proven from code. The same
  limitation applies to today's wildcard-plus-credentials combination: nobody has observed whether
  the library accepts it, rejects it, or silently rewrites it.
- **Failure mode.** The configuration means something other than what it reads as.
- **Effects.** Either the browser path fails after the change, or the edge stays permissive after the
  wildcard is removed — the second being the dangerous one, because it is silent.
- **Causes.** A third-party configuration dialect with no source in the workspace (`ADR-004` A1).
- **Affected.** `api-gateway` (CAP-02).
- **Mitigation · type `mitigate`.** Observe it. Run the gateway with the exact origin and compare the
  response headers for an allowed and a disallowed origin. Record what the library actually does.
- **ADR.** `ADR-021` A1, FA3.
- **Who must act.** Platform owner, when `R-01` is applied.
- **Must verify.** `Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials` on a
  credentialed preflight from both origins.
- **Residual: Low** once observed.

### `R-04` — The allowed-method list omits `get` · **[handled later by HLS]**

- **What it is.** `extensions.cors.allowedMethods` lists `post`, `put` and `delete` only, in all four
  configurations. No cross-origin `GET` is permitted through the edge.
- **Failure mode.** The first browser surface that reads through the gateway is blocked.
- **Effects.** None for `DO1` or `DO2` — their only platform call is `POST /identity/sign-in`. A
  later read surface fails at the browser, loudly.
- **Causes.** A method list written for machine callers that never exercised browser CORS.
- **Affected.** `api-gateway` (CAP-02), any future browser read surface.
- **Mitigation · type `defer`.** Widen it in the same change that adds the first browser read
  surface. Widening it speculatively now enlarges the edge's accepted surface for no delivered need.
- **ADR.** `ADR-021` §6.2 item 5 and `Q2`.
- **Who must act.** Platform architect, when the first browser read surface is designed. **No action
  now.**
- **Must verify.** That whoever designs that surface knows this, which is what this row is for.
- **Residual: Low.**

### `R-11` — A duplicate submission is accepted while a request is in flight · **[handled later by `DO1` implementation]**

- **What it is.** `DO1` targets zero duplicate login submissions accepted while a request is in
  progress.
- **Failure mode.** Repeated clicks or an Enter key held down send several concurrent sign-in
  requests.
- **Effects.** Multiple tokens minted for one intent, multiple refresh tokens created, a confusing
  race between responses. `IdentityService.SignInAsync` has no idempotency guard and does not need
  one — the fix belongs in the client.
- **Causes.** A submit control left enabled during an in-flight request.
- **Affected.** `Pacco.Web` Login screen (CAP-17), `identity-service` (CAP-01).
- **Mitigation · type `mitigate`.** Disable submission for the duration of the request and show a
  processing state.
- **ADR.** `ADR-021` §8 `N7`.
- **Who must act.** The `DO1` implementer.
- **Must verify.** Submit repeatedly against a slowed sign-in response and assert exactly one request
  leaves the browser.
- **Residual: Low** once verified.

---

## 3. Binding constraints

Constraints are not risks. They are properties of the platform that any design must work within.
They are listed here so a designer does not have to rediscover them.

| id | Constraint | Source | What it forbids |
|----|-----------|--------|-----------------|
| `C-01` | Authentication is enforced at the edge. Services behind it trust the caller context they are given and re-check only resource ownership | `ADR-006` §2 | A client that authenticates on its own, or a browser path that bypasses the gateway |
| `C-02` | The gateway configuration is a reviewed architectural artifact. A change to it changes the platform's public contract | `ADR-004` §2 obligation 1 | Treating the CORS origin change as an operations edit |
| `C-03` | Response aggregation and per-client shaping must not be added to the gateway configuration. A client needing either gets a backend-for-frontend service and its own record | `ADR-004` §2 obligation 3 | Stretching the edge to shape responses for the browser |
| `C-04` | Revocation is consulted only by the issuing service, and its routes are not exposed at the edge | `ADR-007` §2 obligation 4 | A logout that claims to invalidate a token |
| `C-05` | Repository per component, with independent per-repository release | `ADR-018` | A client bundled into a backend image or onto a backend's release path |
| `C-06` | Deployment is Docker Compose stacks and PM2 manifests only. No Kubernetes, Helm or Terraform exists | `ADR-017` | Designing a frontend deployment pipeline against a target that does not exist |
| `C-07` | The role vocabulary is the closed, lower-cased set `user` and `admin`, validated case-insensitively | `Identity.Core/Entities/Role.cs` | Any third role, and any role inferred from an identifier |
| `C-08` | Every .NET deployable is pinned to `.NET Core 3.1`, which is out of support | `ADR-020` | Assuming a browser client inherits a platform runtime baseline — it does not |
| `C-09` | There are currently no Dev, QA, Staging or Production frontend environments | `ADR-021` §5 rule 6, reviewer decision | Pre-defining environment origins, DNS names or deployment targets now |

---

## 4. Evidenced gaps

A gap is a missing input, not a risk to score. Each one blocks a decision somebody will have to make.

| id | Gap | Why it matters | Who must supply it | Needed |
|----|-----|----------------|--------------------|--------|
| `G-01` | **[ACTION NOW]** No repository has an owner — no `CODEOWNERS`, no contributing guide, no team metadata anywhere in the fourteen clones | Nothing can be approved. All twenty-one ADRs are stuck at `Proposed`, and `C-02`'s review obligation is unenforceable | Platform owner | Now — it blocks `ADR-021`'s approval |
| `G-02` | **[ACTION NOW]** No environment-to-gateway-configuration mapping is recorded | An exact CORS origin must land in the file the running gateway loads (`R-05`) | Platform owner | Now — before `DO1` is exercised end to end |
| `G-03` | **[ACTION NOW]** No availability target, latency budget or error-rate objective is documented for any service, for the gateway, or for the platform | A client cannot justify a timeout, a retry policy or a loading-state threshold against nothing. Whatever `DO1` picks becomes the de facto target | Platform owner | Now, if the client is to have a justified timeout. Otherwise the first value chosen becomes the standard by accident |
| `G-04` | **[ACTION NOW]** No frontend standard of any kind exists — state ownership, accessibility, error presentation, client logging and redaction, dependency and lockfile policy | The first client sets all of them by accident (`R-09`) | Platform architect | Now — cheapest while the first screens are being written |
| `G-05` | **[handled later by the design-system decision]** `STYLE_README.md` and `pacco-material-you.css` have no versioning, no distribution mechanism and no consumer contract | A second client would copy them or force a design-system decision under time pressure | Platform architect | Later — forced by a second consumer, not before (`ADR-021` FA6) |
| `G-06` | **[handled later by HLS]** The message path into `identity-service` has no evidenced producer: `Program.cs` calls `SubscribeCommand<SignUp>()`, but no gateway route publishes to the `identity` exchange and no service publishes `sign_up` | Pre-existing (`architecture-views.md` GAP-4). It does not affect `DO1` or `DO2`, whose sign-up path is not in scope, but it is the kind of thing a client team will trip over | Platform owner | Later |

---

## 5. Resolved, awaiting prune

**None.** This register was first authored on 2026-09-22 and no item has yet been resolved. The
first revision that closes an item records it here with its closing date and the stage or person that
closed it, and the row is deleted at the revision after that.
