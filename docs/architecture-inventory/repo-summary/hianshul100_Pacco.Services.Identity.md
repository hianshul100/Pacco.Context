# Repository Summary — `hianshul100_Pacco.Services.Identity`

**Primary name:** `identity-service` (aliases used in this file: `Pacco.Services.Identity.Api` — the .NET project and assembly name; `devmentors/pacco.services.identity` — the published Docker image name).

**Repository:** `hianshul100_Pacco.Services.Identity`, path: `src/Pacco.Services.Identity.Api`.

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

The platform's authentication authority. It registers users, signs them in, issues and signs the JWTs that every other service validates, and manages refresh-token and access-token lifecycles including revocation. It is the only service that mints tokens.

## 2. Main runtime / service type

ASP.NET Core 3.1 HTTP microservice (`netcoreapp3.1`) on **Convey** `0.4.*`, with an in-process RabbitMQ consumer. Layered `.Api` / `.Application` / `.Core` / `.Infrastructure`.

It is the only service whose route table uses plain `UseEndpoints` with hand-written delegates rather than Convey's `UseDispatcherEndpoints` — authentication does not fit the command/query dispatcher shape.

## 3. Key entrypoints

| Entrypoint | Path |
|---|---|
| Process entrypoint | `src/Pacco.Services.Identity.Api/Program.cs` |
| Dependency wiring | `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` |
| Container entrypoint | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Identity.Api.dll` |
| Local run script | `scripts/start.sh` |

## 4. Important modules / packages

Four source projects, enumerated from `Pacco.Services.Identity.sln`: `src/Pacco.Services.Identity.Api`, `.Application`, `.Core`, `.Infrastructure`. `.Core` has no NuGet references. `.Infrastructure` carries the standard Convey stack and additionally references `Convey.Auth`.

Notable types registered in `src/Pacco.Services.Identity.Infrastructure/Extensions.cs`:

- `IJwtProvider` → `JwtProvider` — token minting.
- `IPasswordService` → `PasswordService`, backed by ASP.NET Core's `IPasswordHasher<IPasswordService>` → `PasswordHasher<IPasswordService>`.
- `IIdentityService` → `IdentityService` — sign-up and sign-in.
- `IRefreshTokenService` → `RefreshTokenService`.
- `IRng` → `Rng` — random value generation for tokens.
- `IUserRepository` → `UserRepository`, `IRefreshTokenRepository` → `RefreshTokenRepository`.
- `AuthenticateUsingJwtAsync` extension — authenticates against `JwtBearerDefaults.AuthenticationScheme` and returns `Guid.Empty` when authentication fails.

## 5. External integrations

- **RabbitMQ**, **MongoDB**, **Redis**, **Consul**, **Fabio**, **Vault**, **Jaeger**, **Prometheus**, **Seq** — all through Convey extensions.
- No outbound HTTP calls to peer services: `httpClient.services` in `src/Pacco.Services.Identity.Api/appsettings.json` is empty.

## 6. Data stores & state

- **MongoDB.** Database `identity-service`, connection string `mongodb://localhost:27017`, `seed: false`.
- **Collections:** two, the only service besides `orders-service` and `parcels-service` with more than one —
  - `users` — `.AddMongoRepository<UserDocument, Guid>("users")`
  - `refreshTokens` — `.AddMongoRepository<RefreshTokenDocument, Guid>("refreshTokens")`
- **Outbox collections:** `outbox` and `inbox` (`outbox.enabled: true`, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `disableTransactions: true`).
- **Redis** — `connectionString: localhost`, key prefix `identity:`. `UseAccessTokenValidator()` in `UseInfrastructure()` implies revoked access tokens are held in Redis and checked on each request; the exact key layout is **Unknown**.
- **Query mechanism:** Convey `IMongoRepository<TDocument, TId>` over the MongoDB .NET driver. **No ORM.**
- **Migration tool:** none.
- **Cross-domain coupling:** the user identifier minted here becomes the customer identifier everywhere else. `customers-service` consumes `signed_up` and creates a customer from it, and the API gateway binds `@user_id` from the JWT into `customerId` on downstream calls (`ntrada.yml`). So the identity domain's primary key is silently reused as the customer key across four other databases, with no foreign key and no enforcement.

