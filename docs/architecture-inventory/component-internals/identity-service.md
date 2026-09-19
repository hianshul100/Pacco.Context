# Component internals — `identity-service`

| | |
| --- | --- |
| **Component** | `identity-service` |
| **Source repository** | `hianshul100_Pacco.Services.Identity` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository) |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 3 of 7 |
| **Status** | New artifact — no prior `component-internals/identity-service.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `baselines/service-summaries.md` and `baselines/api-inventory.md` remain valid and are **complemented**, not replaced: those catalogue the surface, this document models the internals. Two baseline open questions are resolved here (see §8.3) and one baseline count is corrected. |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a member or line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains the service's own source in full — four
> projects, `src/Pacco.Services.Identity.{Core,Application,Infrastructure,Api}`, confirmed by
> `find src -name '*.csproj'`. It contains **no test project of any kind**: there is no `tests/`
> directory and no `*.Tests.csproj`, yet `scripts/test.sh` and `.travis.yml` still invoke
> `dotnet test` (§3.40). `Convey 0.4.*` — which supplies the CQRS dispatchers, the JWT handler,
> the access-token store, the Mongo repository, the RabbitMQ client, the outbox and the WebApi
> endpoint mapping — is a NuGet reference with **no source in this workspace**. Mechanisms owned by
> Convey are marked `[convey]` and, where their exact semantics change a conclusion, flagged
> `Unverifiable — Missing Source Evidence`. ASP.NET Core primitives (notably
> `Microsoft.AspNetCore.Identity.PasswordHasher<T>`) are marked `[framework]` on the same basis.
> The upstream half of every inbound HTTP contract is modelled in
> `component-internals/api-gateway.md`; the downstream consumer of `SignedUp` is modelled in
> `component-internals/customers-service.md`.

---

## Contents

1. [Purpose & boundary](#1-purpose--boundary)
2. [Core concepts (exhaustive)](#2-core-concepts-exhaustive)
3. [Per concept](#3-per-concept)
4. [Primary control flows](#4-primary-control-flows)
5. [Persistence & schema evolution](#5-persistence--schema-evolution)
6. [Surface → internals map](#6-surface--internals-map)
7. [Change/extension guide](#7-changeextension-guide)
8. [Assumptions, Blockers & Open Questions](#8-assumptions-blockers--open-questions)

---

## 1. Purpose & boundary

### 1.1 What this component is responsible for

`identity-service` is the platform's **credential authority and token mint**. It owns exactly two
aggregates — `User` and `RefreshToken` — and it is the only component in the Pacco workspace that
can turn an email/password pair into a signed JWT. Every other service in the platform *consumes*
those tokens (validating them with a shared key) but none can *issue* one.

Four properties distinguish it from its siblings:

1. **It is a transaction script, not a domain-event-sourced aggregate.** Unlike
   `availability-service`, `deliveries-service` and `orders-service`, whose aggregates call
   `AddEvent(...)` and whose infrastructure drains that buffer, `identity-service` inherits the
   same `AggregateRoot` base (`…Core/Entities/AggregateRoot.cs:5-18`) but **never calls
   `AddEvent` anywhere in the repository**. All business logic lives in
   `IdentityService`/`RefreshTokenService` and integration events are published imperatively
   (§3.3, §3.14).
2. **Its HTTP write path bypasses the command dispatcher.** `Program.cs:47-71` registers raw
   `Post<TCommand>(path, handler)` lambdas that resolve `IIdentityService` /
   `IRefreshTokenService` / `IAccessTokenService` from `ctx.RequestServices` and call them
   directly. It never calls `UseDispatcherEndpoints`, so the `OutboxCommandHandlerDecorator`
   (`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs`) — and its inbox-based
   idempotency — **never runs for an HTTP request** (§3.26, §3.30, §4.1).
3. **It has exactly one AMQP inbound message and zero AMQP producers for it in this workspace.**
   `UseInfrastructure` ends with `.SubscribeCommand<SignUp>()`
   (`…Infrastructure/Extensions.cs:103`), but the API gateway routes `POST /identity/sign-up`
   with `use: downstream` in **both** the sync and the async profile
   (`Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:256-261` and `ntrada-async.yml`), so no
   in-workspace component publishes `sign_up`. The subscription is live but unreachable (§3.29).
4. **It is a write-mostly store with no update path.** `IUserRepository`
   (`…Core/Repositories/IUserRepository.cs:7-12`) declares `GetAsync(Guid)`, `GetAsync(string)`
   and `AddAsync` — **no `UpdateAsync`, no `DeleteAsync`**. There is consequently no password
   change, no password reset, no email change, no role change and no account deletion anywhere in
   the service (§3.21, §7.6).

| Responsibility | Where it lives |
| --- | --- |
| Model a user as id + email + role + password hash + created-at + permissions | `src/Pacco.Services.Identity.Core/Entities/User.cs:8-41` |
| Enforce the closed role vocabulary `{user, admin}` | `…Core/Entities/Role.cs:3-19`; `…Core/Entities/User.cs:29-32` |
| Normalise email and role to lower case on construction | `…Core/Entities/User.cs:35,37` |
| Validate email shape before any repository hit | `…Application/Services/Identity/IdentityService.cs:18-21,51-55,87-91` |
| Hash and verify passwords | `…Infrastructure/Auth/PasswordService.cs:15-19` over `[framework]` `PasswordHasher<T>` |
| Mint access tokens (JWT) with role + optional `permissions` claim | `…Infrastructure/Auth/JwtProvider.cs:18-29` over `[convey]` `IJwtHandler` |
| Generate cryptographically-random refresh tokens | `…Infrastructure/Auth/Rng.cs:12-22` (`RNGCryptoServiceProvider`) |
| Model a refresh token as id + user id + token + created-at + revoked-at | `…Core/Entities/RefreshToken.cs:6-42` |
| Exchange a refresh token for a fresh access token | `…Application/Services/Identity/RefreshTokenService.cs:50-79` |
| Revoke a refresh token (one-way) | `…Core/Entities/RefreshToken.cs:33-41`; `…Application/…/RefreshTokenService.cs:38-48` |
| Revoke an access token (Redis deny-list) | `Program.cs:57-61` over `[convey]` `IAccessTokenService.DeactivateAsync` |
| Reject requests bearing a revoked access token | `…Infrastructure/Extensions.cs:97` (`UseAccessTokenValidator()`) `[convey]` |
| Accept 5 commands over HTTP and 1 (`SignUp`) over AMQP | `Program.cs:47-71`; `…Infrastructure/Extensions.cs:103` |
| Answer 2 read routes (`users/{userId}`, `me`) | `Program.cs:35-46,78-88` |
| Publish `SignedIn` / `SignedUp` integration events | `…Application/…/IdentityService.cs:80,106`; `…Infrastructure/Services/MessageBroker.cs:47-86` |
| Translate exceptions into HTTP responses **and** into rejected events | `…Infrastructure/Exceptions/ExceptionTo{Response,Message}Mapper.cs` |
| Persist users and refresh tokens in MongoDB (`identity-service` DB) | `…Infrastructure/Extensions.cs:85-86`; `…Api/appsettings.json:100-104` |
| Create the sole schema artefact on the platform: a unique index on `users.Email` | `…Infrastructure/Mongo/Extensions.cs:13-28` |
| Publish through a Mongo-backed transactional outbox | `…Infrastructure/Extensions.cs:67-68,80`; `…Api/appsettings.json:105-113` |
| Expose its `[Contract]`-marked message types for other services to read | `…Infrastructure/Extensions.cs:99` (`UsePublicContracts<ContractAttribute>()`) `[convey]` |
| Register with Consul, emit Jaeger spans, expose Prometheus metrics | `…Infrastructure/Extensions.cs:76-77,83-84`; `…Api/appsettings.json:7-22,82-99` |
| Fetch secrets and dynamic Mongo credentials from Vault | `Program.cs:74` (`UseVault`); `…Api/appsettings.json:169-198` |

### 1.2 What this component explicitly is **not**

- **Not an authorizer.** `UseAuthentication()` is called (`…Infrastructure/Extensions.cs:101`) but
  `UseAuthorization()` is **never** called, and no `[Authorize]` attribute exists in the repository
  (there are no controllers at all). The only place this service checks a caller's identity is the
  `me` route, which calls `AuthenticateUsingJwtAsync()` by hand (`Program.cs:38`). In particular
  **`GET users/{userId}` performs no ownership or role check inside the service** — the
  `role: admin` claim requirement exists only at the gateway
  (`Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:243-246`). §3.27, §8.2/B2.
- **Not the owner of the customer profile.** It stores email, role and permissions and nothing
  else. Name, address and VIP status live in `customers-service`, which is bootstrapped by the
  `SignedUp` event (§6.5).
- **Not a session store.** There is no server-side session; the only server-side auth state is the
  Redis access-token deny-list `[convey]` and the `refreshTokens` Mongo collection.
- **Not multi-tenant, not federated, not OAuth.** No client ids, no scopes, no external identity
  providers, no consent, no PKCE. `SignIn` is email + password only
  (`…Application/Commands/SignIn.cs:6-16`).
- **Not a rate limiter.** Nothing in this repository throttles `sign-in`; there is no lockout
  counter, no failed-attempt tracking and no CAPTCHA. Repeated invalid-credential attempts are
  logged (`…Application/…/IdentityService.cs:60`) and otherwise unbounded. §3.14, §8.2/B3.
- **Not a token *validator* for other services.** Every sibling service validates JWTs itself with
  its own `AddJwt()` and the same `issuerSigningKey`; nothing calls back into `identity-service` to
  introspect a token. The one exception is revocation, which is only honoured by services that call
  `UseAccessTokenValidator()` — and `operations-service` does not (see
  `component-internals/operations-service.md` §3.24).

### 1.3 The dual-transport boundary

| | HTTP (`Program.cs`) | AMQP (`…Infrastructure/Extensions.cs:103`) |
| --- | --- | --- |
| Entry point | `WebHost` → Convey `UseEndpoints` `[convey]` | `IBusSubscriber.SubscribeCommand<SignUp>()` `[convey]` |
| Messages accepted | `SignIn`, `SignUp`, `RevokeAccessToken`, `UseRefreshToken`, `RevokeRefreshToken` | `SignUp` only |
| Reads accepted | `GetUser` (`users/{userId}`), `me`, `""` | none |
| Goes through `ICommandDispatcher`? | **No** — direct service calls (`Program.cs:47-71`) | **Yes** — Convey resolves `ICommandHandler<SignUp>` → `SignUpHandler` |
| Goes through the outbox decorator? | **No** (§3.30) | **Yes** (`TryDecorate` at `…Infrastructure/Extensions.cs:67`) |
| Failure surface | `ExceptionToResponseMapper` → HTTP 400 (§3.19) | `ExceptionToMessageMapper` → rejected event, or silent drop (§3.18) |
| Idempotent on retry? | No | Yes, via the outbox inbox `[convey]`, keyed on `messageId` |

### 1.4 Position in the platform

| Serial Number | Fact | Evidence |
| --- | --- | --- |
| 1 | Listens on `5004:80` in Docker Compose | `hianshul100_Pacco/compose/services.yml:42-49` |
| 2 | PM2 app name `identity`, run from `…Identity.Api` | `hianshul100_Pacco/services.yml:18-21` |
| 3 | Reachable only through the gateway prefix `/identity` | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:237-289` |
| 4 | Only 4 of its 8 HTTP routes have a gateway route | §6.2 — the three token-lifecycle routes and `""` are unrouted |
| 5 | Owns the RabbitMQ topic exchange `identity` | `…Api/appsettings.json:136-142` |
| 6 | Its `SignedUp` event has exactly one consumer, `customers-service` | grep of `SignedUp`/`signed_up` across all 14 clones |
| 7 | Its `SignedIn` event has **no** consumer | same grep — `operations-service`'s `messages.json:66-69` subscribes `signed_in` generically, but only to render a UI notification (§6.5) |
| 8 | Scraped by Prometheus as a static target `identity-service` | `hianshul100_Pacco/compose/prometheus/prometheus.yml:26-28` |
| 9 | It is **not** in any other service's `depends_on`; `operations-service` depends on it | `hianshul100_Pacco/compose/services.yml:59-67` |

---

## 2. Core concepts (exhaustive)

Every named idea a maintainer must hold in their head to change this service safely. "Owner" is
the layer that defines the concept; "Modelled in" is the authoritative file. Concepts marked
**(dead)** are present in source but never executed on any reachable path — they are listed because
they *look* load-bearing and will mislead a reader who omits them.

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | `User` aggregate | Core | `…Core/Entities/User.cs:8-41` |
| 2 | `AggregateId` value object + implicit `Guid` conversions | Core | `…Core/Entities/AggregateId.cs:6-50` |
| 3 | `AggregateRoot` domain-event buffer **(dead)** | Core | `…Core/Entities/AggregateRoot.cs:5-18` |
| 4 | `Role` — the closed vocabulary `{user, admin}` | Core | `…Core/Entities/Role.cs:3-19` |
| 5 | `Permissions` — the open, unvalidated vocabulary | Core | `…Core/Entities/User.cs:14,39` |
| 6 | Email canonicalisation (lower-casing) and the `EmailRegex` gate | Core + Application | `…Core/Entities/User.cs:35`; `…Application/…/IdentityService.cs:18-21` |
| 7 | Password hashing (`IPasswordService` over `PasswordHasher<T>`) | Application + Infrastructure | `…Application/Services/IPasswordService.cs`; `…Infrastructure/Auth/PasswordService.cs:15-19` |
| 8 | `RefreshToken` aggregate and its one-way revocation | Core | `…Core/Entities/RefreshToken.cs:6-42` |
| 9 | `Rng` — the cryptographic token generator and its special-char stripping | Infrastructure | `…Infrastructure/Auth/Rng.cs:8-23` |
| 10 | Access-token issuance and the `"N"` subject format | Infrastructure | `…Infrastructure/Auth/JwtProvider.cs:18-29` |
| 11 | `AuthDto` — the wire shape of a successful authentication | Application | `…Application/DTO/AuthDto.cs:3-9` |
| 12 | Access-token revocation (Redis deny-list) | `[convey]` + Api | `Program.cs:57-61`; `…Infrastructure/Extensions.cs:82,97` |
| 13 | Refresh-token lifecycle: create → use (no rotation) → revoke | Application | `…Application/…/RefreshTokenService.cs:29-79` |
| 14 | `IdentityService` — the sign-in/sign-up transaction script | Application | `…Application/…/IdentityService.cs:42-107` |
| 15 | The five commands and the `[Contract]` marker | Application | `…Application/Commands/*.cs`; `…Application/ContractAttribute.cs` |
| 16 | Integration events `SignedIn`, `SignedUp` | Application | `…Application/Events/{SignedIn,SignedUp}.cs` |
| 17 | Rejected events `SignInRejected`, `SignUpRejected` | Application | `…Application/Events/Rejected/*.cs` |
| 18 | `ExceptionToMessageMapper` — the AMQP failure path (contains a type-match defect) | Infrastructure | `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:11-24` |
| 19 | `ExceptionToResponseMapper` — the HTTP failure path (everything is 400) | Infrastructure | `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:15-45` |
| 20 | The exception taxonomy: `DomainException` / `AppException` and the `Code` string | Core + Application | `…Core/Exceptions/*.cs`; `…Application/Exceptions/AppException.cs` |
| 21 | `IUserRepository` / `UserRepository` — an append-and-read-only store | Core + Infrastructure | `…Core/Repositories/IUserRepository.cs:7-12`; `…Infrastructure/Mongo/Repositories/UserRepository.cs` |
| 22 | `IRefreshTokenRepository` / `RefreshTokenRepository` — lookup by an **unindexed** field | Core + Infrastructure | `…Core/Repositories/IRefreshTokenRepository.cs:6-11`; `…Infrastructure/Mongo/Repositories/RefreshTokenRepository.cs:19-28` |
| 23 | Mongo documents and the `AsEntity` / `AsDocument` / `AsDto` mapping seam | Infrastructure | `…Infrastructure/Mongo/Documents/{UserDocument,RefreshTokenDocument,Extensions}.cs` |
| 24 | The startup unique index on `users.Email` — fire-and-forget | Infrastructure | `…Infrastructure/Mongo/Extensions.cs:13-28` |
| 25 | `GetUser` query + `GetUserHandler` **(dead — never dispatched)** | Application + Infrastructure | `…Application/Queries/GetUser.cs`; `…Infrastructure/Mongo/Queries/Handlers/GetUserHandler.cs` |
| 26 | Raw endpoint registration instead of `UseDispatcherEndpoints` | Api | `Program.cs:33-72` |
| 27 | `AuthenticateUsingJwtAsync` — the only in-service identity check | Infrastructure | `…Infrastructure/Extensions.cs:108-113` |
| 28 | `UseAccessTokenValidator()` and `jwt.allowAnonymousEndpoints` | `[convey]` + config | `…Infrastructure/Extensions.cs:97`; `…Api/appsettings.json:44` |
| 29 | `SubscribeCommand<SignUp>()` — the AMQP write path and `SignUpHandler` | Infrastructure + Application | `…Infrastructure/Extensions.cs:103`; `…Application/Commands/Handlers/SignUpHandler.cs` |
| 30 | The outbox decorators (`TryDecorate`) and the enabled/disabled matrix | Infrastructure | `…Infrastructure/Decorators/Outbox*Decorator.cs`; `…Infrastructure/Extensions.cs:67-68` |
| 31 | `MessageBroker` — the publish seam, message ids and header propagation | Infrastructure | `…Infrastructure/Services/MessageBroker.cs:45-86` |
| 32 | Correlation context: the `Correlation-Context` header, `IAppContext`, `IIdentityContext` **(built but never read)** | Infrastructure | `…Infrastructure/Extensions.cs:115-118`; `…Infrastructure/Contexts/*.cs` |
| 33 | `Saga` header forwarding — the only header this service propagates | Infrastructure | `…Infrastructure/Extensions.cs:120-134` |
| 34 | Span context propagation (`span_context`) and Jaeger | Infrastructure | `…Infrastructure/Extensions.cs:136-149`; `…Infrastructure/Services/MessageBroker.cs:57-61` |
| 35 | `UsePublicContracts<ContractAttribute>()` — machine-readable contract publication | `[convey]` | `…Infrastructure/Extensions.cs:99` |
| 36 | Configuration layering: `appsettings.json` + `.local` + `.docker` | Api | `…Api/appsettings*.json` |
| 37 | Vault: KV settings, PKI, and the dynamic Mongo lease | Api | `Program.cs:74`; `…Api/appsettings.json:169-198` |
| 38 | Service discovery (Consul), load balancing (Fabio), metrics (AppMetrics) | Infrastructure | `…Infrastructure/Extensions.cs:76-77,83` |
| 39 | Deployment topology: Dockerfile, Compose, PM2, Travis | repo root + `hianshul100_Pacco` | `Dockerfile`; `.travis.yml`; `scripts/*.sh` |
| 40 | The absence of tests, and the CI that pretends otherwise | repo root | `.travis.yml`; `scripts/test.sh` |

---

## 3. Per concept

Each subsection follows the same six-part shape: **Definition** — what the concept *is*;
**Representation & storage** — how it is held in memory and on disk; **Lifecycle** — how it is
created, mutated and destroyed; **Invariants & enforcement** — what must always hold and whether a
violation *fails loudly* (throws/returns an error the caller sees) or *fails silently* (is swallowed,
defaulted or ignored); **Extension procedure** — the exact edit sequence to add a new instance;
**Failure modes** — what actually goes wrong.

### 3.1 `User` aggregate

**Definition.** The credential record. A `User` is `(AggregateId Id, string Email, string Role,
string Password, DateTime CreatedAt, IEnumerable<string> Permissions)`
(`…Core/Entities/User.cs:10-14`). `Password` holds a **hash**, never a plaintext — the name is a
long-standing trap for readers; the value stored is whatever `PasswordService.Hash` returned
(§3.7).

**Representation & storage.** In memory: a `User` extending `AggregateRoot`, all setters `private`,
constructed only through the single public constructor (`…Core/Entities/User.cs:16-40`). On disk: a
`UserDocument` (`…Infrastructure/Mongo/Documents/UserDocument.cs:7-15`) in the `users` collection of
the `identity-service` database (`…Infrastructure/Extensions.cs:86`; `…Api/appsettings.json:100-104`).
`UserDocument.Id` is a `Guid` and is the Mongo `_id` `[convey]` — `IIdentifiable<Guid>` is the marker
Convey's `IMongoRepository<TDocument,TIdentifiable>` keys on. The two representations are bridged by
`AsEntity` / `AsDocument` (`…Infrastructure/Mongo/Documents/Extensions.cs:9-22`) and, for reads that
skip the domain, `AsDto` (`:24-32`).

