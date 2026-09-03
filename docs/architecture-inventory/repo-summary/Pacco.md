# Repository: `Pacco`

`Pacco` (also known as: Pacco platform root, Pacco solution repository) is the aggregator and
infrastructure-orchestration repository for the Pacco platform. It ships no deployable service of
its own.

- **Repository:** `Pacco`, path: `/` (repository root)
- **Base ref analysed:** `feature/12998/aidlc`
- **Upstream identity in docs:** devmentors.io / `devmentors/Pacco`

---

## README vs repository

`README.md` describes Pacco as an open-source .NET Core 3.1 microservices platform for "exclusive
parcels delivery … limited resources availability", built on Convey, event-driven, cloud-agnostic
CNCF tooling, clean architecture + DDD. It gives a clone list and two `docker-compose` commands.

**Claimed in README, present on disk (confirmed):**

- .NET Core 3.1 — confirmed by every sibling repo's `Dockerfile` and `.travis.yml`.
- `docker-compose -f infrastructure.yml up -d` then `docker-compose -f services-local.yml up` —
  both files exist at `compose/infrastructure.yml` and `compose/services-local.yml`.
- Convey framework — confirmed in every sibling repo's `*.csproj`.
- Clone list of twelve repositories — matches `scripts/git-clone.sh`.

**Claimed in README, not verifiable from this repository:**

- "clean architecture and DDD" — this repository contains no C# source. The claim is verifiable
  only in the service repos, where it holds for nine of eleven services and does **not** hold for
  `pricing-service` or `ordermaker-service` (single-project layouts).
- The pointer to `Pacco-sample-scenario.rest` "in the APIGateway repo" — the file was not located
  in `Pacco.APIGateway`. **Stale doc — needs validation.**

**Present on disk, absent from README (disk-only):**

- `compose/host-infrastructure.yml`, `compose/consul-fabio-vault.yml`,
  `compose/grafana-seq-jaeger-prometheus.yml`, `compose/mongo-rabbit-redis.yml`,
  `compose/services.yml`, `compose/host-prometheus/` — six of the eight compose entry points are
  undocumented, including the split-stack variants.
- `services.yml` and `prod-services.yml` — the PM2-style process manifests that define the
  platform's port allocation (5000–5009). Not mentioned anywhere in the README.
- `docker-images.txt` — the infrastructure runbook, including Vault initialisation, PKI role
  creation and MongoDB dynamic-credential setup.
- `Pacco.sln` — the aggregate solution spanning all sibling repositories.
- `scripts/git-clone-fast.sh`, `scripts/git-pull.sh`, `scripts/git-pull-fast.sh`,
  `scripts/git-clone.ps1`.
- `assets/` — three architecture diagrams (`clean_architecture.png`, `infrastructure.png`,
  `pacco_overview.png`) that are not referenced by any prose in the README beyond the logo.

**Stale doc.** The README's clone list omits `Pacco.Web`, which is in the discovery scope for this
job. The README's "start all services" instruction does not start `ordermaker-service`, because
that service is in none of the compose or process manifests.

**Unknown.** Whether `compose/services.yml` (pulls published images) or `compose/services-local.yml`
(local build) is the intended default; the README names only the latter.

---

## 1. Primary purpose

Aggregate the Pacco platform: hold the cross-repository solution file, define every environment's
infrastructure and service topology as Docker Compose stacks, define the local/production process
manifests, and provide the clone/pull scripts that assemble the multi-repo workspace. It is the
only place where the platform exists as a whole rather than as twelve independent services.

## 2. Main runtime / service type

**Not a runtime.** There is no application entrypoint, no `*.csproj`, no `Dockerfile` producing a
Pacco image, and no CI pipeline. Its outputs are consumed by `docker-compose`, by PM2, and by
developers running the shell scripts.

## 3. Key entrypoints

| Entrypoint | Kind | Effect |
|---|---|---|
| `compose/infrastructure.yml` | Docker Compose | Starts the full backing estate: `consul`, `fabio` (9998/9999), `grafana` (3000), `jaeger`, `mongo` (27017 + named volume), `prometheus`, `rabbitmq` (5672/15672/15692), `redis` (6379 + named volume), `seq` (5341→80), `vault` (8200, dev root token). Network `pacco-network` |
| `compose/services.yml` | Docker Compose | Runs the nine services + gateway from published `devmentors/pacco.*` images |
| `compose/services-local.yml` | Docker Compose | The README's documented service start path |
| `compose/host-infrastructure.yml` | Docker Compose | Host-networking variant of the infrastructure stack |
| `compose/consul-fabio-vault.yml`, `compose/mongo-rabbit-redis.yml`, `compose/grafana-seq-jaeger-prometheus.yml` | Docker Compose | Split stacks — the infrastructure estate in three startable groups |
| `services.yml` | PM2 app manifest | 10 apps, `script: dotnet run`, `max_restarts: 3`, `cwd` pointing at sibling repository source directories |
| `prod-services.yml` | PM2 app manifest | Same 10 apps run as `dotnet <Name>.dll` from `bin/release/netcoreapp3.1/publish`, with explicit `ASPNETCORE_URLS` ports |
| `scripts/git-clone.sh`, `scripts/git-clone.ps1`, `scripts/git-clone-fast.sh` | Shell/PowerShell | Clone the sibling repositories |
| `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` | Shell | Update all sibling clones |
| `Pacco.sln` | .NET solution | Opens every sibling project in one IDE session |

