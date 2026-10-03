# Event schemas — capability `13652`

Empty by design.

Capability `13652` publishes and subscribes to **no** message-broker event. Both delivery outcomes run
entirely in the browser, and neither reaches RabbitMQ directly or indirectly.

`identity-service` continues to publish its own `SignedIn` event as part of the sign-in path this
capability consumes. That event is unchanged, is owned by `identity-service`, and is not a contract
artifact of this capability.

Any artifact added here must first be recorded in `../CONTRACT_LEDGER.md`, and a wave that finds it
needs one must raise a blocker against the high-level ESD rather than adding it silently.
