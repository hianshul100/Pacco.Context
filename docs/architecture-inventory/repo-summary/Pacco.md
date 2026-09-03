---
title: "Repository Summary — Pacco"
project_key: "Common Architecture"
project_name: "Common Architecture"
stage: "architecture_discovery"
repository: "Pacco"
status: "evidence-based inventory"
---

# Pacco

**Primary name:** `Pacco` (aliases used in this file: "the umbrella repository", "the root repository", "the orchestration repository").
Repository: `Pacco`, path: `hianshul100_Pacco/`

> All evidence below is taken from the files on disk in this clone. Where the repository README claims something the tree does not show, the tree wins and the conflict is stated.

---

## 1. Primary purpose

Umbrella / orchestration repository for the Pacco platform. It holds no application source code. It holds the aggregate solution file, the local process manager definitions, the Docker Compose infrastructure stack, the clone/pull helper scripts, and the platform documentation and diagrams.

Evidence: `hianshul100_Pacco/README.md`, `hianshul100_Pacco/Pacco.sln`, `hianshul100_Pacco/compose/`, `hianshul100_Pacco/scripts/`, `hianshul100_Pacco/assets/`.

## 2. Runtime / service type

Not a runtime service. It is a build-and-run harness:

- `Pacco.sln` — Visual Studio solution aggregating the microservice projects by relative path (`..\Pacco.Services.X\src\...`).
- `services.yml` and `prod-services.yml` — PM2-style application lists for running services locally.
- `compose/*.yml` — Docker Compose stacks for infrastructure and for the containerised services.

## 3. Entrypoints

| Entrypoint | Path | What it starts |
|---|---|---|
| `compose/infrastructure.yml` | `hianshul100_Pacco/compose/infrastructure.yml` | Consul, Fabio, Grafana, Jaeger, MongoDB, Prometheus, RabbitMQ, Redis, Seq, Vault |
| `compose/services.yml` | `hianshul100_Pacco/compose/services.yml` | `api-gateway`, `availability-service`, `customers-service`, `deliveries-service`, `identity-service`, `operations-service` (and further service containers) |
| `compose/services-local.yml` | `hianshul100_Pacco/compose/services-local.yml` | Local variant of the service stack |
| `compose/mongo-rabbit-redis.yml` | `hianshul100_Pacco/compose/mongo-rabbit-redis.yml` | Minimal data/broker stack |
| `compose/consul-fabio-vault.yml` | `hianshul100_Pacco/compose/consul-fabio-vault.yml` | Discovery, load balancing and secrets stack |
| `compose/grafana-seq-jaeger-prometheus.yml` | `hianshul100_Pacco/compose/grafana-seq-jaeger-prometheus.yml` | Observability stack |
| `compose/host-infrastructure.yml` | `hianshul100_Pacco/compose/host-infrastructure.yml` | Host-networking variant |
| `services.yml` | `hianshul100_Pacco/services.yml` | 10 apps run with `dotnet run` |
| `prod-services.yml` | `hianshul100_Pacco/prod-services.yml` | 10 apps run with `dotnet <Assembly>.dll` from `bin/release/netcoreapp3.1/publish` |
| `scripts/git-clone.sh`, `scripts/git-clone.ps1`, `scripts/git-clone-fast.sh`, `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` | `hianshul100_Pacco/scripts/` | Clone or refresh all sibling repositories |

## 4. Modules / packages

No compiled modules. Logical groupings on disk: `assets/` (`clean_architecture.png`, `pacco_logo.png`, `infrastructure.png`, `pacco_overview.png`), `compose/` (including `compose/prometheus/`, `compose/host-prometheus/`, `compose/rabbitmq/`), `scripts/`, plus `Pacco.sln`, `services.yml`, `prod-services.yml`, `docker-images.txt`, `LICENSE`.

## 5. External integrations

Declared in `compose/infrastructure.yml` and `docker-images.txt`: Consul, Fabio, Vault, MongoDB, RabbitMQ, Redis, Jaeger, Seq, Prometheus, Grafana. `docker-images.txt` additionally documents SQL Server 2017, PostgreSQL, InfluxDB, Mongo Express, Elasticsearch 6.4.0, Kibana 6.4.0 and Logstash 6.4.0.

