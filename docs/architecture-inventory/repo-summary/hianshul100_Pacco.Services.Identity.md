# Repository summary — `hianshul100_Pacco.Services.Identity`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco.Services.Identity` (also known as: Pacco.Services.Identity, the Identity service)
**Deployable:** `Pacco.Services.Identity.Api` (also known as: `identity-service` — its Consul service name, container name and gateway service key — and `devmentors/pacco.services.identity`, its published image). Repository: `hianshul100_Pacco.Services.Identity`, path: `src/Pacco.Services.Identity.Api`.
**Source of truth for this document:** the files on disk in this repository.

---

## 1. Primary purpose

The platform's authentication authority. It owns user accounts and passwords, issues the JWT access tokens every other service trusts, and manages refresh tokens. It is also the origin of the `signed_up` event that creates a customer profile downstream.

## 2. Main runtime / service type

ASP.NET Core web application on `netcoreapp3.1`, Convey-based. **It is the only service that registers its HTTP routes with plain `UseEndpoints` rather than Convey's `UseDispatcherEndpoints`**, because sign-in has to return a token body rather than dispatch-and-forget. It also consumes one RabbitMQ command in the same process.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entry | `src/Pacco.Services.Identity.Api/Program.cs` |
| Local run | `scripts/start.sh` → `ASPNETCORE_ENVIRONMENT=local`, `cd src/Pacco.Services.Identity.Api`, `dotnet run` |
| Container entry | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Identity.Api.dll` |
| Message subscriptions | `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` → `UseInfrastructure()` |
| Request collection | `Pacco.Services.Identity.rest` |

HTTP routes from `Program.cs`:

| Method | Route | Behaviour |
|---|---|---|
| GET | `users/{userId}` | returns the user |
| GET | `me` | `ctx.AuthenticateUsingJwtAsync()`, 401 if the result is empty |
| POST | `sign-in` | returns the token as JSON |
| POST | `sign-up` | responds Created at `identity/me` |
| POST | `access-tokens/revoke` | responds 204 |
| POST | `refresh-tokens/use` | returns a new token |
| POST | `refresh-tokens/revoke` | responds 204 |

## 4. Important modules / packages

Four projects: `src/Pacco.Services.Identity.Api`, `.Application`, `.Core`, `.Infrastructure`.

- **Core** — `Entities/User.cs`, `Entities/RefreshToken.cs`, `Entities/Role.cs`, `Entities/AggregateRoot.cs`, `Entities/AggregateId.cs`, `Entities/IDomainEvent.cs`.
- **Application** — commands `SignUp`, `SignIn`, `RevokeAccessToken`, `RevokeRefreshToken`, `UseRefreshToken`; only `SignUpHandler` exists as a command handler, the rest are driven directly by `IdentityService` and `RefreshTokenService`; events `SignedUp`, `SignedIn`; rejected events `SignUpRejected`, `SignInRejected`; query `GetUser`; `Services/Identity/IdentityService.cs` holds the sign-in and sign-up logic including a compiled email-validation regular expression.
- **Infrastructure** — `Auth/JwtProvider.cs`, `Auth/PasswordService.cs`, `Auth/Rng.cs`; `Mongo/Documents/UserDocument.cs`, `Mongo/Documents/RefreshTokenDocument.cs`, repositories and the `GetUser` handler; outbox decorators; exception mappers; logging template mapper.

Packages are the platform's standard Convey `0.4.*` set plus `Convey.Auth`, which supplies JWT issuance and the access-token revocation cache.

## 5. External integrations

MongoDB (database `identity-service`), Redis (instance prefix `identity:`), RabbitMQ (exchange `identity`), Consul (service `identity-service`, port 5004, ping endpoint `ping`), Fabio (`http://localhost:9999`), Vault (kv v2 mount `kv`, path `identity-service/settings`; PKI common name `identity.pacco.io`; dynamic MongoDB credentials with auto-renewal), Jaeger (service name `identity`, UDP `localhost:6831`, `sampler: const`), Prometheus, Seq.

`httpClient.services` is **empty** — no outbound service-to-service HTTP.

## 6. Data stores and state handling

**Store:** MongoDB, database `identity-service`. **Query mechanism:** Convey MongoDB repositories over the MongoDB .NET driver. **No ORM. No migration tool.**

| Collection | Fields |
|---|---|
| `users` | `Id` (Guid), `Email` (string), `Role` (string), `Password` (string, hashed by `PasswordService`), `CreatedAt` (DateTime), `Permissions` (string collection) |
| `refreshTokens` | `Id` (Guid), `UserId` (Guid), `Token` (string), `CreatedAt` (DateTime), `RevokedAt` (nullable DateTime) |
| `inbox` | Convey inbox, de-duplication |
| `outbox` | Convey outbox, pending publishes |