## 7. Messaging / async / events

**System:** RabbitMQ topic exchange `identity`, queue template `identity-service/{{exchange}}.{{message}}`, `conventionsCasing: snakeCase`, durable, `context.header: message_context`, `spanContextHeader: span_context`. Mongo-backed outbox; Jaeger plugin registered on the broker.

**Consumed** (`src/Pacco.Services.Identity.Infrastructure/Extensions.cs`, line 103) — exactly one subscription:

| Kind | Message | Wire name |
|---|---|---|
| Command | `SignUp` | `sign_up` |

**Published**, per the `identity-service` block of `hianshul100_Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`: `signed_up`, `signed_in`; rejection events `sign_in_rejected`, `sign_up_rejected`. `messages.json` also lists `sign_in` and `sign_up` as commands on this exchange.

Note the mismatch worth flagging: `messages.json` declares `sign_in` as a command on the `identity` exchange, but `UseInfrastructure()` subscribes only to `SignUp`. Nothing consumes `sign_in`. `api-gateway`'s `ntrada-async.yml` likewise routes only synchronous HTTP for sign-in and sign-up — the `identity` module has no `use: rabbitmq` routes at all.

**Consumer of these events:** `customers-service` subscribes to `SignedUp` and creates the customer record.

**Payload key fields:** `sign_up` and `sign_in` carry credentials. Given that the logging configuration explicitly redacts `Password`, `Email`, `Login`, `Secret` and `Token`, at least those field names are expected in this service's message and request payloads. The complete field set is **unknown — requires runtime capture**.

## 8. APIs exposed & consumed

**Exposed** — plain `UseEndpoints` in `src/Pacco.Services.Identity.Api/Program.cs`:

| Method | Path | Behaviour |
|---|---|---|
| GET | `users/{userId}` | user lookup |
| GET | `me` | calls `ctx.AuthenticateUsingJwtAsync()`; returns `401` when the result is `Guid.Empty` |
| POST | `sign-in` | `IIdentityService.SignInAsync` — returns the token pair |
| POST | `sign-up` | `IIdentityService.SignUpAsync`, then `201 Created` with `Location: identity/me` |
| POST | `access-tokens/revoke` | `IAccessTokenService.DeactivateAsync` → `204 No Content` |
| POST | `refresh-tokens/use` | `IRefreshTokenService.UseAsync` — exchanges a refresh token for a new access token |
| POST | `refresh-tokens/revoke` | `IRefreshTokenService.RevokeAsync` → `204 No Content` |

Swagger UI at `docs`. Consul health endpoint `ping`.

**Consumed by:** `api-gateway` exposes only `POST identity/sign-up` and `POST identity/sign-in`, both with `auth: false`. The token-management routes (`refresh-tokens/*`, `access-tokens/revoke`) and `me` are **not exposed through the gateway** — see open questions.

**Consumes:** nothing over HTTP.

## 9. Deployment & runtime clues

- `Dockerfile`: SDK 3.1 build → ASP.NET 3.1 runtime, `ASPNETCORE_URLS=http://*:80`, `ASPNETCORE_ENVIRONMENT=docker`.
- `.travis.yml`: `dotnet: 3.1.100`, branches `master` and `develop`, `./scripts/build.sh`, `./scripts/test.sh`, `after_success: ./scripts/dockerize.sh`.
- Local port `5004`.
- Consul at `http://localhost:8500`, address `docker.for.win.localhost`, ping interval `3`; Fabio at `http://localhost:9999`.
- Published image `devmentors/pacco.services.identity`.

## 10. Security & auth clues

This is the security-critical service of the platform, and its checked-in configuration deserves direct attention.

