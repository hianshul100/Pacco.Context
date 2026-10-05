# Repository: `Pacco.Web`

`Pacco.Web` is the Pacco platform's **standalone browser client**. It owns the browser presentation and browser-session experience currently implemented for sign-in and the role-aware Welcome/Landing screen.

- **Repository:** `Pacco.Web`, path: `/`
- **Base ref analysed:** `master`
- **Commit analysed:** `697750256b7c211ce31943bab2a4192184c4f0ec`
- **Local browser origin:** `http://localhost:5173`
- **Gateway:** `http://localhost:5000`

---

## README vs repository

`README.md` correctly identifies `Pacco.Web` as a standalone browser application that runs as its own local process alongside the Docker Compose backend. It states that the browser reaches the platform through one API Gateway URL and does not address backend services directly.

**Claimed in README, present on disk (confirmed):**

- `Pacco.Web` is a standalone browser application.
- Local frontend origin is `http://localhost:5173`.
- Local API Gateway is `http://localhost:5000`.
- Sign-in uses `POST /identity/sign-in`.
- Runtime configuration is injected through `public/pacco-config.js`.
- Browser session state is stored in `sessionStorage`.
- Access-token expiry is derived from the JWT `exp` claim.
- The refresh token is deliberately not retained.
- User-facing sign-in failures come from a closed client-owned message set.
- Current environment guidance is local-development only.

**Present on disk, absent or incompletely described in README:**

- `src/Router.tsx` now contains a protected `/welcome` route.
- `src/features/welcome/` contains a complete role-aware Welcome/Landing implementation.
- `RequireSession.tsx` protects authenticated routes.
- `roleDecision.ts` centralises the Admin versus standard-user presentation decision.
- `LogoutAction.tsx` implements client-side Logout.
- Typed Login and Landing telemetry events exist in `src/platform/telemetry.ts`.
- A substantial security negative-test suite exists under `tests/security/`.
- A separate work-item 13652 TypeScript/Playwright automation suite exists under `tests/automation/13652/typescript-playwright/`.
- Browser CORS, visual-fidelity and edge-conformance verification scripts are included.
- Jest coverage has targeted 100% gates for selected security-sensitive modules.

**Stale doc:** parts of `README.md` still describe the Welcome/Landing screen and Logout as future wave-2 work. The current repository now contains `WelcomeRoute.tsx`, `WelcomeCard.tsx`, `TopBar.tsx`, `LogoutAction.tsx`, and `RequireSession.tsx`. The README also says `/welcome` renders nothing, which no longer matches the current repository.

**Unknown:** the non-local deployment model for this browser application. No Dev, QA, Staging or Production frontend hosting configuration exists in the repository.

---

## 1. Primary purpose

Provide Pacco's browser presentation layer and own browser-session behaviour.

The current implementation covers:

- one common Login screen;
- credential submission through the API Gateway;
- browser-session creation;
- authenticated route protection;
- session-expiry handling;
- role-aware Welcome/Landing presentation;
- client-side Logout.

The repository does not currently provide the broader Pacco operational UI for Orders, Deliveries, Parcels, Vehicles, Customers, Pricing or Availability.

## 2. Main runtime / service type

React 19 single-page browser application written in TypeScript and built with Vite.

It runs as a **standalone browser process**, not as an ASP.NET service, backend container, static content hosted inside another Pacco service, or content served by Ntrada/API Gateway.

Primary runtime dependencies from `package.json`:

- `react ^19.1.1`
- `react-dom ^19.1.1`
- `react-router-dom ^7.8.2`
- TypeScript `~5.8.3`
- Vite `^7.1.3`
- Tailwind CSS `^3.4.17`
- Node.js `>=20.19.0`

Tooling includes Jest, React Testing Library, jest-axe, ESLint and Prettier.

## 3. Key entrypoints

- `src/main.tsx` — browser bootstrap; mounts `AppShell` into `#root`.
- `src/AppShell.tsx` — application composition root; reads runtime configuration once, constructs the single `GatewayClient`, and installs `BrowserRouter`.
- `src/Router.tsx` — route table and authenticated-route grouping.
- `src/routes.ts` — central path constants.
- `src/gateway/gatewayClient.ts` — the browser's only platform network client.
- `public/pacco-config.js` — runtime-injected browser configuration.
- `vite.config.ts` — local runtime/build configuration.

Current route vocabulary:

| Route | Behaviour |
|---|---|
| `/` | Shows Login when anonymous; redirects a live session to `/welcome` |
| `/login` | Login screen |
| `/welcome` | Protected role-aware Landing screen |
| other | No application content |

