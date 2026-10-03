# OpenAPI artifacts — capability `13652`

Empty by design.

Capability `13652` authors **no** OpenAPI document. Wave-1 consumes `POST /identity/sign-in` exactly as
`identity-service` already publishes it, and wave-2 calls no API at all. The high-level ESD §9.X
records that this round changes no service contract.

The `operationId` `identitySignIn` used across the spec pack is provisional — the platform publishes no
OpenAPI document, so the id exists for traceability inside `docs/specs/13652/` only.

Any artifact added here must first be recorded in `../CONTRACT_LEDGER.md`, and a wave that finds it
needs one must raise a blocker against the high-level ESD rather than adding it silently.
