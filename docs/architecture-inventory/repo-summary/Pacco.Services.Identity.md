# Repository: `Pacco.Services.Identity`

`identity-service` (also known as: Identity Service, `Pacco.Services.Identity`, Docker image
`devmentors/pacco.services.identity`) is the platform's authentication origin: it owns users,
credentials, roles, JWT issuance and refresh-token lifecycle.

- **Repository:** `Pacco.Services.Identity`, path: `src/Pacco.Services.Identity.Api`
- **Base ref analysed:** `feature/12998/aidlc`
- **Port:** `5004`

---

## README vs repository

`README.md` is the platform boilerplate — logo, shared "What is Pacco?" paragraph, Travis badge,
generic start instructions. It names no entity, endpoint, event, collection or dependency of this
service.

**Claimed in README, present on disk (confirmed):** .NET Core 3.1; Travis CI; the
`scripts/start.sh` local run path.

**Present on disk, absent from README (disk-only):**

- That this repository is the platform's token issuer — nothing in the README indicates that every
  other service's security model depends on it.
- The four-project clean-architecture split.
- `Infrastructure/Auth/JwtProvider.cs`, `PasswordService.cs`, `Rng.cs` and the refresh-token
  mechanism.
- All seven HTTP endpoints and the RabbitMQ `identity` exchange with its six messages.
- MongoDB database `identity-service` and collections `users` and `refreshTokens`.
- The Redis-backed access-token revocation list (`UseAccessTokenValidator()`).
- The unique index created on `users` at startup — the only schema-shaping code in the workspace.
- `UsePublicContracts<ContractAttribute>()`, which publishes machine-readable contract metadata.

**Stale doc:** none identified.

**Unknown:** how this service's signing configuration relates to the symmetric
`issuerSigningKey` committed in the gateway's four `ntrada*.yml` files. Both sides validate
`validIssuer: pacco`, but the trust material is expressed differently here (`certs/localhost.cer`)
than at the edge (a symmetric key). **Needs validation.**

---

## 1. Primary purpose

Register users, verify credentials, issue and refresh JWT access tokens, revoke tokens, and expose
user records. It is the only service that mints the tokens every other component trusts.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP API **and** RabbitMQ consumer in one process.

**Distinguishing detail:** unlike every other service, `Program.cs` uses
`app.UseEndpoints(...)` with explicitly written endpoint delegates rather than Convey's
`UseDispatcherEndpoints`. Routes here are hand-wired because several of them need to read the
incoming token before dispatching (for example `GET me`, which calls
`ctx.AuthenticateUsingJwtAsync()` first).

## 3. Key entrypoints

- `src/Pacco.Services.Identity.Api/Program.cs` — composition root, route table, and the
  `AuthenticateUsingJwtAsync` / `GetCorrelationContext` helpers (the latter reads the
  `Correlation-Context` header).
- `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` — DI composition root.
- `src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs` — creates a unique index on the
  `users` collection at startup.
- `Dockerfile` — `ENTRYPOINT dotnet Pacco.Services.Identity.Api.dll`.
- `scripts/start.sh` — local run with `ASPNETCORE_ENVIRONMENT=local`.

## 4. Important modules / packages

| Project | Role |
|---|---|
| `Pacco.Services.Identity.Api` | Host, hand-written route table, configuration, `certs/` |
| `Pacco.Services.Identity.Application` | Commands (`SignUp`, `SignIn`, `RevokeAccessToken`, `UseRefreshToken`, `RevokeRefreshToken`), queries, events, DTOs, service interfaces (`IIdentityService`, `IRefreshTokenService`, `IPasswordService`, `IJwtProvider`, `IRng`) |
| `Pacco.Services.Identity.Core` | `Entities/User.cs`, `Entities/Role.cs`, `Entities/RefreshToken.cs`, repository interfaces, domain exceptions |
| `Pacco.Services.Identity.Infrastructure` | `Auth/JwtProvider.cs`, `Auth/PasswordService.cs`, `Auth/Rng.cs`; Mongo documents and repositories; RabbitMQ broker; decorators; contexts; logging |