## 4. Important modules / packages

| Area | Role |
|---|---|
| `src/features/login/` | Login form, validation, submit lock, sign-in orchestration and Login messages |
| `src/features/welcome/` | Protected Landing page, role presentation, session guard and Logout |
| `src/session/` | Browser-session model, persistence, JWT expiry parsing and error-message mapping |
| `src/gateway/` | API Gateway client |
| `src/platform/` | Local correlation, telemetry abstraction and redacted diagnostics |
| `src/components/` | Shared Pacco presentation frame, logo/lockup and icons |
| `src/config/` | Runtime application config and local dev-server origin |
| `tests/` | Jest, integration, accessibility, security and cross-repository checks |
| `tests/automation/13652/typescript-playwright/` | Separate browser/API/static/live-platform automation suite |

Important files:

| File | Responsibility |
|---|---|
| `features/login/LoginRoute.tsx` | Owns Login field values and page-level flow |
| `features/login/useSignIn.ts` | Sole sign-in orchestrator and sole application call site of `SessionStore.write()` |
| `features/welcome/RequireSession.tsx` | Authenticated route guard |
| `features/welcome/roleDecision.ts` | Sole role interpretation point |
| `features/welcome/LogoutAction.tsx` | Client-side Logout |
| `session/sessionStore.ts` | `sessionStorage` persistence and liveness |
| `session/jwt.ts` | Reads `exp` from the access token |
| `session/errorMapper.ts` | Closed error classification |
| `session/messageRegistry.ts` | Client-owned user-facing failure strings |
| `gateway/gatewayClient.ts` | Browser → Gateway HTTP boundary |

## 5. External integrations

The Pacco API Gateway is the application's only runtime platform integration.

Runtime configuration declares:

```javascript
window.__PACCO_CONFIG__ = {
  gatewayBaseUrl: 'http://localhost:5000',
  signInTimeoutMs: 15000,
}
```

The browser currently calls only:

```http
POST /identity/sign-in
```

through the Gateway.

No browser source directly addresses `identity-service`, `customers-service`, `orders-service`, `deliveries-service` or any other Pacco backend service.

There is currently no call to `/identity/me`, refresh-token endpoints, access-token revocation, a Logout endpoint, or any protected Pacco business API.

The user's role comes from the successful sign-in response, not from a subsequent identity lookup.

### Browser / Gateway CORS

The local browser origin is `http://localhost:5173`. The Gateway is expected to permit that exact origin. Repository tests and scripts cross-check the Gateway's `ntrada*.yml` CORS configuration when a Pacco.APIGateway checkout is available.

### External browser analytics

None currently wired. A typed telemetry abstraction exists, but no application bootstrap installs an external telemetry sink.

## 6. Data stores / state

There is **no backend database** in this repository. No SQL, MongoDB, Redis or server-side persistence client exists.

The browser stores the authenticated session in:

```text
sessionStorage["pacco.session"]
```

The stored record is defined by `BrowserSession` and contains:

- `accessToken`
- `role`
- `expiresAt`
- optional `expiresRaw`

Important persistence rules:

- `role` is lower-cased on write;
- an unknown role is retained rather than converted into `user` or `admin`;
- `expiresAt` comes from the JWT `exp` claim;
- malformed persisted records fail closed;
- inaccessible browser storage is treated as no usable session;
- password is never persisted;
- refresh token is never persisted.

`localStorage` is not used by application code.

**Migration tool:** not applicable — browser storage contains one session record and no persistent domain data.

## 7. Messaging / async / events

`Pacco.Web` has **no RabbitMQ integration**.

It publishes no Pacco commands directly, subscribes to no events, owns no exchange or queue, and has no inbox/outbox.

Platform interaction is synchronous HTTP through the API Gateway.

### Browser-local telemetry events

Login events:

- `login.viewed`
- `login.validation_blocked`
- `login.submitted`
- `login.duplicate_suppressed`
- `login.succeeded`
- `login.failed`

Landing events:

- `landing.viewed`
- `landing.blocked_unauthenticated`
- `landing.session_expired`
- `landing.logout`

The telemetry types deliberately do not provide fields for password, access token, refresh token, raw backend error reason or raw role.

**Important:** no runtime sink is installed by `main.tsx` or `AppShell.tsx`. With no sink, `Telemetry.emit()` discards the event.

Browser-generated correlation IDs remain client-local and are deliberately not propagated to the Gateway as HTTP headers.

## 8. APIs exposed / consumed

**Exposed:** none. `Pacco.Web` is a browser SPA and does not expose a backend API.

**Consumed:**

