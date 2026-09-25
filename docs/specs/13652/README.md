# 13652 — Pacco Login and Welcome Landing Page

Spec pack for the platform's first browser surface: one common sign-in screen and a role-aware landing page, delivered in two waves.

## Status

| Item | Value |
| --- | --- |
| Work item | 13652 |
| Branch | `feature/13652/spec-generation` |
| Tier | High-level baseline architecture (ESD) |
| Status | Draft — not yet reviewed |
| Waves | wave-1 (DO1), wave-2 (DO2) |
| New records authored | `ADR-022`, `ADR-023` |
| Open blockers | 4, all `[ACTION NOW]` — see the ESD's final section |

## Reading order

| # | File | Responsibility |
| --- | --- | --- |
| 1 | `../../../intents/13652.md` | The validated intent. Owns the Delivery Outcome text, their metrics and targets, and the wave plan. Nothing downstream may restate a DO differently |
| 2 | `SPECIFICATION.md` | The Execution Specification Document. Owns scope, requirements, contracts, workflows, UX, security, operations, tests, acceptance criteria and traceability for this capability, plus the live Assumptions, Blockers & Open Questions list |
| 3 | `../../adr/standalone-browser-client-and-browser-caller-edge-contract.md` | `ADR-021` — owns the client boundary, the browser-caller contract at the edge, and the logout semantics |
| 4 | `../../adr/browser-session-bounded-by-access-token-expiry.md` | `ADR-022` — owns how long a browser session lasts and what happens to the refresh token |
| 5 | `../../adr/browser-boundary-error-presentation-contract.md` | `ADR-023` — owns how a failure is turned into something a user reads |
| 6 | `../../architecture-inventory/architecture-views.md` | The platform map. Owns where this capability sits among existing components |
| 7 | `solution-design.md` | Prior analysis input. Superseded as guidance by `SPECIFICATION.md`, retained for its reasoning trail |
| 8 | `LOW_LEVEL_SPEC-13652-wave-1.md`, `LOW_LEVEL_SPEC-13652-wave-2.md` | Not yet authored. Will own file-level design, component internals and design grounding against the approved style assets |

## Authority precedence

When two documents disagree, the higher rule wins:

1. **Component source code** — for any claim about current behaviour. Code beats every document here.
2. **`intents/13652.md`** — for what is being delivered and why.
3. **ADRs** — for architectural rules. A newer record beats an older one only where it says so explicitly.
4. **`SPECIFICATION.md`** — for how this capability realises the above.
5. **Low-level specs** — for implementation detail within the boundaries the ESD sets.
6. **`solution-design.md` and other analysis** — context only, never authority.

A disagreement is raised in the ESD's Assumptions, Blockers & Open Questions section. It is never resolved by quietly editing the losing document.

## Glossary

| Term | Meaning here |
| --- | --- |
| **ESD** | Execution Specification Document — `SPECIFICATION.md`, the target-state baseline architecture for one work item |
| **DO** | Delivery Outcome — a user-visible outcome with its own metric and target, defined in the intent |
| **Wave** | A sequenced group of DOs handed to one low-level spec |
| **FR / AC** | Functional Requirement / Acceptance Criterion |
| **ABQ** | The Assumptions, Blockers & Open Questions section that closes the ESD and each ADR |
| **`BLOCKING_FOR_LLD`** | An unresolved item that must be answered before low-level design can proceed |
| **The edge** | The single north-south API gateway, locally `http://localhost:5000` |
| **Browser session** | Access token, role and expiry held in the browser. Not a server-side session — Pacco has none |