## 4. Important modules / packages

No code packages. The structural units are `compose/` (8 stack definitions + `prometheus/` and
`rabbitmq/` image build contexts), `scripts/` (5 repo-management scripts), `assets/`
(3 architecture diagrams), and the two root process manifests.

`compose/rabbitmq/Dockerfile` builds a custom RabbitMQ image with a `plugins` file — the exposure
of port `15692` indicates the Prometheus plugin is among them.
`compose/prometheus/prometheus.yml` and `compose/host-prometheus/prometheus.yml` are two scrape
configurations for the two networking modes.

## 5. External integrations

This repository *declares* the platform's entire external estate rather than integrating with it:

RabbitMQ, MongoDB, Redis, Consul, Fabio, HashiCorp Vault, Jaeger, Seq, Prometheus, Grafana
(all instantiated in `compose/infrastructure.yml`); plus SQL Server 2017, PostgreSQL, InfluxDB,
Elasticsearch, Kibana and Logstash, which appear **only** in `docker-images.txt`.

**Conflict — documentation vs code.** `docker-images.txt` documents SQL Server, PostgreSQL,
InfluxDB, Elasticsearch, Kibana and Logstash. No service in any of the twelve source repositories
uses any of them: every persistent service uses MongoDB + Redis; every `appsettings.json` sets
`metrics.influxEnabled: false` and `logger.elk.enabled: false`. **Code is authoritative — these six
technologies are not part of the running platform.** They are best read as an operator's reference
list. Recorded as **Future/Intended State (Not Implemented)** or leftover reference material;
which of the two is **Unknown**.

## 6. Data stores / state

Owns no data store. Provisions two named Docker volumes in `compose/infrastructure.yml` (for
`mongo` and `redis`) and documents Vault's storage setup in `docker-images.txt`.

- **ORM / query mechanism:** not applicable — no code.
- **Migration tool:** none present, and none for the platform. `docker-images.txt` documents Vault
  MongoDB dynamic-credential roles (`availability-service`, `customers-service`) but no schema
  migration step. **Needs validation** — see `repo-inventory.md` §6 G11.
- **Table/collection names:** none owned here. The per-service databases are named
  `<service>-service` and are declared in each service's own `appsettings.json`.
- **Cross-domain coupling:** none introduced by this repository.

## 7. Messaging / async / events

Owns no messages. Provides the broker: `compose/infrastructure.yml` builds and runs the `rabbitmq`
service from `compose/rabbitmq/Dockerfile`, exposing `5672` (AMQP), `15672` (management UI) and
`15692` (Prometheus metrics). Default `guest`/`guest` credentials are used by every service.

Exchange and message names are owned by the service repositories; the catalogue is in
`Pacco.Services.Operations/src/Pacco.Services.Operations.Api/messages.json`.

## 8. APIs exposed / consumed

Exposes none. Consumes none. `compose/prometheus/prometheus.yml` scrapes
`docker.for.mac.localhost:5000/metrics-text` — that is a metrics scrape target, not a platform API
dependency.

## 9. Deployment / runtime clues

- **Container orchestration:** Docker Compose only. **No Kubernetes manifests, no Helm chart, no
  Terraform, no Kustomize** anywhere in this repository or the workspace.
- **Process orchestration:** `services.yml` and `prod-services.yml` are PM2-format app lists.
- **Port allocation (from `prod-services.yml`):** api `5000`, availability `5001`, customers
  `5002`, deliveries `5003`, identity `5004`, operations `5005`, orders `5006`, parcels `5007`,
  pricing `5008`, vehicles `5009`.
- **Missing deployable:** `ordermaker-service` (port 5015 per its own `appsettings.json`) appears
  in **none** of `services.yml`, `prod-services.yml`, `compose/services.yml`, or
  `compose/services-local.yml`.
- **Image registry:** `compose/services.yml` pulls `devmentors/pacco.apigateway`,
  `devmentors/pacco.services.availability`, `devmentors/pacco.services.customers`,
  `devmentors/pacco.services.deliveries`, `devmentors/pacco.services.identity`, and the remaining
  service images from Docker Hub.