**Lifecycle.** Create-only. `new User(...)` happens in exactly one place —
`IdentityService.SignUpAsync` (`…Application/…/IdentityService.cs:102`) — and is immediately
persisted by `_userRepository.AddAsync(user)` (`:103`). There is **no update and no delete path at
all** (§3.21): `IUserRepository` does not declare them, `UserRepository` does not implement them,
and nothing calls the underlying Convey repository's update/delete directly. A `User` row is
therefore immortal and immutable from the moment of sign-up.

**Invariants & enforcement.**

| Invariant | Enforced at | Loud or silent |
| --- | --- | --- |
| Email is non-blank | `…Core/Entities/User.cs:19-22` → `InvalidEmailException` | **Loud** |
| Password (hash) is non-blank | `:24-27` → `InvalidPasswordException` | **Loud** |
| Role ∈ `{user, admin}` | `:29-32` via `Role.IsValid` → `InvalidRoleException` | **Loud** |
| Email is stored lower-cased | `:35` `email.ToLowerInvariant()` | **Silent** (normalisation, not rejection) |
| Role is stored lower-cased | `:37` | **Silent** |
| `Permissions` is never null | `:39` `permissions ?? Enumerable.Empty<string>()` | **Silent** |
| Email is unique | **Not enforced by the aggregate.** Enforced twice, weakly: a read-then-write check in `IdentityService.SignUpAsync:93-98` (a TOCTOU race) and a Mongo unique index that may or may not exist (§3.24) | Loud at the app check, **loud-but-unmapped** at the index (§3.19) |
| Email is *syntactically* an email | **Not enforced by the aggregate** — only by `EmailRegex` in the application service (§3.6). `new User(id, "not-an-email", …)` succeeds | — |
| `Id` is non-empty | `AggregateId(Guid)` (`…Core/Entities/AggregateId.cs:15-23`) throws `InvalidAggregateIdException` on `Guid.Empty`; reached via the implicit conversion at `User.cs:34` | **Loud** |

Note the asymmetry: the *aggregate* is a weak validator (blankness + role), and the *application
service* is the real one (regex + uniqueness). Anyone reconstituting a `User` from a document
(`AsEntity`, `…Mongo/Documents/Extensions.cs:9-11`) re-runs the aggregate's three checks, so a
document that was written with a now-invalid role would throw on read, not on write.

**Extension procedure — adding a field to `User`.**

1. Add the property with a `private set` to `…Core/Entities/User.cs:10-14` and a constructor
   parameter at `:16-17`; add any validation before the assignments at `:34-39`.
2. Add the matching property to `…Infrastructure/Mongo/Documents/UserDocument.cs`.
3. Extend all three mappers in `…Infrastructure/Mongo/Documents/Extensions.cs`
   (`AsEntity:9-11`, `AsDocument:13-22`, `AsDto:24-32`) — `AsEntity` is positional, so an omission
   here is a compile error; `AsDocument`/`AsDto` are object initialisers, so an omission is
   **silent data loss**.
4. If the field should be visible to callers, add it to `…Application/DTO/UserDto.cs:9-13` and to
   the `UserDto(User)` constructor at `:19-26`.
5. If the field must be supplied at sign-up, add it to `…Application/Commands/SignUp.cs:10-23`
   (getter-only property **and** constructor parameter — Convey's WebApi binds JSON through the
   constructor `[convey]`, so the parameter *name* is the JSON property name) and pass it through
   `IdentityService.SignUpAsync:102`.
6. If a downstream service needs it, add it to `…Application/Events/SignedUp.cs` and re-check the
   consumer in `customers-service`.
7. **There is no migration step.** Existing documents will deserialise the new field as its CLR
   default; see §5.3.

**Failure modes.**

- A field added at steps 1–2 but forgotten at step 3's `AsDocument` is written as `null`/default
  and then, on the next `AsEntity`, may trip the aggregate's validation — turning a write-time
  omission into a **read-time exception for every existing row**.
- Because `Password` is a `string` on both sides, swapping the hash algorithm (§3.7) silently
  invalidates every stored credential: `VerifyHashedPassword` returns `Failed`, which surfaces as
  `invalid_credentials` (§3.14) and is indistinguishable from a wrong password.
- `Permissions` is `IEnumerable<string>` end to end with no whitelist; a typo'd permission is
  stored, embedded in the JWT (§3.5, §3.10) and silently never matches anything.

### 3.2 `AggregateId` and the implicit-conversion seam

**Definition.** A single-field value object wrapping a non-empty `Guid`
(`…Core/Entities/AggregateId.cs:6-50`), with structural equality (`:25-41`) and **two implicit
conversions**: `AggregateId → Guid` (`:43-44`) and `Guid → AggregateId` (`:46-47`).

**Representation & storage.** Never persisted as itself. `AggregateRoot.Id` is an `AggregateId`
(`…Core/Entities/AggregateRoot.cs:9`) but both documents declare a plain `Guid Id`
(`UserDocument.cs:9`, `RefreshTokenDocument.cs:8`), and the mappers rely on the implicit conversion
in both directions (`…Mongo/Documents/Extensions.cs:16,39-40`). `RefreshToken.UserId` is typed
`AggregateId` (`…Core/Entities/RefreshToken.cs:8`) while `RefreshTokenDocument.UserId` is a `Guid`
(`RefreshTokenDocument.cs:9`) — again bridged implicitly.

**Lifecycle.** `new AggregateId()` mints a fresh `Guid.NewGuid()` (`:10-13`); `new
AggregateId(guid)` validates. Both are value-typed and never mutated.

**Invariants & enforcement.** `Guid.Empty` is rejected **loudly** with
`InvalidAggregateIdException` (`:17-20`) — but *only* through the `Guid`-taking constructor. The
parameterless constructor cannot produce an empty id. The practical consequence: `User.cs:34`
(`Id = id;` where `id` is a `Guid` parameter) goes through the implicit conversion and therefore
*does* validate; `SignUp`'s constructor pre-empts this by substituting a fresh `Guid` when the
caller sends `Guid.Empty` (`…Application/Commands/SignUp.cs:18`), so the exception is effectively
unreachable from the HTTP sign-up path.

**Extension procedure.** Do not add fields; it is a value object. To introduce a strongly-typed id
for a *new* aggregate, copy the class, or reuse `AggregateId` (the codebase does the latter for
both aggregates).

**Failure modes.** The implicit conversions make type errors invisible: passing a `RefreshToken.Id`
where a `UserId` is expected compiles cleanly and produces a silently wrong lookup. There is no
distinct `UserId`/`RefreshTokenId` type. `RefreshTokenService.UseAsync:63` calls
`_userRepository.GetAsync(token.UserId)` — correct here, but nothing in the type system would have
caught `token.Id`.

### 3.3 `AggregateRoot` and the dead domain-event buffer

**Definition.** An abstract base holding `List<IDomainEvent> _events`, an `AggregateId Id`, an
`int Version`, a protected `AddEvent` and a public `ClearEvents`
(`…Core/Entities/AggregateRoot.cs:5-18`).

**Representation & storage.** Purely in-memory; never persisted. Neither document type has a
`Version` field, and no code reads `AggregateRoot.Events`.

**Lifecycle.** Never used. A repository-wide search for `AddEvent` finds only its declaration at
`…Core/Entities/AggregateRoot.cs:12`; a search for `.Events` finds only the property at `:8`;
`ClearEvents` is never called; `Version` is never assigned beyond its default `0`.

**Invariants & enforcement.** None — the class enforces nothing that runs.

**Extension procedure.** If you want this service to emit domain events the way
`deliveries-service` and `orders-service` do, you must add *all three* halves that are missing
here: (1) call `AddEvent(...)` inside `User`/`RefreshToken` mutators; (2) add an `EventMapper`
translating domain events to integration events — **this repository has none**, unlike its
siblings; and (3) drain and publish the buffer after the repository write in
`IdentityService`/`RefreshTokenService`. Until all three exist, adding `AddEvent` calls changes
nothing observable.

**Failure modes.** The primary failure mode is *the reader's*. This base class is the standard
Pacco domain-event scaffold, so a maintainer arriving from `deliveries-service` will reasonably
assume events are buffered and published. They are not: `SignedIn` and `SignedUp` are constructed
inline in the application service and handed straight to `IMessageBroker`
(`…Application/…/IdentityService.cs:80,106`). A change that adds `AddEvent(new UserPasswordChanged(...))`
will compile, run, and publish nothing.

### 3.4 `Role` — the closed vocabulary

**Definition.** A static class with two constants, `User = "user"` and `Admin = "admin"`, and a
validator `IsValid(string)` that lower-cases and compares (`…Core/Entities/Role.cs:3-19`). This is
the **entire** authorization vocabulary of the Pacco platform.

**Representation & storage.** A lower-cased `string` on `User.Role` (`…Core/Entities/User.cs:11`)
and `UserDocument.Role`. It is emitted into the JWT as the role claim
(`…Infrastructure/Auth/JwtProvider.cs:21`, second argument) `[convey]` and echoed back in
`AuthDto.Role` (`…Application/DTO/AuthDto.cs:7`).

**Lifecycle.** Chosen once, at sign-up, and never changed — there is no update path (§3.1).

**Invariants & enforcement.**

- `IsValid` returns `false` for blank (`:10-13`), so `new User(..., role: "", ...)` throws
  `InvalidRoleException` **loudly**. But `IdentityService.SignUpAsync:100` substitutes `"user"`
  when the command's role is blank, so a blank role from a caller is **silently defaulted**, not
  rejected.
- An unknown role (e.g. `"manager"`) reaches `Role.IsValid`, fails, and throws
  `InvalidRoleException` — **loud over HTTP** (400 `invalid_role`, §3.19) but **silently dropped
  over AMQP** because `ExceptionToMessageMapper` has no arm for `InvalidRoleException` (§3.18).
- **Nothing prevents a client from self-assigning `admin`.** The gateway route
  `POST /identity/sign-up` is `auth: false` and passes the request body through unchanged
  (`Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml:256-261`); `SignUp.Role` is bound from that
  body (`…Application/Commands/SignUp.cs:13,21`); and `SignUpAsync` accepts any value `Role.IsValid`
  accepts. A `POST /identity/sign-up` with `"role": "admin"` creates an administrator. See
  §8.2/B2.

**Extension procedure — adding a role.**

1. Add the constant to `…Core/Entities/Role.cs:5-6` and extend the comparison at `:17`.
2. Decide whether the new role is self-assignable at sign-up; if not, you must add a guard in
   `IdentityService.SignUpAsync` (there is currently no place where roles are gated), or change the
   gateway route to bind the role from a trusted source.
3. Add gateway `claims: role: <new-role>` entries anywhere the role should grant access
   (`ntrada.yml`) — the services themselves do not read roles except through
   `IdentityContext.IsAdmin` (`…Infrastructure/Contexts/IdentityContext.cs:29`), which is itself
   never consumed here (§3.32).
4. Existing documents keep their old role; there is no backfill.

**Failure modes.** `IdentityContext.IsAdmin` hard-codes `"admin"` (`IdentityContext.cs:29`) rather
than referencing `Role.Admin`, so renaming the constant leaves a silently-stale string. Adding a
role without a corresponding gateway `claims` entry grants nothing; adding a gateway entry without
the constant means no one can ever hold it.

### 3.5 `Permissions` — the open vocabulary

**Definition.** An unvalidated `IEnumerable<string>` on `User` (`…Core/Entities/User.cs:14`),
supplied at sign-up and surfaced as a JWT claim.

**Representation & storage.** `UserDocument.Permissions` (`UserDocument.cs:14`), a BSON array of
strings; defaulted to empty on both write (`…Mongo/Documents/Extensions.cs:21`) and read
(`:31`) and in the aggregate (`User.cs:39`).

**Lifecycle.** Set once at sign-up from `SignUp.Permissions`
(`…Application/Commands/SignUp.cs:14`, passed at `…/IdentityService.cs:102`). Read on every
sign-in and refresh: both `IdentityService.SignInAsync:70-75` and
`RefreshTokenService.UseAsync:69-74` build the claim dictionary **only when
`user.Permissions.Any()`**, passing `null` otherwise.

**Invariants & enforcement.** None. There is no whitelist, no format check, no length cap and no
de-duplication. Every failure here is **silent**: an unknown permission string is stored, minted
into the token, and simply never matches.

