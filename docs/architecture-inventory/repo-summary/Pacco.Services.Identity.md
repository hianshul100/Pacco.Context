---
title: "Repository Summary — Pacco.Services.Identity"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco.Services.Identity"
status: "evidence-based inventory"
---

# Pacco.Services.Identity

**Primary name:** `Pacco.Services.Identity` (aliases used in this file: `identity-service` — the value of `app.service`, the Consul registration name, the MongoDB database name and the Compose service name; `identity` — the RabbitMQ exchange, the Jaeger `serviceName` and the gateway module).
Repository: `Pacco.Services.Identity`, path: `hianshul100_Pacco.Services.Identity/`

---

## 1. Primary purpose

Issues and revokes the platform's access tokens. It owns users, passwords, roles and refresh tokens, and it is the origin of the JWT that every other service validates.

Evidence: `src/Pacco.Services.Identity.Core/Entities/User.cs`, `RefreshToken.cs`, `Role.cs`; `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs`.

## 2. Runtime / service type

ASP.NET Core `netcoreapp3.1` HTTP service. **Unlike its siblings it uses `UseEndpoints` with hand-written route delegates rather than only the Convey dispatcher style**, because sign-in must return a token body directly. It also publishes to RabbitMQ. Listens on `5004`.

Evidence: `src/Pacco.Services.Identity.Api/Program.cs`.

## 3. Entrypoints

| Entrypoint | Path |
|---|---|
| `Program.cs` | `src/Pacco.Services.Identity.Api/Program.cs` |
| Container entrypoint | `Dockerfile` |
| `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` | `scripts/` |

## 4. Modules / packages

Four source projects: `Pacco.Services.Identity.Api`, `.Application`, `.Core`, `.Infrastructure`. **No test project exists in this repository.**

- **Core:** `Entities/User.cs`, `Entities/RefreshToken.cs`, `Entities/Role.cs`, `Entities/AggregateId.cs`, `Entities/AggregateRoot.cs`, `Entities/IDomainEvent.cs`; `Repositories/IUserRepository.cs`, `Repositories/IRefreshTokenRepository.cs`; exceptions `EmailInUseException`, `InvalidCredentialsException`, `InvalidRefreshTokenException`, `RevokedRefreshTokenException`, `InvalidRoleException`.
- **Application:** commands `SignIn`, `SignUp`, `RevokeAccessToken`, `RevokeRefreshToken`, `UseRefreshToken`; events `SignedIn`, `SignedUp`; rejected events `SignInRejected`, `SignUpRejected`; DTOs `AuthDto`, `UserDto`; service contracts `IIdentityService`, `IJwtProvider`, `IPasswordService`, `IRefreshTokenService`, `IRng`, with `Identity/IdentityService.cs` and `Identity/RefreshTokenService.cs`.
- **Infrastructure:** `Auth/JwtProvider.cs`, `Auth/PasswordService.cs`, `Auth/Rng.cs`; `Mongo/Documents/UserDocument.cs`, `Mongo/Documents/RefreshTokenDocument.cs`, `Mongo/Repositories/UserRepository.cs`, `Mongo/Repositories/RefreshTokenRepository.cs`, `Mongo/Queries/Handlers/GetUserHandler.cs`.

Convey `0.4.*` packages as used across the platform.

## 5. External integrations

MongoDB, RabbitMQ, Redis, Consul, Fabio, Vault, Jaeger, Prometheus. No outbound HTTP service clients are defined.

## 6. Data stores / state

- **Store:** MongoDB, database `identity-service`.
- **Access mechanism:** no ORM. The MongoDB .NET driver behind `Convey.Persistence.MongoDB`, with explicit document classes (`UserDocument`, `RefreshTokenDocument`) and hand-written repositories.
- **Collections:** a users collection and a refresh-tokens collection derived from those document types, plus `inbox` and `outbox`.
- **Migration tool:** none anywhere in the repository.
- **Cross-domain coupling:** the user identifier produced here becomes the customer identifier used by every other service. There is no foreign key; the identifier is propagated by the `signed_up` event and by the `@user_id` binding in the gateway routes.
- **Cache:** Redis. Revoked access tokens are held in the cache by `Convey.Auth`, so token revocation is a cache-backed deny list rather than a database record.

## 7. Messaging / async / events

**System:** RabbitMQ, topic exchange `identity`, snake-case naming, queue template `identity-service/{{exchange}}.{{message}}`, message context header `message_context`, span context header `span_context`. Transactional outbox and inbox on MongoDB (`inbox`, `outbox`).

**Commands consumed:** `sign_in`, `sign_up` — the asynchronous gateway configuration publishes these to the `identity` exchange.

**Events published:**

| Event name on the wire | Class | Payload key fields |
|---|---|---|
| `signed_up` | `Application/Events/SignedUp.cs` | user identifier, email, role |
| `signed_in` | `Application/Events/SignedIn.cs` | user identifier |

**Rejected events published:** `sign_in_rejected`, `sign_up_rejected`.

**External events consumed:** none.

`signed_up` is the trigger that causes `Pacco.Services.Customers` to create a customer record. Wire names are confirmed against `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`. Exact serialised payload shapes are **unknown — requires runtime capture**.

## 8. APIs exposed / consumed

Exposed (`Program.cs`, hand-written endpoint delegates):

