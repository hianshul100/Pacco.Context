# Repository summary — `Pacco`

**Repository:** `Pacco` (workspace clone path: `hianshul100_Pacco`, also known as: Pacco root repository, Pacco solution repository)
**Upstream URL:** https://github.com/hianshul100/Pacco
**Base ref analysed:** `feature/12915/aidlc`
**Classification:** Infrastructure / composition repository — **contains no application source code**.

---

## 1. Primary purpose of the repo

Umbrella / composition repository for the Pacco platform. It carries no C# projects of its own; it aggregates the other repositories into a single Visual Studio solution (`Pacco.sln`), and holds the Docker Compose stacks, process-manager manifests, and clone/pull helper scripts used to run the whole platform locally.

**Evidence:** `README.md`, `Pacco.sln`, `compose/`, `services.yml`, `prod-services.yml`, `scripts/`, `docker-images.txt`.

## 2. Main runtime/service type

**Not a runtime.** No `Program.cs`, no `Dockerfile` producing an application image, no `*.csproj`. The only container images built here are infrastructure sidecars: `compose/prometheus/Dockerfile`, `compose/host-prometheus/Dockerfile`, `compose/rabbitmq/Dockerfile`.

**Evidence:** repository tree contains zero `.csproj` files; `compose/prometheus/Dockerfile`, `compose/rabbitmq/Dockerfile`.

## 3. Key entrypoints

There is no application entrypoint. The operational entrypoints are:

| Entrypoint | File | What it does |
|---|---|---|
| `docker-compose -f infrastructure.yml up -d` | `compose/infrastructure.yml` | Starts Consul, Fabio, Grafana, Jaeger, MongoDB, Prometheus, RabbitMQ, Redis, Seq, Vault on the `pacco-network` bridge |
| `docker-compose -f services-local.yml up` | `compose/services-local.yml` | Starts the platform services against locally-built images |
| `docker-compose -f compose/services.yml up` | `compose/services.yml` | Starts the platform services from published `devmentors/pacco.*` images |
| `services.yml` / `prod-services.yml` | repo root | Process-manager app manifests (`script:`, `cwd:`, `max_restarts:`) that run each service via `dotnet run` (dev) or `dotnet <Service>.dll` (release publish output) |
| `scripts/git-clone.sh`, `scripts/git-clone-fast.sh`, `scripts/git-clone.ps1`, `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` | `scripts/` | Bulk clone/update of the sibling repositories |

**Evidence:** `README.md` ("Open `Pacco/compose` directory and execute…"), `compose/infrastructure.yml`, `compose/services.yml`, `services.yml`, `prod-services.yml`, `scripts/`.

## 4. Important modules/packages

Not a code repository, so there are no packages. The architecturally significant assets are:

- `Pacco.sln` — aggregate solution referencing projects in sibling clones.
- `compose/infrastructure.yml` — the canonical infrastructure topology.
- `compose/consul-fabio-vault.yml`, `compose/mongo-rabbit-redis.yml`, `compose/grafana-seq-jaeger-prometheus.yml` — split infrastructure stacks.
- `compose/host-infrastructure.yml`, `compose/host-prometheus/` — host-networking variants.
- `compose/prometheus/prometheus.yml`, `compose/host-prometheus/prometheus.yml` — Prometheus scrape configuration.
- `compose/rabbitmq/Dockerfile`, `compose/rabbitmq/plugins` — custom RabbitMQ image with plugins enabled.
- `docker-images.txt` — long-form runbook of `docker run` commands and Vault provisioning steps (database secrets engine, PKI engine, policies).
- `assets/` — `pacco_overview.png`, `infrastructure.png`, `clean_architecture.png`, `pacco_logo.png`.

**Evidence:** repository tree; `docker-images.txt`.

## 5. External integrations

All integrations are infrastructure images pulled from public registries, not business integrations:

`consul`, `fabiolb/fabio`, `grafana/grafana`, `jaegertracing/all-in-one`, `mongo`, `prom/prometheus`, `rabbitmq:3-management`, `redis`, `datalust/seq`, `vault`. `docker-images.txt` additionally documents (but `compose/infrastructure.yml` does **not** start) SQL Server 2017, PostgreSQL, InfluxDB, Elasticsearch, Kibana, Logstash, Mongo Express.

**Evidence:** `compose/infrastructure.yml`, `docker-images.txt`.

## 6. Data stores / state handling

Declares the shared data-store containers for local runs; owns no schema, no ORM, no migrations.

