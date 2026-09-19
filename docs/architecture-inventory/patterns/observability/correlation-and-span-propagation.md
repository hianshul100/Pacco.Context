# Pattern: Correlation And Span Propagation

## Status

**Candidate.** No approval metadata, ADR, or documented human sign-off exists in the workspace, and
the CAKE catalog for tenant `Q5SCXYFS` holds no governing decision.

## Category

observability

## Problem

A single user action crosses a gateway, a queue, three services, and comes back as a push notification.
When it goes wrong, the only thing tying those hops together is whatever identifier each one happens to
have written down. If each hop invents its own, the story ends at the first boundary, and debugging
becomes reading five logs and guessing which lines belong together.

Asynchronous messaging makes this sharper than HTTP does. There is no connection between the publisher
and the consumer to hang a trace on — if the identifiers do not travel inside the message, they do not
travel at all.

## Context

Applies to any platform where work crosses process boundaries, and especially where it crosses them
asynchronously. Pacco generates a correlation id and a trace id at the gateway, carries both — along
with a serialised span context and the caller's identity — in message headers through every hop, and
reports spans to Jaeger from ten of its eleven deployables.

## When to Use

- Requests fan out across services, and at least some hops are asynchronous.
- A user-visible outcome depends on work that finishes after the original request returned.
- Operators need to answer "what happened to this request" without correlating by timestamp.
- Message consumers publish further messages, so causality is a chain rather than a pair.

## When Not to Use

- A single process. In-process call stacks already carry the causality.
- Sampling is not acceptable and volume is high — the pattern's cost scales with traffic, and constant
  sampling in particular does not scale.
- The identifiers would carry data that must not leave the originating service. The context here
  includes the caller's identity and claims, which is a deliberate choice with consequences.

## Architecture Summary

Three identifiers travel together in one header, and a fourth in a header of its own.

The gateway generates a request id and a trace id per incoming request (`generateRequestId: true`,
`generateTraceId: true`) and returns both to the client through exposed response headers. When a route
publishes a message, that context is written into the message.

Every service reads the context from an accessor rather than from the transport, so the same code path
works whether the work arrived over HTTP or from a queue
([[transport-agnostic-caller-context]]). When a service publishes, its message broker wrapper
reassembles the outgoing context: it carries forward the originating message id, the correlation id, and
a span context — falling back to the tracer's currently active span when no span context was supplied —
plus the headers marked for forwarding and the caller's correlation context.

Separately, every service registers Jaeger with a `const` sampler and a RabbitMQ plugin that reads and
writes span context on the wire, so spans link across the broker rather than restarting at each
consumer.