**Conflict — documentation vs code:** no service `appsettings.json` in any repository in this workspace references SQL Server, PostgreSQL, InfluxDB, Elasticsearch, Kibana or Logstash. All persistence in the running services is MongoDB plus Redis. Treat those entries in `docker-images.txt` as a catalogue of optional images, not as deployed platform components. **Needs validation.**

## 6. Data stores / state

None owned by this repository. It provisions the shared stores used by others:

- MongoDB container (`compose/infrastructure.yml`, port `27017`, named volume).
- Redis container (port `6379`, named volume).
- No ORM, no migration tool and no schema definitions exist anywhere in this repository.

## 7. Messaging / async / events

Provisions the broker only: RabbitMQ is built from `compose/rabbitmq/Dockerfile` with a `plugins` file, exposing `5672` (AMQP), `15672` (management UI) and `15692` (Prometheus metrics). This repository declares **no exchanges, no topics and no event payloads** of its own. The platform-wide catalogue of exchange, command and event names lives in `Pacco.Services.Operations` at `src/Pacco.Services.Operations.Api/messages.json`.

## 8. APIs exposed / consumed

None. Port allocation for the services it launches is defined in `prod-services.yml` via `ASPNETCORE_URLS`: `api` `5000`, `availability` `5001`, `customers` `5002`, `deliveries` `5003`, `identity` `5004`, `operations` `5005`, `orders` `5006`, `parcels` `5007`, `pricing` `5008`, `vehicles` `5009`.

## 9. Deployment / runtime clues

- Docker Compose network `pacco`, named `pacco-network`.
- Service images follow `devmentors/pacco.<name>` (for example `devmentors/pacco.apigateway`, `devmentors/pacco.services.availability`), `restart: unless-stopped`.
- `api-gateway` publishes `5000:80`; each service container publishes `500X:80` matching the port table above.
- `operations-service` declares `depends_on: availability-service`.
- `api-gateway` sets `NTRADA_CONFIG=ntrada-async.docker.yml`.
- No Kubernetes manifests, no Helm charts and no Terraform exist in this repository.

## 10. Security / auth clues

- Vault runs in dev mode with `VAULT_DEV_ROOT_TOKEN_ID=secret` and `cap_add: IPC_LOCK` (`compose/infrastructure.yml`).
- `docker-images.txt` contains a worked Vault setup: versioned KV at `secret`, `userpass` auth, a `services` policy, a database secrets engine issuing dynamic MongoDB credentials for `availability-service`, and a PKI engine with `common_name=pacco.io` and roles `availability-service` and `customers-service`.
- **Security finding:** `docker-images.txt` contains example Vault unseal keys and a root token in plain text, and RabbitMQ is configured with the default `guest` / `guest` credentials. These are development values committed to the repository. **[reported, not remediated in this stage]**

## 11. Observability / logging / tracing

- Jaeger container exposes `5775/udp`, `5778`, `6831/udp`, `6832/udp`, `9411`, `14268`, `16686`.
- Seq container maps `5341:80` with `ACCEPT_EULA=Y`.
- Prometheus is built from `compose/prometheus/Dockerfile` with `compose/prometheus/prometheus.yml`; a host variant exists at `compose/host-prometheus/`.
- The Prometheus scrape configuration in `docker-images.txt` defines jobs `prometheus` and `api`, with `metrics_path: '/metrics-text'` against target `docker.for.mac.localhost:5000`.
- Grafana container on `3000`.

## 12. Files carrying major architecture decisions; feature flags

Decision-bearing files: `Pacco.sln` (which projects form the platform), `compose/infrastructure.yml` (which backing services exist), `compose/services.yml` (container topology and the async gateway selection), `services.yml` and `prod-services.yml` (the local run set), `docker-images.txt` (the infrastructure playbook), `README.md` (stated architecture style).

**Feature-flag system: none.** No LaunchDarkly, Unleash, Split, Flagsmith or bespoke flag store appears anywhere in this repository. The only toggles are boolean `enabled` keys per infrastructure integration inside each service's `appsettings.json`, which are configuration switches rather than runtime feature flags. There are therefore no flag keys to list.