- **MongoDB** — `mongo` service, port `27017`, named volume `mongo:/data/db`. This is the shared physical store; each service uses a **separate logical database** (see `repo-inventory.md`).
- **Redis** — `redis` service, port `6379`, named volume `redis:/data`.
- **Migration tooling: none present.** No Alembic / Flyway / Liquibase / EF Core / Doctrine / ActiveRecord artefacts exist anywhere in this clone. Collections are created implicitly by the MongoDB driver at first write.

**Evidence:** `compose/infrastructure.yml` (`mongo`, `redis` service definitions and `volumes:` block); absence of any migrations directory in the tree.

## 7. Messaging / async / event mechanisms

Declares the broker; publishes/consumes nothing itself.

- **RabbitMQ**, built from `compose/rabbitmq/Dockerfile` with the plugin set in `compose/rabbitmq/plugins`. Ports `5672` (AMQP), `15672` (management UI), `15692` (Prometheus metrics plugin).
- Event/topic names are owned by the service repos — the authoritative catalogue is `Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json` (reproduced in `repo-inventory.md`).

**Evidence:** `compose/infrastructure.yml` (`rabbitmq` block), `compose/rabbitmq/Dockerfile`, `compose/rabbitmq/plugins`.

## 8. APIs exposed or consumed

None. This repository exposes no HTTP, gRPC, or messaging API.

## 9. Deployment/runtime clues

- **Docker Compose** is the only orchestration mechanism present: `compose/infrastructure.yml`, `compose/services.yml`, `compose/services-local.yml`, `compose/host-infrastructure.yml`, `compose/consul-fabio-vault.yml`, `compose/mongo-rabbit-redis.yml`, `compose/grafana-seq-jaeger-prometheus.yml`.
- Compose file format `version: "3.7"`; all services attach to an external-style bridge network named `pacco-network`.
- `compose/services.yml` pins published images `devmentors/pacco.apigateway`, `devmentors/pacco.services.<name>` — i.e. images from the **upstream devmentors organisation**, not from the `hianshul100` forks in this workspace.
- `compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml` on `api-gateway`, selecting the **asynchronous (RabbitMQ-publishing)** gateway profile for the composed stack.
- `operations-service` declares `depends_on` for availability, customers, deliveries, identity, orders, ordermaker, parcels, vehicles — the only inter-service ordering constraint expressed in compose.
- `services.yml` / `prod-services.yml` are process-manager manifests (PM2-style `apps:` schema) that reference sibling clone paths such as `../Pacco.Services.Orders/src/Pacco.Services.Orders.Api`, and in the prod variant `.../bin/release/netcoreapp3.1/publish`.
- **No Kubernetes, Helm, Terraform, or GitHub Actions workflows exist in this repository or anywhere in the workspace.** No CI configuration at all in this repo (no `.travis.yml`).

**Evidence:** `compose/services.yml`, `compose/infrastructure.yml`, `services.yml`, `prod-services.yml`.

## 10. Security/auth clues

- **Vault** is started in dev mode with a hard-coded root token: `VAULT_DEV_ROOT_TOKEN_ID=secret` and `cap_add: IPC_LOCK` (`compose/infrastructure.yml`).
- `docker-images.txt` documents production Vault provisioning: `vault secrets enable database` with a MongoDB plugin and per-service roles (`availability-service`), and `vault secrets enable pki` issuing certificates under `common_name=pacco.io` with per-service roles (`availability-service`, `customers-service`) — this is the origin of the `pki.commonName: <service>.pacco.io` settings seen in each service's `appsettings.json`.
- `docker-images.txt` contains **example** Vault unseal keys and an example root token in plain text. They are illustrative sample output copied from a Vault init run, not live credentials for this deployment, but they are real-looking secrets committed to the repository.
- MongoDB, RabbitMQ, and Redis are all started **without authentication** in `compose/infrastructure.yml` (the `MONGO_INITDB_ROOT_*` environment lines are commented out).

**Evidence:** `compose/infrastructure.yml`, `docker-images.txt`.

## 11. Observability/logging/tracing clues

- **Metrics:** Prometheus (`compose/prometheus/Dockerfile` + `prometheus.yml`) scraping `/metrics-text` on the gateway; Grafana on `3000`.
- **Tracing:** Jaeger all-in-one, UDP `6831`/`6832`, UI `16686`, Zipkin-compatible `9411`, collector `14268`.
- **Logging:** Seq on host port `5341` (container `80`), `ACCEPT_EULA=Y`.
- `docker-images.txt` additionally documents an ELK stack (Elasticsearch/Kibana/Logstash) and InfluxDB, neither of which is started by `compose/infrastructure.yml`. Service `appsettings.json` files have `logger.elk.enabled: false` and `metrics.influxEnabled: false`, so ELK/Influx are **Future/Intended State (Not Implemented)** for the default configuration.