**Key packages:** `Convey`, `Convey.CQRS.Commands/.Events/.Queries`, `Convey.Auth`,
`Convey.MessageBrokers.RabbitMQ`, `.MessageBrokers.Outbox`, `.MessageBrokers.Outbox.Mongo`,
`Convey.Persistence.MongoDB`, `.Persistence.Redis`, `Convey.Discovery.Consul`,
`Convey.LoadBalancing.Fabio`, `Convey.HTTP`, `Convey.Logging`, `Convey.Metrics.AppMetrics`,
`Convey.Tracing.Jaeger`, `.Tracing.Jaeger.RabbitMQ`, `Convey.Secrets.Vault`, `Convey.Security`,
`Convey.WebApi`, `.WebApi.CQRS`, `.WebApi.Security`, `.WebApi.Swagger`, plus
`Microsoft.AspNetCore.Identity` for `IPasswordHasher<>`.

**Registrations of note** (`Infrastructure/Extensions.cs`): `IJwtProvider → JwtProvider`,
`IPasswordService → PasswordService`, `IPasswordHasher<IPasswordService> → PasswordHasher<>`,
`IIdentityService`, `IRefreshTokenService`, `IRng → Rng`, `IMessageBroker → MessageBroker`,
`IAppContextFactory`; and the decoration of every `ICommandHandler<>` and `IEventHandler<>` with
`OutboxCommandHandlerDecorator<>` / `OutboxEventHandlerDecorator<>`.

## 5. External integrations

Consul (registration, `pingEndpoint: ping`), Fabio, RabbitMQ (exchange `identity`), MongoDB
(database `identity-service`), Redis (prefix `identity:` — backs the revoked-access-token list),
Vault (kv v2 path `identity-service/settings`, PKI role `identity-service`, common name
`identity-service.pacco.io`, MongoDB dynamic credentials), Jaeger, Seq, Prometheus.

**It calls no other service.** `httpClient.services` is empty.

## 6. Data stores / state

- **Store:** MongoDB, database `identity-service`; plus **Redis** for token revocation state.
- **Query mechanism:** Convey `IMongoRepository<UserDocument, Guid>` and
  `IMongoRepository<RefreshTokenDocument, Guid>` over the MongoDB .NET driver. **Not a relational
  ORM.**
- **Registrations:** `AddMongoRepository<RefreshTokenDocument, Guid>("refreshTokens")` and
  `AddMongoRepository<UserDocument, Guid>("users")` in
  `src/Pacco.Services.Identity.Infrastructure/Extensions.cs`.
- **Collections for the primary domain:** **`users`** (`Mongo/Documents/UserDocument.cs`) and
  **`refreshTokens`** (`Mongo/Documents/RefreshTokenDocument.cs`).
- **Framework collections:** `inbox`, `outbox` (`type: sequential`, `disableTransactions: true`).
- **Migration tool:** **none.** However, this is the only repository in the workspace with any
  schema-shaping code: `src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs` creates a
  unique index on the `users` collection at startup. That is index creation, not a migration
  framework — there is no versioning, no rollback and no history.
- **Cross-domain coupling:** none at the data layer. `RefreshTokenDocument` references a `UserId`
  within the same database. No other service reads this database, and this service replicates no
  other domain's data. The coupling it creates is *contractual*, not relational: the `UserId` it
  mints becomes the `CustomerId` that `customers-service` uses when handling `signed_up`, so a
  single identifier crosses the whole platform without any referential enforcement.

## 7. Messaging / async / events

- **Broker:** RabbitMQ. **Exchange:** `identity`, type `topic`, durable.
- **Conventions:** `snakeCase`; queue template `identity-service/{{exchange}}.{{message}}`;
  headers `message_context` and `span_context`.
- **Outbox:** enabled (`AddMessageOutbox(o => o.AddMongo())`) with outbox decorators on handlers.
- **Explicit subscription:** `UseRabbitMq().SubscribeCommand<SignUp>()` in `Program.cs` — sign-up
  can arrive as a message as well as an HTTP call, which is what the gateway's async mode relies
  on. Note that `sign_in` is declared in `messages.json` but **no `SubscribeCommand<SignIn>()`
  call exists**, and the gateway routes `POST /identity/sign-in` as an HTTP downstream in *both*
  modes — sign-in is deliberately synchronous. **Needs validation** that `sign_in` in the
  catalogue is therefore unreachable as a message.

