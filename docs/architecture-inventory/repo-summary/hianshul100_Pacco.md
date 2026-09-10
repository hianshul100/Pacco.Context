# Repository summary — `hianshul100_Pacco`

**Project:** Common Architecture
**Repository:** `hianshul100_Pacco` (also known as: Pacco, the Pacco solution repository, the platform orchestration repository)
**Source of truth for this document:** the files on disk in this repository. Where the README and the files disagree, the files win and the disagreement is written down in *README vs repository* below.

---

## 1. Primary purpose

This repository holds no application code. It is the platform's assembly point: it carries the solution file that ties every service project together, the Docker Compose files that start the shared infrastructure and the services, the process-manager files used to run everything locally, the clone/pull helper scripts, and the platform README with the architecture diagrams.

Evidence: `hianshul100_Pacco/Pacco.sln`, `hianshul100_Pacco/compose/`, `hianshul100_Pacco/services.yml`, `hianshul100_Pacco/prod-services.yml`, `hianshul100_Pacco/scripts/`, `hianshul100_Pacco/README.md`, `hianshul100_Pacco/assets/`.

## 2. Main runtime / service type

Not a runtime. There is no `.csproj` and no `Dockerfile` in this repository. It produces configuration that other runtimes consume: Docker Compose stacks and PM2-style application lists.

## 3. Key entrypoints

| Entrypoint | Path | What it starts |
|---|---|---|
| Shared infrastructure stack | `hianshul100_Pacco/compose/infrastructure.yml` | consul, fabio, grafana, jaeger, mongo, prometheus, rabbitmq, redis, seq, vault |
| Published-image service stack | `hianshul100_Pacco/compose/services.yml` | all services from `devmentors/pacco.*` images |
| Locally-built service stack | `hianshul100_Pacco/compose/services-local.yml` | same services, built from the sibling repository folders |
| Split infrastructure stacks | `hianshul100_Pacco/compose/consul-fabio-vault.yml`, `hianshul100_Pacco/compose/mongo-rabbit-redis.yml`, `hianshul100_Pacco/compose/grafana-seq-jaeger-prometheus.yml`, `hianshul100_Pacco/compose/host-infrastructure.yml` | subsets of the infrastructure, for starting pieces separately |
| Local dev process list | `hianshul100_Pacco/services.yml` | `dotnet run` per service |
| Release process list | `hianshul100_Pacco/prod-services.yml` | `dotnet <Assembly>.dll` from `bin/release/netcoreapp3.1/publish` |
| Repository fetch helpers | `hianshul100_Pacco/scripts/git-clone.sh`, `git-clone-fast.sh`, `git-pull.sh`, `git-pull-fast.sh` (plus `.ps1` variants) | clone or update all sibling repositories |
| Solution | `hianshul100_Pacco/Pacco.sln` | opens all 41 referenced projects in one IDE session |

## 4. Important modules / packages

There are no code packages. The meaningful units are:

- `compose/` — the infrastructure and service topology.
- `compose/prometheus/` and `compose/host-prometheus/` — a Dockerfile plus a `prometheus.yml` scrape configuration each.
- `compose/rabbitmq/` — a Dockerfile plus a `plugins` file (the broker image is built locally so plugins can be enabled).
- `scripts/` — clone and pull helpers.
- `assets/` — four PNG diagrams referenced by the README.
- `docker-images.txt` — a runbook of `docker run` commands for each backing technology.

## 5. External integrations

Every backing technology the platform depends on is declared here, with the ports it is published on:

| Component | Image / build | Published ports |
|---|---|---|
| Consul | consul | 8500 |
| Fabio | fabio (`FABIO_REGISTRY_CONSUL_ADDR=consul:8500`) | 9998, 9999 |
| Grafana | grafana | 3000 |
| Jaeger | jaeger all-in-one | 5775/udp, 5778, 6831/udp, 6832/udp, 9411, 14268, 16686 |
| MongoDB | mongo (named volume) | 27017 |
| Prometheus | built from `./prometheus` | 9090 |
| RabbitMQ | built from `./rabbitmq` | 5672, 15672, 15692 |
| Redis | redis (named volume) | 6379 |
| Seq | seq (`ACCEPT_EULA=Y`) | 5341 → container 80 |
| Vault | vault (`IPC_LOCK`, dev root token set in the file) | 8200 |