**Evidence:** `compose/infrastructure.yml`, `compose/prometheus/prometheus.yml`, `docker-images.txt`.

## 12. Files that appear to contain major architecture decisions

There are **no ADRs and no `docs/` directory** in this repository. The files that carry de-facto architectural decisions are:

| File | Decision it encodes |
|---|---|
| `README.md` | Microservices architecture on .NET Core 3.1; event-driven asynchronous integration; cloud-agnostic CNCF tooling; Convey as the framework; clean architecture + DDD "depending on the particular microservice complexity" |
| `compose/infrastructure.yml` | The canonical infrastructure set: Consul (discovery), Fabio (load balancing), Vault (secrets), RabbitMQ (broker), MongoDB (store), Redis (cache), Jaeger/Prometheus/Grafana/Seq (observability) |
| `docker-images.txt` | Vault database-secrets and PKI provisioning model; the per-service dynamic Mongo credential pattern |
| `assets/pacco_overview.png`, `assets/infrastructure.png`, `assets/clean_architecture.png` | Diagram-only architecture statements — **content not machine-readable; Needs validation by a human reviewer** |
| `compose/services.yml` | Selection of the async gateway profile (`NTRADA_CONFIG=ntrada-async.docker.yml`) for the composed stack |

**Feature flag system: none.** A workspace-wide search for `launchdarkly`, `unleash`, `flagsmith`, `splitio`, `featureflag`, `feature_flag`, `featureToggle` across `*.cs`, `*.json`, `*.yml` returned zero matches. The closest analogues are static boolean config switches (`consul.enabled`, `fabio.enabled`, `vault.enabled`, `outbox.enabled`, `metrics.enabled`, `jaeger.enabled`, `swagger.enabled`, `logger.elk.enabled`) read at startup from `appsettings.json`. There are no runtime-toggleable flag keys.

## 13. Open questions / ambiguities

- **Q:** `compose/services.yml` pulls `devmentors/pacco.*` images while this workspace holds `hianshul100/*` forks. Which image registry/tag is authoritative for the target deployment? — **Needs validation.**
- **Q:** Compose is the only deployment mechanism found. Is there a production deployment target (Kubernetes, ECS, VM + process manager) defined outside these repositories? — **Unknown.**
- **Q:** `assets/*.png` are the only architecture diagrams; their claims could not be verified against code. — **Needs validation.**
- **Q:** `services.yml` / `prod-services.yml` use an `apps:` schema consistent with PM2, but no process manager is named in the README and no `package.json` exists in the workspace. The intended runner is **Unknown**.
- **Q:** `README.md` lists 12 repositories to clone and **omits `Pacco.Web`**, which is nonetheless in the task's repository list. Is `Pacco.Web` part of the platform? — **Needs validation** (see `repo-summary/Pacco.Web.md`).

## 14. Frontend stack

