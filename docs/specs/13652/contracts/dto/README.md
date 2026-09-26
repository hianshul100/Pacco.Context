# Shared DTOs — capability `13652`

Empty by design.

Capability `13652` authors **no** shared DTO. The one wire shape it reads, `AuthDto`, is owned by
`identity-service` and lives at
`Pacco.Services.Identity/src/Pacco.Services.Identity.Application/DTO/AuthDto.cs`. It is consumed
unchanged: no field is added, removed or renamed.

`BrowserSession`, the closed user-facing message registry, and the `SessionStore` module are browser-
internal. They are owned by wave-1 and consumed by wave-2, they cross no service boundary, and they are
therefore not shared DTOs. Their ownership is recorded in
`../../LOW_LEVEL_SPEC-13652-wave-1.md` §L.8.1.

Any artifact added here must first be recorded in `../CONTRACT_LEDGER.md`, and a wave that finds it
needs one must raise a blocker against the high-level ESD rather than adding it silently.