- **Token signing.** `src/Pacco.Services.Identity.Api/appsettings.json` `jwt` block: `certificate.location: certs/localhost.pfx`, `certificate.password: "test"`, `issuerSigningKey: eiquief5phee9pazo0Faegaez9gohThailiur5woy2befiech1oarai4aiLi6ahVecah3ie9Aiz6Peij`, `expiryMinutes: 60`, `issuer: pacco`, `validateAudience: false`, `validateIssuer: false`, `validateLifetime: true`, `allowAnonymousEndpoints: ["/sign-in", "/sign-up"]`.
- **The signing key is the same literal string hardcoded in the API gateway's `ntrada.yml` and `ntrada-async.yml`.** Both the issuer and the validator carry it in plain text in source control.
- **Four private key materials are committed** under `src/Pacco.Services.Identity.Api/certs/`: `localhost.pfx`, `localhost.key`, `localhost.pem`, `localhost.cer`. The `.pfx` password `test` is in `appsettings.json` beside it.
- `validateIssuer: false` here, while every other service sets `validateIssuer: true` with `validIssuer: pacco`.
- **Vault:** `enabled: true`, `url http://localhost:8200`, `authType: token`, `token: "secret"`, and additionally `username: "user"` / `password: "secret"`. Key-value engine v2 at mount `kv`, path `identity-service/settings`. PKI `roleName: identity-service`, `commonName: identity-service.pacco.io`. `lease.mongo` issues dynamic MongoDB credentials with `autoRenewal: true` and template `mongodb://{{username}}:{{password}}@localhost:27017`.
- **Password storage:** ASP.NET Core `PasswordHasher<T>` — the framework default (PBKDF2). No custom crypto.
- `UseAccessTokenValidator()` runs on every request, so revoked access tokens are rejected before reaching a handler.
- No `security` access-control list and no certificate authentication for inbound service-to-service calls.

## 11. Observability / logging / tracing

- **Tracing:** Jaeger, `serviceName: identity`, UDP `localhost:6831`, `sampler: const`, with `AddJaegerRabbitMqPlugin()` propagating `span_context` across the broker.
- **Correlation:** `Correlation-Context` header read on inbound requests; the `Saga` header is forwarded on outbound messages.
- **Logging:** level `information`, console + rolling file `logs/logs.txt` (daily) + Seq at `http://localhost:5341`. ELK configured but disabled. `excludePaths: ["/", "/ping", "/metrics"]`. The `excludeProperties` redaction list matters more here than anywhere else — it covers `Password`, `Email`, `Login`, `Secret`, `Token`, `ApiKey`, `ApiSecret`, `ClientId`, `ClientSecret`, `ConnectionString`, `api_key`, `access_key`.
- **Metrics:** AppMetrics, `prometheusEnabled: true`, `influxEnabled: false`, database `pacco`, env `local`, interval `5`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `src/Pacco.Services.Identity.Api/appsettings.json` — the `jwt` block is the platform's authentication contract: symmetric key, 60-minute expiry, issuer `pacco`, audience validation off.
- `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` — the composition root, the `AuthenticateUsingJwtAsync` helper other code depends on, and `UseAccessTokenValidator()`, which is the decision to make JWTs revocable rather than purely stateless.
- `src/Pacco.Services.Identity.Api/Program.cs` — the decision to hand-write endpoints rather than use the CQRS dispatcher.
- `Pacco.Services.Identity.rest` — worked API examples.

**Feature-flag system:** none. No flag library and no flag keys exist in this repository.

## 13. Open questions & ambiguities