- **Gateway mode selection:** `compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml`
  on the gateway container — the platform's default local topology is the asynchronous one.
- **CI:** none in this repository. No `.travis.yml`, no GitHub Actions workflow.

## 10. Security / auth clues

- `compose/infrastructure.yml` runs `vault` in dev mode with a root token supplied as an
  environment variable.
- `docker-images.txt` is an operator runbook covering: Vault `operator init` / `unseal`, a
  `userpass` auth method, a policy, PKI mount and role creation for `availability-service` and
  `customers-service` with common name `pacco.io`, and MongoDB dynamic database credential roles.
- **Security observation (do not treat as a finding about intent):** `docker-images.txt` contains
  what appear to be real Vault unseal key shares and a root token in plaintext, and the compose
  files use default `guest`/`guest` RabbitMQ credentials and a fixed Vault dev token. Whether
  these are demo values is **Unknown** — see the Blockers table.
- No authentication or authorisation logic lives here; that is `identity-service` and the gateway.

## 11. Observability / logging / tracing

Provides the whole observability estate as containers: `jaeger` (tracing collector), `seq` (5341,
structured logs), `prometheus` (metrics scrape), `grafana` (3000, dashboards).
`compose/prometheus/prometheus.yml` and `compose/host-prometheus/prometheus.yml` define the scrape
jobs; the target `docker.for.mac.localhost:5000/metrics-text` corresponds to the gateway's
App.Metrics text endpoint. `docker-images.txt` additionally documents an Elasticsearch/Kibana/
Logstash stack that no service is configured to use.

## 12. Architecture-decision files and feature flags

**Files carrying architecture decisions:**

| File | Decision it records |
|---|---|
| `README.md` | Microservices + event-driven integration; Convey as the cross-cutting framework; "clean architecture and DDD … or another style that is the best fit" — which explains why `pricing-service` and `ordermaker-service` deviate |
| `compose/infrastructure.yml` | The backing-service estate: one shared Mongo, Rabbit, Redis, Consul, Fabio, Vault, Jaeger, Seq, Prometheus, Grafana |
| `compose/services.yml` | That the platform runs the **async** gateway configuration by default (`NTRADA_CONFIG=ntrada-async.docker.yml`) |
| `services.yml` / `prod-services.yml` | Port allocation and the authoritative list of processes that constitute a running platform |
| `docker-images.txt` | Vault as the PKI and dynamic-credential authority; the intended (unimplemented) polyglot data estate |
| `assets/clean_architecture.png`, `assets/infrastructure.png`, `assets/pacco_overview.png` | Diagram artefacts; contents not machine-readable from a static pass — **Unknown** whether they agree with the code |

**Feature flag system:** **none detected.** A workspace-wide search for `LaunchDarkly`, `Unleash`,
`Flagsmith`, `Split`, `featureFlag`, `feature_flag` and `featureToggle` across `*.cs`, `*.json` and
`*.yml` returned zero matches. There are therefore **no flag keys to list**.

## 13. Open questions / ambiguities

1. Which compose stack is authoritative for which environment — eight exist, one is documented.
2. Whether the Vault unseal keys and root token in `docker-images.txt` are live credentials.
3. Why `ordermaker-service` is absent from every process and compose manifest.
4. Whether the six documented-but-unused data technologies are planned, abandoned, or reference
   material only.
5. Whether `assets/*.png` still reflect the implemented topology.

## 14. Frontend stack

**No frontend assets detected — checked:** `assets/` (contains three `.png` diagrams only),
`compose/`, `compose/prometheus/`, `compose/host-prometheus/`, `compose/rabbitmq/`, `scripts/`,
and the repository root. There is no `public/`, `public/js/`, `src/`, `resources/js/`, `static/`,
`assets/js/`, `web/`, or `wwwroot/` directory, no `package.json`, no bundler configuration, and no
view templates of any kind. The Grafana container in `compose/infrastructure.yml` serves a
third-party UI on port 3000 but ships no first-party frontend code.

---

## Evidence