Docker network names: `pacco-network` in `compose/infrastructure.yml`, `pacco` in `compose/services.yml` and `compose/services-local.yml`.

`docker-images.txt` additionally documents SQL Server 2017, PostgreSQL, InfluxDB, Elasticsearch, Kibana and Logstash. None of those appear in any compose file or in any service configuration, so they are documented options rather than deployed components.

## 6. Data stores and state handling

No data store is owned here. The compose files provision MongoDB and Redis for the services, both with named Docker volumes. There is no ORM, no query builder, no migration tool, and no table or collection definition anywhere in this repository. Collection names live in the individual service repositories.

## 7. Messaging, async and event mechanisms

RabbitMQ is provisioned here (`compose/rabbitmq/Dockerfile` plus a `plugins` file, ports 5672 / 15672 / 15692). No exchange, queue, routing key or payload is declared in this repository — those live in the service repositories and in the API gateway's YAML. Whether the broker is created with any pre-declared topology beyond what the services declare at startup is **unknown — requires runtime capture**.

## 8. APIs exposed and consumed

None exposed. The compose files fix the externally reachable surface: the API gateway is published on host port 5000, and each service is published on its own host port (5001 availability, 5002 customers, 5003 deliveries, 5004 identity, 5005 operations, 5006 orders, 5007 parcels, 5008 pricing, 5009 vehicles, 5015 ordermaker). Every service container listens on port 80 internally.

## 9. Deployment and runtime clues

- Docker Compose is the only orchestration present. There are **no Kubernetes manifests, no Helm charts, and no Terraform** anywhere in this repository or in any sibling repository.
- `compose/services.yml` runs published images named `devmentors/pacco.*`; `compose/services-local.yml` builds the same services from `../../Pacco.Services.X`, which assumes all repositories are checked out as siblings under a common parent.
- `compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml` on the gateway, so the composed stack runs the **asynchronous** gateway profile by default.
- `operations-service` declares `depends_on` for availability, customers, deliveries, identity, orders, ordermaker, parcels and vehicles — the widest start-up dependency in the platform.
- There is no CI configuration in this repository; each service repository carries its own `.travis.yml`.

## 10. Security and auth clues

- `compose/infrastructure.yml` starts Vault in development mode with a root token value written directly into the file. That is acceptable for a local sandbox and unsafe anywhere else.
- `docker-images.txt` contains example command output that includes what look like real Vault unseal keys and a root token. Those values are **not reproduced in this document**. They should be treated as compromised and rotated, and the example output should be redacted in the file.
- RabbitMQ, MongoDB and Redis are started with default credentials and no TLS.

## 11. Observability, logging and tracing clues

The full observability stack is provisioned here: Jaeger for traces, Prometheus (with a locally built image carrying a `prometheus.yml` scrape config) and Grafana for metrics and dashboards, Seq for structured logs. `compose/grafana-seq-jaeger-prometheus.yml` groups them for starting the observability tier on its own.

## 12. Files holding major architecture decisions; feature flags

Decisions readable from this repository: `README.md` (architecture narrative and diagrams), `Pacco.sln` (which projects are considered part of the solution), `compose/infrastructure.yml` (which backing technologies are canonical), `compose/services.yml` and `compose/services-local.yml` (service topology, ports, start-up order, gateway profile), `services.yml` and `prod-services.yml` (local and release run models), `docker-images.txt` (technology options considered).

**Feature flag system: none.** No LaunchDarkly, Unleash, Split, Flagsmith, ConfigCat or hand-rolled flag store appears in this repository. The closest thing to a switch is the choice of gateway configuration file through the `NTRADA_CONFIG` environment variable, which selects synchronous versus asynchronous edge behaviour for the whole platform. Flag keys: none exist.

## 13. Open questions and ambiguities

Carried into *Assumptions, Blockers & Open Questions* below.

## 14. Frontend stack

**No frontend assets detected — checked:** `hianshul100_Pacco/` (repository root), `hianshul100_Pacco/assets/`, `hianshul100_Pacco/compose/`, `hianshul100_Pacco/scripts/`. The `assets/` directory contains only PNG diagrams used by the README. There is no `package.json`, no bundler configuration, no template file and no static web root in this repository.

---

## README vs repository