| Method | Gateway route | Purpose |
|---|---|---|
| `POST` | `/identity/sign-in` | Authenticate the user |

Request body:

```json
{
  "email": "<identifier>",
  "password": "<password>"
}
```

The browser requires `accessToken` and `role` from a successful response. `expires`, if supplied, is retained only for diagnostics. `refreshToken` is deliberately not read or stored.

Browser-side validation is limited to identifier non-empty after trimming and password non-empty after trimming. The field is labelled **Email or Username**, but the Gateway request property is named `email`; no username-resolution logic exists in this repository.

## 9. Deployment / runtime clues

- `vite.config.ts` serves the application locally from `localhost:5173`.
- `strictPort: true` prevents Vite from silently selecting another port.
- The same port is used by Vite preview.
- Build output is `dist/`.
- Source maps are disabled.
- Runtime configuration is loaded from `public/pacco-config.js` before the application bundle.
- `src/config/devServerOrigin.ts` is the source of truth for the local browser origin.
- `tests/compose/devServerPort.test.ts` verifies that the chosen browser port does not collide with ports published by the Pacco Docker Compose backend.
- Port `3000` is deliberately avoided because Pacco Compose uses it for Grafana.

No application-level Dockerfile, Kubernetes manifest, nginx configuration, cloud hosting configuration, production deployment pipeline or Dev/QA/Staging/Production frontend origin matrix exists in the root application repository.

A Dockerfile and GitHub workflow do exist inside the generated Playwright automation package under `tests/automation/13652/typescript-playwright/`, but those belong to the test suite rather than to the Pacco.Web application deployment.

**CI observation:** the Playwright workflow is located at `tests/automation/13652/typescript-playwright/.github/workflows/test-automation-13652.yml` rather than repository-root `.github/workflows/`. **Needs validation** if automatic GitHub Actions execution is expected.

## 10. Security / auth clues

### Authentication boundary

Authentication is performed through the Gateway:

```text
Browser
  → Pacco.Web
  → POST /identity/sign-in
  → API Gateway
  → Identity service
```

The sign-in request carries no cookie/ambient credential mode.

### Password handling

The password exists in Login component state while the user enters it and is passed as an argument to the sign-in operation.

It is not written to browser storage, telemetry, diagnostics, URLs, accessibility attributes or hidden DOM elements. It is cleared after a settled submission.

### Access token

The access token is stored in the browser session. `src/session/jwt.ts` decodes the JWT payload only to read `exp`; it does **not** verify the JWT signature and does not make an authorisation decision.

### Refresh token

The browser deliberately does not retain the refresh token. There is no refresh-token field in `BrowserSession`, no refresh route, no renewal timer and no retry-on-401 token refresh.

### Route protection

`RequireSession.tsx` reads the current browser session, denies absent/unusable sessions, evaluates expiry on every guard decision, clears an expired session before redirecting, and returns the protected route only for a live session.

The guard is a browser navigation control, not the platform's authorisation boundary.

### Role handling

`roleDecision.ts` is the only application module that interprets role values.

```text
admin → Admin presentation
everything else → standard presentation
```

Unknown or unsupported role values cannot accidentally receive the Admin presentation.

### Logout

Logout consists of:

```text
SessionStore.clear()
→ landing.logout telemetry event
→ navigate /login
```

No revoke or Logout network call is made.

**Security limitation:** clearing the browser session does not invalidate an already-issued access token at the platform level. A copied token can remain usable until its own expiry.

### Error-data handling

The browser does not display backend `reason` strings. For HTTP 400, it reads only `code`.

Recognised codes `invalid_credentials` and `invalid_email` deliberately map to the same user-facing message:

```text
The email or password you entered is incorrect.
```

Other failures map to closed client-owned messages.

## 11. Observability / logging / tracing

### Diagnostics

`src/platform/diagnostics.ts` sends developer diagnostics to `console.warn`. The diagnostic model contains only `stage`, `classification` and `correlationId`; no password, identifier, request body, token or backend reason can be passed through the diagnostic API.

### Telemetry

Typed Login and Landing telemetry exists in `src/platform/telemetry.ts`. No external analytics sink is configured by the normal application bootstrap, so events are discarded unless a sink is installed.

### Correlation

`src/platform/correlation.ts` generates a browser-local correlation ID using `crypto.randomUUID()` when available. Correlation is **not propagated to the Gateway**.

### Tracing