**Extension procedure.** Define the permission string wherever it will be *checked* — note that
**nothing in this workspace checks it**: a grep for `"permissions"` finds only the two claim-
building sites above. The gateway authorizes on `role` only
(`Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `claims:` blocks). Adding a permission is
therefore a two-repository change: emit it here, and add the consuming check wherever it belongs.

**Failure modes.**

- The `Any()` guard means a user with zero permissions gets a token with **no `permissions` claim
  at all**, not an empty array. Any consumer written to read `claims["permissions"]`
  unconditionally will throw or null-deref on exactly the common case.
- Permissions are baked into the access token at issue time and the token lives 60 minutes
  (`…Api/appsettings.json:39`). Because there is no user-update path, permissions cannot change —
  but if one were added, revocation would lag by up to an hour unless the access token were also
  revoked (§3.12).

### 3.6 Email canonicalisation and the `EmailRegex` gate

**Definition.** Two distinct mechanisms that are easy to conflate. **Canonicalisation** lower-cases
the email on aggregate construction (`…Core/Entities/User.cs:35`) and on lookup
(`…Infrastructure/Mongo/Repositories/UserRepository.cs:28`). **Validation** is a compiled
`Regex` held as a `static readonly` field on `IdentityService`
(`…Application/…/IdentityService.cs:18-21`, `RegexOptions.IgnoreCase | Compiled | CultureInvariant`).

**Representation & storage.** The regex is code, not configuration — it cannot be changed without a
redeploy. The canonical email is the stored `UserDocument.Email` value.

**Lifecycle.** The regex is evaluated at the very top of both `SignInAsync` (`:51-55`) and
`SignUpAsync` (`:87-91`), *before* any repository access. Canonicalisation happens at construction
and at query time, so the two always agree.

**Invariants & enforcement.**

- Malformed email → `InvalidEmailException`, **loud** on both paths, but with different fates:
  HTTP 400 `invalid_email` (§3.19) versus, over AMQP, a **silent drop** for `SignUp` because of the
  mapper defect in §3.18.
- Case-insensitive uniqueness holds **only** because both the write (`User.cs:35`) and the read
  (`UserRepository.cs:28`) lower-case. The Mongo unique index (§3.24) is built on the raw stored
  value, so it enforces uniqueness of the canonical form — consistent, but only by construction. If
  a future write path bypassed the `User` constructor, `"A@x.com"` and `"a@x.com"` would both be
  insertable and the index would not stop it.
- `UserRepository.GetAsync(string email)` calls `email.ToLowerInvariant()` with **no null guard**
  (`UserRepository.cs:28`) — a null email would `NullReferenceException`. It is unreachable in
  practice because `EmailRegex.IsMatch(null)` throws `ArgumentNullException` first
  (`…/IdentityService.cs:51,87`) `[framework]` — also **loud**, but as a 500-shaped generic 400
  (`code: "error"`, §3.19), not as `invalid_email`.

**Extension procedure.** To relax or tighten the rule, edit `…Application/…/IdentityService.cs:18-21`
only — the regex has a single definition and two call sites in the same file. To make the check
configurable, add an options class bound in `…Infrastructure/Extensions.cs:54-89` and inject it;
there is no existing options plumbing for the application layer to copy.

**Failure modes.** The regex is the .NET documentation's canonical email pattern and accepts forms
the platform never sees (quoted local parts, bracketed IP domains). It rejects some valid
internationalised addresses. Because it runs *before* the uniqueness check, a caller can learn
"this is not a valid email" and "this email is taken" as two distinguishable errors — an
enumeration oracle (§8.2/B3).

### 3.7 Password hashing

**Definition.** `IPasswordService` declares `IsValid(hash, password)` and `Hash(password)`
(`…Application/Services/IPasswordService.cs:3-7`). `PasswordService`
(`…Infrastructure/Auth/PasswordService.cs:6-20`) delegates both to
`IPasswordHasher<IPasswordService>` — ASP.NET Core Identity's `PasswordHasher<TUser>`
`[framework]`, registered as a singleton at `…Infrastructure/Extensions.cs:58` with the service
interface itself standing in for the "user" type parameter.

**Representation & storage.** The output of `HashPassword` is a base64 string embedding the format
marker, salt and iteration count `[framework]`; it is stored verbatim in `UserDocument.Password`.
No salt column, no algorithm column, no work-factor column exists — everything is inside the
string.

**Lifecycle.** `Hash` is called exactly once per user, at `…/IdentityService.cs:101`. `IsValid` is
called at `:58` and again at `:64` (see below). There is no rehash-on-login, no upgrade path and no
`PasswordVerificationResult.SuccessRehashNeeded` handling: `PasswordService.IsValid:16` collapses
the three-valued result to `!= Failed`, so a "rehash needed" verdict is treated as success and the
old hash is kept forever.

**Invariants & enforcement.**

- A blank password reaches `Hash` unguarded — `SignUp` does not validate it and `SignUpAsync` does
  not either. The resulting hash is non-blank, so `User`'s `InvalidPasswordException` check
  (`User.cs:24-27`) **never fires from the sign-up path**; it only protects direct aggregate
  construction. There is **no minimum length, no complexity rule and no maximum length** anywhere
  in this repository. This is a **silent** acceptance of `""` as a password.
- Hash comparison is constant-time within `PasswordHasher` `[framework]`.

**The duplicated check.** `SignInAsync` verifies the password twice:

```csharp
// …Application/Services/Identity/IdentityService.cs:57-68
var user = await _userRepository.GetAsync(command.Email);
if (user is null || !_passwordService.IsValid(user.Password, command.Password))
{
    _logger.LogError($"User with email: {command.Email} was not found.");   // ← message is wrong
    throw new InvalidCredentialsException(command.Email);
}

if (!_passwordService.IsValid(user.Password, command.Password))             // ← unreachable-as-false
{
    _logger.LogError($"Invalid password for user with id: {user.Id.Value}");
    throw new InvalidCredentialsException(command.Email);
}
```

The second `if` can never evaluate true: the first already returned on a bad password. It is dead
code that costs a **second full PBKDF2 verification on every successful sign-in** — the single
most expensive operation on the hot path, run twice. The log message at `:60` is also wrong: it
says "was not found" for what is usually a wrong password. See §8.2/A2.

**Extension procedure — changing the hashing scheme.**

1. Replace the registration at `…Infrastructure/Extensions.cs:58` with your implementation of
   `IPasswordHasher<IPasswordService>`, or replace `PasswordService` itself
   (`…Infrastructure/Auth/PasswordService.cs`) and its registration at `:57`.
2. **There is no migration path.** Because `User` has no update method (§3.1, §3.21), you cannot
   rehash on next login without first adding `IUserRepository.UpdateAsync` and a
   `UserRepository.UpdateAsync` implementation. Without that, changing the scheme invalidates every
   existing credential permanently.
3. If you add rehash-on-login, change `PasswordService.IsValid:16` to surface
   `SuccessRehashNeeded` rather than collapsing it — the current signature (`bool`) cannot express
   it, so the interface at `…Application/Services/IPasswordService.cs:5` must change too.

**Failure modes.** Silent acceptance of empty passwords; double CPU cost per sign-in; no rehash;
misleading logs; and a permanent lock-out for all users if the scheme is swapped without first
adding an update path.

### 3.8 `RefreshToken` aggregate

**Definition.** `(AggregateId Id, AggregateId UserId, string Token, DateTime CreatedAt,
DateTime? RevokedAt)` with a computed `bool Revoked => RevokedAt.HasValue`
(`…Core/Entities/RefreshToken.cs:8-12`). Note what is **absent**: there is **no expiry field**.

**Representation & storage.** `RefreshTokenDocument`
(`…Infrastructure/Mongo/Documents/RefreshTokenDocument.cs:6-13`) in the `refreshTokens` collection
(`…Infrastructure/Extensions.cs:85`). **No index of any kind is created on this collection**
(§3.24) even though the only lookup is by `Token` (§3.22). A `protected RefreshToken()`
parameterless constructor exists at `:14-16`, unused by the mappers — `AsEntity` uses the public
constructor (`…Mongo/Documents/Extensions.cs:34-35`).

**Lifecycle.**

1. **Create** — `RefreshTokenService.CreateAsync` (`…Application/…/RefreshTokenService.cs:29-36`)
   on every successful sign-in, with a fresh `new AggregateId()` and `_rng.Generate(30, true)`.
2. **Use** — `UseAsync` (`:50-79`) looks the token up, rejects null and revoked, loads the user,
   and mints a **new access token** while returning **the same refresh token** (`:76`).
3. **Revoke** — `RevokeAsync` (`:38-48`) loads, calls `token.Revoke(DateTime.UtcNow)` and persists
   via `UpdateAsync`. This is the **only** `UpdateAsync` call in the service.

**Invariants & enforcement.**

| Invariant | Enforced at | Loud or silent |
| --- | --- | --- |
| Token string is non-blank | `…Core/Entities/RefreshToken.cs:21-24` → `EmptyRefreshTokenException` | **Loud** |
| Revocation is one-way | `:33-38` → `RevokedRefreshTokenException` if already revoked | **Loud** |
| A revoked token cannot be used | `…Application/…/RefreshTokenService.cs:58-61` → `RevokedRefreshTokenException` | **Loud** |
| An unknown token is rejected | `:41-44,53-56` → `InvalidRefreshTokenException` | **Loud** |
| The owning user still exists | `:63-67` → `UserNotFoundException` | **Loud** |
| Token is unique | **Not enforced.** No unique index, no application check. Collision probability is negligible (30 random bytes) but nothing detects one | Silent |
| Token expires | **Not enforced — the concept does not exist.** | — |
| Token rotates on use | **Not enforced — `UseAsync:76` returns the same token.** | — |

**Extension procedure — adding expiry.**

1. Add `DateTime? ExpiresAt` to `…Core/Entities/RefreshToken.cs:8-12` and the constructor at
   `:18-19`; add an `IsExpired(DateTime now)` accessor.
2. Add the field to `RefreshTokenDocument.cs` and to both mappers
   (`…Mongo/Documents/Extensions.cs:34-45`) — `AsEntity` is positional (compile error if missed),
   `AsDocument` is an initialiser (**silent** if missed).
3. Set it in `RefreshTokenService.CreateAsync:32` and check it in `UseAsync` alongside the
   `Revoked` check at `:58`; throw a new `DomainException` subclass with a `Code`.
4. Add the new exception to `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs` only if you
   want a status other than 400 — the `DomainException` arm at `:18-19` already handles the code
   string automatically (§3.19).
5. **Existing documents have no `ExpiresAt`** and will deserialise as `null`; decide explicitly
   whether `null` means "never expires" (backwards compatible) or "expired" (invalidates every
   live session). See §5.3.
6. Consider a Mongo TTL index — there is no precedent for one in this workspace, and the only
   index-creation site is `…Infrastructure/Mongo/Extensions.cs:13-28`.

**Extension procedure — adding rotation.** Change `UseAsync:75-77` to call `RevokeAsync` on the
presented token and `CreateAsync` for a replacement, and return the new value. This changes an
externally-observable contract (`refresh-tokens/use` currently returns the same token), so the
`.rest` sample (`Pacco.Services.Identity.rest:28-33`) and any client must be updated together.

**Failure modes.**

- **A refresh token is a permanent bearer credential.** No expiry, no rotation, no reuse
  detection, no device binding, no cap on the number of live tokens per user, and a new one is
  minted on *every* sign-in (`…/IdentityService.cs:77`). The `refreshTokens` collection therefore
  grows monotonically and is never pruned. Stealing one grants indefinite access-token minting
  until someone calls `refresh-tokens/revoke` — a route **the gateway does not expose**
  (§6.2). See §8.2/B4.
- `RevokeAsync` on an already-revoked token throws `RevokedRefreshTokenException` from the
  aggregate (`RefreshToken.cs:36`) rather than being idempotent — a double sign-out returns
  HTTP 400.
- `UseAsync` performs three round trips (token read, user read, token *not* written) plus a JWT
  signature; the token read is an unindexed collection scan (§3.22).

### 3.9 `Rng` — the token generator, and an interface/implementation divergence

**Definition.** `IRng.Generate(int length = 50, bool removeSpecialChars = false)`
(`…Application/Services/IRng.cs:3-6`), implemented by `Rng`
(`…Infrastructure/Auth/Rng.cs:8-23`) over `RNGCryptoServiceProvider`: fill `length` bytes, base64
them, and optionally strip the seven characters `/ \ = + ? : &` (`:10`).

**Representation & storage.** Registered as a singleton (`…Infrastructure/Extensions.cs:61`). The
generated string becomes `RefreshToken.Token`.

**Lifecycle.** One call site: `RefreshTokenService.CreateAsync:31`, `_rng.Generate(30, true)`.

**Invariants & enforcement.**

- **The default values diverge.** The interface declares `removeSpecialChars = false`
  (`IRng.cs:5`); the implementation declares `= true` (`Rng.cs:12`). In C#, optional-argument
  defaults are resolved at the *call site* from the *static type of the receiver*. `IRng` is what
  is injected, so a hypothetical `_rng.Generate(30)` would strip **nothing**, while
  `new Rng().Generate(30)` would strip. This is a **silent** trap: the two defaults are invisible
  at the call site. The current single call site passes the flag explicitly, so nothing is broken
  today.
- Stripping characters **shortens** the string non-deterministically: 30 bytes → 40 base64
  characters → typically 36–40 after stripping. The *entropy* is still 240 bits (it comes from the
  bytes, not the encoding), but the length is variable, which matters if a column, index or client
  ever assumes a fixed width.
- `RNGCryptoServiceProvider` is disposed via `using var` (`:14`) `[framework]`.

**Extension procedure.** To change token length, edit the literal at
`…Application/…/RefreshTokenService.cs:31`. To add a new random-token use (e.g. a password-reset
token), inject `IRng` and always pass `removeSpecialChars` explicitly until the interface default is
fixed. To fix the divergence, change `Rng.cs:12` to `= false` and audit every call site — there is
exactly one.

**Failure modes.** The divergence above; and the fact that stripping is applied *after* base64
rather than using a URL-safe alphabet, so two different byte sequences can in principle map to the
same stripped string. With 30 random bytes this is not a practical concern, but nothing would
detect a collision because there is no unique index on `refreshTokens.Token` (§3.8).

### 3.10 Access-token issuance and the `"N"` subject format

**Definition.** `IJwtProvider.Create(userId, role, audience = null, claims = null)`
(`…Application/Services/IJwtProvider.cs:14-18`), implemented by `JwtProvider`
(`…Infrastructure/Auth/JwtProvider.cs:9-30`) as a thin adapter over Convey's `IJwtHandler`
`[convey]`, registered as a singleton at `…Infrastructure/Extensions.cs:56` with `AddJwt()`
supplying `IJwtHandler` at `:74`.

**Representation & storage.** Stateless — nothing about an issued access token is stored. The
returned `AuthDto` carries `AccessToken`, `Role` and `Expires` (`JwtProvider.cs:23-28`);
`RefreshToken` is filled in afterwards by the caller (`…/IdentityService.cs:77`).

**The load-bearing line:**

```csharp
// …Infrastructure/Auth/JwtProvider.cs:21
var jwt = _jwtHandler.CreateToken(userId.ToString("N"), role, audience, claims);
```

`"N"` is the **32-hex-digit, dash-less** Guid format. The JWT subject therefore contains
`3fa85f6457174562b3fc2c963f66afa6`, not `3fa85f64-5717-4562-b3fc-2c963f66afa6`. This single format
choice propagates across the whole platform:

| Consumer | Where | Behaviour under `"N"` |
| --- | --- | --- |
| `identity-service` `GET me` | `…Infrastructure/Extensions.cs:112` — `Guid.Parse(Principal.Identity.Name)` | `Guid.Parse` accepts `"N"` `[framework]` — works |
| Gateway `@user_id` binding | `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml` `bind:` entries | binds the dash-less string verbatim into downstream payloads and into the correlation context |
| `identity-service` `IdentityContext` | `…Infrastructure/Contexts/IdentityContext.cs:26` — `Guid.TryParse` | accepts `"N"` — works |
| `operations-service` SignalR groups | `Hubs/PaccoHub.cs:34` (`Guid.Parse(sub).ToUserGroup()` → `users:{N}`) vs. `Services/HubWrapper.cs:18` (`userId.ToUserGroup()` → `users:{raw correlation-context string}`) | **The two group names coincide only because this line emits `"N"`.** Changing it to `userId.ToString()` would make the hub join `users:<dash-less>` while publishes target `users:<dashed>`, and every real-time notification would be **silently** discarded by SignalR. See `component-internals/operations-service.md` §3.19. |

**Lifecycle.** Called from exactly two places: `IdentityService.SignInAsync:76` and
`RefreshTokenService.UseAsync:75`. Tokens expire after `jwt.expiryMinutes: 60`
(`…Api/appsettings.json:39`).

**Invariants & enforcement.** Signing-key selection, claim naming, lifetime stamping and signature
algorithm all live inside Convey `[convey]`. From configuration (`…Api/appsettings.json:32-45`) we
can state:

- `issuer: "pacco"`, but `validateIssuer: false` and `validateAudience: false` — the issuer and
  audience are stamped and then ignored by every validator on the platform. `audience` is a
  parameter of `Create` that **no call site ever supplies** (both pass only `claims:`), so it is
  always `null`.
- `validateLifetime: true` — expiry is honoured.
- The signing material has two possible sources: `jwt.certificate.location: "certs/localhost.pfx"`
  with `password: "test"` (`:33-37`), or the symmetric `jwt.issuerSigningKey` (`:38`). **Which one
  Convey prefers when both are present is not determinable from this workspace** —
  `Unverifiable — Missing Source Evidence`. What *is* determinable: `appsettings.docker.json:28-32`
  and `appsettings.local.json:26-31` both blank the certificate, so **in Docker and locally the
  symmetric key is necessarily used**, and that key is committed in plaintext and is byte-identical
  across every Pacco service (verified by grep across all clones — e.g.
  `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json:36` holds the same
  string). Any service that can read its own configuration can therefore **mint a token for any
  user with any role**. See §8.2/B1.
- `src/Pacco.Services.Identity.Api/certs/` contains `localhost.pfx`, `localhost.key` and
  `localhost.pem` — **private key material committed to source control** — and
  `appsettings.json:35` contains its password.
  `…Api/Pacco.Services.Identity.Api.csproj:20-22` copies the directory into every published image.

**Extension procedure — adding a claim.** Extend the `claims` dictionary built at
`…/IdentityService.cs:70-75` **and** the identical block at `…/RefreshTokenService.cs:69-74` — they
are duplicated and will drift. Note the `Any()` guard means the dictionary is `null` when the user
has no permissions, so a new unconditional claim must be added *outside* that ternary or it will be
dropped for most users.

**Failure modes.** A missing or mistyped `certs/localhost.pfx` in the non-Docker profile; the
`"N"`-format coupling above; a 60-minute window during which revocation depends entirely on the
Redis deny-list (§3.12); and claim-building duplication between the two mint sites.

### 3.11 `AuthDto` — the authentication wire shape

**Definition.** `(string AccessToken, string RefreshToken, string Role, long Expires)`
(`…Application/DTO/AuthDto.cs:3-9`). This is the JSON body returned by `POST sign-in` and
`POST refresh-tokens/use`.

**Representation & storage.** Never persisted. Constructed in `JwtProvider.Create:23-28` with three
of the four fields; `RefreshToken` is assigned by the caller afterwards
(`…/IdentityService.cs:77`, `…/RefreshTokenService.cs:76`). Serialised by
`ctx.Response.WriteJsonAsync(token)` (`Program.cs:50,65`) `[convey]`.

**Lifecycle.** Per request; immediately discarded.

**Invariants & enforcement.** None — a mutable POCO with public setters. `Expires` is a `long`
(Unix seconds `[convey]`), not a `DateTime`, so clients must know the unit; nothing in this
repository documents it.

**Extension procedure.** Add a property here and populate it at both mint sites. A field derivable
from the JWT belongs in `JwtProvider.cs:23-28` (shared); a field derived from the user belongs at
the two callers. It is a response-only type — no gateway or schema change is needed, but any client
parsing it strictly needs updating.

**Failure modes.** The two-step construction (`Create`, then assign `RefreshToken`) means a new code
path that calls `IJwtProvider.Create` directly returns an `AuthDto` with a **null** `RefreshToken`
and no compiler complaint.

### 3.12 Access-token revocation (the Redis deny-list)

**Definition.** `POST access-tokens/revoke` deactivates a *specific* access-token string so that
subsequent requests bearing it are rejected before reaching any handler.

**Representation & storage.** Convey's `IAccessTokenService` `[convey]`, obtained from `AddJwt()`
(`…Infrastructure/Extensions.cs:74`) and backed by `IDistributedCache`. `AddRedis()` at `:82`
supplies that cache, with `redis.instance: "identity:"` and `connectionString: "localhost"`
(`…Api/appsettings.json:156-159`; `redis` in Docker, `appsettings.docker.json:88-91`). The exact key
layout and TTL are internal to Convey — `Unverifiable — Missing Source Evidence` — but the storage
medium is settled: without `AddRedis()` there is no `IDistributedCache` registration in this
service at all.

**Lifecycle.**

- **Write** — `Program.cs:57-61`: `IAccessTokenService.DeactivateAsync(cmd.AccessToken)`, then
  `204`. The route takes the token **in the request body** (`RevokeAccessToken.AccessToken`,
  `…Application/Commands/RevokeAccessToken.cs:7`), not from the `Authorization` header.
- **Read** — `UseAccessTokenValidator()` (`…Infrastructure/Extensions.cs:97`) `[convey]` inspects
  every incoming request and rejects deactivated tokens.

**Invariants & enforcement.**

- **Nothing checks that the caller owns the token in the body.** Convey's validator middleware is
  configured with `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`
  (`…Api/appsettings.json:44`), which implies `/access-tokens/revoke` requires *some* valid bearer
  token to reach the handler — but the token in the **body** is unrelated to the token in the
  **header**. Whether the middleware enforces anything beyond the anonymous list is
  `Unverifiable — Missing Source Evidence` `[convey]`. Either way this is at best "any
  authenticated user can revoke any other user's access token", and at worst an unauthenticated
  denial of service. See §8.2/B5.
- The route has **no gateway entry** (`ntrada.yml:237-289` exposes only `users/{userId}`, `me`,
  `sign-up`, `sign-in`), so it is unreachable from outside the cluster in the deployed topology
  (§6.2).
- Revocation is honoured only by services that call `UseAccessTokenValidator()`. In this workspace
  `identity-service` does (`…Infrastructure/Extensions.cs:97`) and `operations-service` does
  **not** (`Pacco.Services.Operations/src/Pacco.Services.Operations.Api/Infrastructure/Extensions.cs:81-93`),
  so a revoked-but-unexpired token still authenticates a SignalR hub connection there.

**Extension procedure.** To revoke *all* of a user's tokens rather than one, there is no primitive:
Convey's deny-list is per-token-string `[convey]`. You would add a `tokenVersion` claim at
`…/JwtProvider.cs:21`, a matching field on `User`, an update path (§3.21), and a validator in every
consuming service.

**Failure modes.** Redis unavailability — whether the validator fails open or closed is Convey's
behaviour and is `Unverifiable — Missing Source Evidence`; worth settling before relying on
revocation. Revocation state is lost if Redis is flushed, silently restoring every revoked token
until it expires naturally.

### 3.13 Refresh-token lifecycle as an orchestration

**Definition.** The three-method contract `IRefreshTokenService`
(`…Application/Services/IRefreshTokenService.cs:7-12`): `CreateAsync(Guid)`, `RevokeAsync(string)`,
`UseAsync(string)`. §3.8 covers the aggregate; this covers the orchestration around it.

**Representation & storage.** Registered transient (`…Infrastructure/Extensions.cs:60`). Holds no
state; composes `IRefreshTokenRepository`, `IUserRepository`, `IJwtProvider` and `IRng`
(`…Application/…/RefreshTokenService.cs:15-27`).

**Lifecycle & control flow.** See §4.4–4.6 for full traces. The salient structural facts:

- `CreateAsync` is called **only** from `IdentityService.SignInAsync:77` — never from
  `SignUpAsync`. A user who signs up receives `201 Created` with a `Location` header and **no
  tokens** (`Program.cs:52-56`); they must sign in separately.
- `UseAsync` does **not** write to the database at all — it reads twice and mints. This is what
  makes the missing rotation (§3.8) structural rather than incidental.
- `RevokeAsync` is the only writer besides `CreateAsync`.

**Invariants & enforcement.** All four guards are **loud** (§3.8 table). What is *not* enforced:
no check that the user is still permitted to sign in (no "disabled" flag exists), no check of how
old the token is, and no audit trail of use.

**Extension procedure.** See §3.8. Additionally, to issue tokens at sign-up, call
`_refreshTokenService.CreateAsync(user.Id)` in `SignUpAsync` after `AddAsync` and change the
endpoint at `Program.cs:52-56` to return the `AuthDto` instead of `Created`. That changes the
route's status code from 201 to 200 and interacts with the gateway's `resourceId` handling
(`ntrada.yml:259-261`), which generates a `userId` and expects a created-resource response.

**Failure modes.** Both `UseAsync` and `RevokeAsync` begin with an unindexed lookup (§3.22). A user
who signs in *n* times accumulates *n* live refresh tokens with no way to enumerate or bulk-revoke
them — `IRefreshTokenRepository` has no "get by user" method.

### 3.14 `IdentityService` — the transaction script

**Definition.** The service that implements sign-in and sign-up end to end
(`…Application/Services/Identity/IdentityService.cs:16-108`), plus a pass-through read
(`GetAsync:42-47`). Registered transient at `…Infrastructure/Extensions.cs:59`.

**Representation & storage.** Stateless; six injected dependencies (`:23-28`) — user repository,
password service, JWT provider, refresh-token service, message broker, logger.

**Lifecycle — `SignInAsync` (`:49-83`).** Regex gate → load by email → credential check (twice,
§3.7) → build optional `permissions` claim → mint JWT → mint refresh token → log → publish
`SignedIn` → return `AuthDto`.

**Lifecycle — `SignUpAsync` (`:85-107`).** Regex gate → load by email → reject if found → default
role → hash password → construct `User` → `AddAsync` → log → publish `SignedUp`.

**Invariants & enforcement.**

| Invariant | Enforced at | Loud or silent |
| --- | --- | --- |
| Email is well-formed | `:51-55`, `:87-91` → `InvalidEmailException` | **Loud** |
| Credentials match | `:58-62` → `InvalidCredentialsException` | **Loud**, but HTTP **400**, not 401 (§3.19) |
| Email not already registered | `:93-98` → `EmailInUseException` | **Loud**, but racy — see below |
| Role defaults to `user` when blank | `:100` | **Silent** |
| Role is valid | delegated to the `User` constructor → `InvalidRoleException` | **Loud** over HTTP, **silent** over AMQP (§3.18) |
| The event is published only after a successful write | `:103` then `:106` | ordering correct; atomicity is the outbox's job (§3.30) |

**The uniqueness race.** `SignUpAsync` does a read (`:93`) and then a write (`:103`) with no
transaction — and `outbox.disableTransactions` is `true` (`…Api/appsettings.json:112`), consistent
with a single-node Mongo that has no replica set and therefore no multi-document transactions at
all. Two concurrent sign-ups for the same email both read `null` and both call `AddAsync`. The only
backstop is the Mongo unique index — and that index is created fire-and-forget with its failures
swallowed (§3.24), so on a fresh deployment where index creation lost the race, **duplicate users
with the same email are insertable**. `UserRepository.GetAsync(email)` then returns whichever
document Mongo finds first, non-deterministically. See §8.2/B6.

**Extension procedure — adding a new identity operation (e.g. change password).**

1. Add the command to `…Application/Commands/` with getter-only properties and a constructor
   (mirroring `RevokeRefreshToken.cs`), and `[Contract]` if other services should see it.
2. Add the method to `…Application/Services/IIdentityService.cs:6-11` and implement it in
   `IdentityService`.
3. **Add the missing repository method** — `IUserRepository.UpdateAsync` and its `UserRepository`
   implementation (`…Infrastructure/Mongo/Repositories/UserRepository.cs`), delegating to
   `_repository.UpdateAsync(user.AsDocument())` exactly as `RefreshTokenRepository.cs:28` does.
4. Add a mutator to the `User` aggregate — every setter is `private set`, so no field can be
   changed from outside.
5. Register the HTTP route in `Program.cs` inside the `UseEndpoints` chain at `:33-72`, following
   the `Post<TCommand>` shape; choose the status code explicitly — there is no convention, the
   existing routes use 200-with-body, 201 and 204.
6. Add a gateway route in **all four** Ntrada manifests (`ntrada.yml`, `ntrada.docker.yml`,
   `ntrada-async.yml`, `ntrada-async.docker.yml`) or the route is unreachable in production (§6.2).
7. If it should also be reachable over AMQP, add `.SubscribeCommand<T>()` at
   `…Infrastructure/Extensions.cs:103` **and** an `ICommandHandler<T>` (see `SignUpHandler.cs`).
8. If it can fail, add arms to **both** `ExceptionToResponseMapper` and `ExceptionToMessageMapper`
   (§3.18–3.19).

**Failure modes.** The duplicated password check (§3.7); the misleading log at `:60` ("was not
found" for what is usually a wrong password); the uniqueness race; and the fact that
`_logger.LogError($"Invalid email: {command.Email}")` at `:53,89` **writes the email into the log
message** despite `logger.excludeProperties` listing `"Email"` (`…Api/appsettings.json:58`) — the
exclusion applies to structured properties, and these are interpolated strings, so the value lands
in the message body regardless. The same applies at `:60` and `:96`.

### 3.15 The five commands and the `[Contract]` marker

**Definition.** `SignIn`, `SignUp`, `RevokeAccessToken`, `UseRefreshToken`, `RevokeRefreshToken`
(`…Application/Commands/*.cs`), all implementing Convey's `ICommand` marker. `ContractAttribute`
(`…Application/ContractAttribute.cs:5-7`) is a bare marker attribute; only `SignIn` and `SignUp`
carry it (`SignIn.cs:5`, `SignUp.cs:7`), alongside the four event types (§3.16–3.17).

**Representation & storage.** Never persisted, except as an outbox payload on the AMQP path
(§3.30). Bound from JSON by Convey's WebApi `[convey]`.

**Binding mechanics — the shape matters.**

| Command | Properties | Constructor | Consequence |
| --- | --- | --- | --- |
| `SignIn` | `{ get; set; }` (`SignIn.cs:8-9`) | present (`:11-15`) | binds either way |
| `SignUp` | `{ get; }` only (`SignUp.cs:10-14`) | required (`:16-23`) | **constructor-bound**: JSON names must match the *parameter* names `userId, email, password, role, permissions` |
| `RevokeAccessToken` | `{ get; }` (`:7`) | `:9-12` | constructor-bound on `accessToken` |
| `UseRefreshToken` | `{ get; }` (`:7`) | `:9-12` | constructor-bound on `refreshToken` |
| `RevokeRefreshToken` | `{ get; }` (`:7`) | `:9-12` | constructor-bound on `refreshToken` |

`SignUp`'s constructor also **normalises the id**:
`UserId = userId == Guid.Empty ? Guid.NewGuid() : userId` (`SignUp.cs:18`). This is why both the
gateway's `resourceId: {property: userId, generate: true}` (`ntrada.yml:259-261`) and a bare client
POST work — an absent `userId` binds to `Guid.Empty` and is then replaced. It is also why
`AggregateId`'s empty-guid guard (§3.2) is unreachable from this path.

**Lifecycle.** Constructed per request; discarded after the handler returns.

**Invariants & enforcement.** No validation attributes, no `IValidatable`, no FluentValidation
anywhere in the repository. All validation lives in `IdentityService`/`User`. A `null` `Email`
therefore reaches `EmailRegex.IsMatch(null)` and throws `ArgumentNullException` `[framework]`,
which the response mapper flattens to a generic `400 {code: "error"}` (§3.19) — **loud but
uninformative**.

**Extension procedure.** Copy `RevokeRefreshToken.cs` as the template. Add `[Contract]` only if the
type belongs to the platform's published surface — `UsePublicContracts<ContractAttribute>()`
(`…Infrastructure/Extensions.cs:99`) exposes exactly the marked types `[convey]`. Note the
asymmetry already present: the three token-lifecycle commands are **not** marked, so they are
absent from the published contract set even though they are real HTTP endpoints.

**Failure modes.** Renaming a constructor *parameter* (not the property) silently breaks JSON
binding for constructor-bound commands — the property keeps its name, the binder does not find a
match, and the field arrives `null`. There is no test to catch this (§3.40).

### 3.16 Integration events `SignedIn` and `SignedUp`

**Definition.** `SignedIn(Guid UserId, string Role)` (`…Application/Events/SignedIn.cs:7-17`) and
`SignedUp(Guid UserId, string Email, string Role)` (`…Application/Events/SignedUp.cs:7-19`), both
`IEvent`, both `[Contract]`.

**Representation & storage.** Published to the `identity` topic exchange
(`…Api/appsettings.json:136-142`) with routing keys derived by Convey's `snakeCase` convention
(`:118`) → `signed_in`, `signed_up`. With the outbox enabled they are first written to the `outbox`
collection of the `identity-service` database (`:105-113`) and drained by a background dispatcher
`[convey]`.

**Lifecycle.** `SignedIn` is published at `…/IdentityService.cs:80`, after the tokens are minted
but **before** the method returns — so a publish failure fails the sign-in. `SignedUp` at `:106`,
after `AddAsync` — so the user exists even if publication fails and (outbox enabled) the event is
retried.

**Consumers, verified by grep across all fourteen clones.**

| Event | Consumer | Evidence |
| --- | --- | --- |
| `SignedUp` | `customers-service` — bootstraps the customer profile | `Pacco.Services.Customers/.../Events/External/Handlers/SignedUpHandler.cs`; modelled in `component-internals/customers-service.md` |
| `SignedIn` | **none** — no service subscribes it for behaviour | the same grep finds only this service and `operations-service`'s generic `messages.json:66-69` entry, which exists solely to render a UI notification (§6.5) |

**Invariants & enforcement.** Both are immutable (getter-only, constructor-set). Neither carries a
correlation id of its own — correlation travels in AMQP message properties, set by `MessageBroker`
(§3.31).

**Extension procedure — adding a field.** Add the property and constructor parameter here, then
update the consuming handler in `customers-service` **in a separate repository**. Because
Newtonsoft deserialisation tolerates missing members `[convey]`, a rolling deploy in which the
consumer ships first is safe (the field arrives `null`/default); the reverse order silently loses
data until the consumer catches up. Adding a *new* event additionally requires adding its
snake-case name to
`Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json:66-69` if it should
drive the operations UI.

**Failure modes.** `SignedIn` fires on **every** sign-in with no consumer — pure exchange traffic
and, with the outbox on, a Mongo write per sign-in. `SignedUp` copies the user's email into a
second service's database, which matters for data-deletion requests: there is no propagation
mechanism, because there is no deletion path at all (§3.1).

### 3.17 Rejected events `SignInRejected` and `SignUpRejected`

**Definition.** `IRejectedEvent` implementations carrying `(Email, Reason, Code)`
(`…Application/Events/Rejected/SignInRejected.cs:6-18`,
`…Application/Events/Rejected/SignUpRejected.cs:6-18`), both `[Contract]`.

**Representation & storage.** Would be published to the `identity` exchange as `sign_in_rejected` /
`sign_up_rejected` — but see the lifecycle.

**Lifecycle.** They are **never constructed on any path that runs today.** Their only construction
sites are inside `ExceptionToMessageMapper`
(`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:15-21`), which Convey invokes only when a
**subscribed AMQP message** throws `[convey]`. The one subscribed message is `SignUp`
(`…Infrastructure/Extensions.cs:103`), and **nothing in the workspace publishes `sign_up`**
(§3.29). Therefore:

- `SignInRejected` is unreachable — `SignIn` is not subscribed at all.
- `SignUpRejected` is reachable only if something outside the workspace publishes `sign_up`, and
  even then only through the `EmailInUseException` arm (`:15`), because the `InvalidEmailException`
  arm is defective (§3.18).

**Invariants & enforcement.** `IRejectedEvent` `[convey]` requires `Reason` and `Code`; both are
populated from the exception (`ex.Message`, `ex.Code`). `EmailInUseException.Message` is
`"Email {email} is already in use."` (`…Core/Exceptions/EmailInUseException.cs:8`), so the **email
is duplicated into both `Email` and `Reason`** and travels onto the bus.

**Extension procedure.** To make rejections observable, first give the service an AMQP producer
(§3.29, §7.4), then add an arm per exception type to `ExceptionToMessageMapper.Map`. Every rejected
event must also be listed in `operations-service`'s `messages.json:70-73` to reach the UI.

**Failure modes.** Dead contracts a reader will assume are live; and PII (the email address)
propagated onto a topic exchange inside a rejection payload.

### 3.18 `ExceptionToMessageMapper` — the AMQP failure path, and its type-match defect

**Definition.** Convey's hook for turning a handler exception into a rejected event
(`…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:9-25`), registered at
`…Infrastructure/Extensions.cs:78`.

**The code, verbatim (`ExceptionToMessageMapper.cs:11-24`):**

```csharp
public object Map(Exception exception, object message)
    => exception switch
    {
        EmailInUseException ex         => new SignUpRejected(ex.Email, ex.Message, ex.Code),
        InvalidCredentialsException ex => new SignInRejected(ex.Email, ex.Message, ex.Code),
        InvalidEmailException ex       => message switch
        {
            SignIn command         => new SignInRejected(command.Email, ex.Message, ex.Code),
            SignUpRejected command => new SignUpRejected(command.Email, ex.Message, ex.Code),  // ← defect
            _ => null
        },
        _ => null
    };
```

**The defect.** The inner switch matches on the *inbound message*. The inbound message for a
sign-up is the **command** `SignUp`, not the **event** `SignUpRejected`. `SignUpRejected` is never
an inbound message anywhere — it is an *output* type produced by this very method. The arm is
therefore unmatchable, and an `InvalidEmailException` raised while handling an AMQP `sign_up` falls
through to `_ => null`.

**Consequence.** Returning `null` means "no rejected event" `[convey]`; the exception is swallowed
and the message is acknowledged. Nothing is published, nothing is retried, and in an async flow the
UI waits forever for an operation that never reaches a terminal state — `operations-service` only
latches `Completed`/`Rejected` on a message it actually receives (see
`component-internals/operations-service.md` §3.8). This is the archetypal **silent** failure. The
fix is a one-token change: `SignUpRejected command` → `SignUp command`.

**Coverage — ten domain/app exceptions, three arms:**

| Exception | `Code` | Mapped to a rejected event? |
| --- | --- | --- |
| `EmailInUseException` | `email_in_use` | ✔ `SignUpRejected` |
| `InvalidCredentialsException` | `invalid_credentials` | ✔ `SignInRejected` — unreachable (`SignIn` is not subscribed) |
| `InvalidEmailException` | `invalid_email` | ✔ for `SignIn` (unreachable); **✘ for `SignUp` (the defect)** |
| `InvalidRoleException` | `invalid_role` | ✘ silent |
| `InvalidPasswordException` | `invalid_password` | ✘ silent |
| `InvalidAggregateIdException` | `invalid_aggregate_id` | ✘ silent |
| `InvalidRefreshTokenException` | `invalid_refresh_token` | ✘ silent |
| `RevokedRefreshTokenException` | `revoked_refresh_token` | ✘ silent |
| `EmptyRefreshTokenException` | `empty_refresh_token` | ✘ silent |
| `UserNotFoundException` (`AppException`) | `user_not_found` | ✘ silent |

**Extension procedure.** Add an arm per exception type. Prefer matching the *exception* rather than
the *message* wherever the target event is unambiguous — the nested `message switch` exists only
because `InvalidEmailException` can arise from either command, and it is exactly that nesting that
harbours the defect.

**Failure modes.** As above. Note also that this mapper and `ExceptionToResponseMapper` are
maintained independently, so an exception can be well-handled on one transport and invisible on the
other — which is the current state for seven of the ten exceptions.

### 3.19 `ExceptionToResponseMapper` — the HTTP failure path

**Definition.** Maps every exception to an `ExceptionResponse`
(`…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:15-24`), registered at
`…Infrastructure/Extensions.cs:71` and activated by `UseErrorHandler()` (`:93`).

**Representation & storage.** Three arms, **all returning `HttpStatusCode.BadRequest`**:

```csharp
DomainException ex => ({code = GetCode(ex), reason = ex.Message}, 400)
AppException    ex => ({code = GetCode(ex), reason = ex.Message}, 400)
_               => ({code = "error", reason = "There was an error."}, 400)
```

`GetCode` (`:26-45`) prefers the exception's own `Code` and otherwise derives one via
`Type.Name.Underscore().Replace("_exception", "")` (`Underscore()` is Convey's string extension
`[convey]`), memoising into a `static ConcurrentDictionary<Type,string>` (`:13`).

**Lifecycle.** Invoked per unhandled exception in the HTTP pipeline.

**Invariants & enforcement.** The invariant is unfortunate but consistent: **this service never
returns 401, 403, 409 or 500 from the error mapper.** Concretely:

| Situation | Ought to be | Actually is |
| --- | --- | --- |
| Wrong password / unknown email (`InvalidCredentialsException`) | 401 | **400** `invalid_credentials` |
| Email already registered (`EmailInUseException`) | 409 | **400** `email_in_use` |
| Refresh token unknown or revoked | 401 | **400** |
| Mongo duplicate-key from the unique index (§3.24) | 409 | **400** `{code:"error"}` — `MongoWriteException` matches neither typed arm |
| Any infrastructure fault (Mongo, Redis or RabbitMQ down) | 500 | **400** `{code:"error"}` |

The last row is the operationally dangerous one: clients, load balancers and alerting all read 4xx
as "the caller's fault — do not retry, do not page". A total datastore outage in this service is
reported as a client error. The only non-400 responses the service can produce are the ones written
by hand: `401` at `Program.cs:41`, `404` at `:83`, `201` at `:55` and `204` at `:60,70`.

**Extension procedure.** Add an arm *before* the `_` fallback, keyed on the specific exception type,
with the correct status. Because `GetCode` memoises by type in a static dictionary, an exception's
code cannot change at runtime; changing it in source is safe.

**Failure modes.** As tabulated. Additionally `reason = ex.Message` leaks the message verbatim, and
several messages embed the email (`EmailInUseException.cs:8`, `InvalidEmailException.cs:7`), making
the response body an account-existence oracle (§8.2/B3).

### 3.20 The exception taxonomy

**Definition.** Two roots: `DomainException` (`…Core/Exceptions/DomainException.cs:5-12`) for
invariant violations and `AppException` (`…Application/Exceptions/AppException.cs:5-12`) for
application-level faults. Both declare `virtual string Code`.

**Representation & storage.** Ten concrete types: nine in `…Core/Exceptions/` and
`UserNotFoundException` in `…Application/Exceptions/`. Each overrides `Code` with a snake-case
string that becomes the client-visible error code (§3.19).

**A file-layout trap.** `EmptyRefreshTokenException` is **not** in its own file — it is declared as
a second public class inside `…Core/Exceptions/InvalidRoleException.cs:12-19`. A maintainer
grepping for `EmptyRefreshTokenException.cs` will conclude it does not exist.

**Lifecycle.** Thrown, caught by Convey's error middleware, mapped (§3.18–3.19), discarded.

**Invariants & enforcement.** `Code` is a property initialiser
(`public override string Code { get; } = "…";`) rather than a constant, so it is per-instance and
cannot be read without constructing the exception. Nothing enforces uniqueness of code strings
across types; nothing enforces that a new exception gets a `Code` at all. Omitting it yields
`null`, and `GetCode`'s `when !string.IsNullOrWhiteSpace(...)` guard
(`ExceptionToResponseMapper.cs:36-38`) then falls back to the derived name — **silently**.

**Extension procedure.** Subclass `DomainException` (invariant) or `AppException` (orchestration),
override `Code`, and give it its own file — do not follow the `InvalidRoleException.cs` precedent.
Then decide, explicitly, on **both** mappers (§3.18, §3.19).

**Failure modes.** Two exceptions sharing a `Code` would be indistinguishable to clients; the
memoising cache would still work (it is keyed by type), so the collision would be invisible in
review and visible only to a client.

### 3.21 `IUserRepository` / `UserRepository` — an append-and-read-only store

**Definition.** The domain-facing port
(`…Core/Repositories/IUserRepository.cs:7-12`) declaring exactly three operations:
`GetAsync(Guid id)`, `GetAsync(string email)`, `AddAsync(User user)`. The adapter
(`…Infrastructure/Mongo/Repositories/UserRepository.cs:10-34`) wraps Convey's
`IMongoRepository<UserDocument, Guid>` `[convey]` and translates documents to entities with
`AsEntity()`.

**Representation & storage.** The `users` collection, registered at
`…Infrastructure/Extensions.cs:86` (`AddMongoRepository<UserDocument, Guid>("users")`), in the
`identity-service` database (`…Api/appsettings.json:100-104`).

**Lifecycle.** `AddAsync` is called once per user (`…/IdentityService.cs:103`). `GetAsync(Guid)` is
called from `IdentityService.GetAsync:44` and `RefreshTokenService.UseAsync:63`.
`GetAsync(string)` is called from `SignInAsync:57` and `SignUpAsync:93`.

**Invariants & enforcement.**

- **There is no `UpdateAsync` and no `DeleteAsync` on the port at all.** This is not an oversight
  that the adapter papers over — the adapter does not implement them either. Consequently the
  service has, by construction: no password change, no password reset, no email change, no role
  change, no permission change, no account deactivation, and no account deletion.
- `GetAsync(string email)` lower-cases its argument (`:28`) to match the canonical stored form
  (§3.6), with **no null guard** — see §3.6 for why that is unreachable today.
- The `Guid` overload delegates to Convey's `GetAsync(id)`, which queries `_id` `[convey]`, so it
  is index-backed by definition.
- The `string` overload uses a predicate — `x => x.Email == email.ToLowerInvariant()` — which
  Convey translates to a filter `[convey]`. It is index-backed **only if** the startup index
  actually got created (§3.24).

**Extension procedure.** Add the method to the port, implement it in the adapter as a one-liner
over `_repository` (copy `RefreshTokenRepository.cs:28`), and add a corresponding mutator to the
`User` aggregate — private setters mean the entity cannot be changed otherwise. If the new query is
not by `_id` or `Email`, also add an index at `…Infrastructure/Mongo/Extensions.cs:13-28` **and**
read §3.24 first, because that mechanism is unsound.

**Failure modes.** The absence of an update path is the single largest functional gap in the
service, and it is invisible from the outside: the surface (`api-inventory.md` §4.4) simply has no
such endpoint, so a reader may assume the capability exists elsewhere. It does not.

### 3.22 `IRefreshTokenRepository` / `RefreshTokenRepository` — lookup on an unindexed field

**Definition.** `GetAsync(string token)`, `AddAsync`, `UpdateAsync`
(`…Core/Repositories/IRefreshTokenRepository.cs:6-11`), implemented over
`IMongoRepository<RefreshTokenDocument, Guid>`
(`…Infrastructure/Mongo/Repositories/RefreshTokenRepository.cs:10-29`).

**Representation & storage.** The `refreshTokens` collection, registered at
`…Infrastructure/Extensions.cs:85`.

**The load-bearing line:**

```csharp
// …Infrastructure/Mongo/Repositories/RefreshTokenRepository.cs:21
var refreshToken = await _repository.GetAsync(x => x.Token == token);
```

`Token` is **not the `_id`** — the `_id` is a separate `AggregateId`-derived `Guid` minted at
`RefreshTokenService.CreateAsync:32`. And **no index is created on `refreshTokens`**: the only
index-creation site in the entire repository is `…Infrastructure/Mongo/Extensions.cs:13-28`, which
touches `users.Email` and nothing else. Every refresh-token lookup is therefore a **full collection
scan** over a collection that grows by one document per sign-in and is never pruned (§3.8).

**Lifecycle.** `AddAsync` once per sign-in; `GetAsync` once per `refresh-tokens/use` and once per
`refresh-tokens/revoke`; `UpdateAsync` once per revoke.

**Invariants & enforcement.** None beyond the aggregate's. In particular there is no unique index
on `Token`, so nothing detects a duplicate; and `GetAsync` returns the *first* match, so a duplicate
would be resolved arbitrarily.

**Extension procedure.** To add "revoke all tokens for a user", add
`Task<IEnumerable<RefreshToken>> GetAllAsync(Guid userId)` to the port, implement it with
`_repository.FindAsync(x => x.UserId == userId)` `[convey]`, and **add an index on `UserId`** —
otherwise you have converted an O(n) scan into an O(n) scan plus n updates.

**Failure modes.** Linear degradation of both token endpoints as the collection grows; unbounded
storage growth; and no way to enumerate or bulk-revoke a user's tokens.

### 3.23 Mongo documents and the mapping seam

**Definition.** Two `internal sealed` document types —
`UserDocument` (`…Infrastructure/Mongo/Documents/UserDocument.cs:7-15`) and
`RefreshTokenDocument` (`…/RefreshTokenDocument.cs:6-13`) — each implementing Convey's
`IIdentifiable<Guid>` `[convey]`, plus five extension methods in
`…Infrastructure/Mongo/Documents/Extensions.cs:7-46` that convert between documents, entities and
DTOs.

**Representation & storage.** Serialisation is the MongoDB C# driver's default POCO mapping: public
properties become BSON elements with the **C# property name verbatim** (`Id`, `Email`, `Role`,
`Password`, `CreatedAt`, `Permissions`). `Id` maps to `_id` via `IIdentifiable<Guid>` `[convey]`.
Nothing in this repository registers a `BsonClassMap`, a custom convention pack, an element-name
attribute or an `[BsonIgnoreExtraElements]` attribute — so the on-disk field names are exactly the
CLR property names, and **an unknown element in a stored document will throw on deserialisation**
`[framework]` unless the driver's default ignores it. Whether the driver's default is
ignore-or-throw for extra elements is `Unverifiable — Missing Source Evidence` here; the safe
assumption for a maintainer is that **removing a property is riskier than adding one**.

**The five mappers:**

| Mapper | Direction | Style | Risk of a silent omission |
| --- | --- | --- | --- |
| `UserDocument.AsEntity()` (`:9-11`) | doc → entity | **positional** constructor call | none — compile error |
| `User.AsDocument()` (`:13-22`) | entity → doc | object initialiser | **high — silent data loss** |
| `UserDocument.AsDto()` (`:24-32`) | doc → DTO | object initialiser | **high — silent field absence** |
| `RefreshTokenDocument.AsEntity()` (`:34-35`) | doc → entity | **positional** | none |
| `RefreshToken.AsDocument()` (`:37-45`) | entity → doc | object initialiser | **high** |

`AsDto` exists to let a read skip the domain entirely — it is used only by the orphaned
`GetUserHandler` (§3.25). Every live read goes `document → AsEntity → new UserDto(user)`
(`…Application/DTO/UserDto.cs:19-26`), i.e. through the aggregate's validation.

**Lifecycle.** Per repository call.

**Invariants & enforcement.** `AsDocument` defaults `Permissions` to empty (`:21`); `AsDto` does
the same (`:31`); `AsEntity` relies on the `User` constructor's default (`User.cs:39`). All three
therefore agree, but by three independent code paths.

**Extension procedure.** See §3.1 step 3. The rule of thumb: **change all five mappers in one
commit**, and prefer positional construction where possible, because it is the only shape the
compiler checks.

**Failure modes.** The `AsEntity` path re-runs the aggregate's validation on every read
(`User.cs:19-32`), so a document whose `Role` was valid when written but is no longer in
`Role.IsValid` — e.g. after someone removes a role constant — makes **every read of that user throw
`InvalidRoleException`**, surfacing as a `400 invalid_role` on `GET /me`. Removing a role from
`Role.cs` is therefore a breaking data change, not a code-only change.

### 3.24 The startup unique index — the platform's only schema action

**Definition.** `UseMongo(this IApplicationBuilder)`
(`…Infrastructure/Mongo/Extensions.cs:13-28`), called from `UseInfrastructure`
(`…Infrastructure/Extensions.cs:98`). It is the **only** index creation and the **only** schema
operation of any kind anywhere in the fourteen-repository workspace.

**The code, verbatim (`…Infrastructure/Mongo/Extensions.cs:13-28`):**

```csharp
public static IApplicationBuilder UseMongo(this IApplicationBuilder builder)
{
    using (var scope = builder.ApplicationServices.CreateScope())
    {
        var users = scope.ServiceProvider.GetService<IMongoRepository<UserDocument, Guid>>().Collection;
        var userBuilder = Builders<UserDocument>.IndexKeys;
        Task.Run(async () => await users.Indexes.CreateOneAsync(
            new CreateIndexModel<UserDocument>(userBuilder.Ascending(i => i.Email),
                new CreateIndexOptions
                {
                    Unique = true
                })));
    }

    return builder;
}
```

**Three independent defects in twelve lines:**

1. **The task is never awaited.** `Task.Run(...)` is fire-and-forget. `UseMongo` returns
   immediately and the application finishes starting up while index creation is still in flight.
   Requests can therefore be served — including sign-ups — **before the unique index exists**.
2. **Failures are swallowed.** Nothing observes the returned `Task`. If `CreateOneAsync` throws —
   Mongo unreachable, insufficient privileges, or (the common case) **a pre-existing duplicate
   email that makes a unique index impossible to build** — the exception becomes an unobserved task
   exception. The service starts, reports healthy, and runs permanently without its only
   uniqueness guarantee. This is a **silent** failure of the strongest kind: the safety net is
   absent and nothing says so.
3. **The DI scope is disposed while the task is running.** The `using` block ends at `:25`,
   disposing the scope that produced `users`. The `IMongoCollection` reference itself is captured,
   so this happens to work, but the pattern is unsound: any future use of a scoped dependency
   inside that lambda would race against `ObjectDisposedException`.

**Lifecycle.** Once per process start. Note that with multiple replicas, every replica issues the
same `CreateOneAsync`; Mongo makes repeated identical index creation idempotent `[framework]`, so
concurrency between replicas is benign — the race that matters is against the *first request*, not
against the other replicas.

**Invariants & enforcement.** The index is the *only* structural guard on email uniqueness; the
application-level check in `SignUpAsync:93-98` is racy (§3.14). When the index does exist and a
duplicate insert is attempted, the driver throws `MongoWriteException`, which
`ExceptionToResponseMapper` maps to a generic `400 {code:"error"}` (§3.19) rather than
`email_in_use` — so even the *working* path reports the wrong error.

**Extension procedure — adding an index.**

1. Add a second `CreateOneAsync` call inside the same method, following the existing shape.
2. **Fix the shape while you are there**: make `UseMongo` await, or move index creation into an
   `IHostedService`/`StartAsync` so that failures surface and startup is ordered. There is no
   precedent in this workspace for either — see §5.4.
3. There is no migration runner, no version collection and no idempotency ledger, so index
   creation must be safe to re-run on every start. Mongo's `CreateOneAsync` is, provided the index
   *definition* is unchanged; changing the definition of an existing index name throws.

**Failure modes.** All three defects above; plus the fact that this is the only place a reader
would ever look for schema management, and it is easy to miss because it is a two-file extension
method named identically to Convey's own `UseMongo`.

### 3.25 `GetUser` / `GetUserHandler` — a complete, orphaned query path

**Definition.** `GetUser : IQuery<UserDto>` with a single `Guid UserId`
(`…Application/Queries/GetUser.cs:7-10`) and `GetUserHandler : IQueryHandler<GetUser, UserDto>`
(`…Infrastructure/Mongo/Queries/Handlers/GetUserHandler.cs:11-26`), which reads the document
directly and maps with `AsDto` — bypassing the aggregate.

**Representation & storage.** Registered by `AddQueryHandlers()` and dispatchable via
`AddInMemoryQueryDispatcher()` (`…Infrastructure/Extensions.cs:72-73`).

**Lifecycle — it never runs.** `Program.cs:35` registers
`.Get<GetUser>("users/{userId}", (query, ctx) => GetUserAsync(query.UserId, ctx))`. The overload
with a lambda **replaces** dispatch: the lambda receives the bound query and does what it likes
`[convey]`. What it does is call `IIdentityService.GetAsync(id)` (`Program.cs:80`), which goes
`IUserRepository → AsEntity → new UserDto(user)`. `IQueryDispatcher` is **never injected or called
anywhere in the repository** (grep: zero hits outside the registration at `:73`). `GetUserHandler`
is therefore dead.

**Why it matters.** The two paths are not equivalent. `GetUserHandler` uses `AsDto`, which does
**not** re-run the aggregate's validation; the live path uses `AsEntity`, which does (§3.23). A
document with an out-of-vocabulary role reads fine through the dead handler and throws through the
live one. A maintainer who "fixes" `Program.cs:35` to use dispatch would silently change the
service's failure behaviour.

**Invariants & enforcement.** None; it is unreachable.

**Extension procedure.** Either delete `GetUser`/`GetUserHandler` and
`AddQueryHandlers()`/`AddInMemoryQueryDispatcher()` (`…Infrastructure/Extensions.cs:72-73`), or
adopt them by changing `Program.cs:35` to `.Get<GetUser>("users/{userId}")` (the dispatch overload)
and deleting `GetUserAsync` at `:78-88`. Note that the dispatch overload's not-found behaviour is
Convey's, not the hand-written `404` at `:83` — `Unverifiable — Missing Source Evidence` as to
whether it returns 404 or 200-with-null.

**Failure modes.** Reader confusion, and the divergent validation behaviour above.

### 3.26 Raw endpoint registration instead of dispatcher endpoints

**Definition.** All eight HTTP routes are registered inline in `Program.cs:33-72` through Convey's
`UseEndpoints` builder with **explicit lambdas**, resolving services from `ctx.RequestServices`.

**The full route table (`Program.cs`):**

| Line | Route | Handler body |
| --- | --- | --- |
| `:34` | `GET ""` | writes `AppOptions.Name` |
| `:35` | `GET users/{userId}` | `GetUserAsync(query.UserId, ctx)` |
| `:36-46` | `GET me` | `AuthenticateUsingJwtAsync()`; `401` if empty; else `GetUserAsync` |
| `:47-51` | `POST sign-in` | `IIdentityService.SignInAsync(cmd)` → JSON |
| `:52-56` | `POST sign-up` | `IIdentityService.SignUpAsync(cmd)` → `201` + `Location: identity/me` |
| `:57-61` | `POST access-tokens/revoke` | `IAccessTokenService.DeactivateAsync` → `204` |
| `:62-66` | `POST refresh-tokens/use` | `IRefreshTokenService.UseAsync` → JSON |
| `:67-71` | `POST refresh-tokens/revoke` | `IRefreshTokenService.RevokeAsync` → `204` |

**Why this matters — three consequences.**

1. **The command dispatcher is bypassed.** `AddInMemoryCommandDispatcher()` is registered
   (`…Application/Extensions.cs:13`) but `ICommandDispatcher` is never resolved. `SignUpHandler`
   (`…Application/Commands/Handlers/SignUpHandler.cs`) exists but is invoked **only** by the AMQP
   subscription (§3.29).
2. **Therefore the outbox decorator is bypassed on HTTP.** `TryDecorate(typeof(ICommandHandler<>), …)`
   (`…Infrastructure/Extensions.cs:67`) decorates *handlers*. No handler runs on the HTTP path, so
   `OutboxCommandHandlerDecorator.HandleAsync` — and with it the inbox de-duplication keyed on
   `messageId` — never executes for a web request (§3.30). Events published from the HTTP path
   still go through `MessageBroker`, which independently consults `_outbox.Enabled` (§3.31), so
   they *are* written to the outbox; what is lost is **command-level idempotency**, not event
   durability.
3. **Status codes are hand-chosen and inconsistent.** 200-with-body, 201, 204 and a hand-written
   401/404 coexist with the mapper's blanket 400 (§3.19).

**Extension procedure.** Follow the existing shape; see §3.14 step 5. If you want dispatcher
semantics (and the outbox decorator) for a new write, register
`.Post<TCommand>("path")` **without** a lambda and add an `ICommandHandler<TCommand>` — but be
aware that this changes the response to Convey's default (`202`/`Location` behaviour, `[convey]`,
`Unverifiable — Missing Source Evidence`), not the hand-written codes above.

**Failure modes.** The lambdas are untyped with respect to failure: any exception escapes to the
error middleware and becomes a 400 (§3.19). And because routes are declared in `Program.cs` rather
than by attribute, there is no route table to inspect and nothing to generate an accurate OpenAPI
document from — `AddWebApiSwaggerDocs()` (`…Infrastructure/Extensions.cs:87`) documents what Convey
can infer `[convey]`.

### 3.27 `AuthenticateUsingJwtAsync` — the only in-service identity check

**Definition.**

```csharp
// …Infrastructure/Extensions.cs:108-113
public static async Task<Guid> AuthenticateUsingJwtAsync(this HttpContext context)
{
    var authentication = await context.AuthenticateAsync(JwtBearerDefaults.AuthenticationScheme);
    return authentication.Succeeded ? Guid.Parse(authentication.Principal.Identity.Name) : Guid.Empty;
}
```

**Representation & storage.** Stateless; relies on the JWT bearer scheme registered by `AddJwt()`
(`…Infrastructure/Extensions.cs:74`) and activated by `UseAuthentication()` (`:101`).

**Lifecycle.** One call site: `Program.cs:38`, for `GET me`. `Guid.Empty` → `401` (`:39-43`);
otherwise the id is passed to `GetUserAsync`.

**Invariants & enforcement.**

- The sentinel is `Guid.Empty`, so an authenticated principal whose `Identity.Name` happened to be
  the empty Guid would be treated as unauthenticated. Unreachable in practice (`AggregateId`
  forbids empty ids, §3.2), but the sentinel pattern is fragile.
- `Guid.Parse` — not `TryParse` — will **throw** if the subject is not a Guid, and the thrown
  `FormatException` becomes a `400 {code:"error"}` (§3.19) rather than a `401`. Any token minted by
  a different issuer with a non-Guid subject therefore produces a confusing 400.
- **`GET users/{userId}` calls nothing of the sort.** It goes straight to `GetUserAsync`
  (`Program.cs:35`). Whether an unauthenticated caller can reach it depends entirely on Convey's
  `UseAccessTokenValidator()` and `allowAnonymousEndpoints` (§3.28) — and the *authorization* to
  read someone else's record depends entirely on the gateway's `claims: role: admin`
  (`ntrada.yml:243-246`). A caller with direct network access to port 5004 bypasses that check
  completely. See §8.2/B2.

**Extension procedure.** For a new authenticated route, call `AuthenticateUsingJwtAsync()` in the
lambda and handle `Guid.Empty` — there is no attribute-based alternative because there are no
controllers. For a role check, you would have to read `authentication.Principal` claims by hand;
nothing in this repository does so (`IdentityContext.IsAdmin` exists but is never consumed, §3.32).

**Failure modes.** `Guid.Parse` throwing; the `Guid.Empty` sentinel; and the fact that this helper
is the service's entire in-process authorization vocabulary.

### 3.28 `UseAccessTokenValidator()` and `allowAnonymousEndpoints`

**Definition.** A Convey middleware inserted at `…Infrastructure/Extensions.cs:97`, configured by
`jwt.allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` (`…Api/appsettings.json:44`, repeated
identically in `appsettings.docker.json:39`).

**Representation & storage.** Pure middleware; its state is the Redis deny-list (§3.12).

**Lifecycle.** Runs on every HTTP request, positioned *before* `UseAuthentication()` (`:101`) in
the `UseInfrastructure` chain.

**Invariants & enforcement.** The precise contract — whether the middleware *requires* a token on
non-anonymous paths or merely *rejects deactivated* ones — is internal to Convey and is
`Unverifiable — Missing Source Evidence`. Two facts constrain the answer from this side:

- `GET me` re-authenticates by hand and returns `401` itself (`Program.cs:38-43`), which would be
  redundant if the middleware already rejected anonymous requests. This suggests the middleware is
  **deny-list only**.
- The anonymous list contains only `/sign-in` and `/sign-up` — not `/`, not `/users/{userId}`, not
  the three token routes. If the middleware were a hard gate, `GET /` (the name endpoint) and
  Consul's `pingEndpoint: "ping"` (`…Api/appsettings.json:14`) would both require a token, which
  would break health checking. This is further evidence for deny-list-only semantics.

Both arguments are inferential. **Resolving this is the single highest-value verification a
maintainer can do** (§8.3/Q2), because it determines whether `GET users/{userId}` and
`POST access-tokens/revoke` are anonymous at the service port.

**Extension procedure.** Add a path to `allowAnonymousEndpoints` in **both** `appsettings.json:44`
and `appsettings.docker.json:39`, or the behaviour will differ between local and containerised
runs. Note the entries are absolute (`/sign-in`) while `Program.cs` registers relative paths
(`"sign-in"`); Convey normalises `[convey]`.

**Failure modes.** Config drift between the two profiles; and the unresolved semantics above.

### 3.29 `SubscribeCommand<SignUp>()` — the AMQP write path

**Definition.** The final call in `UseInfrastructure`
(`…Infrastructure/Extensions.cs:103`): `.SubscribeCommand<SignUp>()`. Convey binds a queue named
by the template `identity-service/{{exchange}}.{{message}}` (`…Api/appsettings.json:148`) — i.e.
`identity-service/identity.sign_up` — to the `identity` topic exchange with routing key `sign_up`,
and dispatches each message to `ICommandHandler<SignUp>` `[convey]`.

**Representation & storage.** `SignUpHandler`
(`…Application/Commands/Handlers/SignUpHandler.cs:8-18`) is an `internal sealed` "simple wrapper"
(the source comment at `:7` says exactly that) delegating to `IIdentityService.SignUpAsync`.
Because it *is* an `ICommandHandler<>`, it **is** decorated by `OutboxCommandHandlerDecorator`
(§3.30) — the AMQP path is the only idempotent one.

**Lifecycle — currently unreachable.** A grep for `sign_up` / `SignUp` publication across all
fourteen clones finds:

| Candidate producer | Verdict |
| --- | --- |
| API gateway sync manifests (`ntrada.yml:256-261`, `ntrada.docker.yml`) | `use: downstream` — HTTP proxy, not AMQP |
| API gateway async manifests (`ntrada-async.yml`, `ntrada-async.docker.yml`) | **also `use: downstream`** for all four identity routes — unlike every other module, identity is never switched to `use: rabbitmq` |
| Any Pacco service | none publishes `SignUp` |
| `operations-service` `messages.json:62-65` | *subscribes* `sign_in`/`sign_up`, does not publish |

So the subscription is live, the queue is declared and bound, and nothing ever puts a message in
it. `SignIn` is not subscribed at all, which means `operations-service`'s
`operations-service/identity.sign_in` queue is likewise permanently empty.

**Invariants & enforcement.** The handler enforces nothing itself; all invariants are
`IdentityService`'s (§3.14). Failures are mapped by `ExceptionToMessageMapper` — with the defect in
§3.18.

**Extension procedure — making identity async like its siblings.**

1. In `ntrada-async.yml` **and** `ntrada-async.docker.yml`, change the `POST /sign-up` route from
   `use: downstream` to `use: rabbitmq` with `config: {exchange: identity, routing_key: sign_up}`,
   mirroring `ntrada-async.yml:117-121` for `availability`.
2. Fix `ExceptionToMessageMapper.cs:20` (§3.18) first — otherwise invalid-email sign-ups become
   silent, unbounded hangs in the UI.
3. Decide what the caller receives: async routes return `202` with a correlation id and the client
   polls `operations-service` (`ntrada-async.yml`, `component-internals/api-gateway.md` §3.7). The
   existing `resourceId: {property: userId, generate: true}` already generates the id.
4. `sign_up_rejected` and `signed_up` are already in `messages.json:66-73`, so the UI side needs no
   change.

**Failure modes.** A live-but-unfed subscription is invisible in monitoring (RabbitMQ shows a
bound queue with zero throughput, indistinguishable from a quiet one); and the defect in §3.18
means the first message ever delivered to it may fail silently.

### 3.30 The outbox decorators and the enabled/disabled matrix

**Definition.** Two Scrutor-style decorators registered with `TryDecorate`
(`…Infrastructure/Extensions.cs:67-68`): `OutboxCommandHandlerDecorator<TCommand>`
(`…Infrastructure/Decorators/OutboxCommandHandlerDecorator.cs:11-36`) and
`OutboxEventHandlerDecorator<TEvent>`. They wrap `_outbox.HandleAsync(messageId, () => inner(...))`
`[convey]`, which records the message id so a redelivery is a no-op.

**Representation & storage.** The `inbox` and `outbox` collections of the `identity-service`
database (`…Api/appsettings.json:105-113`), registered by `AddMessageOutbox(o => o.AddMongo())`
(`…Infrastructure/Extensions.cs:80`).

**The message-id rule (`OutboxCommandHandlerDecorator.cs:26-29`):**

```csharp
var messageProperties = messagePropertiesAccessor.MessageProperties;
_messageId = string.IsNullOrWhiteSpace(messageProperties?.MessageId)
    ? Guid.NewGuid().ToString("N")
    : messageProperties.MessageId;
```

A message arriving over AMQP carries a `MessageId` and is therefore genuinely de-duplicated. A
handler invoked with no ambient message properties gets a **fresh random id every time**, which
de-duplicates nothing. Since the HTTP path does not run handlers at all (§3.26), the fallback
branch is effectively unreachable here — but it is exactly the branch a maintainer would rely on if
they switched `Program.cs` to dispatcher endpoints, and it would silently do nothing.

**Configuration matrix — note the profile asymmetry:**

| Profile | `outbox.enabled` | Source |
| --- | --- | --- |
| default (`appsettings.json`) | **true**, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true` | `…Api/appsettings.json:105-113` |
| `local` | **false** | `appsettings.local.json:38-40` |
| `docker` | **true** — `appsettings.docker.json` has **no `outbox` section at all**, so the base value stands | absence in `appsettings.docker.json` |

That last row is the trap: a developer who tests locally never exercises the outbox, and the Docker
deployment always does. `disableTransactions: true` means the outbox write and the domain write are
**not atomic** — the platform's Mongo is a single node with no replica set
(`hianshul100_Pacco/compose/mongo-rabbit-redis.yml`), so multi-document transactions are
unavailable. The outbox therefore provides *at-least-once delivery with retry*, **not**
transactional consistency: a crash between `AddAsync` (`…/IdentityService.cs:103`) and the outbox
write (`:106`) loses the `SignedUp` event, and the user exists with no customer profile.

**Extension procedure.** Nothing to do per-message — the decorators are open-generic. To make the
guarantee real you would need a replica set and `disableTransactions: false`, in every service.

**Failure modes.** As above; plus the fact that with the outbox enabled, every publish costs a
Mongo write and is delivered up to 2 seconds late (`intervalMilliseconds: 2000`), and entries
expire after an hour (`expiry: 3600`) — an outbox that cannot drain for an hour **loses messages
silently**.

### 3.31 `MessageBroker` — the publish seam

**Definition.** `IMessageBroker.PublishAsync(params IEvent[])`
(`…Application/Services/IMessageBroker.cs:5-8`), implemented by
`…Infrastructure/Services/MessageBroker.cs:16-87` and registered transient at
`…Infrastructure/Extensions.cs:62`.

**Representation & storage.** Composes `IBusPublisher`, `IMessageOutbox`,
`ICorrelationContextAccessor`, `IHttpContextAccessor`, `IMessagePropertiesAccessor`, `ITracer` and
the RabbitMQ options (`:28-43`).

**Lifecycle — `PublishAsync` (`:47-86`).**

1. Null-guard the collection (`:49-52`) — **silently returns** on `null`.
2. Read the ambient inbound message's id, correlation id and span context (`:54-57`).
3. If no span context is present on the message, fall back to `_tracer.ActiveSpan` (`:58-61`).
4. Collect headers to forward — **`Saga` only** (`:63`, §3.33).
5. Resolve the correlation context: ambient AMQP context first, else the `Correlation-Context`
   HTTP header (`:64-65`, §3.32).
6. For each event: skip `null` (`:69-72`, **silent**); mint a **fresh** `messageId`
   (`:74` — `Guid.NewGuid().ToString("N")`, so every event gets a new id even on republish); then
   either `_outbox.SendAsync(...)` if `_outbox.Enabled` (`:76-81`) or `_busPublisher.PublishAsync(...)`
   (`:83-84`).

**Invariants & enforcement.** The `originatedMessageId` (`:55`) is threaded through so a consumer
can trace causation. The **correlation id is inherited, not minted** — if there is no ambient
message and no HTTP correlation header, `correlationId` is `null`, and a downstream
`operations-service` handler will see a blank `CorrelationId` and **silently return without
recording anything** (`component-internals/operations-service.md` §3.6). This is the mechanism by
which an HTTP-originated `SignedUp` may or may not appear in the operations UI, depending entirely
on whether the gateway supplied a correlation context.

**Extension procedure.** Publish through `IMessageBroker`, never through `IBusPublisher` directly —
the latter would bypass the outbox, the header forwarding and the correlation resolution. Note the
interface declares only the `params` overload (`IMessageBroker.cs:6`) while the implementation also
offers an `IEnumerable<IEvent>` overload (`MessageBroker.cs:47`); the enumerable one is reachable
only through the concrete type.

**Failure modes.** Two silent early-returns (`null` collection, `null` element); a fresh
`messageId` per publish that defeats any consumer-side de-duplication on republish; and inherited-
or-absent correlation ids.

### 3.32 Correlation context — built, never read

**Definition.** Three cooperating pieces: `CorrelationContext`
(`…Infrastructure/Contexts/CorrelationContext.cs:6-24`, mirroring the gateway's shape:
`CorrelationId`, `SpanContext`, `User`, `ResourceId`, `TraceId`, `ConnectionId`, `Name`,
`CreatedAt`), `AppContextFactory` (`…Infrastructure/Contexts/AppContextFactory.cs:8-34`), and the
`IAppContext`/`IIdentityContext` pair (`…Application/IAppContext.cs`,
`…Application/IIdentityContext.cs`).

**Representation & storage.** Transported as the `Correlation-Context` HTTP request header, read by
`GetCorrelationContext(this IHttpContextAccessor)` (`…Infrastructure/Extensions.cs:115-118`), or as
the AMQP `message_context` header (`…Api/appsettings.json:150-153`) surfaced through Convey's
`ICorrelationContextAccessor` `[convey]`.

**Lifecycle.** `AppContextFactory.Create()` prefers the ambient AMQP context, round-tripping it
through JSON to re-type it (`AppContextFactory.cs:23-27` — a serialise-then-deserialise because the
Convey type and the local type are structurally identical but nominally different), and otherwise
falls back to the HTTP header (`:30-32`). It is registered as a factory-backed transient at
`…Infrastructure/Extensions.cs:65-66`.

**It is never consumed.** A grep for `IAppContext` finds the interface, the implementation, the
factory and the registration — and **no injection site anywhere**. `IdentityContext.IsAdmin`
(`…Infrastructure/Contexts/IdentityContext.cs:29`) and `IdentityContext.Claims` are likewise never
read. The only *live* use of correlation data is `MessageBroker`'s direct call to
`_httpContextAccessor.GetCorrelationContext()` (`MessageBroker.cs:65`), which uses the raw
`CorrelationContext`, not `IAppContext`.

**Invariants & enforcement.** `GetCorrelationContext` uses
`Headers.TryGetValue(...) is true ? JsonConvert.DeserializeObject<CorrelationContext>(json.FirstOrDefault()) : null`
(`Extensions.cs:116-117`) — **malformed JSON throws** `JsonReaderException`, which becomes a
`400 {code:"error"}` (§3.19). A missing header yields `null`, handled everywhere.
`IdentityContext`'s constructor uses `Guid.TryParse` and defaults to `Guid.Empty`
(`IdentityContext.cs:26`) — **silent** on a malformed id.

**Extension procedure.** To use the caller's identity inside the application layer, inject
`IAppContext` — the plumbing already exists and works; only the consumers are missing. This is the
intended seam for, say, "only an admin may sign someone up with `role: admin`" (§3.4).

**Failure modes.** A complete, tested-looking abstraction that does nothing, which will lead a
reader to believe identity propagation is handled. The one real consumer (`MessageBroker`) reaches
around it.

### 3.33 `Saga` header forwarding — a one-header allow-list

**Definition.**

```csharp
// …Infrastructure/Extensions.cs:120-134
private static IDictionary<string, object> GetHeadersToForward(IMessageProperties messageProperties)
{
    const string sagaHeader = "Saga";
    if (messageProperties?.Headers is null || !messageProperties.Headers.TryGetValue(sagaHeader, out var saga))
    {
        return null;
    }

    return saga is null
        ? null
        : new Dictionary<string, object> { [sagaHeader] = saga };
}
```

**Representation & storage.** An AMQP header on the inbound message; copied verbatim onto every
event published while handling it (`MessageBroker.cs:63`, then `:78`/`:84`).

**Lifecycle.** Read once per `PublishAsync` call, from the ambient `IMessageProperties`.

**Invariants & enforcement.** Exactly one header is forwarded. `Correlation-Context`, `span_context`
and everything else are *not* forwarded as headers — the first two are handled explicitly by
`MessageBroker` (`:56-57`, `:64-65`) and re-attached as their own arguments. The allow-list is
deliberate but hard-coded: adding a header to the platform's conventions requires editing this
method **in every service**, and there are eight copies of it across the workspace.

**Why the `Saga` header exists.** It carries the saga state (`pending`/`completed`/`rejected`) that
`operations-service` reads in `Handlers/Extensions.cs:9-26` to decide the `OperationState` it
records. Because identity's HTTP path has no inbound message properties, `GetHeadersToForward`
returns `null` for every HTTP-originated `SignedUp` — so `operations-service` falls back to its
per-handler default (`Completed` for events, `…/Handlers/GenericEventHandler.cs:40`), which happens
to be right.

**Extension procedure.** To forward a new header, extend the dictionary here; to forward it
*platform-wide*, repeat the edit in each service's `Infrastructure/Extensions.cs`.

**Failure modes.** Silent divergence between services if the allow-lists drift; and the fact that a
`null` return means "forward nothing", which is indistinguishable from "there was no inbound
message".

### 3.34 Span context and Jaeger tracing

**Definition.** `GetSpanContext<T>(IMessageProperties, T message)`
(`…Infrastructure/Extensions.cs:136-149`) plus `AddJaeger()`/`UseJaeger()`
(`…Infrastructure/Extensions.cs:76`, `:94`) and the `jaeger` configuration block
(`…Api/appsettings.json:19-31`).

**The resolution order (`Extensions.cs:136-149`):**

1. If the inbound message properties carry a `span_context` header **and** it is non-blank, return
   it (`:138-145`).
2. Otherwise, build a span from the active tracer scope via `_tracer.BuildSpan(...)` `[convey]` and
   return its serialised context (`:147-148`).

`MessageBroker` applies the same two-step fallback inline at `:56-61`, so the helper and the broker
implement the rule twice.

**Representation & storage.** The `span_context` AMQP header name is configured at
`…Api/appsettings.json:151` (`spanContextHeader: "span_context"`). Jaeger is configured with
`serviceName: "identity-service"`, `udpHost: "jaeger"`, `udpPort: 6831`, `maxPacketSize: 0`,
`sampler: "const"`, `maxStackTraceLength: 0` (`…Api/appsettings.json:19-31`), and
`appsettings.local.json:19-21` sets `jaeger.enabled: false`.

**Lifecycle.** Per request (HTTP middleware) and per message (publish/consume).

**Invariants & enforcement.** `sampler: "const"` with the default parameter samples **every** trace
`[framework]` — acceptable for a reference platform, expensive under load. Nothing enforces that a
span is closed; that is Convey/Jaeger's middleware.

**Extension procedure.** No per-feature work: tracing is ambient. To trace an internal step, inject
`ITracer` and `BuildSpan` — nothing in this repository does so outside `MessageBroker`.

**Failure modes.** With `jaeger.enabled: false` (local), `ITracer` resolves to a no-op tracer
`[convey]`, so `GetSpanContext`'s fallback branch returns an empty/valueless context — meaning
locally-published messages carry a `span_context` header with no useful content. Harmless, but it
means "the header is present" does not imply "the trace is joinable".

### 3.35 `UsePublicContracts<ContractAttribute>()` — the contract registry

**Definition.** `…Infrastructure/Extensions.cs:99`, paired with the marker attribute
`…Application/ContractAttribute.cs:6-9` (`[AttributeUsage(AttributeTargets.Class)] public class ContractAttribute : Attribute`).

**Representation & storage.** Convey scans the loaded assemblies for types bearing the attribute
and exposes them at a discovery endpoint `[convey]` (the path is Convey's default and is
`Unverifiable — Missing Source Evidence` from this repository).

**Which types are marked.** Exactly the five commands (§3.15) and the four events (§3.16, §3.17) —
`SignIn`, `SignUp`, `RevokeAccessToken`, `UseRefreshToken`, `RevokeRefreshToken`, `SignedIn`,
`SignedUp`, `SignInRejected`, `SignUpRejected`. `GetUser` is **not** marked, consistent with it
being a query.

**Lifecycle.** Assembly scan at startup; static thereafter.

**Invariants & enforcement.** Nothing verifies that the set of `[Contract]` types matches the set
of messages actually published or subscribed. `SignedIn` is a contract with no consumer (§3.16);
`SignIn` is a contract with no subscription (§3.29). The registry documents *intent*, not
*wiring* — a maintainer must not treat it as a dependency graph.

**Extension procedure.** Add `[Contract]` to any new command or event class. Omitting it has no
runtime effect on messaging — it only removes the type from the discovery endpoint — which is
exactly why the omission is easy to make and hard to notice.

**Failure modes.** Drift between the registry and reality, as above.

### 3.36 Configuration layering and the three profiles

**Definition.** `Program.cs:26-31` builds the host with Convey's `ConfigureServices`/`Configure`
lambdas; configuration itself is ASP.NET Core's default provider chain plus Convey's Vault and
Consul providers (`…Api/appsettings.json:6-17`, `:169-198`).

**The three profiles.**

| File | Selected by | Deltas from base |
| --- | --- | --- |
| `appsettings.json` | always | the base; 198 lines, every section |
| `appsettings.local.json` | `ASPNETCORE_ENVIRONMENT=local` | jaeger off (`:19-21`), **outbox off** (`:38-40`), vault off, `jwt.certificate.location: ""`, consul/fabio off, hosts → `localhost` |
| `appsettings.docker.json` | `ASPNETCORE_ENVIRONMENT=docker` (set in `Dockerfile:8`) | `jwt.certificate.location: ""`, vault off, hosts → container names; **no `outbox` section** |

**The load-bearing sections of the base file:**

| Section | Lines | Notable values |
| --- | --- | --- |
| `app` | `:2-5` | `name: "Identity Service"`, `displayBanner`, `displayVersion` |
| `consul` | `:6-17` | `service: identity-service`, `pingEndpoint: ping`, `pingInterval: 5`, `removeAfterInterval: 10` |
| `jaeger` | `:19-31` | §3.34 |
| `jwt` | `:32-45` | §3.10 — `expiryMinutes: 60`, `issuer: pacco`, `validateAudience: false`, `validateIssuer: false`, `allowAnonymousEndpoints`, `certificate.location: "certs/localhost.pfx"`, `certificate.password: "test"` |
| `logger` | `:46-95` | console + file + seq; `excludeProperties: ["api_key","access_key","Authorization"]` |
| `mongo` | `:100-104` | `connectionString: "mongodb://localhost:27017"`, `database: "identity-service"`, `seed: false` |
| `outbox` | `:105-113` | §3.30 |
| `rabbitMq` | `:114-155` | `namespace: identity`, exchange `identity` (topic, declare true, durable true), `queueTemplate: "identity-service/{{exchange}}.{{message}}"`, `conventionsCasing: snakeCase`, `messageProcessor.enabled: false`, `spanContextHeader`, `messageContextHeader` |
| `redis` | `:156-159` | `connectionString: "localhost"`, `instance: "identity:"` |
| `security` | `:160-168` | encryption/certificate placeholders |
| `vault` | `:169-198` | §3.37 |

**Invariants & enforcement.** None. Convey binds sections to options types by name; a **misspelled
or missing section binds to a default-constructed options object** `[convey]`, which is the general
silent-failure shape of this whole layer. Two concrete instances documented elsewhere: an absent
`outbox` section leaves the base value in force (§3.30), and in `operations-service` an absent
`requests` section would yield `TimeSpan.Zero`
(`component-internals/operations-service.md` §3.4).

**Extension procedure.** Add the section to `appsettings.json` **and** decide explicitly whether
each of the other two profiles needs an override; the `outbox` asymmetry shows what happens when
that decision is skipped. If the value is a secret, add it to the Vault block too (§3.37) — and
read §8.2/B1 first.

**Failure modes.** Silent default binding; profile drift; and the fact that `logger.excludeProperties`
protects only header-shaped secrets, not the **email addresses interpolated into log messages** by
`IdentityService` (`…/IdentityService.cs:62`, `:79`, `:105`) — see §8.2/B3.

### 3.37 Vault integration and the committed secrets

**Definition.** `…Api/appsettings.json:169-198` configures Convey's Vault provider: `enabled: true`,
`url: "http://localhost:8200"`, **`authType: "token"`**, **`token: "secret"`**, `kv.enabled: true`,
`kv.path: "identity-service/settings"`, plus PKI and lease blocks.

**Representation & storage.** Values fetched from Vault at startup are merged into
`IConfiguration` `[convey]`, overriding `appsettings.json`.

**Lifecycle.** Startup only; leases are renewed by Convey's background service if
`lease.enabled` `[convey]`.

**Invariants & enforcement.** None from this repository's side. Both `appsettings.local.json` and
`appsettings.docker.json` set `vault.enabled: false`, so **Vault is inert in every runnable
profile in this workspace** — the compose stack (`hianshul100_Pacco/compose/`) does not start a
Vault container. The block is aspirational.

**What is actually committed.** Three distinct classes of secret live in source control:

| Item | Location | Nature |
| --- | --- | --- |
| Vault root-ish token `"secret"` | `…Api/appsettings.json:174` | dev token, but committed and enabled-by-default |
| `jwt.issuerSigningKey` | `…Api/appsettings.json:36` | **the symmetric signing key**, and it is *byte-identical* in `operations-service` (`…Operations.Api/appsettings.json:36`) and every other Pacco service |
| `certs/localhost.pfx`, `.key`, `.pem`, `.cer` | `src/Pacco.Services.Identity.Api/certs/` | **private key material**, published into the image by `…Api.csproj:20-22` (`<Content Include="certs\**" CopyToPublishDirectory="Always" />`) |

The shared signing key is not merely a hygiene issue: it means **any service that can read its own
config can mint a token that identity-service will accept**, because `validateIssuer` and
`validateAudience` are both `false` (§3.10). There is no asymmetry and no key rotation anywhere.

**Extension procedure.** For a real deployment: move `issuerSigningKey` and the certificate out of
the repository, switch to an asymmetric algorithm so only identity holds the private key, enable
`validateIssuer`/`validateAudience`, and give each service a distinct Vault path. All four are
out of scope for this document and are recorded as blockers (§8.2/B1).

**Failure modes.** As above. Note also that with `vault.enabled: true` in the base profile and no
Vault reachable, startup behaviour is `Unverifiable — Missing Source Evidence` (Convey may throw or
may log-and-continue) — nobody runs that combination today.

### 3.38 Consul, Fabio and metrics

**Definition.** `AddConsul()` + `AddFabio()` (`…Infrastructure/Extensions.cs:57-58`),
`UseMetrics()` (`:100`), and the `consul`/`fabio` configuration blocks
(`…Api/appsettings.json:6-17` and the `fabio` section).

**Representation & storage.** Consul registration is by service name `identity-service` with a
health check hitting `pingEndpoint: "ping"` every `pingInterval: 5` seconds and deregistering after
`removeAfterInterval: 10`. Fabio registers a routing tag so the edge proxy can reach the service by
name. Metrics are AppMetrics over Prometheus; `hianshul100_Pacco/compose/prometheus/prometheus.yml:26-32`
scrapes `identity-service` and `operations-service` explicitly.

**Lifecycle.** Registration at startup, deregistration at shutdown `[convey]`.

**Invariants & enforcement.** The `ping` endpoint is provided by Convey `[convey]`, **not** by any
route in `Program.cs` — a reader grepping `Program.cs` for a health endpoint will not find one. It
is also **not** in `allowAnonymousEndpoints` (§3.28), which is one of the arguments that the
access-token middleware is deny-list-only.

**Extension procedure.** Nothing per-feature. If the service is renamed, the name appears in at
least six places: `consul.service`, `jaeger.serviceName`, `rabbitMq.queueTemplate`, `mongo.database`,
`compose/services.yml:42-49`, and `prometheus.yml:26-32`.

**Failure modes.** A Consul health check that passes because `ping` returns 200 while the service is
functionally broken — e.g. the startup index never got built (§3.24) — is the concrete instance
here: **liveness is not readiness anywhere in this platform.**

### 3.39 Deployment topology

**Definition.** How the built artefact runs, per the repository's own scripts and the platform
compose files.

| Facet | Evidence | Value |
| --- | --- | --- |
| Container base / env | `Dockerfile:8-9` | `ASPNETCORE_ENVIRONMENT docker`, `ASPNETCORE_URLS http://*:80` |
| Compose port | `hianshul100_Pacco/compose/services.yml:42-49` | `5004:80` |
| PM2 app | `hianshul100_Pacco/services.yml:18-25` | app name `identity` |
| Local run | `scripts/start.sh` | `cd src/Pacco.Services.Identity.Api && dotnet run` |
| CI | `.travis.yml:12-14` | `./scripts/build.sh` then `./scripts/test.sh` |
| Gateway route base | `…APIGateway/…/ntrada.yml:237-242` | module `identity`, downstream `identity-service` |
| Dependencies declared | `compose/services.yml` | **none** — identity has no `depends_on` |

**Lifecycle.** Standard container lifecycle; no init containers, no migration job, no readiness
gate.

**Invariants & enforcement.** identity-service starts independently of Mongo, RabbitMQ and Redis;
Convey's clients retry `[convey]`. The one ordering hazard is §3.24 — the unique index races the
first request regardless of dependency ordering, because it races *within* the process.

**Extension procedure.** A new environment variable must be added to `Dockerfile`, `compose/services.yml`
and the matching `appsettings.<env>.json`, or it will be silently absent (§3.36).

**Failure modes.** Only port 80 is exposed inside the container, so anything that binds another port
(as `operations-service` does for gRPC — `component-internals/operations-service.md` §3.14) is
unreachable. identity-service binds nothing else, so it is unaffected.

### 3.40 The absent test suite

**Definition.** There is **no test project** in this repository. `Pacco.Services.Identity.sln`
references four projects — `Api`, `Application`, `Core`, `Infrastructure` — and nothing else. There
is no `tests/` directory, no `*.Tests.csproj`, no xUnit/NUnit/MSTest package reference in any
`.csproj`, and no `Shouldly`/`FluentAssertions`/`Moq`.

**And CI runs tests anyway.** `.travis.yml:12-14`:

```yaml
script:
  - ./scripts/build.sh
  - ./scripts/test.sh
```

`scripts/test.sh` is a one-line `dotnet test`. With no test project in the solution, `dotnet test`
finds nothing to run and **exits 0** `[framework]`. CI is therefore green by construction and has
been for the life of the repository.

**Lifecycle.** N/A.

**Invariants & enforcement.** None. Every invariant documented in §3 is enforced only by the code
that implements it; none is pinned by a test. This is why the defects in §3.13 (dead second
password check), §3.18 (`SignUpRejected` type-match), §3.24 (unawaited index) and §3.11 (`IRng`
default divergence) could all survive indefinitely: each is invisible to the compiler and there is
nothing else looking.

**Extension procedure — adding the first test.**

1. `dotnet new xunit -o tests/Pacco.Services.Identity.Tests` and add it to the `.sln`.
2. `Core` is the natural first target: it has no dependencies at all, and `User`, `Role`,
   `RefreshToken` and `AggregateId` are entirely deterministic. Every invariant table in §3.6,
   §3.4, §3.8 and §3.2 is directly executable as a test case.
3. `Application` needs `IUserRepository`, `IPasswordService`, `IJwtProvider`, `IRefreshTokenService`
   and `IMessageBroker` doubles — all five are interfaces, so no framework is required.
4. `Infrastructure` needs a live Mongo/Redis; there is no precedent for integration testing
   anywhere in this workspace and `hianshul100_Pacco/compose/` is not wired for it.

**Failure modes.** A CI badge that means nothing; and the false confidence that follows from it.

---

## 4. Primary control flows

Each flow below is traced from the true entry point through every function that executes, to the
datastore writes and side-effects it produces. Line references are to the files named in §3.

### 4.1 `POST /identity/sign-up` — account creation

**Entry.** Gateway route `ntrada.yml:256-261` (`auth: false`,
`resourceId: {property: userId, generate: true}`, `use: downstream`) → HTTP
`POST http://identity-service/sign-up` → `Program.cs:52-56`.

| # | Location | Action |
| --- | --- | --- |
| 1 | `Program.cs:52` | Convey binds the JSON body to `SignUp` via its **single constructor** (`SignUp.cs:14-23`); `UserId` is normalised to a fresh `Guid` if empty (`SignUp.cs:18`) |
| 2 | `Program.cs:53-54` | resolves `IIdentityService` from `ctx.RequestServices`, calls `SignUpAsync(cmd)` |
| 3 | `IdentityService.cs:87-91` | `EmailRegex.IsMatch(email)` — on failure throws `InvalidEmailException` |
| 4 | `IdentityService.cs:93` | `_userRepository.GetAsync(email)` → `UserRepository.cs:26-29` → Mongo `users` find on `Email` |
| 5 | `IdentityService.cs:94-99` | if found: `_logger.LogError($"Email already in use: {email}")` then `throw new EmailInUseException(email)` |
| 6 | `IdentityService.cs:100` | `role = string.IsNullOrWhiteSpace(command.Role) ? Role.User : command.Role.ToLowerInvariant()` |
| 7 | `IdentityService.cs:101` | `_passwordService.Hash(command.Password)` → `PasswordService.cs:13` → ASP.NET `IPasswordHasher<User>.HashPassword` (PBKDF2) |
| 8 | `IdentityService.cs:102` | `new User(command.UserId, command.Email, password, role, DateTime.UtcNow, command.Permissions)` — **this is where `User`'s three validations run** (`User.cs:19-32`) and where email/role are lower-cased (`:35`, `:37`) |
| 9 | `IdentityService.cs:103` | `_userRepository.AddAsync(user)` → `UserRepository.cs:31-32` → `AsDocument()` (`Documents/Extensions.cs:13-22`) → Mongo insert into `users` |
| 10 | `IdentityService.cs:105` | `_logger.LogInformation($"Created an account for the user with id: {user.Id}.")` |
| 11 | `IdentityService.cs:106` | `_messageBroker.PublishAsync(new SignedUp(user.Id, user.Email, user.Role))` |
| 12 | `MessageBroker.cs:54-65` | resolves messageId, correlationId, span context, `Saga` header (all `null` on the HTTP path — no ambient message) |
| 13 | `MessageBroker.cs:76-84` | `_outbox.Enabled` → Mongo insert into `outbox`; else direct `IBusPublisher.PublishAsync` to exchange `identity`, routing key `signed_up` |
| 14 | `Program.cs:55` | `ctx.Response.Created("identity/me")` → **`201`** with `Location: identity/me` |

**Side-effects:** one `users` document; one `outbox` document (docker/default) or one AMQP publish
(local); two log lines, **one of which contains the email address** (step 5) — §8.2/B3.

**Failure branches.** Step 3 → `400 invalid_email`; step 5 → `400 email_in_use`; step 8 →
`400 invalid_email` / `400 invalid_password` / `400 invalid_role`; step 9 duplicate-key →
**`400 {code:"error"}`** (§3.24); any Mongo/Rabbit fault → `400 {code:"error"}` (§3.19).

**The race.** Steps 4–9 are not atomic. Two concurrent sign-ups for the same address both pass
step 4 and both reach step 9; only the unique index stops the second — and only if it exists
(§3.24). If it does not, two users share an email and `GetAsync(email)` (§3.21) returns an
arbitrary one thereafter, making sign-in non-deterministic.

**The AMQP variant.** `SubscribeCommand<SignUp>()` (§3.29) delivers the same command to
`SignUpHandler.cs:16` → the identical `SignUpAsync`, but wrapped by
`OutboxCommandHandlerDecorator` (§3.30) so redelivery is idempotent, and with failures routed to
`ExceptionToMessageMapper` (§3.18) instead of the HTTP mapper. **No producer exists today.**

### 4.2 `POST /identity/sign-in` — token issuance

**Entry.** `ntrada.yml:265+` (`auth: false`, `use: downstream`) → `Program.cs:47-51`.

| # | Location | Action |
| --- | --- | --- |
| 1 | `Program.cs:47` | binds body to `SignIn` (`SignIn.cs`) |
| 2 | `IdentityService.cs:51-55` | `EmailRegex.IsMatch` → `InvalidCredentialsException` on failure (**not** `InvalidEmailException` — deliberate, avoids distinguishing "bad format" from "wrong password") |
| 3 | `IdentityService.cs:57-62` | `_userRepository.GetAsync(email)`; if `null`, log `$"User with email: {email} was not found."` and throw `InvalidCredentialsException` |
| 4 | `IdentityService.cs:63` | `_passwordService.IsValid(user.Password, command.Password)` → `PasswordService.cs:15-19` → `VerifyHashedPassword(...) != Failed` |
| 5 | `IdentityService.cs:64-68` | **the duplicated check** — the identical `IsValid` call and identical throw, executed a second time on every successful sign-in. Doubles the PBKDF2 cost (§3.13) |
| 6 | `IdentityService.cs:70` | `var claims = user.Permissions.Any() ? new Dictionary<string, IEnumerable<string>> {["permissions"] = user.Permissions} : null` |
| 7 | `IdentityService.cs:71` | `_jwtProvider.Create(user.Id, user.Role, claims: claims)` → `JwtProvider.cs:21` → `_jwtHandler.CreateToken(userId.ToString("N"), role, audience, claims)` — **`"N"` format, no dashes** (§3.10) |
| 8 | `IdentityService.cs:72` | `auth.RefreshToken = await _refreshTokenService.CreateAsync(user.Id)` |
| 9 | `RefreshTokenService.cs:31-35` | `_rng.Generate(30, true)` → `Rng.cs:12-28` → `RNGCryptoServiceProvider` → strip `/ \ = + ? : &`; `new RefreshToken(Guid.NewGuid(), userId, token, DateTime.UtcNow)`; `_refreshTokenRepository.AddAsync` → Mongo insert into `refreshTokens` |
| 10 | `IdentityService.cs:79` | `_logger.LogInformation($"User with id: {user.Id} has been authenticated.")` |
| 11 | `IdentityService.cs:80` | `_messageBroker.PublishAsync(new SignedIn(user.Id))` → outbox or bus, exchange `identity`, key `signed_in` — **no consumer anywhere** (§3.16) |
| 12 | `Program.cs:49` | `ctx.Response.WriteJsonAsync(auth)` → `200` with `AuthDto` (access token, refresh token, expiry, role, claims) |

**Side-effects:** one `refreshTokens` document per sign-in, **never pruned** (§3.8, §3.22); one
outbox/AMQP message with no consumer; two PBKDF2 verifications.

**Failure branches.** Steps 2/3/4/5 all → `400 invalid_credentials`. Note this is **400, not 401**
(§3.19) — a client cannot distinguish an authentication failure from a malformed request without
reading `code`.

### 4.3 `GET /identity/me` — the only authenticated read

**Entry.** `ntrada.yml:250-252` (`auth: true`, no role constraint) → `Program.cs:36-46`.

1. `Program.cs:38` — `ctx.AuthenticateUsingJwtAsync()` (`Extensions.cs:108-113`): ASP.NET validates
   the bearer token against `issuerSigningKey`, with `validateIssuer`/`validateAudience` **off**
   (§3.10); returns `Guid.Parse(Principal.Identity.Name)`.
2. `Program.cs:39-43` — `Guid.Empty` → `ctx.Response.StatusCode = 401`, return.
3. `Program.cs:45` → `GetUserAsync(userId, ctx)` (`Program.cs:78-88`).
4. `Program.cs:80` — `IIdentityService.GetAsync(id)` → `IdentityService.cs:44-47` →
   `_userRepository.GetAsync(id)` → Mongo `users` find by `_id` → `AsEntity()`
   (`Documents/Extensions.cs:9-11`, **re-running `User`'s validation**, §3.23) → `new UserDto(user)`.
5. `Program.cs:81-87` — `null` → `404`; else `WriteJsonAsync(user)` → `200`.

Note the middleware ordering: `UseAccessTokenValidator()` (`Extensions.cs:97`) runs *before*
`UseAuthentication()` (`:101`), so a revoked-but-otherwise-valid token is rejected by the Redis
deny-list before signature validation ever happens (§3.12).

### 4.4 `GET /identity/users/{userId}` — the admin read

**Entry.** `ntrada.yml:243-246` — `auth: true` **plus** `claims: {role: admin}`. The gateway is the
**only** place that check exists.

`Program.cs:35` → `GetUserAsync(query.UserId, ctx)` → identical to steps 4–5 of §4.3. There is **no
authentication call, no role check and no ownership check inside the service.** A caller with
direct access to `identity-service:80` reads any user record by id, subject only to whatever
`UseAccessTokenValidator()` enforces (§3.28, §8.2/B2).

### 4.5 `POST /identity/refresh-tokens/use` — token refresh

**Entry.** **No gateway route exists** for this or the other two token endpoints (§3.7). Reachable
only by direct service access. → `Program.cs:62-66`.

1. `RefreshTokenService.cs:52-55` — blank token → `EmptyRefreshTokenException`.
2. `:57` — `_refreshTokenRepository.GetAsync(refreshToken)` → **unindexed collection scan** on
   `Token` (`RefreshTokenRepository.cs:21`, §3.22).
3. `:58-61` — `null` → `InvalidRefreshTokenException`.
4. `:62` — `token.Revoked` → `RevokedRefreshTokenException`.
5. `:63-67` — `_userRepository.GetAsync(token.UserId)`; `null` → `UserNotFoundException`.
6. `:69` — `claims` built from `user.Permissions` exactly as in §4.2 step 6.
7. `:70` — `_jwtProvider.Create(user.Id, user.Role, claims: claims)` — a **new access token**.
8. `:76` — `auth.RefreshToken = refreshToken` — **the same refresh token is returned**. No rotation,
   no new document, no expiry check, no reuse detection (§3.8).
9. `:77` — `_logger.LogInformation(...)`; `Program.cs:64` writes the `AuthDto` as `200`.

**The security shape.** A refresh token is a **permanent** credential: it never expires, is never
rotated, and grants a fresh 60-minute access token indefinitely, until someone calls
`refresh-tokens/revoke` — for which **no gateway route exists**. §8.2/B4.

### 4.6 `POST /identity/refresh-tokens/revoke`

`Program.cs:67-71` → `RefreshTokenService.RevokeAsync` → same lookup as §4.5 steps 1–3 →
`RefreshToken.Revoke(DateTime.UtcNow)` (`RefreshToken.cs:33-41`, throws
`RevokedRefreshTokenException` if already revoked) → `_refreshTokenRepository.UpdateAsync(token)`
(`RefreshTokenRepository.cs:28`) → `204`. No event is published.

### 4.7 `POST /identity/access-tokens/revoke`

`Program.cs:57-61` → `IAccessTokenService.DeactivateAsync(...)` `[convey]` → Redis `SET` under the
`identity:` instance prefix (`appsettings.json:156-159`) with the token's remaining lifetime as
TTL `[convey]` → `204`. Thereafter `UseAccessTokenValidator()` (`Extensions.cs:97`) rejects that
token on every Pacco service that shares the same Redis instance and prefix. Also no gateway route.

### 4.8 Consuming a `SignUp` command over AMQP

RabbitMQ delivers to queue `identity-service/identity.sign_up` → Convey's subscriber → the
`OutboxCommandHandlerDecorator` (`Decorators/OutboxCommandHandlerDecorator.cs:31-34`) computes the
message id (`:26-29`) and calls `_outbox.HandleAsync(messageId, () => _handler.HandleAsync(command))`
`[convey]`, which records the id in the `inbox` collection and skips the body on redelivery →
`SignUpHandler.cs:16` → `IdentityService.SignUpAsync` (§4.1 steps 3–13). On exception,
`ExceptionToMessageMapper.Map` (`:11-24`) is consulted — and returns `null` for every
`InvalidEmailException` because of the `SignUpRejected command` pattern at `:20` (§3.18), so the
rejection is dropped and the caller waits forever. **No producer exists today**, so this path has
never run in anger.

### 4.9 Startup

`Program.cs:26-31` → Convey host builder → `ConfigureServices`: `AddConvey()` →
`AddApplication()` (`…Application/Extensions.cs:9-16`: command handlers, event handlers,
in-memory dispatchers, `IdentityService`, `RefreshTokenService`) → `AddInfrastructure()`
(`…Infrastructure/Extensions.cs:54-89`) → `Configure`: `UseInfrastructure()`
(`…Infrastructure/Extensions.cs:91-106`) → `UseEndpoints(...)` (`Program.cs:33-72`).

`UseInfrastructure`'s order is load-bearing:

```csharp
// …Infrastructure/Extensions.cs:93-104
app.UseErrorHandler()          // must be outermost — §3.19
   .UseSwaggerDocs()
   .UseJaeger()
   .UseConvey()
   .UseAccessTokenValidator()  // Redis deny-list, before signature validation — §3.12
   .UseMongo()                 // fire-and-forget unique index — §3.24
   .UsePublicContracts<ContractAttribute>()
   .UseMetrics()
   .UseAuthentication()
   .UseRabbitMq()
   .SubscribeCommand<SignUp>();
```

`UseMongo()` is the one call that is not middleware — it is an initialisation step in the middle of
a middleware chain, and it returns before its work is done (§3.24).

---

## 5. Persistence & schema evolution

### 5.1 The stores

| Store | Purpose | Registered at | Configured at |
| --- | --- | --- | --- |
| MongoDB `identity-service` | `users`, `refreshTokens`, `inbox`, `outbox` | `…Infrastructure/Extensions.cs:79-86` | `…Api/appsettings.json:100-104` |
| Redis (`identity:` prefix) | revoked access tokens | `…Infrastructure/Extensions.cs:56` (`AddRedis()`) | `…Api/appsettings.json:156-159` |

There is no relational database, no ORM, no cache besides Redis and no file storage.

### 5.2 Collection shapes, as written by the mappers

**`users`** — from `User.AsDocument()` (`…Infrastructure/Mongo/Documents/Extensions.cs:13-22`) and
`UserDocument` (`…/Documents/UserDocument.cs:7-15`):

| Field | CLR type | Source | Notes |
| --- | --- | --- | --- |
| `_id` | `Guid` | `User.Id` (via `IIdentifiable<Guid>.Id`) | `AggregateId`, never empty (§3.2) |
| `Email` | `string` | `User.Email` | **always lower-case** (`User.cs:35`); the only indexed non-`_id` field (§3.24) |
| `Role` | `string` | `User.Role` | lower-case; `user` or `admin` (§3.4) |
| `Password` | `string` | `User.Password` | ASP.NET PBKDF2 composite hash (§3.9) |
| `CreatedAt` | `DateTime` | `User.CreatedAt` | UTC, set by the caller (`IdentityService.cs:102`) |
| `Permissions` | `IEnumerable<string>` | `User.Permissions` | defaults to empty, never `null` |

**`refreshTokens`** — from `RefreshToken.AsDocument()` (`…/Documents/Extensions.cs:37-45`) and
`RefreshTokenDocument` (`…/Documents/RefreshTokenDocument.cs:6-13`):

| Field | CLR type | Notes |
| --- | --- | --- |
| `_id` | `Guid` | fresh `Guid.NewGuid()` per sign-in (`RefreshTokenService.cs:32`) |
| `UserId` | `Guid` | **not indexed** |
| `Token` | `string` | 30-byte RNG, special characters stripped (§3.11); **not indexed**, yet it is the only lookup key (§3.22) |
| `CreatedAt` | `DateTime` | UTC |
| `RevokedAt` | `DateTime?` | `null` until revoked; `Revoked => RevokedAt.HasValue` is a computed property on the entity (`RefreshToken.cs:12`), **not** a stored field |

**`inbox` / `outbox`** — shape owned by Convey `[convey]`, configured at
`…Api/appsettings.json:105-113`. Not mapped by this repository.

### 5.3 What field names actually depend on

Nothing in this repository registers a `BsonClassMap`, an element-name attribute, a convention pack
or `[BsonIgnoreExtraElements]`. Therefore:

- **On-disk element names are the C# property names, verbatim and case-sensitive.** Renaming
  `UserDocument.Email` to `EmailAddress` silently stops matching existing documents — reads return
  `null` for that field rather than failing.
- **The `Guid` representation** is the driver's default (`CSharpLegacy` binary subtype 3 unless
  configured otherwise) `[framework]`. Nothing configures it, so it is consistent within the
  platform but not necessarily interoperable with tooling that assumes subtype 4. This is
  `Unverifiable — Missing Source Evidence` at the repository level and matters only if someone
  queries the collections from outside the service.

### 5.4 Schema evolution — there is no mechanism

This is the most important subsection in §5 and the shortest, because there is nothing to describe.

| Facility | Present? | Evidence |
| --- | --- | --- |
| Migration framework (EF Core, Fluent Migrator, mongock, …) | **no** | no package reference in any `.csproj` in this repository |
| A `migrations/` directory | **no** | absent from the repository tree |
| A schema-version document or collection | **no** | no read or write of any such key |
| A seeder | **no** | `mongo.seed: false` (`…Api/appsettings.json:103`); Convey supports `IMongoDbSeeder` `[convey]` but none is implemented |
| Index management | **one index, fire-and-forget** | `…Infrastructure/Mongo/Extensions.cs:13-28` (§3.24) |
| A `dotnet ef`-style CI step | **no** | `.travis.yml:12-14` runs build and test only |

**This is true of the entire fourteen-repository workspace**, not just identity-service — the
startup unique index in §3.24 is the *only* schema action on the platform (baseline gap **G11**,
`baselines/service-summaries.md`, confirmed by this analysis).

**The practical consequence.** Schema change is therefore a **code-and-data problem handled by
hand**, and it has a specific safe/unsafe boundary in this codebase:

| Change | Safety | Why |
| --- | --- | --- |
| **Add** a nullable/defaulted property to a document | **safe** | absent element deserialises to the CLR default; add it to all mappers (§3.23) |
| **Add** a required property | **unsafe** | existing documents deserialise it as `null`/`default`, then `User`'s constructor validation (`User.cs:19-32`) throws on **every read of every old document** |
| **Rename** a property | **unsafe** | silent data loss on read; requires a backfill script that does not exist |
| **Remove** a property | **conditionally unsafe** | depends on the driver's extra-element behaviour (§3.23); no `[BsonIgnoreExtraElements]` is present to make it safe |
| **Change** a property's type | **unsafe** | deserialisation throws; surfaces as `400 {code:"error"}` (§3.19) |
| **Add** a value to `Role` | **safe** | new value is simply never present in old documents |
| **Remove** a value from `Role` | **unsafe** | `AsEntity` re-validates on read (§3.23), so every affected user becomes unreadable |
| **Add** an index | **safe but unsound** | see §3.24; add it there and fix the awaiting while you are at it |

**The procedure a maintainer must follow, since no tooling exists:**

1. Make the change additive and defaulted, deployed first.
2. Backfill by hand — there is no runner, so this is an ad-hoc `mongosh` script or a throwaway
   console project. Note that `IUserRepository` has **no update method** (§3.21), so a backfill
   cannot go through the application's own code.
3. Only after the backfill completes, tighten validation in `User`'s constructor.
4. There is no rollback path other than restoring a database snapshot; nothing records which
   version of the schema a given database is at.

### 5.5 Retention and growth

| Collection | Growth driver | Pruned? |
| --- | --- | --- |
| `users` | one document per sign-up | never — no delete path exists (§3.21) |
| `refreshTokens` | **one document per sign-in**, forever | **never** — no expiry field, no TTL index, no cleanup job (§3.8, §3.22) |
| `outbox` | one document per published event | by Convey's `expiry: 3600` (`appsettings.json:109`) `[convey]` |
| `inbox` | one document per consumed message | same `expiry` `[convey]` |
| Redis `identity:*` | one key per revoked access token | by TTL — bounded by `jwt.expiryMinutes: 60` (§3.12) |

`refreshTokens` is the one unbounded, un-indexed, ever-scanned collection. Its growth rate is the
sign-in rate, and every `refresh-tokens/use` and `refresh-tokens/revoke` call scans all of it. This
is the service's single clearest scaling cliff, and the fix is two lines: an index on `Token` in
`…Infrastructure/Mongo/Extensions.cs`, and an expiry field on `RefreshToken` with a TTL index.

---

## 6. Surface → internals map

Every externally reachable entry point, traced to the code that implements it. "Gateway" is
`…APIGateway/src/Pacco.APIGateway/ntrada.yml` (and its `.docker` twin; the two async manifests are
**identical for identity**, §3.29).

### 6.1 HTTP surface

| Gateway route | Gateway auth | Service route | Implementation | Concepts | Returns |
| --- | --- | --- | --- | --- | --- |
| — | — | `GET /` | `Program.cs:34` | §3.36 | `200` `AppOptions.Name` |
| `GET /identity/users/{userId}` (`ntrada.yml:243-246`) | `auth: true`, `claims: role=admin` | `GET /users/{userId}` | `Program.cs:35` → `GetUserAsync` `:78-88` | §3.25, §3.26, §3.21, §3.23 | `200` `UserDto` / `404` |
| `GET /identity/me` (`:250-252`) | `auth: true` | `GET /me` | `Program.cs:36-46` | §3.27, §3.21, §3.23 | `200` / `401` / `404` |
| `POST /identity/sign-up` (`:256-261`) | `auth: false`, generates `userId` | `POST /sign-up` | `Program.cs:52-56` → `IdentityService.SignUpAsync:85-107` | §3.14, §3.6, §3.9, §3.16 | `201` + `Location: identity/me` |
| `POST /identity/sign-in` (`:265+`) | `auth: false` | `POST /sign-in` | `Program.cs:47-51` → `IdentityService.SignInAsync:49-83` | §3.13, §3.10, §3.8, §3.11 | `200` `AuthDto` |
| **none** | — | `POST /access-tokens/revoke` | `Program.cs:57-61` | §3.12 | `204` |
| **none** | — | `POST /refresh-tokens/use` | `Program.cs:62-66` → `RefreshTokenService.UseAsync:50-79` | §3.8, §3.22, §3.10 | `200` `AuthDto` |
| **none** | — | `POST /refresh-tokens/revoke` | `Program.cs:67-71` → `RevokeAsync` | §3.8, §3.22 | `204` |
| — | — | `GET /ping` | Convey `[convey]`, not in `Program.cs` | §3.38 | `200` |
| — | — | Swagger UI + JSON | `AddWebApiSwaggerDocs()` `Extensions.cs:87`, `UseSwaggerDocs()` `:95` | §3.26 | — |
| — | — | public-contracts endpoint | `UsePublicContracts<ContractAttribute>()` `Extensions.cs:99` | §3.35 | — |
| — | — | metrics endpoint | `UseMetrics()` `Extensions.cs:100` | §3.38 | — |

**Three of the eight service routes have no gateway route.** They are reachable only from inside
the compose network (or from the host, since `compose/services.yml:42-49` publishes `5004:80`).

### 6.2 AMQP surface

| Direction | Message | Exchange / key | Wiring | Concepts |
| --- | --- | --- | --- | --- |
| **In** | `SignUp` | `identity` / `sign_up`, queue `identity-service/identity.sign_up` | `Extensions.cs:103` `.SubscribeCommand<SignUp>()` → `SignUpHandler.cs:16` | §3.29, §3.30 |
| **Out** | `SignedUp` | `identity` / `signed_up` | `IdentityService.cs:106` → `MessageBroker.cs:76-84` | §3.16, §3.31 |
| **Out** | `SignedIn` | `identity` / `signed_in` | `IdentityService.cs:80` | §3.16 |
| **Out (dead)** | `SignInRejected`, `SignUpRejected` | `identity` / `*_rejected` | only via `ExceptionToMessageMapper` — unreachable, §3.18 | §3.17 |

**Verified consumers across all fourteen clones:**

| Message | Consumer | Evidence |
| --- | --- | --- |
| `SignedUp` | `customers-service` | its `SignedUpHandler` subscribes and creates a customer profile |
| `SignedUp` | `operations-service` | `messages.json:66-69` (event, for UI status) |
| `SignedIn` | **none** — no service subscribes | grep across all clones |
| `SignInRejected` / `SignUpRejected` | `operations-service` | `messages.json:70-73` — but they are never published (§3.17) |
| `SignIn` (command) | **none** — identity does not subscribe it | `Extensions.cs:103` subscribes only `SignUp`; yet `operations-service/messages.json:62-65` declares it |

### 6.3 Store surface

| Store | Touched by | Concepts |
| --- | --- | --- |
| Mongo `users` | `UserRepository.cs:10-34` | §3.21, §3.23, §3.24 |
| Mongo `refreshTokens` | `RefreshTokenRepository.cs:10-29` | §3.22 |
| Mongo `inbox`/`outbox` | Convey, via the decorators and `MessageBroker` | §3.30, §3.31 |
| Redis `identity:*` | `IAccessTokenService` `[convey]` | §3.12 |

### 6.4 The cross-service coupling that is easiest to break

`JwtProvider.cs:21` emits the subject as `userId.ToString("N")` — 32 hex characters, no dashes.
Three separate consumers parse that string back:

1. `AuthenticateUsingJwtAsync` (`…Identity.Infrastructure/Extensions.cs:111`) — `Guid.Parse`.
2. `PaccoHub.InitializeAsync` (`…Operations.Api/Hubs/PaccoHub.cs:34`) —
   `Guid.Parse(payload.Subject).ToUserGroup()` → `users:{N-format}`.
3. `HubWrapper` (`…Operations.Api/Services/HubWrapper.cs:18`) —
   `Clients.Group(userId.ToUserGroup())` on the **`string`** overload
   (`…Operations.Api/Infrastructure/Extensions.cs:33`), which does **not** re-format.

Path 2 and path 3 produce the same group name **only because** path 1's producer already used
`"N"`. Changing `JwtProvider.cs:21` to `userId.ToString()` would leave identity-service working
perfectly and silently break every real-time notification in `operations-service` — a defect that
would appear in a different repository from the change that caused it. See
`component-internals/operations-service.md` §3.11.

---

## 7. Change/extension guide

Ordered by how often the change is likely to be wanted. Each entry lists the files to touch, in
order, and the traps specific to this codebase.

### 7.1 Add a field to the user record

1. `…Core/Entities/User.cs` — constructor parameter + private-set property; add validation in the
   `:19-32` block if it is required.
2. `…Infrastructure/Mongo/Documents/UserDocument.cs` — the property.
3. `…Infrastructure/Mongo/Documents/Extensions.cs` — **all three** user mappers (`:9-11` positional,
   `:13-22` initialiser, `:24-32` initialiser). Only the first is compiler-checked (§3.23).
4. `…Application/DTO/UserDto.cs` — if the field should be visible on `GET /me`.
5. `…Application/Commands/SignUp.cs` — if it is supplied at sign-up; the constructor is the binding
   contract (§3.15).
6. **Trap:** if the field is required, every existing document fails validation on read (§5.4).
   Make it optional, backfill by hand, then tighten.

### 7.2 Add a role

1. `…Core/Entities/Role.cs` — add the constant and extend `IsValid` (`:14-18`).
2. Decide gateway policy: `ntrada.yml` uses `claims: {role: admin}` (`:243-246`); a new role that
   should reach admin routes must be added there in **all four manifests**.
3. **Trap:** roles are stored lower-cased (`User.cs:37`) and compared exactly. **Never remove** an
   existing role — `AsEntity` re-validates on read and would make those users unreadable (§3.23).

### 7.3 Add an HTTP endpoint

1. `…Application/Commands/` (or `Queries/`) — the message type, single constructor, `[Contract]`
   if it is part of the public contract (§3.35).
2. `…Program.cs:33-72` — register the route with a lambda, following the existing shape (§3.26).
3. Choose the status code by hand; there is no convention beyond what `Program.cs` already does.
4. `…APIGateway/src/Pacco.APIGateway/ntrada.yml` **and** `ntrada.docker.yml`, `ntrada-async.yml`,
   `ntrada-async.docker.yml` — otherwise the endpoint is not externally reachable (§3.7). All four
   need the entry; identity uses `use: downstream` in all four.
5. If the endpoint requires authentication, call `ctx.AuthenticateUsingJwtAsync()` inside the
   lambda (§3.27) — there is no attribute-based alternative and the gateway check is not enough
   for a defence-in-depth posture (§8.2/B2).
6. **Trap:** every exception becomes `400` (§3.19). If you need another status, set it explicitly
   in the lambda.

### 7.4 Add a domain exception

1. `…Core/Exceptions/YourException.cs` — derive from `IdentityException`, set `Code` to a unique
   snake_case string.
2. `…Infrastructure/Exceptions/ExceptionToResponseMapper.cs:15-45` — add an arm, or accept the
   default `400 {code:"error"}`.
3. `…Infrastructure/Exceptions/ExceptionToMessageMapper.cs:11-24` — add an arm **and check the
   existing type patterns**; `:20` matches `SignUpRejected` where it should match `SignUp`
   (§3.18).
4. **Trap:** do not add the class to an unrelated file. `EmptyRefreshTokenException` lives inside
   `InvalidRoleException.cs:12-19` and is effectively unfindable (§3.20).

### 7.5 Publish a new event

1. `…Application/Events/YourEvent.cs` — `IEvent`, `[Contract]`, single constructor.
2. Publish through `IMessageBroker.PublishAsync` — never `IBusPublisher` (§3.31).
3. `…Operations.Api/messages.json`, `identity` block (`:60-74`) — add it so the UI can display it.
4. The exchange and routing key need no configuration: `rabbitMq.conventionsCasing: snakeCase`
   (`appsettings.json:146`) derives `signed_up` from `SignedUp` `[convey]`.
5. **Trap:** publishing is fire-and-forget from the caller's perspective. If the outbox is enabled
   the event is delayed up to 2 s (`intervalMilliseconds: 2000`) and lost if the outbox cannot
   drain within `expiry: 3600` (§3.30).

### 7.6 Consume a message from another service

1. `…Application/Events/External/` — mirror the producing service's contract type exactly (name,
   namespace-independent, property names).
2. `…Application/Events/External/Handlers/` — an `IEventHandler<T>`.
3. `…Infrastructure/Extensions.cs:103` — chain `.SubscribeEvent<T>()`.
4. The handler **will** be wrapped by `OutboxEventHandlerDecorator` (`:68`), so redelivery is
   idempotent, provided the producer sets a stable `MessageId` (§3.30).
5. **Trap:** identity currently consumes nothing. There is no existing external-event directory to
   copy from in this repository — use `customers-service` as the pattern.

### 7.7 Change token behaviour

- **Access-token lifetime:** `jwt.expiryMinutes` (`appsettings.json:41`). Note this also bounds
  the Redis revocation TTL (§3.12).
- **Refresh-token rotation:** modify `RefreshTokenService.UseAsync:76` to mint and persist a new
  token and revoke the old one; add `UpdateAsync` calls. This is the highest-value security change
  in the service (§8.2/B4).
- **Refresh-token expiry:** add a field to `RefreshToken` (`…Core/Entities/RefreshToken.cs`), all
  mappers (§3.23), a check in `UseAsync`, and a TTL index (§5.5). Note the entity has no expiry
  concept at all today.
- **Trap:** `Rng.Generate`'s default differs between interface and implementation (§3.11); always
  pass the flag explicitly.

### 7.8 Make sign-up asynchronous

See §3.29's extension procedure — four steps, and **fix `ExceptionToMessageMapper.cs:20` first**,
or invalid sign-ups become silent hangs.

### 7.9 Things you cannot do without new code

| Wanted | Why it is not possible today |
| --- | --- |
| Change a password | `IUserRepository` has no update method (§3.21) |
| Reset a password | ditto; also no email transport anywhere on the platform |
| Change an email or role | ditto |
| Delete or deactivate an account | ditto; no delete method, no `IsActive` flag |
| Revoke all of a user's sessions | no query by `UserId` on `refreshTokens` (§3.22) |
| List users | no such repository method or endpoint |
| Enforce role checks in-service | `IdentityContext.IsAdmin` exists but is never consumed (§3.32) |
| Rate-limit sign-in | no such middleware in the chain (§4.9) |
| Lock an account after failed attempts | nothing records failed attempts |

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

Statements this document relies on that are **not** provable from `hianshul100_Pacco.Services.Identity`
alone. Each is marked with what would confirm it.

| ID | Assumption | Basis | How to confirm |
| --- | --- | --- | --- |
| **A1** | Convey `0.4.*` behaves as described where marked `[convey]` — dispatcher registration, `IMongoRepository`, `IJwtHandler`, `IAccessTokenService`, outbox `HandleAsync`, `SubscribeCommand` queue binding, `UseAccessTokenValidator`. | Package references in the four `.csproj` files; consistent usage across all fourteen clones. | Read the Convey source at the pinned version. The package is not vendored here. |
| **A2** | `mongo.seed: false` plus the absence of any `IMongoDbSeeder` means no data is written at startup. | `…Api/appsettings.json:103`; grep finds no seeder. | Confirmed within this repository; Convey's default seeder behaviour is `[convey]`. |
| **A3** | The MongoDB driver's default POCO mapping produces element names identical to CLR property names. | No `BsonClassMap`, attribute or convention pack anywhere in the repository. | Inspect a document in a running `identity-service` database. |
| **A4** | `dotnet test` with no test project exits 0, so `.travis.yml` is green regardless of code quality. | `.travis.yml:12-14`; `scripts/test.sh`; no test project in the `.sln`. | Run `./scripts/test.sh` locally. |
| **A5** | `SignedUp` is consumed by `customers-service`, creating the customer profile that the rest of the platform depends on. | Cross-repo read of `customers-service`'s subscription and handler. | Already verified by direct source read; carried over from `baselines/service-summaries.md` **A5**. |
| **A6** | No migration tooling exists anywhere on the platform; the §3.24 index is the only schema action. | Absence across all fourteen clones (no migration packages, no `migrations/`, no version collection). | Confirmed by this analysis; resolves baseline gap **G11**. |
| **A7** | The gateway is the only enforcement point for `role: admin` on `GET /users/{userId}`. | `ntrada.yml:243-246`; no role check in `Program.cs` or `IdentityService`. | Confirmed within the repositories read; see **B2** for the consequence. |

### 8.2 Blockers

Findings that would block a production deployment. Each is code-grounded; none is speculative.

| ID | Blocker | Evidence | Impact |
| --- | --- | --- | --- |
| **B1** | **Secrets committed to source control.** The JWT symmetric `issuerSigningKey` is in `appsettings.json:36` and is byte-identical across every Pacco service; a Vault token literal `"secret"` sits at `:174`; and `src/Pacco.Services.Identity.Api/certs/` contains `localhost.pfx`, `.key`, `.pem`, `.cer`, published into the image by `…Api.csproj:20-22`. | Direct file read. | Any party with repository access can mint tokens that every service accepts, since `validateIssuer` and `validateAudience` are both `false` (`appsettings.json:38-39`). Confirms baseline **G13/B1**. |
| **B2** | **`GET /users/{userId}` has no in-service authorization.** `Program.cs:35` calls `GetUserAsync` directly, with no `AuthenticateUsingJwtAsync` and no role check. The only `role: admin` constraint is in the gateway (`ntrada.yml:243-246`). | §3.27, §4.4. | Anything that can reach `identity-service:80` reads any user record by id. `compose/services.yml:42-49` publishes `5004:80` to the host. |
| **B3** | **Email addresses are written to logs.** `IdentityService.cs:62` (`$"User with email: {email} was not found."`), `:96` (`$"Email already in use: {email}"`). `logger.excludeProperties` (`appsettings.json`) filters `api_key`, `access_key` and `Authorization` — it cannot filter an interpolated message body. | Direct file read. | PII in Seq/console/file sinks; also an account-enumeration oracle for anyone with log access. |
| **B4** | **Refresh tokens never expire and are never rotated.** `RefreshToken` has no expiry field (`…Core/Entities/RefreshToken.cs:6-42`), `UseAsync` returns the same token (`RefreshTokenService.cs:76`), and no gateway route exists for `refresh-tokens/revoke` (§3.7). | §3.8, §4.5. | A leaked refresh token is a permanent credential with no externally reachable revocation path. |
| **B5** | **The unique email index may never be created, silently.** `…Infrastructure/Mongo/Extensions.cs:13-28` is a fire-and-forget `Task.Run` whose exception is never observed, inside a scope that is disposed before it completes. | §3.24, verbatim code block. | The only structural uniqueness guarantee can be absent while the service reports healthy; combined with the TOCTOU window in `SignUpAsync:93-103`, duplicate accounts become possible and sign-in becomes non-deterministic. |
| **B6** | **The AMQP sign-up rejection path is broken by a one-token type error.** `ExceptionToMessageMapper.cs:20` matches `SignUpRejected command` inside the `SignUp` arm; it should match `SignUp command`. | §3.18, verbatim code block. | If sign-up is ever switched to `use: rabbitmq` (§7.8), every invalid-email sign-up is silently dropped and the caller polls `operations-service` forever. Currently latent only because no producer exists. |
| **B7** | **No tests exist, and CI reports success anyway.** No test project in the `.sln`; `.travis.yml:12-14` runs `dotnet test` against nothing. | §3.40. | Every invariant in §3 is unpinned. B5, B6 and the §3.13/§3.11 defects all survived because nothing was checking. |

### 8.3 Open questions

| ID | Question | Why it matters | Where the answer lives |
| --- | --- | --- | --- |
| **Q1** | Does `UseAccessTokenValidator()` reject *anonymous* requests on non-allow-listed paths, or only *deactivated* tokens? | Determines whether **B2** is "reachable by anyone on the network" or "reachable by any authenticated user". Two inferential arguments in §3.28 point to deny-list-only, but neither is conclusive. | Convey `0.4.*` source, or an empirical `curl` against a running instance. |
| **Q2** | What does Convey's `.Get<TQuery>(path)` dispatch overload return when the handler yields `null` — `404` or `200 null`? | Decides whether §3.25's cleanup ("adopt the dead handler") preserves the hand-written `404` at `Program.cs:83`. | Convey source. |
| **Q3** | Why does the identity module use `use: downstream` in **both** async gateway manifests when every other module switches to `use: rabbitmq`? | Determines whether the AMQP `SignUp` subscription (§3.29) is unfinished work or a deliberate exception for the credential path. | Product/architecture decision; not recorded in any repository read. |
| **Q4** | Is `SignedIn` intended to have a consumer? | It is published on every sign-in, is marked `[Contract]`, and is declared in `operations-service/messages.json:66-73` — but nothing subscribes it (§3.16, §6.2). | Product intent. |
| **Q5** | Is the `Rng` default-parameter divergence (`IRng.cs:5` → `false`, `Rng.cs:12` → `true`, §3.11) deliberate — i.e. is special-character stripping *wanted* — or an accident? | Stripping `/ \ = + ? : &` reduces the effective alphabet of a 30-byte token. The single call site passes `true` explicitly (`RefreshTokenService.cs:32`), so behaviour is currently well-defined; the divergence is a trap for the next caller. | Author intent; no comment or test records it. |
| **Q6** | What is the intended lifecycle of the `IAppContext`/`IIdentityContext` abstraction, which is fully built and entirely unconsumed (§3.32)? | It is the natural seam for in-service authorization (**B2**) and for the "only an admin may create an admin" rule (§3.4). | Product intent. |
| **Q7** | Should `refreshTokens` documents be reused per user rather than appended per sign-in? | Decides whether the fix for §5.5's unbounded growth is a TTL index or a change to `CreateAsync`'s semantics. | Product intent. |

### 8.4 Baseline reconciliation

Corrections and confirmations this document makes to
`docs/architecture-inventory/baselines/service-summaries.md`:

| Baseline item | Status after this analysis |
| --- | --- |
| **G11 / A6** — "no migration tooling identified" | **Confirmed and closed.** The `…Infrastructure/Mongo/Extensions.cs:13-28` unique index is the only schema action on the platform (§3.24, §5.4). |
| **G13 / B1** — "committed Vault keys and a shared JWT signing key" | **Confirmed and expanded.** Also includes committed private key material under `…Api/certs/` published into the image by `…Api.csproj:20-22` (§3.37, **B1**). |
| **A5** — "`SignedUp` drives customer-profile creation" | **Confirmed** by direct cross-repo read (§6.2). |
| **D1** — identity's dependency direction | **Confirmed**: identity depends on nothing; three services depend on its tokens (§3.39 — no `depends_on` in `compose/services.yml:42-49`). |
| **Q1** (baseline) — access-token validator semantics | **Carried forward as Q1 above**; still unresolved, and it is the highest-value verification remaining (§3.28). |

---

*End of `identity-service` component-internals model.*
