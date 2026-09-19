# ADR-022: Client-owned browser session custody, with a client-side guard and a client-side logout

| | |
| --- | --- |
| **Status** | Proposed |
| **Date** | 2026-09-19 |
| **ADR id** | `ADR-022` |
| **Backlog candidate** | — (no candidate exists; this record originates from work item 13155, `intents/13155.md`) |
| **Source** | `DO1` — Pacco Common Login & Role-Aware Welcome Landing; `DO2` — Pacco Session Continuity, Landing-Page Guard & Logout |
| **Originating NFRs** | `NFR-22` (decision `needs_adr`); `NFR-7`, `NFR-21` (`at_risk`); `NFR-5`, `NFR-6`, `NFR-8`, `NFR-10` (constraints honoured) |
| **Category / Impact** | Security / high |
| **Supersedes / Superseded by** | — |
| **Deciders** | Work item 13155 input gate, questions 1, 3 and 4, answered by a human. Repository ownership remains unassigned — no `CODEOWNERS`, contributing guide or team metadata exists in any of the fourteen clones (blocker B1) |
| **Base ref of all cited source** | `feature/12998/aidlc` |
| **Related** | `ADR-006` (the authentication boundary this client sits outside), `ADR-007` (why revocation cannot make logout real), `ADR-021` (the surface that holds the session), `ADR-023` (the origin the session is established from) |
| **Known defect** | This record's logout semantics are bounded by a live platform defect it does not fix: a revoked access token is still accepted by the gateway and by all eight domain services until it expires (`ADR-007` §2 rule 4, B3). The decision is honest about that rather than claiming a stronger logout than the platform can enforce |

## Notation

| Symbol | Meaning |
| --- | --- |
| ✅ | Confirmed current behaviour — observed in source at the cited path and line range |
| 🎯 | Target state — intended design, not in production today |
| ❓ | Needs validation — assumed or inferred, not observed in source |
| `[INFERRED]` | Conclusion drawn from observed evidence rather than stated by it |

## Contents