**No frontend assets detected in this repository — checked:** `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view templates (`*.phtml`, `*.blade.php`, `*.erb`, `*.twig`, `*.cshtml`). The `assets/` directory exists but contains only four PNG documentation images (`pacco_logo.png`, `pacco_overview.png`, `infrastructure.png`, `clean_architecture.png`) — no JavaScript, no stylesheets, no templates. No `package.json`, no build tooling, no module-federation or `single-spa` markers anywhere in the clone.

**Evidence:** `assets/` directory listing; workspace-wide `find` for `package.json` returned no results.

---

## README vs repository

**What the README claims:**
- Pacco is a microservices platform on **.NET Core 3.1**, event-driven, using **Convey** and cloud-agnostic CNCF tooling. — **Confirmed** by every service `*.csproj` (`<TargetFramework>netcoreapp3.1</TargetFramework>`, `Convey.* 0.4.*`) and by `compose/infrastructure.yml`.
- "Clean architecture + DDD… or another style that is the best fit." — **Confirmed and materially important.** Eight services use the four-project `Api`/`Application`/`Core`/`Infrastructure` layering; `Pacco.Services.Pricing` and `Pacco.Services.OrderMaker` are deliberately single-project, and `Pacco.Services.Operations` is a single `Api` project plus a gRPC client. The README's hedge is accurate.
- `Pacco.sln` "aggregates all the microservices under a single solution". — **Present on disk**, but it references project paths in sibling clones and therefore only loads when all repos are cloned side by side, as the README instructs.
- Start via `compose/infrastructure.yml` then `services-local.yml`. — **Confirmed**; both files exist.

**Components on disk but not in the README:**
- `prod-services.yml` and `services.yml` (process-manager manifests) — never mentioned.
- `compose/host-infrastructure.yml`, `compose/consul-fabio-vault.yml`, `compose/mongo-rabbit-redis.yml`, `compose/grafana-seq-jaeger-prometheus.yml`, `compose/host-prometheus/` — the split and host-networking stacks are undocumented.
- `docker-images.txt` — a substantial infrastructure and Vault-provisioning runbook, unreferenced by the README.
- `compose/rabbitmq/` and `compose/prometheus/` custom image builds — undocumented.

**README claims not reflected in the clone — Stale doc:**
- All README links point to `github.com/devmentors/...` and all image references to `devmentors/pacco.*`, whereas this workspace is a `hianshul100` fork on branch `feature/12915/aidlc`. The README was not re-pointed when the fork was taken. **Stale doc.**
- The README's clone list of 12 repositories **omits `Pacco.Web`**, which the task's repository list includes. **Stale doc / scope gap.**
- All three architecture images are hot-linked to `raw.githubusercontent.com/devmentors/Pacco/master/assets/...` rather than to the local `assets/` copies that exist in this clone. **Stale doc** (cosmetic, but it means the rendered README depends on the upstream repo remaining public).

**Unknown (neither pass yielded proof):**
- Whether `compose/services-local.yml` and `compose/services.yml` are kept in sync with the service set — not verified beyond file existence.
- The intended production runtime. Nothing in either pass establishes one.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `compose/infrastructure.yml` describes the intended infrastructure set for the platform, not just one developer's laptop setup. | It is the only complete infrastructure definition in the workspace, and the README points to it as *the* way to start the solution. | If a different production topology exists (managed MongoDB Atlas, managed RabbitMQ, a service mesh instead of Consul+Fabio), every dependency and failure-mode conclusion drawn from this file is wrong. | Ask the platform owner for the production deployment definition, or confirm that Compose is genuinely the only target. |
| A2 | The Vault unseal keys and root token printed in `docker-images.txt` are illustrative sample output, not live credentials. | They appear inside a step-by-step tutorial narrative ("You will see the similar output"), and Vault prints exactly this format on `operator init`. | If they are real, a live Vault instance is compromised by a public repository. | A security owner checks whether any Vault instance was ever initialised with these keys, and rotates if so. |
| A3 | The `apps:` manifests `services.yml` and `prod-services.yml` are consumed by a PM2-style process manager. | The schema (`apps:` with `name`/`script`/`cwd`/`max_restarts`/`env`) matches PM2's ecosystem file format. | Anyone told to "run the platform without Docker" will pick the wrong tool and the manifests will not execute. | Confirm with the maintainer which runner reads these files; the answer belongs in the README. |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** The container images this repo composes (`devmentors/pacco.*`) are published by a third-party organisation that this project does not control, while the source in this workspace is a `hianshul100` fork. Nobody has said which images the target deployment should actually run. | Any deployment or DevOps stage that has to build, tag, and push images; also any statement about what version of the code is actually running. | Platform owner / release engineering | Decide whether to keep consuming upstream `devmentors/*` images or to stand up a registry for the fork, then update `compose/services.yml` accordingly. | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Is there a production deployment target for Pacco outside Docker Compose (Kubernetes, ECS, VMs)? Nothing in any of the thirteen clones defines one. | If production runs on something else, that definition lives in a system nobody has given us, and every deployment-related conclusion here is incomplete. | No — Compose plus a process manager appears to be the whole story, but this cannot be proven from the repositories. | Platform owner |
| Q2 | **[handled later by architecture_evolution_generation]** What do `assets/pacco_overview.png`, `assets/infrastructure.png`, and `assets/clean_architecture.png` actually assert, and does any of it contradict the code? | They are the platform's only architecture diagrams; if they disagree with the code, downstream ADRs could inherit a wrong picture. | — | Architecture team |
| Q3 | **[ACTION NOW]** Should the README be re-pointed from `devmentors/*` to the `hianshul100` fork, and should `Pacco.Web` be added to its clone list? | New joiners following the README will clone the wrong repositories and miss one entirely. | Yes — update the twelve links and add `Pacco.Web`, or explicitly record that `Pacco.Web` is out of scope. | Repository maintainer |