Outbox: `inboxCollection: inbox`, `outboxCollection: outbox`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`.

**Cross-domain coupling.** `users.Id` is the identifier every other service knows as the customer identifier — the gateway binds `customerId:@user_id` from the JWT subject on customer, order, parcel and reservation routes. This service is therefore the root of identity for the whole platform, and its `Guid` propagates into `customers`, `orders` and `parcels` documents without any referential integrity between the databases.

`refreshTokens` is append-only in practice: revocation sets `RevokedAt` rather than deleting, and nothing removes expired rows, so the collection grows without bound.

Redis is registered and, unlike the other services, has a concrete use here: `Convey.Auth` keeps revoked access tokens in a distributed cache so revocation takes effect before the token's natural expiry. The exact key scheme is internal to Convey and is **unknown — requires runtime capture**.

## 7. Messaging, async and event mechanisms

**System:** RabbitMQ topic exchanges via `Convey.MessageBrokers.RabbitMQ`, with the Mongo-backed transactional outbox and the Jaeger RabbitMQ plugin.

**Broker settings:** exchange `identity`, `conventionsCasing: snakeCase`, queue name template `identity-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`, credentials `guest`/`guest`.

**Published on exchange `identity`:**

- Events: `signed_up`, `signed_in`
- Rejected events: `sign_up_rejected`, `sign_in_rejected`

Observable payload: `SignedUp { Guid UserId, string Email, string Role }`. This is the widest-reaching event in the platform's start-of-life flow — `customers-service` turns it into a customer record.

**Consumed:**

| Kind | Message | Origin exchange |
|---|---|---|
| Command | `SignUp` | `identity` |

`SignUp` is the only subscription. Sign-in is deliberately synchronous, because the caller needs the token back.

## 8. APIs exposed and consumed

**Exposed:** the seven HTTP routes in section 3, Swagger at route prefix `docs`, and a `ping` endpoint for Consul. Local base URL `http://localhost:5004`; port 80 in the container.

The gateway exposes only four of the seven: `GET /identity/users/{userId}` (role `admin`), `GET /identity/me`, `POST /identity/sign-up` (`auth: false`), `POST /identity/sign-in` (`auth: false`). **`access-tokens/revoke`, `refresh-tokens/use` and `refresh-tokens/revoke` are not routed through the gateway at all**, so no external client can refresh or revoke a token.

Both gateway profiles route sign-up and sign-in as `downstream` HTTP, even in the async profile — these are the only write operations in the platform that stay synchronous end to end.

**Consumed:** no outbound HTTP calls.

## 9. Deployment and runtime clues

- `Dockerfile`: `sdk:3.1` build → `aspnet:3.1` runtime; publishes `src/Pacco.Services.Identity.Api`; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`.
- Composed as `identity-service` on host port 5004.
- Configuration: `appsettings.json`, `appsettings.development.json`, `appsettings.docker.json`, `appsettings.local.json`.
- CI: `.travis.yml` — dotnet 3.1.100, branches master and develop, `./scripts/build.sh` then `./scripts/test.sh`, `./scripts/dockerize.sh` on success, pushing `$DOCKER_USERNAME/pacco.services.identity`.

## 10. Security and auth clues

This is the security-critical service in the platform, and the findings below are the most serious in this inventory.

- **Hard-coded signing key.** `appsettings.json` carries `jwt.issuerSigningKey` as a symmetric key written in plain text. The identical value appears in `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml`, and in the Operations service settings. Anyone with read access to any of those repositories can mint a valid token for any user identifier and any role, including `admin`.
- JWT settings: `expiryMinutes: 60`, `issuer: pacco`, `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`.
- **Committed private key material.** This service uses `certs/localhost.pfx` with `password: "test"` in configuration — a private key file and its password, both committed. Every other service references the public `certs/localhost.cer`.
- Password hashing goes through `Infrastructure/Auth/PasswordService.cs`; `Infrastructure/Auth/Rng.cs` supplies randomness for refresh tokens.
- `Application/Services/Identity/IdentityService.cs` logs the submitted email address on failed sign-in (`_logger.LogError($"User with email: {command.Email} was not found.")`) and on invalid-email errors. That writes user-supplied personal data into Seq on every failed attempt, and it also distinguishes "unknown email" from "wrong password" in the logs.
- `logger.excludeProperties` keeps secret-like property names out of logs; `httpClient.requestMasking` is configured.
- Vault supplies settings and dynamic MongoDB credentials, yet the JWT key — the single most sensitive value in the platform — is not among them.

## 11. Observability, logging and tracing clues

Jaeger under service name `identity`; Prometheus metrics via `Convey.Metrics.AppMetrics` (`metrics.database: pacco`, interval 5); structured logs to console, file and Seq with `/`, `/ping` and `/metrics` excluded from request logging.

## 12. Files holding major architecture decisions; feature flags

`src/Pacco.Services.Identity.Api/appsettings.json` (the JWT policy — issuer, lifetime, signing key, anonymous endpoints), `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` (technology choices and the single subscription), `src/Pacco.Services.Identity.Api/Program.cs` (the endpoint style, which differs from every other service), `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs` and `PasswordService.cs` (token and credential handling), `src/Pacco.Services.Identity.Core/Entities/Role.cs` (the role vocabulary the whole platform authorises against).

**Feature flag system: none.** No flag library, no flag store, no flag keys. The only switches are `ASPNETCORE_ENVIRONMENT` and the `enabled` booleans on Convey integrations — integration toggles, not product flags.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** repository root, `src/Pacco.Services.Identity.Api/`, `src/Pacco.Services.Identity.Application/`, `src/Pacco.Services.Identity.Core/`, `src/Pacco.Services.Identity.Infrastructure/`, `scripts/`. No `package.json`, no `wwwroot`, no template file, no bundler configuration, no static asset directory. The only browser-facing surface is the generated Swagger UI at `/docs`.

---

## README vs repository

**Claimed in the README and confirmed on disk:** part of the Pacco solution; runs via `dotnet run` or `./scripts/start.sh`; available on `http://localhost:5004`; buildable from the local `Dockerfile` or pullable as `devmentors/pacco.services.identity`; `Pacco.Services.Identity.rest` lists the HTTP requests.