**Commands consumed:** `sign_up` (via subscription), `sign_in` (declared in the catalogue only).

**Events published:**

| Event | Observable payload fields |
|---|---|
| `signed_up` | `UserId`, `Email`, `Role` |
| `signed_in` | `UserId` |

**Rejected events published:** `sign_up_rejected`, `sign_in_rejected` — each with `Reason` and
`Code`.

**External events consumed:** **none.**

**Consumers of this service's events:** `signed_up` → `customers-service`, where it creates the
customer record. This single edge is how a user becomes a customer. `signed_in` has **no domain
consumer**; only `operations-service` observes it.

## 8. APIs exposed / consumed

**Exposed** (from `src/Pacco.Services.Identity.Api/Program.cs`, verbatim):

| Method | Route | Behaviour |
|---|---|---|
| `GET` | `users/{userId}` | Fetch a user |
| `GET` | `me` | Reads the caller's own record via `ctx.AuthenticateUsingJwtAsync()` |
| `POST` | `sign-in` | Issues an access token and a refresh token |
| `POST` | `sign-up` | Registers a user |
| `POST` | `access-tokens/revoke` | Adds the access token to the Redis revocation list |
| `POST` | `refresh-tokens/use` | Exchanges a refresh token for a new access token |
| `POST` | `refresh-tokens/revoke` | Invalidates a refresh token |

Swagger UI at route prefix `docs`.

**Consumed:** none.

**Inbound synchronous callers:** none from other services. The gateway is the only caller.

**Upstream:** the gateway module `identity` fronts `GET /users/{userId}` (**role `admin`**),
`GET /me`, `POST /sign-up` and `POST /sign-in`. All four are HTTP `downstream` calls in **both**
sync and async gateway configurations — authentication is never fire-and-forget.
The three token-management routes (`access-tokens/revoke`, `refresh-tokens/use`,
`refresh-tokens/revoke`) have **no gateway route at all** and are therefore not reachable from
outside. **Needs validation.**

## 9. Deployment / runtime clues

- `Dockerfile`: multi-stage `sdk:3.1` → `aspnet:3.1`; `ASPNETCORE_URLS http://*:80`;
  `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Identity.Api.dll`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master`/`develop`, `./scripts/build.sh`,
  `after_success: ./scripts/dockerize.sh` → `$DOCKER_USERNAME/pacco.services.identity`.
- Port `5004` in `Pacco/prod-services.yml`, `Pacco/compose/services.yml` (`5004:80`), and the
  gateway `localUrl`.
- Consul service name `identity-service`; `httpClient.type: fabio`.
- Environments: `appsettings.json`, `.local.json`, `.docker.json`.
- **Runtime dependency to note:** because token revocation lives in Redis, a Redis outage does not
  stop token *issuance* but does affect revocation checking. **Needs validation** of the failure
  mode.

## 10. Security / auth clues

This is the security-critical repository of the platform.

- **Token issuance:** `Infrastructure/Auth/JwtProvider.cs` mints access tokens; configuration sets
  `validIssuer: pacco` and points at `certs/localhost.cer`.
- **Password handling:** `Infrastructure/Auth/PasswordService.cs` wraps ASP.NET Core's
  `IPasswordHasher<>` — passwords are hashed with the framework's algorithm, not a hand-rolled one.
- **Randomness:** `Infrastructure/Auth/Rng.cs` generates refresh-token material.
- **Revocation:** `app.UseAccessTokenValidator()` in `Program.cs` gives Redis-backed access-token
  blacklisting, so a revoked token stops working before it expires.
- **Refresh tokens:** persisted in the `refreshTokens` collection with use and revoke commands.
- **Roles:** `Core/Entities/Role.cs`; the `admin` role is what the gateway's `claims.role: admin`
  gates check.
- **Public contracts:** `app.UsePublicContracts<ContractAttribute>()` exposes marked message
  contracts over HTTP for consumers to discover.
- **Vault:** kv v2 settings, PKI role `identity-service`, MongoDB dynamic credentials with lease
  auto-renewal.
- **Log masking:** `logger.excludeProperties` explicitly removes password and token properties —
  materially important in this service specifically.
- **Observation:** the trust material differs between this service (a certificate file) and the
  gateway (a committed symmetric `issuerSigningKey`). Whether they describe the same trust root is
  **Unknown**; see Blockers.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: identity`, UDP `6831`, `const` sampler rate 1, with the
  `Convey.Tracing.Jaeger.RabbitMQ` plugin propagating `span_context` across AMQP.
