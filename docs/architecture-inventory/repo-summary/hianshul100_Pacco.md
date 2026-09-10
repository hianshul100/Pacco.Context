# Repository Summary — `hianshul100_Pacco`

**Primary name:** `hianshul100_Pacco` (aliases used in this file: `Pacco` — the name used inside the repository's own `README.md` and `.sln`; "root repository" — informal description only, never an entity name).

**Repository:** `hianshul100_Pacco`, path: `/` (repository root; this repository has no `src/` directory and produces no deployable of its own).

**Branch inspected:** `feature/13106/aidlc`.

---

## 1. Primary purpose

Composition and orchestration repository for the Pacco platform. It holds no application source code. It contains: the aggregate Visual Studio solution `Pacco.sln` that references projects living in sibling repositories, the Docker Compose files that stand up infrastructure and services, the clone/pull helper scripts, a PM2-style process list for running everything locally, and the platform-level `README.md` and architecture diagrams.

Evidence: `Pacco.sln`, `compose/`, `scripts/git-clone.sh`, `services.yml`, `prod-services.yml`, `README.md`, `assets/`.

## 2. Main runtime / service type

Not a runtime. No `Dockerfile`, no `.csproj`, no entrypoint. It is a developer tooling and infrastructure-definition repository consumed by humans and by `docker-compose`.

## 3. Key entrypoints

| Entrypoint | Path | What it does |
|---|---|---|
| Infrastructure compose | `compose/infrastructure.yml` | Brings up the full backing-services stack. |
| Services compose | `compose/services.yml` | Runs the ten services plus `api-gateway` from published Docker images. |
| Local services compose | `compose/services-local.yml` | Same set, built from local sources. |
| Split compose files | `compose/consul-fabio-vault.yml`, `compose/mongo-rabbit-redis.yml`, `compose/grafana-seq-jaeger-prometheus.yml`, `compose/host-infrastructure.yml` | Subsets of the infrastructure stack. |
| Clone helpers | `scripts/git-clone.sh`, `scripts/git-clone.ps1`, `scripts/git-clone-fast.sh` | Clone every sibling repository. |
| Pull helpers | `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` | Update every sibling repository. |
| Process list | `services.yml`, `prod-services.yml` | PM2-style app list, each entry `script: dotnet run` with a relative `cwd` pointing into a sibling repository. |
| Aggregate solution | `Pacco.sln` | Opens all service projects together via `..\` relative paths. |

## 4. Important modules / packages

None — no code. The meaningful units are the compose file set, the `prometheus/` and `rabbitmq/` Docker build contexts under `compose/`, and `docker-images.txt`.

- `compose/prometheus/{Dockerfile,prometheus.yml}` and `compose/host-prometheus/{Dockerfile,prometheus.yml}` — a customised Prometheus image with scrape configuration baked in.
- `compose/rabbitmq/{Dockerfile,plugins}` — a customised RabbitMQ image with a plugin list.
- `docker-images.txt` — a free-text runbook of `docker run` invocations and HashiCorp Vault setup commands.

## 5. External integrations

Declared in `compose/infrastructure.yml`: Consul, Fabio, Grafana, Jaeger, MongoDB, Prometheus, RabbitMQ, Redis, Seq, Vault. All are containers on the Docker network named `pacco-network` (compose network key `pacco`).

`docker-images.txt` additionally documents SQL Server 2017, PostgreSQL, InfluxDB, Elasticsearch, Kibana and Logstash. **No service in the workspace connects to any of those** — see "README vs repository" below.

## 6. Data stores & state

This repository stores no data. It *declares* the platform's stores:

- **MongoDB** — `compose/infrastructure.yml`, port `27017`, named volume `mongo:/data/db`.
- **Redis** — port `6379`, named volume.

There is **no ORM and no migration tool anywhere in this repository** — no Entity Framework, Flyway, Liquibase, Alembic or equivalent. Table/collection names are defined in the individual service repositories, not here. No cross-domain foreign keys are declared here.

## 7. Messaging / async / events

Declares the broker only: `rabbitmq` service in `compose/infrastructure.yml`, built from `compose/rabbitmq/Dockerfile`, ports `5672` (AMQP), `15672` (management UI), `15692` (Prometheus metrics). No exchange, queue, routing-key or payload definitions live in this repository; those are in the service repositories and in `hianshul100_Pacco.APIGateway/src/Pacco.APIGateway/ntrada-async.yml`.

## 8. APIs exposed & consumed

None exposed. None consumed.

## 9. Deployment & runtime clues

- Documented start-up sequence (`README.md`): `docker-compose -f infrastructure.yml up -d`, then `docker-compose -f services-local.yml up`.
- `compose/services.yml` pulls published images: `devmentors/pacco.apigateway`, `devmentors/pacco.services.availability`, `.customers`, `.deliveries`, `.identity`, `.operations`, `.orders`, `.ordermaker`, `.parcels`, `.pricing`, `.vehicles`.
- Port allocations for local runs are held per service in each service's own `appsettings.json`, not here.
- No Kubernetes manifests, no Helm charts, no Terraform, no CI configuration in this repository (there is no `.travis.yml` here, unlike every service repository).

## 10. Security & auth clues

- `compose/infrastructure.yml` runs Vault in dev mode with `VAULT_DEV_ROOT_TOKEN_ID=secret` and capability `IPC_LOCK`.
- `docker-images.txt` contains Vault role definitions, including `vault write database/roles/availability-service`, `pki/roles/availability-service` and a `customers-service` PKI role with `allowed_domains=pacco.io`.
- `docker-images.txt` also contains example Vault unseal keys and root tokens copied from HashiCorp's own documentation. They are illustrative sample strings in a runbook, not live credentials, but they are checked into source control and are worth flagging to whoever owns the repository.

## 11. Observability / logging / tracing

Declared, not implemented, here:

- **Jaeger** — `jaegertracing/all-in-one`, ports `5775/udp`, `5778`, `6831/udp`, `6832/udp`, `9411`, `14268`, `16686`.
- **Prometheus** — built from `compose/prometheus`, port `9090`.
- **Grafana** — port `3000`.
- **Seq** — `datalust/seq`, port `5341:80`, `ACCEPT_EULA=Y`.

## 12. Architecture-decision files & feature flags

**Files carrying architecture decisions:**

- `README.md` — states the platform is microservices, event-driven, .NET Core 3.1, "cloud agnostic", and built on the Convey framework.
- `compose/infrastructure.yml` — the authoritative statement of which backing services the platform assumes.
- `Pacco.sln` — the authoritative statement of which projects were meant to make up the solution.
- `assets/clean_architecture.png`, `assets/pacco_overview.png`, `assets/infrastructure.png` — diagrams. Images only; no text was extracted from them for this inventory.

**Feature-flag system:** none. A search across `*.cs`, `*.json`, `*.csproj` and `*.yml` in the whole workspace for `launchdarkly`, `unleash`, `flagsmith`, `split.io`, `featureflag`, `feature_flag`, `featuremanagement` and `IFeatureManager` returned zero matches. There are no flag keys to list.

## 13. Open questions & ambiguities

- `Pacco.sln` references `..\Pacco.APIGateway.Ocelot\src\Pacco.APIGateway.Ocelot\Pacco.APIGateway.Ocelot.csproj`. That repository is not in this workspace and is not in the `README.md` clone list. Whether it was deleted, renamed, or is simply not cloned here is **Unknown**.
- `docker-images.txt` documents relational databases and an ELK stack that no service uses. Whether these are leftovers, aspirations, or used by something outside this workspace is **Unknown**.
- The `assets/*.png` diagrams may contain architectural claims not reflected in the code. They were not read. **Needs validation.**

## 14. Frontend stack

No frontend assets detected — checked: `/` (repository root), `assets/`, `compose/`, `scripts/`. `assets/` contains only PNG diagrams (`clean_architecture.png`, `pacco_logo.png`, `infrastructure.png`, `pacco_overview.png`). There is no `package.json`, no bundler configuration, and no HTML, CSS or JavaScript in this repository.

## README vs repository

Source code and configuration on disk are authoritative; the README is treated as a claim to be checked.

| Claim in `README.md` | What the repository shows | Verdict |
|---|---|---|
| Lists 12 repositories to clone: `Pacco`, `Pacco.APIGateway`, and ten `Pacco.Services.*`. | The workspace contains 14 repositories. `hianshul100_Pacco.Web` and `hianshul100_Pacco.Context` are present but absent from the clone list. | **Stale doc.** The list is incomplete relative to what exists. |
| Implicitly, the solution builds. | `Pacco.sln` references `Pacco.APIGateway.Ocelot`, which does not exist in the workspace. | **Conflict — dangling reference.** The solution cannot open cleanly as checked out here. |
| `docker-compose -f infrastructure.yml up -d` and `docker-compose -f services-local.yml up`. | Both files exist, at `compose/infrastructure.yml` and `compose/services-local.yml`. The README's paths omit the `compose/` prefix, so the commands must be run from inside `compose/`. | **Needs validation** — minor path ambiguity, not a contradiction. |
| Points to `Pacco-sample-scenario.rest` in the API gateway repository. | `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest` exists. | **Confirmed.** |
| The README describes a `docs/` folder for the project. | No `docs/` directory exists in this repository. | **Docs-only claim.** |

**On disk but not mentioned in `README.md`:** `docker-images.txt`, `services.yml`, `prod-services.yml`, the six split compose files under `compose/`, and the `scripts/git-*` helpers.

**No `CONTRIBUTING.md`, no `CHANGELOG.md`, no `docs/` directory** exist in this repository. The documentation pass therefore covered `README.md` only.

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | The example Vault unseal keys and root tokens in `docker-images.txt` are sample strings copied from HashiCorp documentation, not live secrets. | They match the well-known example values printed in Vault's own tutorial output, and the compose file uses a separate dev token `secret`. | If they are real, credentials for a live Vault are in source control. | Ask the repository owner; check whether any Vault instance accepts them. |
| A2 | The PNG diagrams in `assets/` do not contradict the code. | They were not read; only the file names were observed. | An architectural claim in a diagram could be missed or misrepresented. | Open the four images and compare against `compose/infrastructure.yml` and the service repositories. |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Does `Pacco.APIGateway.Ocelot` still exist anywhere, or should the reference be removed from `Pacco.sln`? | The aggregate solution does not open cleanly as checked out. Anyone told to "open Pacco.sln" hits an error immediately. | Most likely a replaced gateway implementation — the live gateway uses Ntrada, not Ocelot. | Repository owner |
| Q2 | **[handled later by the platform inventory review]** Are SQL Server, PostgreSQL, InfluxDB and the ELK stack in `docker-images.txt` still intended parts of the platform? | They shape any statement about what data stores the platform uses. No service code touches them today. | They look like leftovers from earlier experiments. | Platform architect |