No frontend OpenTelemetry, Jaeger, Application Insights, Datadog, New Relic or equivalent tracing SDK is configured.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `src/AppShell.tsx` | Browser configuration is read once and one Gateway client is created |
| `src/Router.tsx` | Current route vocabulary and grouped authenticated-route guard |
| `src/config/appConfig.ts` | One Gateway URL and sign-in timeout; runtime-injected configuration |
| `src/config/devServerOrigin.ts` | Exact local browser origin is `http://localhost:5173` |
| `src/gateway/gatewayClient.ts` | Browser reaches Pacco only through the API Gateway |
| `src/features/login/useSignIn.ts` | Sign-in state machine, submit lock and sole successful session-write path |
| `src/session/sessionStore.ts` | Browser session is tab-scoped `sessionStorage` state |
| `src/session/jwt.ts` | Browser lifetime comes from JWT `exp` |
| `src/session/errorMapper.ts`, `messageRegistry.ts` | Failure presentation is client-owned and backend reason text is not rendered |
| `src/features/welcome/RequireSession.tsx` | Protected routes require a live session |
| `src/features/welcome/roleDecision.ts` | Admin is an explicit allow-list |
| `src/features/welcome/LogoutAction.tsx` | Logout is local browser-session discard only |
| `tailwind.config.js`, `src/index.css` | Current design-token foundation is approximated from supplied static references |

**Feature flag system:** **none detected.** No LaunchDarkly, Unleash, Flagsmith, Split or in-house runtime feature-toggle mechanism appears in `package.json` or the application source. There are therefore **no flag keys to list**.

## 13. Open questions / ambiguities

1. Whether Login truly supports **Username**. The field is labelled `Email or Username`, but `gatewayClient.ts` sends a property named `email` and there is no username-resolution logic in this repository.
2. Where the typed Login/Landing telemetry is intended to be sent. The event abstraction exists, but no production sink is installed.
3. Whether the nested Playwright GitHub Actions workflow is meant to execute automatically; it is not under repository-root `.github/workflows/`.
4. What the non-local hosting/deployment model for `Pacco.Web` will be.
5. When the approved style assets named by the architecture will replace the currently approximated Tailwind design tokens.
6. When authenticated business APIs will be added to the Gateway client. The browser stores an access token today, but the current Welcome page makes no authenticated API request.
7. Whether `README.md` will be refreshed now that wave-2 Welcome/Landing and Logout are implemented.

## 14. Frontend stack

**Frontend assets are present and constitute the primary purpose of this repository.**

### Framework

- React 19
- React DOM
- React Router
- TypeScript
- Vite
- Tailwind CSS

### Main browser features

Login:

- Pacco branded Login screen;
- Email/Username field;
- password field;
- password show/hide control;
- required-field validation;
- in-flight submit lock;
- processing state;
- authentication failure messages;
- session-expiry notice;
- post-login navigation.

Landing:

- protected `/welcome` route;
- Admin presentation: `Welcome to Admin Area`;
- standard presentation: `Welcome`;
- role indicator;
- Logout;
- role allow-list;
- session-expiry redirect.

### Accessibility

The repository contains visible label associations, `aria-invalid`, `aria-describedby`, `role="alert"`, `role="status"`, password-toggle accessible labels, focus movement after route transition, keyboard-operable controls, visible focus styles, and responsive behaviour.

Jest includes `jest-axe` tests for Login and Welcome. The Playwright suite adds browser-level accessibility, keyboard, zoom and contrast test cases.

### Styling

`tailwind.config.js` contains semantic Pacco design tokens for brand, ink, surfaces, borders, danger, notices, decorative arcs, radii, card shadow and layout widths.

The source explicitly labels the current design foundation as **approximated**, because the expected canonical `STYLE_README.md` and `pacco-material-you.css` assets are not present in the analysed workspace.

### Design references

Committed reference files include:

- `docs/design-reference/01_pacco-logo-1.png`
- `docs/design-reference/02_login-page-ux.png`
- `docs/design-reference/03_welcome-page-ux.png`

Application image assets include:

- `src/assets/office-background.png`
- `src/assets/pacco-logo.png`
- `src/assets/pacco-mark.png`
- `src/assets/pacco-wordmark.png`

---

## Evidence