- **Correlation:** `GetCorrelationContext` in `Program.cs` reads the `Correlation-Context` header
  set by the gateway.
- **Logging:** console, file and Seq sinks enabled; ELK sink present but `enabled: false`; the
  `excludeProperties` masking list is the platform's standard set.
- **Metrics:** App.Metrics with `prometheusEnabled: true`, `influxEnabled: false`, database
  `pacco`; `/metrics` and `/metrics-text`.

## 12. Architecture-decision files and feature flags

| File | Decision it records |
|---|---|
| `Pacco.Services.Identity.sln` | Four-project clean-architecture split |
| `src/Pacco.Services.Identity.Api/Program.cs` | That this service alone uses hand-written `UseEndpoints` rather than Convey dispatcher endpoints, because routes must inspect the token before dispatching; also that sign-up is subscribable as a command while sign-in is not |
| `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` | The full capability chain and the outbox decorators; the choice of `IPasswordHasher<>` from ASP.NET Core Identity |
| `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs` | The token shape every other service and the gateway depend on |
| `src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs` | That schema constraints are applied as startup index creation rather than by a migration tool — the only such code in the workspace |
| `src/Pacco.Services.Identity.Api/appsettings.json` | JWT issuer and certificate, Vault PKI, outbox with `disableTransactions: true` |

**Feature flag system:** **none detected.** No flag library or in-house toggle mechanism appears in
the code or configuration, so **there are no flag keys to list**.

## 13. Open questions / ambiguities

1. Whether the gateway's symmetric signing key and this service's certificate describe the same
   trust root.
2. Why the three token-management routes have no gateway route — how a client revokes or refreshes
   a token from outside.
3. Whether `sign_in` in `messages.json` is reachable, given no `SubscribeCommand<SignIn>()`.
4. Why `signed_in` has no domain consumer.
5. The failure mode when Redis is unavailable and revocation cannot be checked.
6. Whether `outbox.disableTransactions: true` is deliberate.

## 14. Frontend stack

**No frontend assets detected — checked:** `src/Pacco.Services.Identity.Api/` (contains only
`certs/`, `Properties/` and configuration files), `src/Pacco.Services.Identity.Application/`,
`src/Pacco.Services.Identity.Core/`, `src/Pacco.Services.Identity.Infrastructure/`, and the
repository root. There is no `wwwroot/`, `public/`, `public/js/`, `static/`, `assets/`,
`resources/js/`, or `web/` directory; no `package.json` or bundler configuration; and no view
templates (`.cshtml`, `.html`, Razor) — notably, there is no hosted login page: sign-in is a JSON
API only. The single browser-facing surface is the Convey Swagger UI at `/docs`.

---

## Evidence

