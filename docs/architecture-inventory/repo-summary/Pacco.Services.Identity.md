# Repository summary — `Pacco.Services.Identity`

**Repository:** `Pacco.Services.Identity` (workspace clone path: `hianshul100_Pacco.Services.Identity`)
**Deployable:** `identity-service` (also known as: Identity Service, `Pacco.Services.Identity.Api`, image `devmentors/pacco.services.identity`). **Repository: `Pacco.Services.Identity`, path: `src/Pacco.Services.Identity.Api`.**
**Upstream URL:** https://github.com/hianshul100/Pacco.Services.Identity
**Base ref analysed:** `feature/12915/aidlc`

---

## 1. Primary purpose of the repo

The platform's **authentication authority**. It owns user credentials, issues and refreshes JWT access tokens, revokes them, and emits the `signed_up` event that causes `customers-service` to create a customer record. Every other service in Pacco validates tokens minted here.

**Evidence:** `src/Pacco.Services.Identity.Core/Entities/User.cs`, `Entities/RefreshToken.cs`, `src/Pacco.Services.Identity.Api/Program.cs`, `src/Pacco.Services.Identity.Api/appsettings.json`.

## 2. Main runtime/service type

ASP.NET Core (`netcoreapp3.1`) HTTP microservice plus RabbitMQ consumer, in one process, using the canonical four-project layering (`.Api`, `.Application`, `.Core`, `.Infrastructure`) on Convey.

**One structural difference from every other service:** `Program.cs` uses raw `app.UseEndpoints(...)` with explicit lambdas rather than Convey's `UseDispatcherEndpoints`. Authentication endpoints need to shape their own responses (returning a token object, reading the `Authorization` header, returning `401`), which the dispatcher convention does not accommodate.

## 3. Key entrypoints

| Entrypoint | File |
|---|---|
| `Program.Main` | `src/Pacco.Services.Identity.Api/Program.cs` — `AddConvey().AddWebApi().AddApplication().AddInfrastructure()`, then `UseInfrastructure()` and `UseEndpoints(...)` |
| `GET me` handler | `Program.cs` — calls `ctx.AuthenticateUsingJwtAsync()` and returns `401` when the resulting identity is empty |
| RabbitMQ subscriptions | `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` → `UseInfrastructure` (`SubscribeCommand<SignUp>()`) |
| Container | `Dockerfile` → `ENTRYPOINT dotnet Pacco.Services.Identity.Api.dll` |
| Scripts | `scripts/build.sh`, `scripts/test.sh`, `scripts/dockerize.sh`, `scripts/start.sh` |

## 4. Important modules/packages

**Projects (authoritative list from `Pacco.Services.Identity.sln`):**

| Project | Role |
|---|---|
| `src/Pacco.Services.Identity.Api` | Host, explicit endpoint map, `appsettings.json`, `certs/` |
| `src/Pacco.Services.Identity.Application` | `Commands/` (`SignUp`, `SignIn`, `RevokeAccessToken`, `UseRefreshToken`, `RevokeRefreshToken`, `ChangeCredentials`) + handlers; `Queries/` (`GetUser`) + handler; `Events/`, `Events/Rejected/`; `DTO/`; `Services/` (identity, password, refresh-token, access-token services) |
| `src/Pacco.Services.Identity.Core` | `Entities/User.cs`, `Entities/RefreshToken.cs`, `Exceptions/`, `Repositories/IUserRepository`, `IRefreshTokenRepository` |
| `src/Pacco.Services.Identity.Infrastructure` | `Mongo/Documents/UserDocument.cs`, `RefreshTokenDocument.cs`, `Mongo/Repositories/`, `Auth/` (JWT provider, access-token service), `Services/PasswordService`, `Decorators/` (outbox), `Contexts/`, `Exceptions/ExceptionToResponseMapper.cs` + `ExceptionToMessageMapper.cs`, `Logging/`, `Extensions.cs` |

**No test projects exist in this repository** — notable for the service that owns authentication.

Convey package set matches the platform standard, plus **`Convey.Auth`** carrying more weight here than elsewhere: this service is the only token *issuer*; everywhere else `Convey.Auth` only validates.

## 5. External integrations