1. [Context](#1-context)
2. [Decision Drivers](#2-decision-drivers)
3. [Architecture Fit Evaluation](#3-architecture-fit-evaluation)
4. [Decision](#4-decision)
5. [Options Considered](#5-options-considered)
6. [Consequences](#6-consequences)
7. [Compliance Considerations](#7-compliance-considerations)
8. [Non-Functional Requirements & Testing](#8-non-functional-requirements--testing)
9. [Relationship to the implementation pattern catalog](#9-relationship-to-the-implementation-pattern-catalog)
10. [Follow-Up Actions](#10-follow-up-actions)
11. [Evidence](#11-evidence)
12. [Assumptions, Blockers & Open Questions](#assumptions-blockers--open-questions)

---

## 1. Context

`ADR-021` puts a browser surface outside the edge. A browser that has signed in holds something, and
the platform has never had to say what, where, or for how long. Six facts constrain the answer, and
four of them are defects the platform already carries.

1. ✅ **No session mechanism exists anywhere on the platform.** No route in either gateway
   configuration uses a session cookie, a session id or an API key; no cookie is read or written by
   any existing UI asset; `identity-service` is not a session store
   (`docs/architecture-inventory/baselines/ui-inventory.md` §9;
   `docs/architecture-inventory/component-internals/identity-service.md` §1).
2. ✅ **No client-side persistence exists anywhere on the platform.** The one existing browser asset
   uses no `localStorage`, no `sessionStorage`, no `document.cookie` and no in-memory store beyond a
   live DOM input value — reloading the page loses the token
   (`…/baselines/ui-inventory.md` §9, §11.5).
3. ✅ **Sign-in returns the role in the response body as well as in the token.** `POST sign-in`
   returns `AuthDto = (string AccessToken, string RefreshToken, string Role, long Expires)`
   (`…/component-internals/identity-service.md` §3.11).
4. ✅ **The access token's absolute lifetime is 60 minutes and there is no refresh path through the
   edge.** Tokens expire after `jwt.expiryMinutes: 60`, and `POST /refresh-tokens/use` has no route
   in either gateway configuration
   (`…/component-internals/identity-service.md` §3.10; `…/baselines/api-inventory.md` §4.4).
5. ✅ **Revocation does not reach the enforcement point.** `identity-service` maintains an
   access-token deny-list and rejects revoked tokens itself, but the gateway validates with a
   symmetric key and consults no deny-list, and the eight domain services validate against a
   certificate and consult none either. `ADR-007` §2 rule 4 states the consequence directly: "Today a
   revoked token is still accepted by the gateway and by all eight domain services, because only the
   issuing service checks the revocation store."
6. ✅ **The authentication boundary is the edge, and in-service authorization fails open.** `ADR-006`
   §2 places authentication at the gateway; §4.2 records that the ownership guard passes for
   unauthenticated callers. Nothing a browser does can make that stronger or weaker.

One more fact shapes the role logic. ✅ The gateway gates routes on an exact-string claim comparison
(`claims: {role: admin}` against the aliased claim URI,
`…/component-internals/api-gateway.md` §3.8), while `IdentityContext.IsAdmin` compares
case-insensitively, and `identity-service` lower-cases the role on construction before minting it
(`…/component-internals/identity-service.md` §3.4). Three components, three comparison
behaviours, one claim.

### 1.1 What this record does *not* cover

It does not change any backend behaviour: no revocation is added, no refresh route is opened, no
`GET /identity/me` call is introduced, and no identity-service or authorization code is touched
(`ASM-6`). It does not decide where the surface is built or released — that is `ADR-021` — nor which
origins the edge admits, which is `ADR-023`. It does not make the client-side guard an access
control, and §4 rule 5 says so explicitly.

## 2. Decision Drivers

| # | Driver | Source |
| --- | --- | --- |
| D1 | The signed-in session must survive a page refresh and end when the tab closes | Work item 13155 input gate question 1, answered by a human; `NFR-22` |
| D2 | Logout must be honest — the UI must never claim a server-side session termination the platform cannot perform | Work item 13155 input gate question 3, answered by a human; `NFR-7`; `ADR-007` §2 rule 4 |
| D3 | Protected-route entry must be resolved from the held token with no server round-trip and no additional identity call | Work item 13155 input gate question 4, answered by a human; `NFR-8`; `ASM-8` |
| D4 | Admin determination binds solely to the token `role` claim value `admin`, with no email or username inference, no hierarchy and no "any of" semantics; unrecognised or absent values resolve to normal user | `NFR-5`; `ASM-2` |
| D5 | The bearer token must never reach a URL, a log line, an analytics event or an error report | `NFR-10`; `ASM-10` |
| D6 | The sign-in call must be bounded client-side, because no total request timeout is configured anywhere at the edge and the gateway retry policy can add up to six seconds on top of the downstream's own timeouts | `NFR-21`; `ASM-16` |
| D7 | No backend change is permitted beyond one edge configuration value | `ASM-6`; `DO3` |

## 3. Architecture Fit Evaluation

**No existing component can hold a browser session, because the platform has no session concept at
all.** Each candidate was evaluated before proposing that the client own it.

| # | Existing component or extension point | Evaluated as | Primary reason for rejection |
| --- | --- | --- | --- |
| 1 | `identity-service` as a server-side session store | It already issues tokens and holds refresh tokens and an access-token deny-list, so it is the nearest thing to a session authority | It is explicitly not a session store: it holds credential records and token records, not sessions, and it has no session endpoint (`…/component-internals/identity-service.md` §1). Making it one is a backend change that `ASM-6` forbids outright, and it would put a synchronous dependency on the landing page that `NFR-8` and the human gate both rule out |
| 2 | The existing access-token deny-list (`POST access-tokens/revoke`, Redis) as the logout mechanism | It exists, it works inside the issuing service, and it is the platform's only revocation primitive | It does not reach the enforcement point. The gateway and all eight domain services accept a revoked token until it expires (`ADR-007` §2 rule 4), so calling it on logout would produce a revocation nobody checks and a UI claim the platform cannot honour. Using it would make the logout *appear* stronger while changing nothing — the worst outcome available |
| 3 | The refresh-token lifecycle as a session-extension mechanism | `identity-service` mints a refresh token on every successful sign-in and can exchange it | `POST /refresh-tokens/use` has no route in either gateway configuration (`…/baselines/api-inventory.md` §4.4), so the browser cannot reach it, and opening one is a new edge route that `ASM-6` forbids. The refresh token the client receives is therefore unusable through the edge |
| 4 | `GET /identity/me` as a per-entry validation call | It exists at the edge, it is authenticated, and it would make the guard server-validated | Rejected at the human gate (question 4). It would also buy less than it appears to: `ADR-006` §4.2 records that in-service authorization fails open, so a validated `me` call proves the token parses at the edge and nothing about what the landing page may show. The landing page carries only a welcome message, so there is no data whose exposure the call would gate |
| 5 | A cookie set by the UI | The conventional browser session mechanism | No cookie exists anywhere on the platform, and a credentialed cross-origin cookie collides directly with the edge's `allowCredentials: true` plus wildcard-origin defect (`…/component-internals/api-gateway.md` §3.18). It would also require the edge or a service to set it, which is a backend change `ASM-6` forbids |
| 6 | The `operations-service` SignalR hub's token handling | The only precedent on the platform for a browser holding a token | It is not a store. The token lives in a DOM input value, is read fresh at click time, and is lost on reload (`…/baselines/ui-inventory.md` §11.5). It is a developer harness pattern, not a session model, and copying it would fail `NFR-22`'s refresh-survival target |

**Why a new boundary is required.** There is no existing session boundary to extend — the platform's
authentication model ends at token issuance and token validation, with nothing in between. The
client is the only component in the system that knows a session is in progress, because it is the
only component that holds the token between two requests. Locating custody anywhere else requires
a backend change that the agreed scope forbids.

**Why the boundary stays stable.** The client's session contract is exactly three things: a token
string, a role string, and an expiry instant — all three supplied by one response body the platform
already returns (`AuthDto`). None of them is a Pacco-specific structure; all three would survive a
change of framework, a change of store, or a move to a server-side session later.

**Why extending an existing component would violate cohesion, ownership or deployment
independence.** Rows 1, 3 and 5 all move work into `identity-service` or the edge, which makes a
browser surface's session model a backend release — precisely the coupling `ADR-021` was written to
remove. Row 2 violates something worse than cohesion: it would make the architecture claim a control
that does not exist.

## 4. Decision

**The browser owns its own session. The access token is held in `sessionStorage`, so the session
survives a page refresh and ends when the tab closes. Protected-route entry is resolved by a
client-side guard reading the held token. Logout is a local discard of the held token and nothing
else. Expiry is detected client-side from the token's own expiry instant. No cookie is set, no
server-side session is created, and no backend call is added.**

This follows the human answers at the work item 13155 input gate, questions 1, 3 and 4. It is not
re-opened by this record.

Seven rules follow from the decision and are part of it:

1. 🎯 **Exactly one client-side token store exists, and it is `sessionStorage`.** Not `localStorage`,
   not a cookie, not both. A second store is a second copy of a bearer credential with a second
   lifetime, and the platform has no way to invalidate either.
2. 🎯 **The token is written to that store and to the `Authorization` header, and nowhere else.** Not
   a URL, not a query string, not a log line, not an analytics event, not an error report, not the
   DOM (`NFR-10`).
3. 🎯 **Role resolution is a single comparison against the token's `role` claim**: the value `admin`
   compared case-insensitively resolves to the administrator landing message; every other value,
   including an absent claim, resolves to the normal-user message. No inference from email or
   username, no hierarchy, no "any of" semantics, and never a default to administrator (`NFR-5`).
4. 🎯 **Logout discards the held token and returns the user to Login, and the interface says exactly
   that.** No copy anywhere in the surface may state or imply that the session has been ended
   server-side, because it has not been (`NFR-7`).
5. 🎯 **The guard is a usability affordance, not an access control.** No security decision is
   delegated to the browser, and no data beyond what the held token already carries may be gated
   client-side. The authentication boundary remains the edge (`NFR-8`, `ADR-006` §2).
6. 🎯 **The surface issues no refresh call and implements no silent renewal.** The session ends at the
   token's absolute lifetime — 60 minutes — and the user is returned to Login with a session-expired
   message (`NFR-6`, `ASM-9`).
7. 🎯 **Every backend call the surface makes carries an explicit client-side timeout**, after which
   the processing state clears and the form returns to a usable state. The edge configures no total
   request timeout and its retry policy can add several seconds on top of the downstream's own
   (`NFR-21`, `NFR-12`).

## 5. Options Considered

Rows 1, 4 and 6 were settled by a human at the work item 13155 input gate. The table records why the
discarded options were discarded, so the reasoning is inheritable rather than only the outcome.

| # | Option | Why it was rejected |
| --- | --- | --- |
| 1 | **`sessionStorage` — survives refresh, ends with the tab** | **Chosen.** Governing source: work item 13155 input gate question 1, answered by a human. It is the only store whose lifetime matches the stated session semantics, and its tab-scoped expiry is the closest thing the platform has to a revocation, given that real revocation does not reach the gateway |
| 2 | **`localStorage`** | Rejected because it outlives the tab and the browser session, extending the window in which a stolen token is usable — and the platform cannot cut that window short, because a revoked token is still accepted until it expires (`ADR-007` §2 rule 4). It also fails the stated requirement that the session end when the tab closes |
| 3 | **In-memory only, lost on reload** | Rejected because it fails the refresh-survival requirement in `DO2` and `NFR-22`, and it reproduces the existing browser asset's behaviour, where reloading the page loses the token (`…/baselines/ui-inventory.md` §9) |
| 4 | **A cookie, or a server-side session** | Rejected at the human gate and independently by evaluation row 5 in §3: no cookie or session mechanism exists on any Pacco route, setting one requires a backend change `ASM-6` forbids, and a credentialed cross-origin cookie collides with the edge's existing wildcard-with-credentials defect |
| 5 | **Server-validated guard — `GET /identity/me` on every protected-route entry** | Rejected at the human gate (question 4). It adds a round-trip to every protected entry, and it would not strengthen the boundary: in-service authorization fails open (`ADR-006` §4.2), and the landing page holds no data the call would gate |
| 6 | **Server-side logout via the access-token deny-list** | Rejected by evaluation row 2 in §3: the deny-list is consulted only by the issuing service, so the revocation would be real but unenforced, and the interface would be claiming a control the platform does not have |
| 7 | **Silent token renewal before expiry** | Rejected because `POST /refresh-tokens/use` has no gateway route in either configuration, so the browser cannot reach it, and adding one is a new edge route that `ASM-6` forbids |

## 6. Consequences

### 6.1 Positive

1. 🎯 The session model is entirely inside one deployable. Nothing about it requires a backend
   release, which keeps `ADR-021`'s deployment independence real rather than nominal.
2. 🎯 The logout claim matches the platform's actual capability. A reader of the interface is not
   misled about whether a token stops working, which is the honest position given `ADR-007` B3.
3. 🎯 Tab-scoped custody bounds the exposure window without any server mechanism. Closing the tab is
   the only user-initiated action on this platform that reliably makes a token unreachable.
4. 🎯 Role resolution has exactly one input — the token claim — so the administrator landing message
   cannot be reached by any path the token does not already authorise.
5. 🎯 The client-side timeout gives the sign-in call the bound the edge does not provide, so a slow
   or hung downstream produces a usable form rather than an indefinite processing state.

### 6.2 Negative

1. 🎯 **This is the platform's first client-side persistence.** Today no `localStorage` or
   `sessionStorage` crosses any hand-off anywhere in the fourteen clones
   (`…/baselines/ui-inventory.md` §11.5). The novelty is deliberate and bounded to the token, but the
   platform now has a client-side credential store where it had none, and nothing governs a second
   one.
2. 🎯 **A bearer token in `sessionStorage` is readable by any script running on the origin.** The
   platform has no content-security-policy standard, no dependency scanning in any pipeline
   (`…/baselines/architecture-baseline.md` §9.5) and no frontend rule family catalogued anywhere, so
   the compensating controls that normally accompany this choice are not inherited — they must be
   authored. Carried as `GAP-13155-04`.
3. 🎯 **Logout is not revocation, and a stolen token survives it.** Discarding the local copy does
   nothing to a copy taken earlier; that copy stays valid at the gateway and at all eight domain
   services until its 60 minutes elapse. This is a platform defect (`ADR-007` B3), not a defect of
   this decision, but this decision is where a reader will first meet it.
4. 🎯 **The session ends abruptly at 60 minutes with no renewal path.** A user mid-task is returned
   to Login. With no business functionality behind the landing page this costs nothing today; the
   first surface that carries real work will need `ADR-007` rule 4 or a routed refresh path resolved
   first.
5. ❓ **Three components compare the same role claim three ways.** The gateway compares exact strings,
   `IdentityContext` compares case-insensitively, and this surface is required to compare
   case-insensitively (`ASM-2`). A token minted outside the normal path with `Admin` would be an
   administrator to this surface and to the domain services, and not to the gateway's claim gate.
   Carried as `GAP-13155-05`.
6. 🎯 **The guard can be bypassed by anyone who opens developer tools.** That is inherent to rule 5
   and is accepted, because nothing behind the guard is protected — but it becomes a real exposure
   the moment business functionality lands on a protected route, which is why rule 5 is written as a
   constraint on future work and not only a description of this one.

### 6.3 Neutral

1. The refresh token returned in the sign-in response body is received and unused. The surface has no
   route to exchange it, so it is discarded rather than stored — holding an unusable second
   credential would only widen rule 1's surface.
2. The role value is available in the response body as well as in the token claim. Rule 3 binds to
   the claim regardless, so the two sources can never diverge in the surface's logic.
3. Nothing about this decision constrains the framework or the state-management approach; it
   constrains only what is stored, where, and for how long.

## 7. Compliance Considerations

| Governing source | Rule as it applies here | How this decision conforms |
| --- | --- | --- |
| `docs/adr/edge-enforced-authentication-with-fail-open-authorization.md` §2 — "Pacco enforces authentication at the gateway… Services behind the edge trust the caller context they are given and re-check only resource ownership" | The browser is a caller outside the boundary; it may not become an enforcement point | Conforms. Rule 5 states the guard is a usability affordance and delegates no security decision to the browser |
| `docs/adr/split-jwt-trust-root-gateway-and-services.md` §2 rule 4 — "Revocation is consulted at the boundary that enforces authentication. Today a revoked token is still accepted by the gateway and by all eight domain services" | Any logout claim must be bounded by what revocation actually achieves | Conforms. Rule 4 forbids any interface copy claiming server-side termination, and §6.2.3 records the residual exposure explicitly rather than mitigating it on paper |
| `docs/adr/declarative-configuration-driven-api-gateway.md` §2 obligation 3 — per-client shaping "must not be added to this configuration" | Nothing in the session model may push work into the edge | Conforms. No manifest change is made by this record; the only edge change in this work item is `ADR-023`'s allowed-origins value |
| `docs/architecture-inventory/patterns/security/transport-agnostic-caller-context.md` | Handlers derive the caller from an envelope or header, never from the body | Unaffected. The surface sends a standard `Authorization` header; no new caller-context mechanism is introduced |
| Security & privacy standards (token storage, content security policy, dependency scanning) | No `docs/standards/` directory exists in this repository, and no security rule family covering browser clients is catalogued | No silent default is taken. The absence is recorded as `GAP-13155-04` in `docs/architecture-inventory/risk-constraint-gap-register.md`, and the compensating controls are named as follow-up F2 rather than assumed |
| Frontend standards (state ownership, host/shell contract) | Not catalogued anywhere in this repository | No silent default is taken. Recorded as `GAP-13155-08`; this record fixes only the session slice of state ownership and says nothing about the rest |

## 8. Non-Functional Requirements & Testing

| NFR | Target | How this decision addresses it | How it is verified |
| --- | --- | --- | --- |
| `NFR-22` [security] | Exactly one client-side token store, scoped to the tab, with no cookie set by the UI | §4 rule 1 — `sessionStorage` and nothing else | After sign-in, the token is present in `sessionStorage` and absent from `localStorage` and `document.cookie`; it survives a reload and is gone in a newly opened tab |
| `NFR-7` [security] | Client-side discard is the only enforceable logout | §4 rule 4 — local discard, and no interface copy claiming otherwise | After logout the token is absent from the client store, and a copy-review check confirms no logout or session-expiry string asserts server-side termination |
| `NFR-6` [security] | 60 minutes | §4 rule 6 — no refresh, session ends at the token's absolute lifetime | A token past its expiry instant produces the session-expired message on Login with no backend call issued |
| `NFR-8` [security] | Zero security decisions delegated to the browser | §4 rule 5 — the guard is an affordance; the boundary is the edge | The landing page issues no authorization call and renders no data not already in the held token; a review confirms every protected value is public or token-carried |
| `NFR-5` [security] | 100% of role decisions derived from the token claim | §4 rule 3 — one comparison, no inference, no default to administrator | Tokens carrying `admin`, `ADMIN`, `user`, `manager`, and no role claim at all each produce the expected message, with the last three all producing "Welcome" |
| `NFR-10` [privacy_compliance] | Zero token occurrences outside the chosen client store and the `Authorization` header | §4 rule 2 | A full sign-in, guard and logout pass produces no token substring in any URL, console line, DOM node or error payload |
| `NFR-21` [performance] | An explicit client-side request timeout, with the processing state clearing and the form returning to a usable state at that bound | §4 rule 7 | With the gateway base URL pointed at a black-hole endpoint, the processing state clears at the configured bound and the form accepts input again |
| `NFR-11` [concurrency] | At most one in-flight sign-in request per session | The processing state that rule 7 bounds also blocks re-submission for the whole time a request is in flight | Repeated submits during an in-flight request produce exactly one network call |
| `NFR-12` [reliability_resilience] | 100% of failure paths return to a usable form | Rules 4, 6 and 7 each end in a usable Login form | Invalid credentials, a network failure, a 5xx response and a client-timeout each leave the form usable with the processing state cleared |

**Testing obligations this decision creates.** `OQ-4` records that the platform has no house
standard to inherit — the build is not broken by a failing test suite and the suite is not run in
CI. The `hls` and `lld` stages must therefore specify, rather than assume:

1. A store-assertion test per store (`sessionStorage` present, `localStorage` and cookie absent) run
   after sign-in and again after logout. This is the only mechanical guard on rule 1.
2. Role-resolution cases for `admin`, a differently-cased `admin`, a valid non-admin role, an
   unrecognised role and an absent claim — five cases, because `NFR-5` and `OQ-3` both turn on the
   last two behaving identically.
3. A token-leak sweep across URL, console, DOM and error payloads, because rule 2 has no other
   enforcement.
4. One end-to-end run per role covering sign-in, guard, refresh-survival, expiry and logout.

## 9. Relationship to the implementation pattern catalog

| Pattern | Relationship |
| --- | --- |
| [`security/edge-enforced-authentication-with-identity-binding.md`](../architecture-inventory/patterns/security/edge-enforced-authentication-with-identity-binding.md) | **Consumed from outside.** The surface is a caller that presents a token to the boundary this pattern describes. It adds no enforcement and it weakens none; rule 5 exists to keep that true |
| [`security/transport-agnostic-caller-context.md`](../architecture-inventory/patterns/security/transport-agnostic-caller-context.md) | **Unaffected.** The surface uses the ordinary `Authorization` header path; no new caller-context population route is introduced |
| [`orchestration/acknowledge-then-notify-completion.md`](../architecture-inventory/patterns/orchestration/acknowledge-then-notify-completion.md) | **Not used, deliberately.** `POST /identity/sign-in` is a synchronous route, so the surface reads a response body rather than correlating an operation id. This record does not adopt the async acknowledgement pattern and the first surface that issues an async write will have to |

Every pattern in the catalog is `Candidate`, so none of these relationships constitutes approval.

## 10. Follow-Up Actions

| # | Action | Owner | Due | Blocks |
| --- | --- | --- | --- | --- |
| F1 | Answer `OQ-3` — confirm that a token with no `role` claim shows "Welcome", identically to an unrecognised role | Product owner (unassigned — see B1) | 2026-09-26 | `NFR-5` test case 5; `DO1` acceptance |
| F2 | Name the compensating controls for a script-readable token store — at minimum a content-security policy and dependency scanning in the `Pacco.Web` pipeline — since no platform standard supplies them | Platform security owner (unassigned — see B1) | 2026-10-17 | `GAP-13155-04`; §6.2.2 |
| F3 | Decide whether revocation is enforced at the gateway, which is the only change that would make logout real | Platform security owner (unassigned — see B1) | 2026-11-14 | `ADR-007` B3; §6.2.3; any future surface carrying business functionality |
| F4 | Reconcile the three role-comparison behaviours — gateway exact-string, `IdentityContext` case-insensitive, this surface case-insensitive | Platform security owner (unassigned — see B1) | 2026-11-14 | `GAP-13155-05` |
| F5 | Set the concrete client-side timeout value for the sign-in call, bounded by the gateway retry policy's worst case of six seconds plus the downstream's own | `hls` stage | 2026-10-03 | `NFR-21` |

## 11. Evidence

| # | Claim | Notation | Source (path:lines) |
| --- | --- | --- | --- |
| E1 | No route on the platform uses a session cookie, a session id or an API key, and no existing UI asset reads or writes a cookie | ✅ | `docs/architecture-inventory/baselines/ui-inventory.md` §9 |
| E2 | The existing browser asset uses no `localStorage`, `sessionStorage`, cookie or in-memory store; reloading loses the token | ✅ | `…/baselines/ui-inventory.md` §9, §11.5 |
| E3 | `POST sign-in` returns `AuthDto = (string AccessToken, string RefreshToken, string Role, long Expires)` | ✅ | `docs/architecture-inventory/component-internals/identity-service.md` §3.11 |
| E4 | Tokens expire after `jwt.expiryMinutes: 60` | ✅ | `…/component-internals/identity-service.md` §3.10 |
| E5 | `POST /refresh-tokens/use` has no route in either gateway configuration | ✅ | `docs/architecture-inventory/baselines/api-inventory.md` §4.4; `…/component-internals/identity-service.md` §3.13 |
| E6 | A revoked access token is still accepted by the gateway and by all eight domain services, because only the issuing service checks the revocation store | ✅ | `ADR-007` §2 rule 4; `ADR-007` B3 |
| E7 | `identity-service` maintains an access-token deny-list and rejects revoked tokens itself | ✅ | `…/component-internals/identity-service.md` §3.12 (`UseAccessTokenValidator()`) |
| E8 | Authentication is enforced at the gateway and in-service authorization passes for unauthenticated callers | ✅ | `ADR-006` §2; `ADR-006` §4.2 |
| E9 | The gateway gates on an aliased `role` claim with exact-string per-route values | ✅ | `…/component-internals/api-gateway.md` §3.8 |
| E10 | `identity-service` lower-cases the role on construction and enforces the closed vocabulary `{user, admin}` | ✅ | `…/component-internals/identity-service.md` §3.4 |
| E11 | `GET /identity/me` exists at the edge as an authenticated read route | ✅ | `…/component-internals/api-gateway.md` §6.1 |
| E12 | No pipeline on the platform performs dependency or image scanning | ✅ | `docs/architecture-inventory/baselines/architecture-baseline.md` §9.5 |
| E13 | `identity-service` is not a session store and exposes no session endpoint | ✅ | `…/component-internals/identity-service.md` §1 |

### 11.1 Documentation-versus-code conflicts

| # | Conflict | Resolution |
| --- | --- | --- |
| X1 | The work item's input-gate question 2 states that "the identity token endpoints have no edge route today". `POST /identity/sign-in` and `POST /identity/sign-up` both have public routes in the synchronous manifest; it is the refresh-token endpoints that have none | The manifest wins. The statement is true of `POST /refresh-tokens/use` and `POST /refresh-tokens/revoke` and false of sign-in, which is why §4 rule 6 is a constraint on renewal and not on sign-in |

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
| --- | --- | --- | --- | --- |
| A1 | The access token carries an expiry claim the client can read without calling the backend | The sign-in response body carries an `Expires` value, and the token is a standard JWT minted by the platform's toolkit with lifetime validation enabled at the gateway and in eight services | Rule 6 has no client-side trigger. Expiry would become observable only from a rejected call, and the landing page issues no call — so an expired session would sit on screen indefinitely | Decode a token issued by a running `identity-service` and confirm a readable expiry instant; the `hls` stage records which of the token claim or the response `Expires` value is used |
| A2 | `sessionStorage` is available and writable in every browser the surface supports | It is a baseline browser capability, but no browser support matrix exists for this platform in any of the fourteen clones | The store silently fails and the session does not survive a refresh, with no error path defined for it | The `hls` stage defines the supported browser matrix and the behaviour when the store is unavailable |
| A3 | No second browser tab is expected to share the session | `sessionStorage` is tab-scoped by definition, and the stated semantics are "ends with the tab" | A user opening the landing page in a second tab is sent to Login and reads it as a bug. `DO2`'s acceptance says nothing about a second tab | Confirm with the product owner that a second tab requiring a second sign-in is acceptable |
| A4 | The gateway's retry policy worst case of six seconds is the correct upper bound to size the client timeout against | It is the figure recorded in `ASM-16` and `NFR-21`, derived from the edge's configured retry behaviour, and no total request timeout exists at the edge to bound it further | The client timeout is set too low and cancels calls the platform would have completed, producing spurious failures on a slow but healthy path | Measure sign-in latency at the edge under a slow downstream before fixing the value in F5 |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
| --- | --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** No repository has a named owner, so no follow-up action in §10 has an accountable person and this record cannot leave `Proposed` | F1 through F5, and this ADR's ratification | Platform owner | Name an owner per subsystem using the six groupings in `docs/architecture-inventory/repo-inventory.md` §4 and record them in this repository | At the work item review that receives this record — it blocks ratification |
| B2 | **[ACTION NOW]** A revoked token is accepted by the gateway and by all eight domain services until it expires. Logout therefore cannot be made real by any client-side decision | Any future surface that puts business functionality behind the guard, and any security review that expects logout to terminate access | Platform security owner | F3 — enforce revocation at the gateway, which `ADR-007` §2 rule 4 already commits the platform to | 2026-11-14 |
| B3 | **[ACTION NOW]** `POST /identity/sign-up` accepts `"role": "admin"` from the request body, so any anonymous caller can create an administrator account. This surface's role-aware landing page is downstream of that, and would faithfully show the administrator message to a self-elevated account | Any claim that the administrator landing message reflects a trustworthy authorization state | Platform security owner | Gate role assignment at sign-up, or bind the role from a trusted source at the gateway route, per `…/component-internals/identity-service.md` §3.4 | Before the platform is reachable by untrusted callers |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** What does a user whose token carries no `role` claim at all see, as distinct from one carrying an unrecognised role value? | `NFR-5` and rule 3 treat them identically, but that is a product judgement this stage cannot make. It is `OQ-3` in the work item and it has not been answered | Treat both identically as a normal user and show "Welcome", never Admin. It is the safe reading of the stated guardrail and needs no new backend behaviour | Product owner |
| Q2 | **[handled later by the `hls` stage]** Is expiry read from the token's own claim or from the sign-in response `Expires` value? | Both are available and they should agree, but nothing verifies that they do; picking one and stating it prevents two code paths disagreeing about when the session ended | Read it from the token claim, so the single source of truth for both role and expiry is the token itself, matching rule 3 | `hls` stage |
| Q3 | **[handled later by the `lld` stage]** What concrete client-side timeout value bounds the sign-in call? | `NFR-21` requires an explicit bound, and A4 records that the figure it should be sized against is itself unmeasured | Size it above the gateway retry worst case of six seconds plus the downstream's own timeout, and record the measurement it was derived from rather than a round number | `lld` stage, per F5 |
| Q4 | **[ACTION NOW]** Is a tab-scoped session acceptable to the product, given that a second tab requires a second sign-in? | It is a direct consequence of the chosen store and it will be read as a defect if nobody has agreed to it. See A3 | Yes, on the grounds that the tab close is the only user-initiated action that reliably ends access while revocation is unenforced | Product owner |