| Fact | File |
|---|---|
| Route table, `UseEndpoints`, JWT helper, correlation header, `SubscribeCommand<SignUp>()`, `UseAccessTokenValidator()`, `UsePublicContracts` | `src/Pacco.Services.Identity.Api/Program.cs` |
| DI composition, service registrations, capability chain, outbox decorators, Mongo repository registrations | `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` |
| Startup index creation on `users` | `src/Pacco.Services.Identity.Infrastructure/Mongo/Extensions.cs` |
| Token issuance, password hashing, random generation | `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs`, `Auth/PasswordService.cs`, `Auth/Rng.cs` |
| Domain entities and roles | `src/Pacco.Services.Identity.Core/Entities/User.cs`, `Entities/Role.cs`, `Entities/RefreshToken.cs` |
| Persistence documents | `src/Pacco.Services.Identity.Infrastructure/Mongo/Documents/UserDocument.cs`, `RefreshTokenDocument.cs` |
| Commands | `src/Pacco.Services.Identity.Application/Commands/SignUp.cs`, `SignIn.cs`, and the token commands |
| Published events and payloads | `src/Pacco.Services.Identity.Application/Events/SignedUp.cs`, `SignedIn.cs` |
| Rejected events | `src/Pacco.Services.Identity.Application/Events/Rejected/*.cs` |
| JWT, Vault, exchange, outbox, logging, metrics, tracing configuration | `src/Pacco.Services.Identity.Api/appsettings.json`, `appsettings.local.json`, `appsettings.docker.json` |
| Signing certificate | `src/Pacco.Services.Identity.Api/certs/localhost.cer` |
| Package set | `src/Pacco.Services.Identity.Infrastructure/Pacco.Services.Identity.Infrastructure.csproj`, `src/Pacco.Services.Identity.Api/Pacco.Services.Identity.Api.csproj` |
| Project list | `Pacco.Services.Identity.sln` |
| Container build and CI | `Dockerfile`, `.travis.yml`, `scripts/build.sh`, `scripts/test.sh`, `scripts/start.sh`, `scripts/dockerize.sh` |
| The one consumer of `signed_up` | `../hianshul100_Pacco.Services.Customers/src/Pacco.Services.Customers.Application/Events/External/Handlers/` |
| Edge JWT configuration and admin gate | `../hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml` |
| Message catalogue cross-check | `../hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | Every token accepted anywhere on the platform was issued by this service | It is the only component with token-minting code, and every service configures `validIssuer: pacco` | If another issuer is trusted, the platform's trust boundary is wider than described here | Confirm no other issuer is configured in any deployed environment |
| A2 | The user identifier minted here is the same identifier used as `CustomerId` throughout the platform | `signed_up` carries `UserId`, and `customers-service` creates its customer record from it; the gateway then binds `@user_id` into downstream customer, order, parcel and pricing calls | Identity would not resolve consistently across services, and the gateway's `@user_id` rewriting would fetch the wrong records | Trace one sign-up through to an order in a running environment |
| A3 | Redis-backed revocation is checked on every authenticated request | `UseAccessTokenValidator()` is in the pipeline and Redis is configured with the `identity:` prefix | Revoked tokens would keep working until natural expiry, so signing a user out would not actually stop them | Read the Convey 0.4 access-token validator, or revoke a token and retry it against a running instance |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** The API gateway validates tokens using a symmetric signing key committed in plaintext in four configuration files, while this service is configured with a certificate. Whether these are the same trust root cannot be settled from the code, and if the committed key does sign valid tokens, anyone holding it can mint an `admin` token | Any statement that the platform's authentication is sound, and any later work that depends on the token model | Whoever owns Pacco authentication | Establish which key material actually signs and validates tokens in each environment. If the committed symmetric key is live, rotate it, move it to Vault, and purge it from git history | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** How does a client revoke or refresh a token from outside the platform? | This service exposes `access-tokens/revoke`, `refresh-tokens/use` and `refresh-tokens/revoke`, but the gateway routes none of them. As far as the configuration shows, a signed-in user cannot sign out or renew a session through the public edge | Either gateway routes are missing, or these operations are intended for internal use only | Whoever owns Pacco authentication |
| Q2 | **[ACTION NOW]** Is `sign_in` reachable as a message, as `messages.json` claims? | The catalogue declares `sign_in` as an identity command, but this service subscribes only to `SignUp`, and the gateway sends sign-in over HTTP in both modes. A published `sign_in` command would go nowhere | Sign-in appears to be deliberately synchronous, and the catalogue entry is likely aspirational | Domain owner for Identity |
| Q3 | **[ACTION NOW]** Should anything consume `signed_in`? | The event is published on every successful sign-in and no service listens. If session tracking, fraud signals or last-seen data were intended, none of it exists | Either a consumer is missing, or the event is emitted purely for the operations feed | Domain owner for Identity |
| Q4 | **[handled later by HLD]** What happens to authentication when Redis is unavailable? | Revocation state lives only in Redis. Depending on how the validator fails, an outage either rejects every request or silently accepts revoked tokens — and neither behaviour is written down | Determine and document the intended failure mode | Platform architect |
| Q5 | **[handled later by HLD]** Is startup index creation the intended substitute for schema migrations? | This is the only place in the workspace where any schema constraint is applied. It runs on every start, has no version history, and cannot be rolled back — so the uniqueness guarantee on `users` depends on a startup side effect | Confirm this is the accepted approach or introduce an explicit schema-management step | Platform architect |