| Integration | Direction | Mechanism |
|---|---|---|
| RabbitMQ | in + out | exchange `identity`, topic |
| MongoDB | out | database `identity-service` |
| Redis | out | instance prefix `identity:` |
| Consul | out | registers `identity-service` on port `5004` |
| Fabio | out | `http://localhost:9999` |
| Vault | out | KV v2 `kv/identity-service/settings`; PKI role `identity-service`, CN `identity-service.pacco.io`; dynamic Mongo credentials |
| Jaeger / Seq / Prometheus | out | tracing / logs / metrics |

`httpClient.services` is **empty** — no outbound HTTP calls to other services.

**No external identity provider.** There is no OAuth2/OIDC federation, no SAML, no social login, no LDAP or Active Directory binding, and no MFA provider. Identity is entirely self-contained: local credentials, locally issued JWTs.

## 6. Data stores / state handling

- **Store:** MongoDB, database `identity-service`.
- **Collections — two, the only service with more than one domain collection besides `orders-service` and `parcels-service`:**
  - `users` — `AddMongoRepository<UserDocument, Guid>("users")`
  - `refreshTokens` — `AddMongoRepository<RefreshTokenDocument, Guid>("refreshTokens")` (note the camelCase collection name, inconsistent with every other collection name in the platform, which is lowercase single-word)
  - plus `inbox` and `outbox`.
- **Access mechanism:** Convey `IMongoRepository<>` over `MongoDB.Driver`. **No ORM.**
- **Migration tool: none.** No Flyway, Liquibase, Alembic, or EF Core migrations.
- **Document shapes:** `UserDocument` holds the user id, email, role, password hash, and creation timestamp; `RefreshTokenDocument` holds the token value, the owning `UserId`, creation/revocation timestamps.
- **Cross-domain coupling:** none inbound. Outbound, the `UserId` minted here becomes the `CustomerId` used by `customers-service`, `orders-service`, and `parcels-service` — **the same GUID is the user identity and the customer identity across the platform**. This is an implicit, undeclared identity contract: nothing enforces it, and the gateway's `bind: customerId: @user_id` depends on it holding.
- **Password storage:** hashed via `Infrastructure/Services/PasswordService` using ASP.NET Core's `IPasswordHasher<T>`; no plaintext storage.
- **Outbox:** enabled, `type: sequential`, `expiry: 3600`, `intervalMilliseconds: 2000`, `inboxCollection: inbox`, `outboxCollection: outbox`, `disableTransactions: true`.

## 7. Messaging / async / event mechanisms

**System:** RabbitMQ topic exchange `identity`; `conventionsCasing: snakeCase`; queue template `identity-service/{{exchange}}.{{message}}`; retries `3` every `2` seconds; `spanContextHeader: span_context`.

**Consumed — commands:**

| Message | Wire name | Key payload fields |
|---|---|---|
| `SignUp` | `sign_up` | `UserId`, `Email`, `Password`, `Role` |

`messages.json` also declares `sign_in` on the `identity` exchange, and the gateway's async profile does **not** route `POST /sign-in` to RabbitMQ (it stays `use: downstream`, because the caller needs the token synchronously). The service subscribes only to `SignUp`. **`sign_in` is therefore a declared but unused message contract.**

**Consumed — external events: none.**

**Published — events:**

| Event | Wire name | Key payload fields |
|---|---|---|
| `SignedUp` | `signed_up` | `UserId`, `Email`, `Role` |
| `SignedIn` | `signed_in` | `UserId`, `Role` |

**Published — rejection events:** `sign_up_rejected`, `sign_in_rejected`, each carrying `Email`, `Reason`, `Code` — produced by `Infrastructure/Exceptions/ExceptionToMessageMapper.cs`. Note these rejection events carry the **email address** in the payload, so it travels through the broker and into any subscriber's logs.

**Downstream effect:** `signed_up` is consumed by `customers-service`, which creates the customer record. It is the first link in the platform's onboarding chain: `signed_up` → `customer_created` → three services register the customer locally.

**Reliability:** outbox/inbox decorators wrap every command and event handler.

## 8. APIs exposed or consumed

**Exposed** (`Program.cs`, explicit `UseEndpoints`; base URL `http://localhost:5004`, container port `80`):