| Fact | File |
|---|---|
| Browser bootstrap | `src/main.tsx` |
| Application composition and Gateway-client construction | `src/AppShell.tsx` |
| Browser route table | `src/Router.tsx`, `src/routes.ts` |
| Runtime Gateway configuration | `public/pacco-config.js`, `src/config/appConfig.ts` |
| Local browser origin | `src/config/devServerOrigin.ts`, `vite.config.ts` |
| Sign-in HTTP contract | `src/gateway/gatewayClient.ts` |
| Login page flow | `src/features/login/LoginRoute.tsx`, `useSignIn.ts` |
| Login presentation | `src/features/login/LoginCard.tsx`, `IdentifierField.tsx`, `PasswordField.tsx` |
| Browser session model | `src/session/browserSession.ts` |
| Session persistence and liveness | `src/session/sessionStore.ts` |
| JWT expiry decoding | `src/session/jwt.ts` |
| Closed failure mapping | `src/session/errorMapper.ts`, `messageRegistry.ts` |
| Protected route | `src/features/welcome/RequireSession.tsx` |
| Role-aware Landing | `src/features/welcome/WelcomeRoute.tsx`, `WelcomeCard.tsx`, `roleDecision.ts` |
| Client-side Logout | `src/features/welcome/LogoutAction.tsx` |
| Browser-local telemetry | `src/platform/telemetry.ts` |
| Browser-local correlation | `src/platform/correlation.ts` |
| Redacted diagnostics | `src/platform/diagnostics.ts` |
| Design tokens | `tailwind.config.js`, `src/index.css` |
| Unit/integration/accessibility/security tests | `tests/` |
| Browser automation | `tests/automation/13652/typescript-playwright/` |
| CORS browser verification | `scripts/cors-browser-check.mjs` |
| Edge conformance | `scripts/edge-conformance-check.mjs` |
| Visual fidelity | `scripts/visual-fidelity-check.mjs` |
| Design approximation record | `docs/DESIGN_APPROXIMATION.md` |
| CORS verification record | `docs/CORS_VERIFICATION.md` |
| Build and package versions | `package.json` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** or **[handled later by <stage>]** where appropriate.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The API Gateway remains the only allowed platform endpoint for browser code | `GatewayClient` contains one Gateway URL and no backend service URL; this is reinforced by tests | A future feature could bypass edge authentication, routing and CORS assumptions | Keep future browser network operations inside the Gateway client and retain static checks |
| A2 | The `role` returned by the sign-in `AuthDto` is the authoritative input for the current Landing presentation | No `/identity/me` request exists; `useSignIn` stores the response role and `roleDecision.ts` consumes it | A different authoritative identity source could make the Landing presentation stale or incorrect | Confirm with Identity/Gateway owners before adding richer role-dependent UI |
| A3 | Logout is intentionally a browser-only session discard | `LogoutAction.tsx` makes no network request and source/tests explicitly preserve the already-issued token | If server-side revocation is required, current Logout does not satisfy it | Confirm expected Logout semantics with platform security owner |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[handled later by deployment architecture]** No Dev/QA/Staging/Production frontend hosting model exists | Deploying Pacco.Web outside local development | Platform/DevOps | Define hosting, environment configuration, DNS and deployment pipeline | TBD |
| B2 | **[ACTION NOW if visual foundation must be considered final]** Approved Pacco style assets named by the architecture are absent; current tokens are explicitly approximated | Treating current Tailwind tokens as the canonical Pacco design system | UX / Frontend architecture | Supply the canonical style assets or formally approve the current design-token set | TBD |
| B3 | **[ACTION NOW if Playwright CI is expected]** The generated GitHub Actions workflow is nested below `tests/automation/...` rather than repository-root `.github/workflows/` | Automatic execution of the Playwright suite by GitHub Actions | Test automation / DevOps | Wire the suite into repository-level CI | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does Pacco authentication support Username as well as Email? | The UI explicitly says `Email or Username`, but the request contract is `{ email, password }` and no username resolution exists in this client | Either confirm the Identity endpoint accepts username values in `email`, or change the visible label to Email | Product + Identity owner |
| Q2 | **[handled later by observability architecture]** Where should `login.*` and `landing.*` telemetry be sent? | The client defines useful typed telemetry but installs no runtime sink, so events disappear in normal execution | Connect the abstraction to the selected browser observability/analytics platform | Platform architect |
| Q3 | **[ACTION NOW]** Should `README.md` be updated after wave 2? | It still says `/welcome` is empty and Logout is future work even though both are implemented | Yes — align the runbook with the current code | Pacco.Web owner |
| Q4 | **[handled later by deployment architecture]** What hosts Pacco.Web outside local development? | No production deployment artefact or environment-specific origin exists | Define when non-local frontend environments are introduced | Platform/DevOps |
| Q5 | **[handled later by frontend architecture]** How should future authenticated business API calls attach the access token? | The access token is persisted today, but the current `GatewayClient` only performs anonymous sign-in and never sends `Authorization: Bearer ...` | Extend the single Gateway-client boundary when the first authenticated business surface is implemented | Frontend architect |