**Claimed in the README and confirmed on disk:** microservices on .NET Core 3.1; event-driven integration; Docker Compose start-up sequence (`docker-compose -f infrastructure.yml up -d`, then `docker-compose -f services-local.yml up`); clean architecture with DDD; the Convey framework as the cross-cutting glue.

**Claimed only in documentation, not verifiable from this repository:**

- The README describes the platform as cloud-agnostic and built on CNCF tooling. The compose files support this for local running, but there is no cloud deployment artifact of any kind to verify the claim against. Treat the cloud-agnostic statement as **Future/Intended State (Not Implemented)** until a deployment target exists.

**Present on disk but absent from the README:**

- `prod-services.yml` — a release run model the README never mentions.
- `docker-images.txt` — a technology runbook the README never mentions.
- The split infrastructure compose files (`consul-fabio-vault.yml`, `mongo-rabbit-redis.yml`, `grafana-seq-jaeger-prometheus.yml`, `host-infrastructure.yml`) and the `host-prometheus/` directory.

**Conflicts to surface:**

- **Stale doc / missing source.** `Pacco.sln` references `..\Pacco.APIGateway.Ocelot\src\Pacco.APIGateway.Ocelot\Pacco.APIGateway.Ocelot.csproj`. That repository is not in this workspace and the README's clone list does not include it. The solution therefore cannot be opened or built as written. Status: **Unverifiable — Missing Source Evidence**.
- **Stale doc.** `services.yml` and `prod-services.yml` each list ten applications and omit `ordermaker`, while `compose/services.yml` and `compose/services-local.yml` both include `ordermaker-service`. Running the platform through `services.yml` silently leaves the saga orchestrator out, which breaks the asynchronous order-making flow.
- **Stale doc.** The README's clone list names twelve repositories and does not mention `Pacco.Web` or `Pacco.Context`, both of which are part of this workspace.
- **Stale doc.** The README does not mention that the composed stack runs the asynchronous gateway profile; `compose/services.yml` sets `NTRADA_CONFIG=ntrada-async.docker.yml`, so writes go over RabbitMQ rather than HTTP in every composed environment.

---

## Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Everything below is unresolved. Each blocker and each question names who acts next. Nothing here has been quietly reconciled — where the documents and the files disagree, both readings are recorded above.

### Assumptions

| ID | Assumption | Why we made it |
|---|---|---|
| A1 | The repositories in this workspace are the complete set the platform is built from. | The workspace was handed to us as the source set, and every compose reference except one resolves inside it. |
| A2 | `compose/services-local.yml` expects all repositories to sit side by side under one parent folder. | Its build paths are written as `../../Pacco.Services.X`. |
| A3 | The technologies listed in `docker-images.txt` but absent from every compose file (SQL Server, PostgreSQL, InfluxDB, Elasticsearch, Kibana, Logstash) were evaluated and not adopted. | Nothing in any service configuration points at them, and the metrics configuration explicitly disables InfluxDB. |

### Blockers

| ID | Blocker | Impact | Owner |
|---|---|---|---|
| B1 | **[ACTION NOW]** `docker-images.txt` contains example output holding what appear to be real Vault unseal keys and a root token. | Anyone with read access to the repository holds the keys to a Vault instance. | Platform security owner: rotate the Vault credentials and redact the example output in the file. |
| B2 | **[ACTION NOW]** `Pacco.sln` points at `Pacco.APIGateway.Ocelot`, which does not exist in the workspace. | The solution cannot be opened or built as committed. | Platform maintainer: either restore the missing repository or remove the project reference from the solution file. |

### Open Questions

| ID | Question | Why it matters | Owner |
|---|---|---|---|
| Q1 | **[handled later by the deployment-architecture stage]** Is Docker Compose the intended production runtime, or is there a target platform that simply has no artifacts committed yet? | Decides whether the platform needs a deployment story written from scratch. | Deployment-architecture stage. |
| Q2 | **[ACTION NOW]** Should `ordermaker` be added to `services.yml` and `prod-services.yml`, or is its absence deliberate? | Developers following the README get a platform with no saga orchestrator running. | Platform maintainer. |
| Q3 | **[ACTION NOW]** `compose/services.yml` joins network `pacco`, declared as `external: true` with real name `pacco-network`, which `compose/infrastructure.yml` creates. Is the infrastructure stack always required to start first? | Starting the service stack on its own fails with a missing-network error, and nothing in the file says so. | Platform maintainer: state the ordering in the README or drop the `external` flag. |