| Method | Path | Behaviour | Gateway exposure |
|---|---|---|---|
| POST | `sign-up` | creates the user, publishes `signed_up` | `/identity/sign-up` — **`auth: false`**, gateway generates `userId` |
| POST | `sign-in` | returns the JWT access token as JSON | `/identity/sign-in` — **`auth: false`**, response `content-type: application/json` |
| GET | `me` | `ctx.AuthenticateUsingJwtAsync()`; returns `401` when the identity is empty | `/identity/me` |
| GET | `users/{userId}` | `GetUser` query | `/identity/users/{userId}` — claim `role: admin` |
| POST | `access-tokens/revoke` | revokes the presented access token → `204 No Content` | **not routed at the gateway** |
| POST | `refresh-tokens/use` | exchanges a refresh token for a new access token | **not routed at the gateway** |
| POST | `refresh-tokens/revoke` | revokes a refresh token → `204 No Content` | **not routed at the gateway** |
| GET | `docs`, `ping`, `metrics` | Swagger / health / Prometheus | not routed publicly |

**Gateway gap:** three of the seven endpoints — the entire refresh-token and revocation surface — have **no route in `ntrada.yml` or `ntrada-async.yml`**. Access tokens expire after 60 minutes (`jwt.expiryMinutes: 60`) and there is no public way to refresh them, so a browser client must re-authenticate every hour with the user's password. There is also no public logout.

**Consumed:** none over HTTP.

## 9. Deployment/runtime clues

- `Dockerfile`: sdk:3.1 → aspnet:3.1; `ASPNETCORE_URLS http://*:80`, `ASPNETCORE_ENVIRONMENT docker`; `ENTRYPOINT dotnet Pacco.Services.Identity.Api.dll`.
- Composed as `identity-service` on `5004:80` (`Pacco/compose/services.yml`); present in `Pacco/services.yml` and `Pacco/prod-services.yml` on `5004`.
- CI: `.travis.yml` (`dotnet: 3.1.100`, `branches.only: [master, develop]`, `./scripts/build.sh`, `after_success: ./scripts/dockerize.sh`). **No GitHub Actions.**
- **No Kubernetes, Helm, or Terraform.**
- **Signing material is baked into the image.** `certs/localhost.pfx` is committed and referenced by `jwt.certificate.location`, so the token-signing certificate ships inside the container rather than being mounted or fetched from Vault at start-up.

## 10. Security/auth clues

This is the platform's security-critical service; its `jwt` configuration differs materially from every other service's.

- **Token issuance:** `jwt.issuer: pacco`, `jwt.expiryMinutes: 60`, `jwt.certificate.location: certs/localhost.pfx` with **`certificate.password: "test"` committed in `appsettings.json`**.
- **A symmetric `issuerSigningKey` literal is also committed in `appsettings.json`.** The identical literal appears in `Pacco.APIGateway/src/Pacco.APIGateway/ntrada.yml`, `ntrada-async.yml`, and `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/appsettings.json`. The value is not reproduced here. Because it is the key that signs platform access tokens, anyone with repository read access can mint tokens with arbitrary user ids and the `admin` role, which every service will accept.
- **`jwt.validateIssuer: false`** in this service. **Correction to an earlier reading:** this is not unique to `identity-service`. `operations-service` sets it to `false` as well; the remaining **eight** services set it to `true`. Verified by `grep -rn '"validateIssuer"' hianshul100_Pacco.Services.*/src/*/appsettings.json` across all ten service repositories. Issuer validation is relaxed on the issuer itself and on the operation tracker, and nowhere else.
- `jwt.allowAnonymousEndpoints: ["/sign-in", "/sign-up"]` — the anonymous surface is declared in configuration as well as at the gateway.
- **Roles:** the `Role` on a user is set at sign-up and carried in the token. The gateway grants `role: admin` access to customer listing, customer state changes, and user lookup. Nothing in this service restricts which role a caller may request at sign-up, so whether an ordinary caller can self-assign `admin` is **Needs validation** — `SignUpHandler` and the gateway's `create_customer.schema` are the places to check.
- **Passwords** are hashed with `IPasswordHasher<T>` (PBKDF2 by default in ASP.NET Core). No password-strength policy, lockout, rate limit, or CAPTCHA was found on `sign-in`.
- **Refresh tokens** are persisted in `refreshTokens` with revocation timestamps, so revocation is stateful and enforceable — but unreachable through the gateway (§8).
- **Vault token `secret`** committed in `appsettings.json` (dev Vault root token).
- Log redaction via `logger.excludeProperties`, which includes `Password`, `Email`, `Secret`, and `Token` — but the `sign_up_rejected` / `sign_in_rejected` **message payloads** carry `Email` and are not covered by log-property redaction.

## 11. Observability/logging/tracing

