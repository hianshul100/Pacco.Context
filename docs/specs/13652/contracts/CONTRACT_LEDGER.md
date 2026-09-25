# Contract Ledger — Capability `13652`

Every shared-contract change made by any wave of capability `13652` is recorded here, one row per
change. A wave that authors, modifies or retires a contract artifact **must** add a row before the
artifact is used.

**Contracts root:** `docs/specs/13652/contracts/`
**Sub-roots:** `openapi/`, `events/`, `dto/`

> [!IMPORTANT]
> The high-level ESD (§9.X) records that this round changes **no** service contract. Both waves
> consume `POST /identity/sign-in` exactly as `identity-service` already publishes it. If
> implementation reveals that a shared contract artifact is genuinely required, that is a **blocker to
> raise** against the high-level ESD — it is not an artifact to author quietly under this ledger.

## Ledger

| Round | Wave | Date (UTC) | Artifact (file · path · operationId · event name) | Change kind | Classification | Spec source | Approval citation |
|---|---|---|---|---|---|---|---|
| 1 | wave-1 | 2026-09-25 | none | none | not-applicable | `docs/specs/13652/LOW_LEVEL_SPEC-13652-wave-1.md` §L.8.4 | HL ESD `docs/specs/13652/SPECIFICATION.md` §9.X |
| 1 | wave-2 | 2026-09-25 | none | none | not-applicable | `docs/specs/13652/LOW_LEVEL_SPEC-13652-wave-2.md` §L.8.5 | HL ESD `docs/specs/13652/SPECIFICATION.md` §9.X |

## Consumed, unchanged

Recorded for traceability. These are **not** ledger changes — no row above corresponds to them,
because nothing about them was altered by this capability.

| Operation | Method · edge path | Declared at | Owner | Consumed by |
|---|---|---|---|---|
| `identitySignIn` | `POST /identity/sign-in` | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:263-269` | `identity-service` | wave-1 (`DO1`) |

`operationId` `identitySignIn` is provisional and exists for traceability inside this spec pack only —
the platform publishes no OpenAPI document (HL ESD §9.1, ASM-5).

## Browser-internal surfaces

`BrowserSession`, the closed user-facing message registry, and the `SessionStore` module are owned by
wave-1 and consumed by wave-2. They cross no service boundary and are therefore **not** shared
contracts under this ledger. Their ownership is recorded in
`LOW_LEVEL_SPEC-13652-wave-1.md` §L.8.1 and `LOW_LEVEL_SPEC-13652-wave-2.md` §L.8.1.