| Fact | File |
|---|---|
| Platform description, clone list, start commands | `README.md` |
| Infrastructure estate and network | `compose/infrastructure.yml` |
| Published-image service topology, gateway async default | `compose/services.yml` |
| Documented local start path | `compose/services-local.yml` |
| Split infrastructure stacks | `compose/consul-fabio-vault.yml`, `compose/mongo-rabbit-redis.yml`, `compose/grafana-seq-jaeger-prometheus.yml`, `compose/host-infrastructure.yml` |
| Custom broker and metrics images | `compose/rabbitmq/Dockerfile`, `compose/rabbitmq/plugins`, `compose/prometheus/Dockerfile`, `compose/prometheus/prometheus.yml`, `compose/host-prometheus/prometheus.yml` |
| Development process list and working directories | `services.yml` |
| Production process list and port allocation | `prod-services.yml` |
| Infrastructure runbook, Vault PKI/DB setup, unused data technologies | `docker-images.txt` |
| Cross-repository solution | `Pacco.sln` |
| Workspace assembly scripts | `scripts/git-clone.sh`, `scripts/git-clone.ps1`, `scripts/git-clone-fast.sh`, `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` |
| Architecture diagrams | `assets/clean_architecture.png`, `assets/infrastructure.png`, `assets/pacco_overview.png` |
| Licence | `LICENSE` |

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This document contains unresolved items that require attention before or during implementation. Review and resolve before merging downstream artifacts. Each item below is tagged **[ACTION NOW]** (a human must decide or confirm it before this work can safely proceed) or **[handled later by <stage>]** (a named later stage owns and will prove it) — read the tags first to see what, if anything, is yours to act on.

### Assumptions

| # | Assumption | Rationale | Impact if Wrong | Validation Path |
|---|------------|-----------|-----------------|-----------------|
| A1 | `compose/infrastructure.yml` describes the backing services the platform actually runs against | It is the only stack the README names, and every service's `appsettings.json` points at the hostnames it defines (`mongo`, `rabbitmq`, `redis`, `consul`, `fabio`, `jaeger`, `vault`) | If production uses a different or per-service estate, the shared-infrastructure picture and its blast radius are wrong for every service in this inventory | Compare against the production deployment definition, which is not in this workspace |
| A2 | `prod-services.yml` reflects the intended production port allocation | It is the only file that assigns explicit `ASPNETCORE_URLS` ports to all ten processes, and the ports match every service's `consul` configuration | Port and routing statements elsewhere in this inventory would be wrong | Confirm with whoever operates the platform |
| A3 | SQL Server, PostgreSQL, InfluxDB, Elasticsearch, Kibana and Logstash are not part of the running platform | No service references them; `influxEnabled` and `elk.enabled` are `false` in every `appsettings.json`. Only `docker-images.txt` mentions them | If any is genuinely in use, an entire data or logging path is missing from this inventory | Ask the platform owner whether `docker-images.txt` describes a planned estate or is reference material |

### Blockers

| # | Blocker | Blocks | Owner | Resolution Path | Target Date |
|---|---------|--------|-------|-----------------|-------------|
| B1 | **[ACTION NOW]** `docker-images.txt` holds five Vault unseal key shares and a Vault root token in plaintext, and the compose files use a fixed Vault dev token and default RabbitMQ credentials. Nobody on this side can tell whether any of it still opens a real system | Treating this repository as safe to share, fork or publish, and any security conclusion drawn from it later | Whoever administers the Pacco Vault instance | A person with Vault access must test whether those keys still unseal it; if they do, rotate them and purge the values from git history | TBD |
| B2 | **[ACTION NOW]** `ordermaker-service` exists as a service repository but is in none of this repository's four service manifests, so the definition of "the running platform" is incomplete here | Any statement about what a full Pacco deployment contains, and the order-creation saga's operational status | Platform owner / operations | Confirm whether `ordermaker-service` should be added to the manifests or is intentionally not deployed | TBD |

### Open Questions

| # | Question | Why It Matters | Proposed Answer (if any) | Decision Owner |
|---|----------|----------------|--------------------------|----------------|
| Q1 | **[ACTION NOW]** Which of the eight compose stacks is the one people are meant to use, and for what? | Only `infrastructure.yml` and `services-local.yml` are documented. Someone starting the platform from the other six gets a different topology with no warning — including a different gateway mode | Document one supported path per environment and mark the rest as variants | Platform owner |
| Q2 | **[ACTION NOW]** Are the six data technologies listed in `docker-images.txt` but used by nothing a plan, a leftover, or a mistake? | Reading them as a plan would put a polyglot data estate into the architecture picture that no code supports | They look like leftover reference material, since no service configuration references them at all | Platform architect |
| Q3 | **[handled later by HLD]** Should the platform have a deployment definition beyond Docker Compose and PM2? | There is no Kubernetes, Helm or Terraform anywhere. Compose plus PM2 is a development-scale answer; whether it is also the production answer is not stated anywhere in the repository | Establish what the real production deployment mechanism is and where it is defined | Platform architect |
| Q4 | **[ACTION NOW]** Where is `Pacco-sample-scenario.rest`, which the README points to in the APIGateway repository? | It is the README's only worked example of how to drive the platform end to end, and it is not at the stated location | Locate the file or correct the README | Platform owner |