- **Tracing:** Jaeger (`serviceName: identity-service`, UDP `localhost:6831`, `sampler: const`) with the RabbitMQ Jaeger plugin.
- **Logging:** console + rolling file `logs/logs.txt` (daily) + Seq (`http://localhost:5341`); ELK sink present but `enabled: false`. `excludePaths: ["/", "/ping", "/metrics"]`. Handler logging via `.AddHandlersLogging()`.
- **Correlation:** `Correlation-Context` header; `Saga` header forwarded.
- **Metrics:** App.Metrics + Prometheus at `/metrics`. No custom metrics, and in particular **no authentication-specific metrics** — no counters for failed sign-ins, token issuance, or revocations, so brute-force activity would not be visible on any dashboard.

## 12. Files with major architecture decisions; feature flags

| File | Decision |
|---|---|
| `src/Pacco.Services.Identity.Api/appsettings.json` | Token lifetime (60 minutes), signing material (`certs/localhost.pfx` + committed symmetric key), `validateIssuer: false`, anonymous endpoint list |
| `src/Pacco.Services.Identity.Api/Program.cs` | The decision to hand-write endpoints instead of using `UseDispatcherEndpoints`, so auth responses can be shaped explicitly |
| `src/Pacco.Services.Identity.Infrastructure/Auth/` | JWT construction: claims, issuer, expiry — the shape of the token every other service trusts |
| `src/Pacco.Services.Identity.Core/Entities/RefreshToken.cs` | Stateful, revocable refresh tokens rather than stateless rotation |
| `src/Pacco.Services.Identity.Infrastructure/Services/PasswordService.cs` | Password hashing strategy |
| `src/Pacco.Services.Identity.Infrastructure/Extensions.cs` | Composition; subscribes to `SignUp` only |

**Feature flag system: none.** No LaunchDarkly / Unleash / Flagsmith / Split / OpenFeature dependency or configuration. Switches are startup-time booleans in `appsettings.json` (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `vault.pki.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.*.enabled`) plus the JWT validation booleans (`validateAudience`, `validateIssuer`, `validateLifetime`). No business or security behaviour can be toggled at runtime.

## 13. Open questions / ambiguities

- **Can a caller self-assign the `admin` role at sign-up?** The `Role` field is part of the sign-up payload and the gateway's schema was not inspected field-by-field. If unconstrained, privilege escalation is trivial. **Needs validation — treated as a blocker below.**
- **The refresh-token and revocation endpoints are not exposed at the gateway.** Whether this is deliberate (tokens are meant to be short-lived and re-obtained by password) or an oversight is **Unknown**.
- **`sign_in` is declared in `messages.json` but no handler subscribes to it.** Dead contract or future intent — **Unknown**.
- **No tests** in the service that owns authentication.
- **No rate limiting or lockout** on `sign-in` anywhere in the repository or the gateway configuration. **Needs validation** that something upstream provides it.
- **Same GUID used as user id and customer id** across the platform, with nothing enforcing it. **Needs validation.**
- Whether `certs/localhost.pfx` is replaced at deployment time (mounted, or fetched from Vault PKI) is **Unknown** — the Dockerfile copies the build output as-is.

## 14. Frontend stack