- `messages.json` lists `sign_in` as a command on the `identity` exchange, but nothing subscribes to it and the gateway never publishes it. Whether asynchronous sign-in was planned and abandoned, or is handled elsewhere, is **Unknown**. **Needs validation.**
- `refresh-tokens/use`, `refresh-tokens/revoke`, `access-tokens/revoke` and `me` exist on the service but are absent from `ntrada.yml`. How a client refreshes a 60-minute token through the gateway is **Unknown**. **Needs validation** — this looks like a real gap in the public API surface.
- `validateIssuer: false` in this service versus `true` everywhere else is unexplained. **Unknown** whether deliberate.
- Whether the certificate in `certs/localhost.pfx` or the `issuerSigningKey` is the key actually used to sign tokens is **Unknown** from configuration alone; both are present and Convey may prefer one.
- Full payload field sets are **unknown — requires runtime capture**.

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `src/`, `src/Pacco.Services.Identity.Api/`, `src/Pacco.Services.Identity.Api/certs/`, `src/Pacco.Services.Identity.Application/`, `src/Pacco.Services.Identity.Core/`, `src/Pacco.Services.Identity.Infrastructure/`, `scripts/`. There is no `wwwroot`, no `package.json`, and no HTML, CSS or JavaScript.

## README vs repository

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| "`dotnet run` … executed in the `/src/Pacco.Services.Identity` directory". | No such directory. The runnable project is `src/Pacco.Services.Identity.Api`. | **Stale doc.** The documented command fails as written. |
| "By default, the service will be available under `http://localhost:5004`." | `appsettings.json` sets the Consul port to `5004`. | **Confirmed.** |
| `./scripts/start.sh`, `docker build`, `docker pull devmentors/pacco.services.identity`. | `scripts/start.sh` and `Dockerfile` exist; the image name matches `compose/services.yml` in `hianshul100_Pacco`. | **Confirmed.** |
| HTTP requests listed in `Pacco.Services.Identity.rest`. | The file exists at the repository root. | **Confirmed.** |
| The README is the shared platform template and describes the service only as "the microservice being part of Pacco solution". | This service issues every token the platform trusts, holds four private key files, and its `jwt` configuration is depended on by all ten other deployables. | **Docs gap — the most serious in the workspace.** Nothing tells a reader that this is the security root of the platform. |

**On disk but not mentioned in `README.md`:** the token model, the committed key material, the revocation mechanism, the `sign_up` subscription, and the Vault integration.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The committed key material under `src/Pacco.Services.Identity.Api/certs/` is development-only and is replaced at deployment, most likely from Vault's PKI engine. | A Vault PKI role `identity-service` with common name `identity-service.pacco.io` is configured, which is the normal way to supply real certificates. | Private keys for a live token issuer would be in source control, and anyone with the repository could mint valid tokens for any user. | Check the deployment pipeline and the Vault path `identity-service/settings`. |
| A2 | The user identifier issued here is reused as the customer identifier across `customers-service`, `orders-service` and `parcels-service`. | `api-gateway` binds `customerId:@user_id` from the JWT in `ntrada.yml`, and `customers-service` creates a customer on `signed_up`. | The platform's identity-to-customer mapping would be misdescribed, affecting every data-ownership statement. | Read the `SignedUp` event class and the `customers-service` handler for it. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** Private key files (`localhost.pfx`, `localhost.key`, `localhost.pem`) and the `.pfx` password `test` are committed to this repository, alongside a JWT signing key that is duplicated verbatim in the API gateway configuration. | Any statement that the platform's authentication is production-ready. | Security owner | Confirm these are development-only; if they are used anywhere reachable, rotate the key material and move it to Vault. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** How does a client refresh an expiring token, given that `refresh-tokens/use` is not exposed through `api-gateway`? | Access tokens expire after 60 minutes. Without a public refresh route, every session ends at the hour mark. | The routes may be reached directly, bypassing the gateway, or the gateway configuration is incomplete. | Platform architect |
| Q2 | **[handled later by the platform inventory review]** Why is `sign_in` declared as a command in `messages.json` when nothing publishes or subscribes to it? | It appears in the platform's event catalogue as if it were live. | Left over from a planned asynchronous sign-in. | Service owner |
| Q3 | **[handled later by the platform inventory review]** Is `validateIssuer: false` deliberate here, when every other service sets it to `true`? | Turning off issuer validation on the token issuer widens what it will accept. | Likely an oversight. | Security owner |