**Present on disk but absent from the README:** the entire authentication model — token lifetime, refresh tokens, revocation, roles and permissions; the `signed_up` event that bootstraps a customer; the Vault integration; the transactional outbox. The README for the platform's security keystone describes only how to start it.

**Conflicts to surface:**

- **Stale doc.** The README instructs running `dotnet run` in `/src/Pacco.Services.Identity`. That directory does not exist — the host project is `src/Pacco.Services.Identity.Api`. `scripts/start.sh` has the correct path.
- **Catalogue conflict.** `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` lists `sign_in` and `sign_up` as commands for `identity-service`, but `Infrastructure/Extensions.cs` subscribes only to `SignUp`. A `sign_in` command published to the `identity` exchange would be delivered to no one and the caller would wait forever. The catalogue promises an async sign-in path that does not exist.
- **Reachability conflict.** Three of the seven HTTP routes — access-token revocation and both refresh-token routes — have no gateway route. The token lifecycle the service implements cannot be exercised by any external client.
- **CI conflict.** `.travis.yml` runs `./scripts/test.sh`, which invokes `dotnet test`, but this repository contains **no test project at all**. The platform's authentication service has no automated tests.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next. The first three blockers concern credentials that are readable by anyone with repository access.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The user identifier issued here is the customer identifier used across the platform. | The gateway binds `customerId:@user_id` from the token subject, and the `SignedUp` handler in Customers creates the record from it. |
| A2 | The committed certificate and signing key are development values, not production ones. | They are named `localhost` and the password is the literal string `test`. |
| A3 | Sign-in is intended to stay synchronous even under the asynchronous gateway profile. | Both gateway profiles route it as `downstream`, and the caller needs the token in the response. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** The JWT signing key is stored in plain text in this service's settings and repeated in the gateway and Operations settings. | Anyone who can read any of those files can forge an administrator token for any user. Every authorisation decision in the platform rests on this one value. | Platform security owner: rotate the key and move it into Vault, which the platform already runs. |
| B2 | **[ACTION NOW]** A private key file (`certs/localhost.pfx`) and its password are committed to the repository. | The private key must be treated as compromised. | Platform security owner: remove the key from the repository and issue a new one. |
| B3 | **[ACTION NOW]** Failed sign-in attempts log the submitted email address at error level. | Personal data is written to the log store on every failed attempt, and the log distinguishes unknown accounts from wrong passwords, which helps an attacker enumerate users. | Identity service maintainer. |
| B4 | **[ACTION NOW]** The platform's authentication service has no automated tests, while CI reports a passing test step. | Any regression in token issuance or password handling ships unnoticed. | Identity service maintainer. |
| B5 | **[ACTION NOW]** Token revocation and refresh are implemented but unreachable through the gateway. | A stolen token cannot be revoked by any client, and sessions cannot be extended without re-entering credentials every hour. | Gateway maintainer with the Identity service maintainer. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[ACTION NOW]** Is an asynchronous `sign_in` command meant to exist? | The message catalogue advertises it and nothing consumes it. | Identity service maintainer. |
| Q2 | **[handled later by the security-architecture stage]** What algorithm and work factor does `PasswordService` use? | Determines whether stored password hashes are adequate. | Security-architecture stage. |
| Q3 | **[handled later by the domain-model stage]** What are the valid values of `Role` and what does the `Permissions` list on a user do? | Roles drive gateway authorisation; permissions appear on the document but no consumer of them was found. | Domain-model stage. |
| Q4 | **[handled later by the data-architecture stage]** Is anything cleaning up expired or revoked refresh tokens? | The collection grows without bound. | Data-architecture stage. |