## Structure / Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as api-gateway
    participant X as Exchange
    participant A as Service A
    participant B as Service B
    participant J as Jaeger

    C->>GW: request
    GW->>GW: generate Request-ID, Trace-ID
    GW->>X: message + message_context + span_context
    GW-->>C: 202 with Request-ID, Trace-ID headers
    X->>A: deliver
    A->>J: span (child of incoming span context)
    A->>X: publish + same correlation id, new span context
    X->>B: deliver
    B->>J: span (child of A's span)
    Note over C,J: one correlation id across every hop;<br/>spans chained by span_context
```

## Key Components

| Component | Responsibility |
|-----------|----------------|
| Gateway `generateRequestId` / `generateTraceId` | Mint the identifiers at the edge |
| Gateway `exposedHeaders` | Return `Request-ID`, `Trace-ID`, `Resource-ID`, `Total-Count` to the client |
| `rabbitMq.context.header` (`message_context`) | Carries correlation id, trace id, connection id, operation name, resource id, and the user context |
| `rabbitMq.spanContextHeader` (`span_context`) | Carries the serialised span for parent/child linking |
| `MessageBroker` wrapper | Reassembles the outgoing context on every publish |
| `ICorrelationContextAccessor` | Supplies the inbound context without touching the transport |
| `AddJaegerRabbitMqPlugin()` | Reads and writes span context across the broker |
| `JaegerCommandHandlerDecorator<T>` | Opens a named span per command — **in one service only** |

## Data / Event / API Contracts

- **`message_context` payload:** `CorrelationId`, `SpanContext`, `TraceId`, `ConnectionId`, `Name`,
  `ResourceId`, `CreatedAt`, and a nested `User` object (`Id`, `IsAuthenticated`, `Role`, `Claims`).
- **`span_context` header:** the serialised OpenTracing span context, written and read by the RabbitMQ
  plugin.
- **Publish-time resolution** (`MessageBroker`): `originatedMessageId` from the inbound message
  properties; `correlationId` likewise; `spanContext` from the inbound properties, falling back to
  `_tracer.ActiveSpan.Context.ToString()`; `headers` from `GetHeadersToForward()`; the correlation
  context from the broker accessor, falling back to the HTTP context.
- **Jaeger configuration**, identical in ten deployables: `enabled: true`, `udpHost: localhost` (or
  `jaeger` in the container variants), `udpPort: 6831`, `sampler: const`,
  `excludePaths: ["/", "/ping", "/metrics"]`. `serviceName` is the short service name —
  `orders`, `parcels`, `availability` — not the `<name>-service` form used everywhere else.
- **Gateway tracing:** a `tracing` block with `serviceName: api-gateway`, `sampler: const`,
  `useEmptyTracer: false`, present in all four Ntrada configurations.
- **`ordermaker-service` has no `jaeger` block at all** and is the one deployable outside the tracing
  picture.
- **Correlation id in the operations projection:** the operation record is keyed on the correlation id,
  which is what lets a client poll or be pushed the outcome of work it started
  ([[acknowledge-then-notify-completion]]).

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Message header names | snake_case | `message_context`, `span_context` |
| Response header names | Title-Case with hyphens | `Request-ID`, `Trace-ID` |
| Jaeger service name | Short service name, no suffix | `orders`, `pricing` |
| Span name | `handling-<message_name>` in underscore case | `handling-add_parcel_to_order` |
| Span tag | Hyphenated lower case | `message-type` |
| Excluded paths | Leading slash, health and metrics endpoints | `["/", "/ping", "/metrics"]` |

The Jaeger `serviceName` convention is the one inconsistency: everywhere else the platform writes
`orders-service`, but traces are tagged `orders`. Anyone joining traces to logs or metrics has to know
that.

## Service / Boundary Guidance

- **Mint identifiers once, at the edge, and never regenerate them.** A service that mints its own on
  receipt breaks the chain at exactly the boundary the chain exists to cross.
- **Propagate on publish, not just on receive.** The broker wrapper is the single place that does this,
  which is why every service routes publishing through it rather than calling the bus directly
  ([[service-owned-topic-exchange-messaging]]).
- **Let the fallback to the active span be a fallback.** When a message arrives with no span context,
  the wrapper uses the currently active span, which keeps a chain intact rather than starting a new
  root. That is correct for a consumer and wrong for a scheduled job, where the "active" span may belong
  to unrelated work.
- **Exclude health and metrics paths.** All three excluded paths are polled continuously; tracing them
  produces volume and no information.
- **A component that publishes without going through the wrapper appears as an orphan trace**, which is
  the observable symptom of the same gap that lets it bypass authorization.

## Security / Compliance Considerations

- **The correlation context carries the caller's identity and full claims dictionary** into every
  message header and, from there, into Jaeger and Seq. Anything placed in a token spreads with it.
- **Trace and request identifiers are returned to clients** through `exposedHeaders`. They are opaque,
  but they are also the key to a stored trace — anyone who can query Jaeger and holds a client's request
  id can read that request's internals.
- **Jaeger is reached over UDP on localhost with no authentication**, and the Jaeger UI in the compose
  stack is unauthenticated.
- **Seq is configured with a committed API key** (`logger.seq.apiKey`) identical in all eleven
  deployables. Recorded by path only; the value is not reproduced.
- **`sampler: const` means every request is traced and stored**, so trace storage becomes a durable
  record of user activity with no retention policy configured anywhere in these repositories.
- **Span logs include exception messages** in the one decorator that exists, so anything a service puts
  in an exception message lands in the trace.

## Observability Considerations

- **Constant sampling is a development setting.** At 100% it guarantees the trace you want exists and
  guarantees the cost scales linearly with traffic. It is the right default for local work and the wrong
  one for a busy environment.
- **Command spans exist in one service.** `availability-service` registers
  `JaegerCommandHandlerDecorator<T>`; the other six layered services register the outbox decorators only.
  So traces in six services show the broker hop but not the handler that did the work — the trace stops
  precisely where the interesting part starts.
- **Nothing traces query handlers** anywhere. Reads are invisible to tracing.
- **Metrics and traces share no identifier.** Metrics carry `env: local` and a service name; traces carry
  the short name. Correlating a latency spike in Grafana to a trace in Jaeger is manual.
- **No trace storage persists.** The compose stack's volume declarations for Jaeger, Seq, Prometheus and
  Grafana are commented out, so a restart discards every trace, log and metric collected.
- The correlation id doubles as the key of the operation projection, so an operator can move from a
  client-visible operation id to the trace that produced it — the single most useful property of this
  design.

## Failure Handling

- **No span context on an inbound message:** the wrapper falls back to the tracer's active span, so the
  chain continues rather than breaking.
- **No active span either:** the outgoing span context is null and the next hop starts a new root trace.
  The work still happens; only the linkage is lost.
- **Jaeger unreachable:** spans are reported over UDP, which does not confirm delivery, so a failure is
  silent and lossy by design. The service is unaffected.
- **Missing correlation id:** in `operations-service` the generic handlers return early rather than
  proceed — three separate `if (string.IsNullOrWhiteSpace(correlationId)) return;` guards — so a message
  without one is silently dropped from the projection.
- **Malformed context payload:** deserialisation yields nulls and the context degrades to empty; nothing
  reports it.

Every failure mode here degrades quietly. That is the right choice for telemetry — observability should
not take the system down — but it means the absence of traces is never itself an alert.

## Trade-offs

| Gain | Cost |
|------|------|
| One identifier follows a request across every hop, including asynchronous ones | Every publisher must go through the wrapper; anything that does not is invisible |
| Span context on the wire links spans across the broker | The link depends on a header surviving every intermediary |
| Correlation id doubles as the operation key, so clients can be told the outcome | The client-facing id and the internal trace id are the same value, so an opaque identifier is also a lookup key |
| `const` sampling means the trace always exists | Trace volume scales one-to-one with traffic, and storage has no retention policy |
| Health and metrics paths excluded, so noise stays out | Those paths are also where a slow start-up would show |
| Telemetry failures never affect the request | A telemetry outage is undetectable from the telemetry itself |

## Variants

- **Command spans via a decorator** (`availability-service`) versus **broker-level spans only** (the
  other six layered services). Both work; only the first shows handler timing.
- **`const` sampler** versus rate-based or probabilistic sampling. Only `const` is configured.
- **`udpHost: localhost`** in the host-mode configurations versus **`udpHost: jaeger`** in the container
  variants — the same pattern with an environment-substituted endpoint
  ([[composable-per-concern-environment-stacks]]).
- **Gateway-level tracing** through the Ntrada `tracing` extension rather than application code.

## Anti-patterns

- **Regenerating a correlation id mid-chain.** The empty-context fallback mints a fresh `RequestId`,
  which keeps logs attributable but produces an identifier that correlates with nothing — easy to mistake
  for a real trace.
- **Tracing the transport but not the work.** Six of seven layered services do exactly this: the span
  ends at the broker and the handler is dark.
- **Constant sampling carried from development into a real environment.**
- **Telemetry stores with their persistence commented out.** Every trace, log and metric is lost on
  restart, which is when they are most wanted.
- **A committed API key for the log store**, identical across eleven deployables.
- **Two naming schemes for the same service** — `orders` in traces, `orders-service` everywhere else.
- **Putting the caller's full claims into a header that fans out to every telemetry backend.**

## Evidence

- **ADR:** none — no ADR exists in any cloned repository or in the CAKE catalog for tenant `Q5SCXYFS`.
- **Repo:** all eleven deployable repositories.
- **Service:** `api-gateway` and all ten services; `ordermaker-service` participates in logging and
  metrics but not tracing.
- **File:**
  `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:58-64` — `tracing` block with
  `serviceName: api-gateway`, `udpPort: 6831`, `sampler: const`, `useEmptyTracer: false`; the same block
  at `ntrada.docker.yml:58-64` (`udpHost: jaeger`), `ntrada-async.yml:83-89`,
  `ntrada-async.docker.yml:83-89`; `ntrada.yml` also sets `generateRequestId: true`,
  `generateTraceId: true`, and `exposedHeaders: [Request-ID, Resource-ID, Trace-ID, Total-Count]`;
  `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Services/MessageBroker.cs:45-86`
  — resolution of `originatedMessageId`, `correlationId`, `spanContext` with the
  `_tracer.ActiveSpan.Context.ToString()` fallback, `GetHeadersToForward()`, and the correlation-context
  fallback to the HTTP context;
  `hianshul100_Pacco.Services.Availability/src/Pacco.Services.Availability.Infrastructure/Jaeger/JaegerCommandHandlerDecorator.cs:24-60`
  — `handling-<command>` span, `message-type` tag, `References.ChildOf` on the active span (`:50-53`),
  error tag and re-throw (`:36-41`), underscore-case naming (`:58-60`);
  `.../Availability.Infrastructure/Extensions.cs:9-12` (`AddJaegerDecorators`, the only registration of
  its kind), `:81` (`AddRabbitMq(plugins: p => p.AddJaegerRabbitMqPlugin())`), `:87-88`;
  `.AddJaeger()` at `Identity` Extensions.cs:84, `Vehicles` :69, `Operations` :74, and equivalently in
  `Deliveries`, `Customers`, `Orders`, `Parcels`, `Pricing`;
  `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Handlers/GenericCommandHandler.cs`
  — the `if (string.IsNullOrWhiteSpace(correlationId)) return;` guard, repeated in
  `GenericEventHandler.cs` and `GenericRejectedEventHandler.cs`;
  `hianshul100_Pacco.Services.Orders/src/Pacco.Services.Orders.Infrastructure/Contexts/CorrelationContext.cs:6-24`
  — the propagated shape.
- **API/Event:** every message published through `MessageBroker` carries `message_context` and
  `span_context`; the exchange and message inventory is in
  [`../../baselines/architecture-baseline.md`](../../baselines/architecture-baseline.md) §4.2.
- **Deployment/Config:** `jaeger` blocks in ten `appsettings.json` files with `sampler: const` and
  `excludePaths: ["/", "/ping", "/metrics"]`; **no `jaeger` block** in
  `hianshul100_Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker/appsettings.json`;
  `rabbitMq.context.header: "message_context"` and `rabbitMq.spanContextHeader: "span_context"` in every
  service; `hianshul100_Pacco/compose/grafana-seq-jaeger-prometheus.yml:26-40` —
  `jaegertracing/all-in-one` with its volume declaration commented out, as are those for Grafana,
  Prometheus and Seq.
- **Notes:** `architecture-baseline.md` §9.1–§9.2, §11.3.

## Related ADRs

None. No ADR, decision record, or RFC exists in the workspace, and the CAKE graph for tenant
`Q5SCXYFS` returned zero nodes.

## Related Patterns

- [[transport-agnostic-caller-context]] — the other half of the same header payload.
- [[structured-logging-with-property-redaction]] — where the correlation id lands in log output.
- [[service-owned-topic-exchange-messaging]] — the publish path that carries the context.
- [[acknowledge-then-notify-completion]] — keys its projection on the correlation id.
- [[declarative-configuration-driven-api-gateway]] — where the identifiers are minted.
- [[composable-per-concern-environment-stacks]] — where the telemetry backends are declared.

## Recommendation

**Adopt; it is the strongest observability decision in the platform.** Carrying a correlation id, a
trace id, and a span context through message headers is what makes an asynchronous platform debuggable
at all, and reusing the correlation id as the client-facing operation key is a genuinely good idea — it
gives an operator a direct path from a user's complaint to a trace. Three changes would make it deliver
on its shape: register the command-handler tracing decorator in the other six layered services, so
traces do not stop at the broker; replace the `const` sampler with a rate-based one before any
meaningful traffic; and either enable persistence on the telemetry stores or state plainly that
telemetry is ephemeral, because a restart currently discards everything collected.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The RabbitMQ Jaeger plugin reads the inbound `span_context` header and makes the consumer's span a child of the publisher's | Every service registers the plugin, and the header is written on publish, which only makes sense if something reads it | Traces would break at every broker hop, and the whole propagation chain would be per-service islands rather than one story | Publish a message through the gateway and check in Jaeger whether the consumer span appears under the gateway span or as a new root |
| A2 | Trace and log volume at `const` sampling is acceptable at the traffic this platform actually sees | All configuration reads as a local development stack — `localhost` endpoints, `env: local`, no retention settings | Trace storage would grow without bound and Jaeger would become the platform's largest data store | Measure spans per request and multiply by expected request volume, then compare to the Jaeger deployment's capacity |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should the command-handler tracing decorator be registered in the other six layered services? | Only `availability-service` has it. Everywhere else the trace shows the message arriving and then goes dark exactly where the work happens, which is the part anyone investigating actually wants | Yes — it is one line per service and the decorator already exists and can be copied verbatim | Platform owner, with the owners of the six layered services |
| Q2 | **[ACTION NOW]** What sampling rate should be used outside local development? | `const` traces everything. It is the reason a trace is always there and the reason storage grows linearly with traffic, and nothing in these repositories sets a retention policy | Rate-limited or probabilistic sampling with an explicit retention period; keep `const` for local work only | Platform owner, with the operator |
| Q3 | **[handled later by the design stage]** Should `ordermaker-service` report traces? | It is the only deployable with no `jaeger` block, and it is also the component that starts saga flows — so the traces that would explain a saga start at the second hop | Yes, if the service is meant to run at all. Its runtime status is itself unresolved | Platform owner |
| Q4 | **[handled later by the design stage]** Should the Jaeger `serviceName` match the `<name>-service` form used everywhere else? | Traces say `orders`, logs and metrics say `orders-service`, so joining across the three requires knowing the mapping by heart | Yes — align on one name; the mismatch has no upside | Platform owner |
| Q5 | **[handled later by the design stage]** Should telemetry stores persist across restarts? | Volume declarations for Jaeger, Seq, Prometheus and Grafana are all commented out, so every trace, log and metric is lost on restart — including the restart being investigated | Enable persistence for at least Seq and Jaeger; a restart is exactly when the history matters | Operator for the Pacco runtime |