**No frontend assets detected — checked:** `public/`, `public/js/`, `src/` (four C# projects only), `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.cshtml`, `*.razor`, `*.html`). None of these web-asset directories exist. No `package.json`, no bundler configuration, no JavaScript or CSS. There is no hosted login page — sign-in is a JSON API only, so any login UI lives in a client outside this repository. The only browser-facing surface is the runtime-generated Swagger UI at `/docs`.

---

## README vs repository

**What the README claims:**
- Identity service, part of Pacco, .NET Core 3.1, runnable with `dotnet run` or Docker, available at `http://localhost:5004`. — **Confirmed** (`appsettings.json` `consul.port: 5004`, `Pacco/compose/services.yml` `5004:80`).

**README claims not reflected in the clone — Stale doc:**
- The README instructs running the command **"in the `/src/Pacco.Services.Identity` directory"**; the actual host project is **`/src/Pacco.Services.Identity.Api`**. The documented path does not exist. **Stale doc** — the same systematic error found in nine of the ten service repositories.
- Links, Travis badge, and Docker Hub image reference the upstream `devmentors` organisation rather than the `hianshul100` fork analysed here. **Stale doc.**

**Components on disk but not in the README:**
- **The whole authentication contract** — token lifetime, claims, roles, the refresh-token model, and the revocation endpoints. For the service every other service depends on for security, none of this is documented anywhere in the repository.
- **The committed signing material** (`certs/localhost.pfx`, its password, and the symmetric signing key in `appsettings.json`) and the fact that the same key is duplicated in three other files across two other repositories.
- The `signed_up` → `customer_created` onboarding chain, and the convention that the user id *is* the customer id.
- The transactional outbox/inbox and the handler decorators.
- The departure from `UseDispatcherEndpoints` used by every other service.
- `scripts/` (`build.sh`, `test.sh`, `dockerize.sh`).

**Unknown (neither pass yielded proof):**
- Whether the deployment process replaces the committed certificate.
- Whether role assignment at sign-up is constrained anywhere.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The user id issued at sign-up is the same GUID used as the customer id everywhere else in the platform. | The gateway binds `customerId: @user_id` on customer, order, and parcel creation, and `customers-service` creates its record straight from the `signed_up` event. | Every ownership check and every `@user_id` binding across the platform would associate work with the wrong person. | Trace a sign-up end to end and compare the id in `users` with the id in `customers`. |
| A2 | The committed `certs/localhost.pfx` and its password are development material, replaced at deployment. | The filename says `localhost` and the password is the literal `test`, which no production process would choose. | The production signing key would be public, and any holder of the repository could mint valid tokens for any user and role. | Ask the platform owner how signing material is provisioned in each environment. |
| A3 | Access tokens are meant to be short-lived (60 minutes) with clients re-authenticating by password, because the refresh endpoints are not publicly routed. | `jwt.expiryMinutes: 60`, and none of the three refresh or revocation endpoints appear in any `ntrada*.yml`. | Clients would face an unexpected hourly logout, and a built, tested revocation capability would sit unusable. | Confirm the intended client session model with the API owner. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** The JWT signing key is committed in plain text in this service's `appsettings.json`, and the same value is duplicated in the gateway's two Ntrada configurations and in the Operations service settings. Anyone who can read the repository can issue tokens the entire platform trusts, including `admin` tokens. | Any deployment to an environment reachable by untrusted users; security sign-off on the whole platform. | Security owner | Move the key into Vault — every service already has a `vault` configuration block — rotate it, and remove it from all four files and from git history. | TBD |
| B2 | **[ACTION NOW]** It is not established whether a caller can choose their own `Role` at sign-up. If they can, anyone can register as `admin` and reach every role-gated route at the gateway. | Security sign-off; any exposure of the sign-up endpoint to the public. | Security owner / service owner | Read `SignUpHandler` and the gateway's `create_customer.schema`; if the role is caller-supplied, force it to a fixed default and add a separate administrative path for elevation. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Should the refresh-token and revocation endpoints be routed through the gateway? | They are implemented and stateful but unreachable, so today there is no logout and no way to refresh a session without the user's password. | Yes — route `refresh-tokens/use`, `refresh-tokens/revoke`, and `access-tokens/revoke`, or record why they are intentionally internal. | API owner |
| Q2 | **[ACTION NOW]** What protects `POST /sign-in` from password guessing? No rate limit, lockout, or CAPTCHA exists in this service or in the gateway configuration. | The sign-in endpoint is public and unthrottled, and no metric would reveal an attack in progress. | Nothing found; likely assumed to be an edge concern. | Security owner |
| Q3 | **[handled later by architecture_evolution_generation]** Should the rejection events `sign_up_rejected` and `sign_in_rejected` carry the email address in their payloads? | The address travels through the broker to every subscriber and into their logs, outside the log-redaction rules that protect it elsewhere. | Replace the address with the user id, or accept and document the exposure. | Security owner |
| Q4 | **[ACTION NOW]** Is it acceptable that the platform's authentication service has no tests at all? | Token construction, password hashing, and revocation are exactly the behaviours where a silent regression is most damaging. | No — token issuance, refresh, and revocation warrant tests at minimum. | Service owner |
| Q5 | **[handled later by architecture_evolution_generation]** Why do `identity-service` and `operations-service` set `validateIssuer: false` when the other eight services set it to `true`? | It is a deliberate-looking inconsistency in the platform's most security-sensitive configuration, and nothing explains it. The two services that disable it are the token issuer and the one service that validates tokens itself inside a SignalR hub — which suggests a shared reason, but the reason is written down nowhere. | Unknown. | Security owner |