## 13. Open questions / ambiguities

Mirrored in the final section of this file.

## 14. Frontend stack

No frontend assets detected — checked: `public/`, `public/js/`, `src/`, `resources/js/`, `static/`, `assets/`, `web/`, `wwwroot/`, and view template directories. The `assets/` directory exists but contains only PNG diagrams (`clean_architecture.png`, `pacco_logo.png`, `infrastructure.png`, `pacco_overview.png`), not web assets. There is no `package.json`, no bundler configuration and no HTML in this repository.

---

## README vs repository

| Claim in the documentation | What the repository shows | Marker |
|---|---|---|
| README describes ".NET Core 3.1", "microservices architecture", "event-driven", "clean architecture + DDD", built on Convey | Every project on disk targets `netcoreapp3.1`; every service references `Convey.*` `0.4.*`; services are split into `.Api` / `.Application` / `.Core` / `.Infrastructure` | Confirmed |
| README lists **12** repositories to clone: `Pacco`, `Pacco.APIGateway` and ten `Pacco.Services.*` | The workspace contains **13** in-scope repositories. `Pacco.Web` is present as a clone but is **absent from the README clone list** | Stale doc |
| README clone list | `Pacco.Services.OrderMaker` is present in the README clone list but is **absent from `services.yml`, `prod-services.yml` and the `compose/services.yml` container set** | Stale doc |
| `Pacco.sln` enumerates 41 projects | Only 40 project files exist across the workspace. `Pacco.APIGateway.Ocelot` (`..\Pacco.APIGateway.Ocelot\src\Pacco.APIGateway.Ocelot\Pacco.APIGateway.Ocelot.csproj`, `Pacco.sln:152`) has **no repository in this workspace** | Unverifiable — Missing Source Evidence |
| `docker-images.txt` documents SQL Server, PostgreSQL, InfluxDB and the Elasticsearch/Kibana/Logstash stack | No service configuration references any of them; InfluxDB and the Elastic stack are explicitly disabled in service settings | Stale doc |
| The Jira backlog item states branch `master` for all repositories | The clones in this workspace are on `feature/12915/aidlc` | Needs validation |

**Docs-only claims:** the SQL Server / PostgreSQL / InfluxDB / ELK stack; the 12-repository clone list.
**Disk-only components:** `compose/host-infrastructure.yml`, `compose/host-prometheus/`, `compose/services-local.yml`, `scripts/git-clone-fast.sh`, `scripts/git-pull-fast.sh` — present on disk, not described in the README.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> This section lists what we assumed, what is blocking us, and what we still need to find out. Everything here is written in plain words so anyone can read it.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The Docker Compose files in `compose/` describe how the platform is meant to run locally, not in production. | The files use development passwords, a development Vault token, and host-machine hostnames. |
| A2 | The extra database and logging images listed in `docker-images.txt` are a reference catalogue, not part of the running platform. | No service configuration file points at any of them. |
| A3 | The port numbers in `prod-services.yml` are the authoritative port map for the platform. | They are the only place where every service port is listed together, and they match each service's own settings. |

### Blockers

| ID | Blocker | Owner and next step |
|---|---|---|
| B1 | The solution file points at a project, `Pacco.APIGateway.Ocelot`, whose source code is not in this workspace. Anyone opening the solution will hit a missing-project error. | **[ACTION NOW]** Architecture discovery stage records it as missing source evidence; the requesting team must confirm whether that repository still exists or should be removed from the solution. |

### Open Questions

| ID | Question | Owner and next step |
|---|---|---|
| Q1 | Which branch is the real source of truth: the `master` branch named in the work item, or the `feature/12915/aidlc` branch that was actually cloned? | **[ACTION NOW]** Confirm with the requesting team before any later stage quotes line numbers. |
| Q2 | Why is the order-making service missing from every run list, even though it has working code and a documented port? | **[handled later by the ADR authoring stage]** Decide and record whether it is an optional demo component or a supported service. |
| Q3 | Is there a production deployment target (Kubernetes, cloud service) anywhere outside these repositories? | **[handled later by the ADR authoring stage]** Only Docker Compose exists here, so the production story cannot be described from this workspace alone. |