| Method | Path | Behaviour |
|---|---|---|
| `GET` | `users/{userId}` | returns a user; the gateway restricts this to the `admin` role |
| `GET` | `me` | calls `ctx.AuthenticateUsingJwtAsync()` and returns `401` when the resolved identifier is `Guid.Empty` |
| `POST` | `sign-in` | returns the token as a JSON body |
| `POST` | `sign-up` | responds `Created` at `identity/me` |
| `POST` | `access-tokens/revoke` | responds `204` |
| `POST` | `refresh-tokens/use` | exchanges a refresh token for a new access token |
| `POST` | `refresh-tokens/revoke` | responds `204` |

Consumed by: `Pacco.APIGateway` (module `identity`; `POST /sign-up` and `POST /sign-in` are the platform's only two `auth: false` write routes).

This service consumes no other service's HTTP API.

## 9. Deployment / runtime clues

Container image `devmentors/pacco.services.identity`, published `5004:80` in `hianshul100_Pacco/compose/services.yml`, `restart: unless-stopped`, network `pacco`. Consul registration on port `5004`.

CI: `.travis.yml` runs `./scripts/build.sh`, `./scripts/test.sh`, then `./scripts/dockerize.sh` on success.

## 10. Security / auth clues

This service's `jwt` configuration differs from every other service, because it is the token **issuer** rather than a validator:

- `certificate.location: "certs/localhost.pfx"`
- `certificate.password: "test"`
- `expiryMinutes: 60`
- `issuer: "pacco"`
- `validateIssuer: false`
- `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`

Password hashing uses `Auth/PasswordService.cs`; random values come from `Auth/Rng.cs`. Refresh tokens can be revoked and are checked for revocation (`RevokedRefreshTokenException`). Vault KV path `identity-service/settings`.

**Security findings** (reported, not remediated in this stage):

1. Private key material is committed to the repository: `certs/localhost.key`, `certs/localhost.pem`, `certs/localhost.pfx`, alongside `certs/localhost.cer`.
2. The password protecting the signing certificate is committed in clear text in `appsettings.json`.
3. `validateIssuer: false` here, while the gateway and every other service set `validateIssuer: true` with `validIssuer: pacco`.
4. Separately, `Pacco.APIGateway` validates tokens using a symmetric key committed in its routing YAML, not the certificate configured here. The two token schemes are configured differently. **Needs validation.**

## 11. Observability / logging / tracing

Jaeger tracing with `serviceName: identity`, including RabbitMQ span propagation; structured logging via `Convey.Logging` and `Convey.Logging.CQRS` with a message-to-log-template mapper; Prometheus metrics via `Convey.Metrics.AppMetrics`.

## 12. Files carrying major architecture decisions; feature flags

- `src/Pacco.Services.Identity.Infrastructure/Auth/JwtProvider.cs` — how the platform's tokens are minted.
- `src/Pacco.Services.Identity.Application/Services/Identity/RefreshTokenService.cs` — the refresh and revocation model.
- `src/Pacco.Services.Identity.Api/Program.cs` — the decision to use hand-written endpoints instead of the platform's dispatcher style.
- `src/Pacco.Services.Identity.Api/appsettings.json` — the token settings that the rest of the platform depends on.

**Feature-flag system: none.** No flag provider package is referenced. The only switches are per-integration `enabled` booleans plus `validateIssuer` and `allowAnonymousEndpoints`, which are deployment configuration rather than runtime feature flags. There are no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. `src/` contains only the four C# projects, plus a `certs/` directory holding certificate files. There is no `package.json`, no bundler configuration, no HTML and no view templates.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes an identity service on .NET Core 3.1 built with Convey, following clean architecture | Confirmed: four layered projects, Convey `0.4.*`, `netcoreapp3.1` | Confirmed |
| The platform uses a single JWT scheme issued here and validated everywhere | This service signs with a certificate; the gateway validates with a committed symmetric key | Needs validation — the token settings on the two sides do not describe the same scheme |
| Every service is built the same way with the Convey dispatcher endpoints | This service uses hand-written endpoint delegates instead | Stale doc |
| The build script chain includes `./scripts/test.sh` | There is no test project in this repository, so the step has nothing to execute | Needs validation |

**Docs-only claims:** none identified.
**Disk-only components:** the refresh-token lifecycle (`refresh-tokens/use`, `refresh-tokens/revoke`, `access-tokens/revoke`) — present in code, not exposed through the gateway and not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The certificates committed here are development-only samples, not production material. | They are self-signed local host certificates and the password protecting them is the word "test". |
| A2 | The user identifier created at sign-up is the same identifier used as the customer identifier everywhere else. | The gateway binds the customer identifier from the signed-in user, and the customers service creates its record from the sign-up event. |
| A3 | The refresh-token endpoints are reachable only inside the platform network. | The public gateway does not expose any route for them. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | Private key files and the password that protects them are stored in the repository, so anyone with repository access can sign tokens. | **[ACTION NOW]** Report to the security owner of the platform; this stage records the location but does not change any file. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Which token scheme is really in use across the platform: certificate signing configured here, or the shared secret key configured in the gateway? | **[ACTION NOW]** Confirm with the requesting team; the answer decides how authentication is described for every service. |
| Q2 | Are the refresh-token and revocation endpoints meant to be usable by clients? They are not exposed through the public gateway. | **[handled later by the ADR authoring stage]** Record whether the token lifecycle is complete or partly unfinished. |
| Q3 | How are roles assigned? A role type exists, and the gateway checks for an administrator role, but nothing in this workspace shows how someone becomes an administrator. | **[handled later by the ADR authoring stage]** Record the role assignment path. |
