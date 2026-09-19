# Component internals — `platform-infrastructure-orchestration`

| | |
| --- | --- |
| **Component** | `platform-infrastructure-orchestration` |
| **Source repository** | `hianshul100_Pacco` (read-only clone; inspected, never modified) |
| **Scoped path** | `.` (whole repository — **29 tracked files**, none of them source code) |
| **Base ref** | `feature/12998/aidlc` (`HEAD` = `182c054`, *Merge branch 'master' of github.com:devmentors/Pacco*, 2020-07-03) |
| **Batch** | 7 of 7 |
| **Status** | New artifact — no prior `component-internals/platform-infrastructure-orchestration.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `repo-summary/Pacco.md` and `baselines/architecture-baseline.md` remain useful surface catalogues and are **complemented**, not replaced; four statements in `repo-summary/Pacco.md` are corrected here with evidence (§8.4). |
| **Grounding** | Every load-bearing claim below cites a file and, where relevant, a line range. Statements that could not be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** This repository contains **no `*.csproj`, no C#, no `Dockerfile` that
> produces a Pacco application image, and no CI definition**. Its entire content is *composition*:
> seven Docker Compose stacks, two PM2 process manifests, three image build contexts, one aggregate
> `.sln`, five workspace scripts, one operator runbook, a README, a LICENSE, a `.gitignore` and four
> PNGs. Consequently this model is a model of **declarations and the contracts they impose on the
> twelve sibling repositories** — not of an executable's control flow.
>
> Three classes of claim are therefore *not* decidable inside this repository and are tagged where
> they occur:
>
> 1. `[compose]` — Docker Compose / Docker Engine semantics (network creation order, `depends_on`
>    without health gating, unpinned-tag resolution). Behaviour of the tool, not of this repository.
> 2. `[image]` — behaviour of an upstream image that is referenced by an **untagged** name and whose
>    Dockerfile is not in this workspace (`mongo`, `consul`, `vault`, `redis`, `datalust/seq`,
>    `fabiolb/fabio`, `grafana/grafana`, `jaegertracing/all-in-one`, `prom/prometheus`,
>    `rabbitmq:management`).
> 3. `[convey]` — what a Pacco service actually does with the configuration this repository's
>    hostnames feed into. Convey `0.4.*` is NuGet-only in this workspace; the per-service side of
>    each contract is modelled in the twelve sibling models listed in `component-internals/index.md` §1.
>
> Wherever a contract has two ends, this document models **this repository's end** and cites the
> sibling model that owns the other end, rather than re-deriving it.

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

`platform-infrastructure-orchestration` is the repository in which **the Pacco platform exists as a
whole**. Every other repository knows only itself: it builds one image, publishes one set of routes
or messages, and reads a configuration file full of hostnames it never resolves. This repository is
the only place that says *which processes constitute a running Pacco*, *what those hostnames resolve
to*, *which backing services exist*, *on which network*, *at which ports*, and *in which order*.

| Responsibility | Where it lives |
| --- | --- |
| Declare the backing-service estate (broker, databases, cache, discovery, secrets, observability) | `compose/infrastructure.yml:1-147`; host variant `compose/host-infrastructure.yml:1-104` |
| Declare the same estate as three independently startable groups | `compose/mongo-rabbit-redis.yml`, `compose/consul-fabio-vault.yml`, `compose/grafana-seq-jaeger-prometheus.yml` |
| Declare the application topology from published images | `compose/services.yml:1-118` |
| Declare the application topology from sibling working copies | `compose/services-local.yml:1-118` |
| Own the shared network name every stack joins | `compose/infrastructure.yml:129-131` and the `external: true` declarations in the dependent stacks |
| Own the DNS aliases the services' `docker` profiles are written against | the `container_name` key of every service in `compose/*.yml` |
| Own the host-port allocation | the `ports` key of every service in `compose/services*.yml`; `prod-services.yml` for the process mode |
| Own the process-mode topology (PM2) | `services.yml:1-41` (development), `prod-services.yml:1-61` (production) |
| Build the two infrastructure images Pacco cannot use off the shelf | `compose/rabbitmq/Dockerfile`, `compose/prometheus/Dockerfile`, `compose/host-prometheus/Dockerfile` |
| Own the Prometheus scrape target list — the definitive "what is expected to be running" | `compose/prometheus/prometheus.yml:1-56`, `compose/host-prometheus/prometheus.yml:1-56` |
| Assemble the multi-repository working copy | `scripts/git-clone.sh`, `scripts/git-clone-fast.sh`, `scripts/git-clone.ps1`, `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` |
| Present all twelve repositories to one IDE | `Pacco.sln` (733 lines, 41 project references, 36 solution folders) |
| Carry the operator runbook for the estate, including Vault initialisation and PKI | `docker-images.txt:1-460` |

### 1.2 What this component explicitly is **not**

1. **Not a deployable.** There is no `*.csproj` anywhere in the repository (`git ls-files` returns 29
   files, listed in §3.1), no application `Dockerfile`, no entrypoint and no published image named
   after it. It produces nothing that runs; it describes what runs.
2. **Not a build system.** It compiles nothing. `Pacco.sln` is an IDE aggregation artefact (§3.3); the
   authoritative build of each service is that service's own `scripts/build.sh` and `Dockerfile`
   (e.g. `hianshul100_Pacco.Services.Availability/Dockerfile:1-11`).
3. **Not a CI pipeline.** This repository has **no `.travis.yml`, no GitHub Actions workflow and no
   pipeline definition of any kind** — confirmed against the full 29-file tracked list (§3.1). Every
   sibling repository carries its own `.travis.yml`; the platform has none. Release is therefore
   per-repository and uncoordinated ([[independent-per-repository-release]], §3.13, §3.40).
4. **Not a Kubernetes, Helm, Terraform or Kustomize deployment.** No manifest, chart, module or
   overlay exists in the repository or anywhere in the workspace. **Docker Compose and PM2 are the
   only two orchestrators the platform has** (§3.9, §3.31).
5. **Not a service registry or a router.** It *runs* Consul and Fabio as containers
   (`compose/infrastructure.yml:4-25`) but holds no registration, no tag, no route and no KV seed.
   Registration is performed by each service at start-up from its own configuration
   ([[registry-mediated-discovery-and-routing]]; per-service end in
   `component-internals/vehicles-service.md` §3.42).
6. **Not a secret store.** It runs Vault in **dev mode** with the root token supplied as a plain
   environment variable (`compose/infrastructure.yml:115-127`) and, separately, documents a
   production Vault in the runbook (`docker-images.txt:200-321`). The two are different systems
   (§3.27) and **no service in the compose topology reads either** (§3.27, evidence: every
   `appsettings.docker.json` sets `vault.enabled` to `false`).
7. **Not an API surface.** It exposes no HTTP route, no message and no RPC. Its "surface" is a set of
   **files an operator invokes** — modelled as such in §6.
8. **Not the owner of any application configuration value.** With exactly one exception —
   `NTRADA_CONFIG` on the gateway container (§3.16) — this repository sets **no** application
   environment variable and overrides **no** application setting. Everything else is chosen inside
   the sibling repositories' `appsettings*.json`; this repository only has to make the hostnames in
   those files resolve (§3.11).
9. **Not health-aware.** There is no `healthcheck` key in any of the seven stacks, and the single
   `depends_on` (§3.15) is start-order only. Nothing in the platform waits for anything to be ready.
10. **Not a source of truth for the platform's own documentation.** The README's four diagrams are
    referenced by **absolute upstream URLs** (`README.md:1,8,19,25`), not by the `assets/` copies
    committed beside them (§3.37).

### 1.3 The two orchestration modes, and why the distinction matters

The repository declares the *same* platform twice, in two mutually exclusive modes. Almost every
defect and every extension cost in this document follows from that duplication.

| | Compose mode | PM2 mode |
| --- | --- | --- |
| Declared in | `compose/services.yml` / `compose/services-local.yml` | `services.yml` (dev) / `prod-services.yml` (prod) |
| Unit | container | OS process |
| How many application units | **11** (ten services + gateway) | **10** — `ordermaker-service` is absent from both manifests (§3.33) |
| Listening port | `80` inside the container, republished to `5000`–`5009`/`5015` on the host | `5000`–`5009` directly (`prod`), or whatever `launchSettings.json` says (`dev`, §3.31) |
| `ASPNETCORE_ENVIRONMENT` | `docker`, baked into each service image (e.g. `hianshul100_Pacco.Services.Availability/Dockerfile:9`) | `local` via `launchSettings.json` in dev; **unset — the base profile** in prod (§3.32) |
| Hostnames the services resolve | container aliases on `pacco-network` (`mongo`, `rabbitmq`, `redis`, `consul`, `fabio`, `seq`, `jaeger`) | `localhost` / `docker.for.win.localhost` from the base profile (§3.32, §3.35) |
| Backing services | any of the four infrastructure stacks | must be started separately; `compose/mongo-rabbit-redis.yml` is the matching minimal stack (§3.35) |
| Supervision | `restart: unless-stopped` on every container | `max_restarts: 3` per app |
| Who sets the port | **this repository** | **the sibling repository** (dev) / **this repository** (prod) |

The last row is the trap. In the Compose mode this repository is authoritative for ports; in the PM2
development mode it is not authoritative for anything except the working directory — the port and the
configuration profile both come from a file in another repository that this manifest never names
(§3.31).

### 1.4 Position in the platform

| Direction | Counterpart | Mechanism | Evidence |
| --- | --- | --- | --- |
| Outbound (declaration) | all 11 application deployables | container name, port, image or build context | `compose/services.yml:4-112`, `compose/services-local.yml:4-112` |
| Outbound (declaration) | all 10 backing services | image, port, network, volume, environment | `compose/infrastructure.yml:4-127` |
| Outbound (contract) | every service's `appsettings.docker.json` | the container aliases those files hard-code must equal the `container_name` values here | e.g. `hianshul100_Pacco.Services.Availability/src/…Api/appsettings.docker.json` (consul, fabio, seq, jaeger, mongo, rabbitmq, redis hostnames) |
| Outbound (contract) | `api-gateway` | `NTRADA_CONFIG` selects the sync or async gateway topology | `compose/services.yml:9`; consumed at `component-internals/api-gateway.md` §3.5 |
| Outbound (contract) | Prometheus | job names and targets fix the expected process list | `compose/prometheus/prometheus.yml:6-56` |
| Inbound (dependency) | the twelve sibling clones | relative paths `../../Pacco.X` (build contexts), `../Pacco.X` (PM2 `cwd`), `..\Pacco.X` (solution) | `compose/services-local.yml:5`, `services.yml:4`, `Pacco.sln:6` |
| Inbound (dependency) | Docker Hub | eleven `devmentors/pacco.*` images published by the siblings' own `scripts/dockerize.sh` | `compose/services.yml:5,16,25,…` |
| Consumed by | operators and developers only | there is no programmatic consumer of this repository | — |

---

## 2. Core concepts (exhaustive)

Forty-five concepts. The list is not capped: it is every distinct mechanism, contract or declared
absence that a competent engineer must understand before changing this repository. "Owner" is the
file in which the concept is *defined*, not every file that touches it.

| # | Concept | Owner | Modelled in |
| --- | --- | --- | --- |
| 1 | Repository file inventory — the 29-file composition surface | repository root | §3.1 |
| 2 | Multi-repository workspace layout contract (sibling directories) | `scripts/git-clone.sh`, all relative paths | §3.2 |
| 3 | Aggregate solution as IDE-only assembly | `Pacco.sln` | §3.3 |
| 4 | The phantom project — `Pacco.APIGateway.Ocelot` | `Pacco.sln`, all five scripts | §3.4 |
| 5 | Three divergent repository lists (README / scripts / backlog) | `README.md`, `scripts/*` | §3.5 |
| 6 | Clone scripts: sequential and parallel variants | `scripts/git-clone.sh`, `scripts/git-clone-fast.sh` | §3.6 |
| 7 | Pull scripts and the two-branch assumption | `scripts/git-pull.sh`, `scripts/git-pull-fast.sh` | §3.7 |
| 8 | PowerShell clone variant and its divergence | `scripts/git-clone.ps1` | §3.8 |
| 9 | Compose stack taxonomy — seven stacks, four bring-up paths | `compose/` | §3.9 |
| 10 | `pacco-network` and the creator-versus-consumer rule | `compose/infrastructure.yml:129-131` | §3.10 |
| 11 | `container_name` → DNS alias binding contract | every `container_name` in `compose/*.yml` | §3.11 |
| 12 | Host-port allocation map | `ports` keys across `compose/*.yml`, `prod-services.yml` | §3.12 |
| 13 | Image reference and tagging policy (untagged ⇒ `:latest`) | `image` keys across `compose/*.yml` | §3.13 |
| 14 | Build-context topology of the local stack | `compose/services-local.yml` | §3.14 |
| 15 | Start ordering — the single `depends_on` block | `compose/services.yml:59-67` | §3.15 |
| 16 | Gateway mode selection via `NTRADA_CONFIG`, and the sync/async divergence | `compose/services.yml:9`, `compose/services-local.yml:9` | §3.16 |
| 17 | Container supervision (`restart: unless-stopped`) | every service in `compose/*.yml` | §3.17 |
| 18 | The environment-variable injection surface | `environment` keys across `compose/*.yml` | §3.18 |
| 19 | Consul container | `compose/infrastructure.yml:4-13` | §3.19 |
| 20 | Fabio container and its Consul binding | `compose/infrastructure.yml:15-25` | §3.20 |
| 21 | MongoDB container | `compose/infrastructure.yml:53-65` | §3.21 |
| 22 | RabbitMQ container and the custom plugin image | `compose/infrastructure.yml:78-89`, `compose/rabbitmq/` | §3.22 |
| 23 | Redis container | `compose/infrastructure.yml:91-100` | §3.23 |
| 24 | Prometheus container and its custom image | `compose/infrastructure.yml:67-76`, `compose/prometheus/` | §3.24 |
| 25 | The Prometheus scrape model (jobs, targets, path, interval) | `compose/prometheus/prometheus.yml` | §3.25 |
| 26 | Host-networking Prometheus variant | `compose/host-prometheus/prometheus.yml` | §3.26 |
| 27 | Vault: dev-mode container versus the production runbook | `compose/infrastructure.yml:115-127`, `docker-images.txt` | §3.27 |
| 28 | Grafana container and the absence of provisioning | `compose/infrastructure.yml:27-36` | §3.28 |
| 29 | Jaeger all-in-one container | `compose/infrastructure.yml:38-51` | §3.29 |
| 30 | Seq container and its port inversion | `compose/infrastructure.yml:102-113` | §3.30 |
| 31 | PM2 development manifest and its implicit `launchSettings.json` dependency | `services.yml` | §3.31 |
| 32 | PM2 production manifest, the publish-path literal and the missing environment | `prod-services.yml` | §3.32 |
| 33 | OrderMaker's asymmetric membership across the four topologies | `compose/services.yml:78-85` vs `services.yml` | §3.33 |
| 34 | Host-networking infrastructure variant | `compose/host-infrastructure.yml` | §3.34 |
| 35 | The three split infrastructure stacks | `compose/mongo-rabbit-redis.yml`, `compose/consul-fabio-vault.yml`, `compose/grafana-seq-jaeger-prometheus.yml` | §3.35 |
| 36 | Volume and data-durability model (two live, five commented out) | `compose/infrastructure.yml:133-147` | §3.36 |
| 37 | README and asset drift (upstream URLs, unreferenced `assets/`) | `README.md`, `assets/` | §3.37 |
| 38 | `.gitignore` posture in a code-free repository | `.gitignore` | §3.38 |
| 39 | `docker-images.txt` as the operator runbook | `docker-images.txt` | §3.39 |
| 40 | Release and versioning: the absence of platform-level CI | repository root (declared absence) | §3.40 |
| 41 | Committed credential material and estate-wide default credentials | `docker-images.txt`, `compose/infrastructure.yml` | §3.41 |
| 42 | Health, readiness and the absence of gating | declared absence across `compose/*.yml` | §3.42 |
| 43 | Scale-out constraint imposed by `container_name` | every service in `compose/*.yml` | §3.43 |
| 44 | Absent operational concerns (TLS, resource limits, log driver, backup) | declared absences | §3.44 |
| 45 | The drift register — where this repository disagrees with itself | cross-file | §3.45 |

---

## 3. Per concept

Each entry follows the fixed six-part shape used throughout `component-internals/`: **Definition**,
**Representation & storage**, **Lifecycle**, **Invariants & enforcement**, **Extension procedure**,
**Failure modes**. Where a concept is a *declared absence*, "Representation & storage" records what
would have to exist and does not, with the search that establishes it.

### 3.1 Repository file inventory — the 29-file composition surface

1. **Definition.** The complete content of the component. Because there is no code, the file list
   *is* the architecture: every mechanism in this document is one of these 29 files.
2. **Representation & storage.** Tracked files, grouped by role:
   1. Composition — `compose/infrastructure.yml` (147 lines), `compose/host-infrastructure.yml` (104),
      `compose/services.yml` (116), `compose/services-local.yml` (116),
      `compose/mongo-rabbit-redis.yml` (47), `compose/consul-fabio-vault.yml` (47),
      `compose/grafana-seq-jaeger-prometheus.yml` (64).
   2. Image build contexts — `compose/prometheus/Dockerfile` + `compose/prometheus/prometheus.yml`,
      `compose/host-prometheus/Dockerfile` + `compose/host-prometheus/prometheus.yml`,
      `compose/rabbitmq/Dockerfile` + `compose/rabbitmq/plugins`.
   3. Process manifests — `services.yml` (40), `prod-services.yml` (61).
   4. Workspace scripts — `scripts/git-clone.sh`, `scripts/git-clone-fast.sh`, `scripts/git-clone.ps1`,
      `scripts/git-pull.sh`, `scripts/git-pull-fast.sh`.
   5. IDE aggregation — `Pacco.sln` (733).
   6. Documentation and repository hygiene — `README.md` (67), `docker-images.txt` (460), `LICENSE`
      (21), `.gitignore` (331), `assets/` (four PNGs).
3. **Lifecycle.** Files are added by hand and take effect the next time an operator invokes the tool
   that reads them. Nothing in the repository is generated, and nothing validates it: there is no
   linter, schema check, `docker compose config` invocation or test anywhere in the workspace that
   parses these files.
4. **Invariants & enforcement.** The intended invariant — *the four topologies (compose published,
   compose local, PM2 dev, PM2 prod) describe the same platform* — is **enforced by nothing**. It is
   already violated in at least four ways (§3.16 gateway mode, §3.33 OrderMaker, §3.32 environment,
   §3.35 RabbitMQ image). This is the single most important property of the component: **every
   consistency rule in it is a convention held together by review, not by a check.**
5. **Extension procedure.** Adding a mechanism means adding a file in one of the six groups above and
   wiring it into the relevant §6 entry point. There is no registry to update inside this repository;
   the closest thing to one is the Prometheus job list (§3.25), which should be treated as the
   canonical roster (§7.2).
6. **Failure modes.**
   1. A file added but never referenced by an entry point is dead weight that reviewers will assume is
      live — `compose/host-infrastructure.yml` and `compose/host-prometheus/` are referenced by **no**
      documentation in the repository (`README.md` names only `infrastructure.yml` and
      `services-local.yml`), and are in exactly this state (§3.34).
   2. Because no tool validates the YAML, a syntax or key error surfaces only at the operator's
      terminal, at the moment of a bring-up.

### 3.2 Multi-repository workspace layout contract

1. **Definition.** Every relative path in this repository assumes one specific on-disk layout: a
   parent directory containing this repository *and* its twelve siblings as peers.
2. **Representation & storage.** Three different relative-path idioms encode the same assumption:
   1. PM2 `cwd: ../Pacco.APIGateway/src/Pacco.APIGateway` (`services.yml:4`) — one level up, resolved
      from the repository root.
   2. Compose `build: ../../Pacco.APIGateway` (`compose/services-local.yml:5`) — **two** levels up,
      because the compose files live in `compose/` and build contexts resolve relative to the compose
      file's directory `[compose]`.
   3. MSBuild `..\Pacco.APIGateway\src\…` (`Pacco.sln:6`) — one level up, backslash-separated.
3. **Lifecycle.** The layout is created by any of the three clone scripts (§3.6, §3.8) run **from the
   parent directory**, which is the only invocation that produces the required peer arrangement.
   `README.md:31` states the requirement in prose ("put them into the same working directory") and
   `README.md:46` gives the operating instruction — copy the scripts to the directory *next to*
   `Pacco` and run them there. That instruction exists because the scripts live at `scripts/` inside
   this repository, so the natural invocation (`./scripts/git-clone.sh` from the repository root)
   produces the **wrong** layout: it clones the siblings *inside* this repository. The correct
   procedure is documented only in the README, is a manual copy step, and is not encoded in the
   scripts themselves.
4. **Invariants & enforcement.** Invariant: `dirname(this-repo) == dirname(sibling)` for all twelve
   siblings, and every sibling directory is named exactly as its GitHub repository. Enforced by
   nothing; violated silently. Note the asymmetry — `README.md:33` includes `Pacco` in the list to be
   cloned, so the README's procedure is self-consistent, while the scripts' procedure requires the
   operator to already have this repository in order to find the scripts (§3.5).
5. **Extension procedure.** A new sibling repository must be added in **five** places that each encode
   the layout independently: the repository list in `scripts/git-clone.sh:2`,
   `scripts/git-clone-fast.sh:2`, `scripts/git-clone.ps1:2`, `scripts/git-pull.sh:2` and
   `scripts/git-pull-fast.sh:2` — plus `README.md`, `Pacco.sln`, both PM2 manifests, both service
   stacks and both Prometheus configs (§7.1 gives the full ordered checklist).
6. **Failure modes.**
   1. Running a clone script from the repository root nests the siblings, after which every
      `../Pacco.X` and `../../Pacco.X` path misses and `docker compose -f compose/services-local.yml
      build` fails with a missing-context error `[compose]`.
   2. Renaming a sibling directory (e.g. to add a fork prefix — which is exactly what this workspace
      does, `hianshul100_Pacco.Services.Vehicles`) breaks **all three** idioms at once. In this
      workspace none of the relative paths in this repository resolve; the model is derived from the
      declarations, not from an executed bring-up (see B-1 in §8.2).

### 3.3 Aggregate solution as IDE-only assembly

1. **Definition.** `Pacco.sln` is a single Visual Studio solution that reaches across repository
   boundaries to present all twelve codebases as one tree.
2. **Representation & storage.** 733 lines containing **77 `Project(` entries — 41 `.csproj`
   references and 36 solution folders**. All 41 project GUIDs appear in
   `GlobalSection(ProjectConfigurationPlatforms)`, where `Debug` and `Release` are each mapped for
   `Any CPU`, `x64` and `x86`, and **every one of those platform mappings targets `Any CPU`** — the
   x64/x86 solution platforms are aliases, not real target platforms.
   `GlobalSection(NestedProjects)` carries 65 parent/child mappings, giving the per-repository folder
   grouping. The 41 project paths include both PACT contract-test projects and all five Availability
   test projects, so the solution is a *complete* view of the platform's compilable units — with one
   exception (§3.4).
3. **Lifecycle.** Hand-maintained. It is loaded by an IDE and is never read by a build script: no
   `.travis.yml` in any sibling repository references it, and each sibling's `scripts/build.sh`
   builds only its own solution.
4. **Invariants & enforcement.** Invariant: the 41 project paths exist on disk relative to the parent
   directory. Verified for this workspace: **40 of 41 resolve** to real `.csproj` files across the
   sibling clones; the 41st does not (§3.4). Nothing enforces this — an unresolvable path produces an
   IDE load error, never a build failure, because nothing builds through this file.
5. **Extension procedure.** Adding a project means adding a `Project(…)` line, six
   `ProjectConfigurationPlatforms` lines and (usually) one `NestedProjects` line, with a fresh GUID.
   In practice this is done by the IDE. Adding a *repository* additionally means adding a solution
   folder. Because this file is never built, a maintainer may reasonably choose to leave it stale;
   §7.1 marks it **optional but conventional**.
6. **Failure modes.**
   1. A path that does not resolve makes the IDE mark the project unavailable — silent, and easy to
      leave in place for years (§3.4 is a live instance).
   2. Because all platforms map to `Any CPU`, selecting `x64` in the IDE changes nothing; an engineer
      debugging a platform-specific issue will find the setting inert.

### 3.4 The phantom project — `Pacco.APIGateway.Ocelot`

1. **Definition.** A twelfth repository that the workspace-assembly machinery treats as real and that
   does not exist in this workspace's scope.
2. **Representation & storage.** It appears in **six** files: the repository list in all five scripts
   (`scripts/git-clone.sh:2`, `scripts/git-clone-fast.sh:2`, `scripts/git-clone.ps1:2`,
   `scripts/git-pull.sh:2`, `scripts/git-pull-fast.sh:2`) and as the 41st project reference in
   `Pacco.sln`. It appears in **neither** `README.md`'s clone list nor the 13-repository scope in the
   discovery backlog, and there is no `Pacco.APIGateway.Ocelot` directory in this workspace.
3. **Lifecycle.** Historical: Pacco's gateway was implemented twice (Ocelot, then Ntrada). The Ntrada
   gateway is the one wired into every compose stack (`compose/services.yml:4-13`); the Ocelot variant
   survived only in the assembly machinery.
4. **Invariants & enforcement.** The implied invariant — *the script list, the solution and the README
   describe the same set of repositories* — is false (§3.5). Nothing checks it.
5. **Extension procedure.** To retire it: remove one array element from each of the five scripts, and
   remove the project entry, its six configuration mappings and its nesting entry from `Pacco.sln`.
   To keep it: it must be added to `README.md` and to the discovery scope, and a model must be written
   for it. **Whether it should be retired is Q-1 (§8.3)** — the decision belongs to the platform
   owner, not to this document.
6. **Failure modes.**
   1. `scripts/git-clone.sh:10` runs `git clone` without checking the exit status, so a repository
      that has been deleted or renamed upstream produces an error line in an otherwise successful run
      — the operator ends up with eleven repositories and a green-looking log.
   2. `scripts/git-pull.sh:9` chains with `&&` starting at `cd $REPOSITORY`, so a missing directory
      short-circuits that iteration *and leaves the shell in the parent directory* — the loop
      continues correctly, but the missing repository is reported only as a `cd` error.
   3. In `scripts/git-clone-fast.sh:4` and `scripts/git-pull-fast.sh:4` the work runs under
      `xargs -P 0` (unbounded parallelism), so the failing repository's output is interleaved with
      eleven others' and is very likely to be missed entirely.

### 3.5 Three divergent repository lists

1. **Definition.** The platform's membership is declared three times, and the three declarations do
   not agree.
2. **Representation & storage.**
   1. `README.md:29-45` — twelve `git clone` lines: the eleven application repositories plus
      **`Pacco` itself**, and **without** `Pacco.APIGateway.Ocelot`.
   2. `scripts/*` (all five) — twelve entries: the eleven application repositories plus
      **`Pacco.APIGateway.Ocelot`**, and **without** `Pacco`.
   3. The discovery backlog — thirteen repositories under `hianshul100/*`: the eleven application
      repositories, `Pacco` and **`Pacco.Web`**, without `Pacco.APIGateway.Ocelot`.
   The three lists differ by exactly the symmetric difference `{Pacco, Pacco.Web}` versus
   `{Pacco.APIGateway.Ocelot}`.
3. **Lifecycle.** Each list is edited independently when a repository is added or removed; there is no
   generated list and no single source.
4. **Invariants & enforcement.** None. The lists are three hand-maintained copies.
5. **Extension procedure.** Treat `scripts/git-clone.sh:2` as the primary list (it is the one an
   operator actually executes), mirror it into the other four scripts, then reconcile `README.md` —
   noting that the README's list is deliberately different in one respect: it includes `Pacco`,
   because the README is written for someone who has not yet cloned anything.
6. **Failure modes.**
   1. An operator who follows `README.md` gets a working platform without the Ocelot gateway; an
      operator who runs `scripts/git-clone.sh` gets the Ocelot gateway and **no copy of this
      repository** — and therefore no compose files. Neither path alone produces the layout §3.2
      requires.
   2. Any tooling that infers platform membership from one list is wrong about the other two — the
      `repo-summary/Pacco.md` claim that the README list "matches `scripts/git-clone.sh`" is an
      instance of exactly this error and is corrected in §8.4.

### 3.6 Clone scripts: sequential and parallel variants

1. **Definition.** Two Bash scripts that materialise the sibling repositories.
2. **Representation & storage.**
   1. `scripts/git-clone.sh` — a Bash array at line 2, a `for` loop at lines 4-12, and
      `git clone https://github.com/devmentors/$REPOSITORY.git` at lines 9-10. Line 11 is
      `cd $REPOSITORY && cd ..`, which is a **no-op** in the success case and a silent error in the
      failure case; it changes nothing about the resulting layout.
   2. `scripts/git-clone-fast.sh` — the same array, then a single pipeline at line 4 that splits the
      array on whitespace with `sed` and feeds it to `xargs -I {} -n 1 -P 0 sh -c '…'`. `-P 0` means
      *as many processes as possible*, i.e. twelve concurrent clones.
3. **Lifecycle.** Run once, by hand, when a developer sets up a workspace. Never re-run in place —
   `git clone` into an existing non-empty directory fails.
4. **Invariants & enforcement.**
   1. Both scripts clone from **`https://github.com/devmentors/…`** — the upstream organisation,
      hard-coded at `scripts/git-clone.sh:9` and `scripts/git-clone-fast.sh:4`. In a fork (this
      workspace is `hianshul100/*`), running these scripts produces **upstream** clones, not the
      fork's. There is no variable, argument or environment override for the organisation.
   2. Both clone the **default branch only**; no `--branch` argument is passed. The pull scripts
      (§3.7) assume `develop` and `master` both exist, so the clone and pull scripts disagree about
      which branches a repository has.
5. **Extension procedure.** To make the organisation configurable, introduce a variable with a default
   (`ORG=${ORG:-devmentors}`) in the three clone scripts and the two pull scripts, and document it in
   `README.md:29-45`. To add a repository, edit line 2 of each script (§3.2 point 5).
6. **Failure modes.**
   1. Fork users silently get upstream code — the most consequential defect in the script set, because
      the failure is invisible: everything clones, everything builds, and the developer is working on
      the wrong remote.
   2. `-P 0` with twelve concurrent HTTPS clones can exhaust connection limits or credential-helper
      capacity; there is no retry, and a partially cloned workspace looks complete.
   3. Neither script checks `git clone`'s exit status, so the loop's overall exit code is that of the
      last operation.

### 3.7 Pull scripts and the two-branch assumption

1. **Definition.** Two Bash scripts that refresh an existing workspace.
2. **Representation & storage.**
   1. `scripts/git-pull.sh:9` — per repository:
      `cd $REPOSITORY && git checkout develop && git pull && git checkout master && git pull && cd ..`.
      The `&&` chain means a failure at any step skips the rest for that repository, and the final
      state is **`master`**.
   2. `scripts/git-pull-fast.sh:4` — the `xargs -P 0` form, using `git -C {}` rather than `cd`, and
      with the steps separated by `;` rather than `&&` so each runs regardless of the previous. It
      ends with an extra `git -C {} checkout develop`, so the final state is **`develop`**.
3. **Lifecycle.** Run by hand whenever a developer wants the workspace current.
4. **Invariants & enforcement.** Both assume every repository has both a `develop` and a `master`
   branch and that neither has local modifications. Neither assumption is checked.
5. **Extension procedure.** If the platform moves to a single default branch, both scripts collapse to
   one `git -C {} pull`; if it keeps two, the *final checked-out branch* should be made consistent
   between them — see Q-2 (§8.3).
6. **Failure modes.**
   1. **The two scripts leave the workspace on different branches** (`master` versus `develop`). A
      developer who alternates between them gets different code with no indication that anything
      changed. This is a genuine behavioural divergence between two files that present themselves as
      fast/slow variants of one operation.
   2. A dirty working tree makes `git checkout` fail. In `git-pull.sh` the `&&` chain then skips that
      repository entirely (safe but silent); in `git-pull-fast.sh` the `;` sequencing runs
      `git pull` anyway on whatever branch is checked out (unsafe, and interleaved into eleven other
      repositories' output).
   3. Neither script handles a repository that is missing from the workspace beyond a `cd`/`git -C`
      error line.

### 3.8 PowerShell clone variant and its divergence

1. **Definition.** A Windows-native equivalent of `scripts/git-clone.sh`.
2. **Representation & storage.** `scripts/git-clone.ps1` — a twelve-element array at line 2 and a
   `foreach` loop at lines 4-10 issuing `git clone` against the same hard-coded `devmentors`
   organisation (line 8).
3. **Lifecycle.** Same as §3.6: run once at workspace setup.
4. **Invariants & enforcement.** The array must stay identical to the four Bash arrays. At the base
   ref it is — all five lists contain the same twelve names in the same order.
5. **Extension procedure.** Any change to the repository set must touch this file too; it is the one
   most likely to be forgotten because a Linux/macOS maintainer never runs it.
6. **Failure modes.**
   1. **There is no PowerShell counterpart to the pull scripts.** A Windows developer can create a
      workspace with the supplied tooling but cannot refresh one; they must fall back to a shell or do
      it by hand.
   2. `git clone` failures are not detected (`$LASTEXITCODE` is never inspected), matching §3.6.

### 3.9 Compose stack taxonomy — seven stacks, four bring-up paths

1. **Definition.** The `compose/` directory holds seven independently invocable Compose files. They
   are not layers to be merged with repeated `-f` flags; they are **alternatives**, and choosing the
   wrong pair produces a platform that starts but does not work.
2. **Representation & storage.** All seven declare `version: "3.7"` on line 1. Their roles:

   | # | Stack | Role | Network declaration | Creates or consumes `pacco-network` |
   | --- | --- | --- | --- | --- |
   | 1 | `infrastructure.yml` | all ten backing services, bridge networking | lines 129-131 | **creates** |
   | 2 | `host-infrastructure.yml` | the same ten, host networking | **no `networks:` block at all** | neither (§3.34) |
   | 3 | `mongo-rabbit-redis.yml` | data plane only | lines 38-40 | **creates** |
   | 4 | `consul-fabio-vault.yml` | discovery + secrets | lines 41-44 | consumes (`external: true`) |
   | 5 | `grafana-seq-jaeger-prometheus.yml` | observability | lines 54-57 | consumes (`external: true`) |
   | 6 | `services.yml` | 11 applications from published images | lines 114-117 | consumes (`external: true`) |
   | 7 | `services-local.yml` | 11 applications built from siblings | lines 114-117 | consumes (`external: true`) |

3. **Lifecycle.** An operator picks **one** infrastructure path and **one** application path:
   1. `infrastructure.yml` → `services.yml` — published images, the "just run it" path.
   2. `infrastructure.yml` → `services-local.yml` — locally built images; this is the pair documented
      in `README.md`, which names `infrastructure.yml` and `services-local.yml` and nothing else.
   3. `mongo-rabbit-redis.yml` (+ optionally stacks 4 and 5) → either application stack — the
      à-la-carte path.
   4. `mongo-rabbit-redis.yml` alone → PM2 (§3.31), the no-container-for-my-code path.
   5. `host-infrastructure.yml` → PM2 — the only sound use of stack 2 (§3.34).
4. **Invariants & enforcement.**
   1. Exactly one *creator* stack must be started before any *consumer* stack (§3.10). Enforced by
      Docker, as a hard error, not by this repository `[compose]`.
   2. Stacks 1 and 3 both define `mongo`, `rabbitmq` and `redis` with the same `container_name`
      values, so they are mutually exclusive — starting both fails on the duplicate name `[compose]`.
   3. Stack 1 is the union of stacks 3, 4 and 5 *in membership* but **not in content**: stack 3's
      RabbitMQ is a different image from stack 1's (§3.22, §3.35).
5. **Extension procedure.** A new backing service must be added to `infrastructure.yml`, to
   `host-infrastructure.yml` (with `network_mode: host` and no `ports`), and to whichever of the three
   split stacks it thematically belongs to — three edits for one capability. §7.3 gives the order.
6. **Failure modes.**
   1. Starting stack 6 or 7 before any creator stack fails immediately with a "network declared as
      external, but could not be found" error `[compose]` — this is the *good* failure, because it is
      loud.
   2. Starting stack 3 instead of stack 1 succeeds and silently removes the RabbitMQ Prometheus
      endpoint the observability stack scrapes (§3.22). This is the *bad* failure: the platform runs
      and one scrape job is permanently down.
   3. Nothing prevents mixing stack 2 (host networking) with stack 6 (bridge, external network); the
      result is that no service can resolve `mongo`, `rabbitmq` or any other backing hostname (§3.34).

### 3.10 `pacco-network` and the creator-versus-consumer rule

1. **Definition.** A single user-defined bridge network is the entire connectivity model of the
   platform. Every container in the bridge-mode stacks joins it; nothing else exists.
2. **Representation & storage.** Each stack declares a Compose-local network key `pacco` and maps it
   to the fixed external name `pacco-network` via `name:`. Creators
   (`compose/infrastructure.yml:129-131`, `compose/mongo-rabbit-redis.yml:38-40`) stop there;
   consumers (`compose/consul-fabio-vault.yml:41-44`, `compose/grafana-seq-jaeger-prometheus.yml:54-57`,
   `compose/services.yml:114-117`, `compose/services-local.yml:114-117`) add `external: true`.
   Because `name:` pins the real network name, the usual `<project>_<network>` prefixing does not
   apply and the network is shared across Compose projects regardless of directory name `[compose]`.
3. **Lifecycle.** Created by the first creator stack's `up`; **not removed** by that stack's `down`
   while another stack still has containers attached; removed when the creating project is torn down
   with no attached endpoints `[compose]`.
4. **Invariants & enforcement.**
   1. The network name is the literal `pacco-network` in all six declarations — verified identical
      across the six files listed above.
   2. Exactly one creator must run first. Enforced by Docker at consumer start-up.
   3. There is **no `driver:` key anywhere**, so the default bridge driver applies; there is no subnet,
      IPAM, alias or `internal: true` declaration. Every container on the network can reach every
      other container on every port — **there is no network segmentation between the application tier
      and the data tier** (§3.44).
5. **Extension procedure.** To segment (e.g. a separate data network), add a second network key to
   `infrastructure.yml`, attach the data containers to both, and attach the application stacks only to
   the front network — but note that every service's `appsettings.docker.json` resolves `mongo`,
   `rabbitmq` and `redis` by name, so any segmentation must keep those three reachable from every
   application container. Nothing else needs to change.
6. **Failure modes.**
   1. Tearing down the creator stack while consumers are running leaves the network in place but
       removes the backing services — the application containers stay up, healthy from Docker's
       perspective, and fail every request `[compose]`.
   2. A previously created `pacco-network` from a different Compose project (or one created by hand)
      satisfies `external: true` regardless of its driver or subnet; there is no validation that it is
      the network this repository intended.

### 3.11 `container_name` → DNS alias binding contract

1. **Definition.** The most load-bearing contract in the repository, and the one with no local
   evidence of its own correctness: the `container_name` values chosen here are the hostnames
   hard-coded in twelve other repositories' `docker` configuration profiles.
2. **Representation & storage.** Two sets of names:
   1. Backing services — `consul`, `fabio`, `grafana`, `jaeger`, `mongo`, `prometheus`, `rabbitmq`,
      `redis`, `seq`, `vault` (`compose/infrastructure.yml:6,17,29,40,55,69,80,93,104,117`).
   2. Applications — `api-gateway`, `availability-service`, `customers-service`, `deliveries-service`,
      `identity-service`, `operations-service`, `orders-service`, `ordermaker-service`,
      `parcels-service`, `pricing-service`, `vehicles-service`
      (`compose/services.yml:6,17,26,35,44,53,71,80,89,98,107`).
   The other end of the contract is each service's `appsettings.docker.json`, whose Consul, Fabio,
   Seq, Jaeger, Mongo, RabbitMQ, Redis and Vault addresses are exactly these strings — e.g. Consul at
   `http://consul:8500`, Fabio at `http://fabio:9999`, Seq at `http://seq:5341`, Mongo at
   `mongodb://mongo:27017`, RabbitMQ hostname `rabbitmq`, Redis `redis`, Jaeger UDP host `jaeger`.
   Each service additionally sets its own Consul *service address* to its own container name, so the
   application-name set is load-bearing in both directions.
3. **Lifecycle.** Resolved by Docker's embedded DNS on the user-defined network, at every lookup, for
   the lifetime of the container `[compose]`.
4. **Invariants & enforcement.**
   1. For each of the ten backing services: `container_name` **must** equal the hostname the sibling
      repositories use. Enforced by nothing; a mismatch is a runtime connection failure in twelve
      repositories at once.
   2. For each application: `container_name` **must** equal the `consul.address` value in that
      service's `appsettings.docker.json`, or Fabio will route to an address that does not resolve.
   3. Because `container_name` is explicit, it is also the container's network alias; renaming the
      Compose *service* key alone would not break DNS, but renaming `container_name` would.
5. **Extension procedure.** **Never rename a `container_name` in isolation.** The procedure is: change
   `compose/services.yml` and `compose/services-local.yml`, change the matching
   `appsettings.docker.json` in the owning repository, change the Prometheus target in
   `compose/prometheus/prometheus.yml`, change the `depends_on` entry if the service appears in
   `compose/services.yml:59-67`, and change the Fabio-facing configuration of every consumer that
   addresses it. §7.4 is the ordered checklist.
6. **Failure modes.**
   1. A rename applied only here yields a platform where every container starts and every outbound
      call fails DNS resolution — with no aggregated error, because each service logs its own failure
      to its own Seq stream.
   2. Because `container_name` is set, only one instance of each service can exist (§3.43).
   3. In host-networking mode none of these names resolve at all (§3.34).

### 3.12 Host-port allocation map

1. **Definition.** The platform's fixed port plan, split across three files and two modes.
2. **Representation & storage.**

   | Range | Owner | Ports |
   | --- | --- | --- |
   | Applications, compose mode | `compose/services.yml:11,20,29,38,47,56,74,83,92,101,110` | `5000` gateway, `5001`–`5009` services, `5015` ordermaker — all mapped to container port `80` |
   | Applications, PM2 prod | `prod-services.yml:7,13,19,25,31,37,43,49,55,61` | `5000`–`5009` via `ASPNETCORE_URLS` |
   | Applications, PM2 dev | *not here* — each sibling's `launchSettings.json` (§3.31) | `5000`–`5009`, `5015` |
   | Backing services | `compose/infrastructure.yml:11,24-25,34,45-51,63,74,85-87,98,111,127` | 8500 Consul; 9998/9999 Fabio; 3000 Grafana; 5775/udp, 5778, 6831/udp, 6832/udp, 9411, 14268, 16686 Jaeger; 27017 Mongo; 9090 Prometheus; 5672/15672/15692 RabbitMQ; 6379 Redis; 5341 Seq; 8200 Vault |

   Every application mapping is `host:80`, because every service image fixes the listener at port 80
   (`ENV ASPNETCORE_URLS http://*:80` in each sibling `Dockerfile`). Every backing-service mapping is
   identity (`X:X`) **except Seq**, which is `5341:80` (`compose/infrastructure.yml:111`).
3. **Lifecycle.** Bound at container start; released at stop. PM2 ports are bound by Kestrel at
   process start.
4. **Invariants & enforcement.**
   1. The compose port for a service and the PM2 port for the same service must agree, because
      `compose/host-prometheus/prometheus.yml` scrapes the PM2 ports and the base `appsettings.json`
      of each service records its own port as its Consul registration port. They do agree at the base
      ref for all ten PM2 apps.
   2. `5015` is allocated to `ordermaker-service` and is out of the contiguous block — it is the only
      application port above 5009 (§3.33).
   3. No port is bound to `127.0.0.1`; every mapping is on all interfaces `[compose]`.
5. **Extension procedure.** A new service takes the next free port (`5010` onwards; note `5015` is
   taken) and must be added to: both service stacks, both Prometheus configs, both PM2 manifests, and
   its own `appsettings.json`/`launchSettings.json`. §7.1.
6. **Failure modes.**
   1. Running compose mode and PM2 mode simultaneously collides on every port in the 5000–5009 range;
      the second binder fails.
   2. Because all mappings are on all interfaces, a developer machine exposes Mongo (27017), Redis
      (6379), RabbitMQ (5672) and Vault (8200) to the local network with default or dev credentials
      (§3.41, §3.44).
   3. Changing a service's port in `prod-services.yml` without changing
      `compose/host-prometheus/prometheus.yml` silently stops metric collection for that service.

### 3.13 Image reference and tagging policy

1. **Definition.** How the compose stacks name the images they run — and the fact that none of them
   pins a version.
2. **Representation & storage.**
   1. Application images — eleven `devmentors/pacco.<name>` references in `compose/services.yml`
      (lines 5, 16, 25, 34, 43, 52, 70, 79, 88, 97, 106), **all untagged**, therefore `:latest`
      `[compose]`.
   2. Backing-service images — `consul`, `mongo`, `redis`, `vault` (official, untagged),
      `fabiolb/fabio`, `grafana/grafana`, `jaegertracing/all-in-one`, `datalust/seq` (untagged), and
      `rabbitmq:3-management` in `compose/mongo-rabbit-redis.yml:16` — **the only tagged image
      reference in the entire repository**.
   3. Locally built images — `build: ./prometheus`, `build: ./host-prometheus`, `build: ./rabbitmq`.
3. **Lifecycle.** `docker compose up` uses a locally cached image if one exists and pulls otherwise;
   it does **not** re-pull `:latest` on subsequent `up` invocations without `--pull` or an explicit
   `docker compose pull` `[compose]`. Publication of the eleven application images happens entirely
   outside this repository, in each sibling's `scripts/dockerize.sh` (which derives its tag from
   `TRAVIS_BRANCH`: `master` → `latest`, `develop` → `dev`).
4. **Invariants & enforcement.** The implicit invariant is that `:latest` on Docker Hub is the
   `master` build of each sibling. Enforced by the siblings' CI, not by this repository. There is no
   digest, no tag, no lock file and no manifest recording which eleven image versions constitute a
   given platform version.
5. **Extension procedure.** To make the platform reproducible, replace each `image:` value with a
   tag or digest and introduce a variable per service
   (`image: devmentors/pacco.services.vehicles:${VEHICLES_TAG:-latest}`), which Compose resolves from
   the environment or a `.env` file `[compose]`. There is currently **no `.env` file and no variable
   interpolation anywhere in the seven stacks** — verified by absence of `${` in `compose/`.
6. **Failure modes.**
   1. **There is no way to state, or to reproduce, "which Pacco was running".** Two machines running
      the same command on the same day can run different code. This is the component's most serious
      release-engineering gap and is recorded as B-2 (§8.2).
   2. A stale local `:latest` silently pins a developer to an old build until they explicitly pull.
   3. Because the backing-service images are untagged too, a Mongo, Consul or Vault **major** version
      change arrives unannounced on any machine with a cold cache.

### 3.14 Build-context topology of the local stack

1. **Definition.** How `services-local.yml` turns eleven sibling working copies into eleven images.
2. **Representation & storage.** Eleven `build: ../../Pacco.X` keys (`compose/services-local.yml:5,
   16, 25, 34, 43, 52, 70, 79, 88, 97, 106`). The short form means: context = that directory,
   Dockerfile = `<context>/Dockerfile` `[compose]`. Each sibling repository provides exactly that file
   at its root (e.g. `hianshul100_Pacco.Services.Availability/Dockerfile`), which restores the NuGet
   packages, publishes to `/app/out` and runs on `mcr.microsoft.com/dotnet/core/aspnet:3.1` with
   `ASPNETCORE_URLS` and `ASPNETCORE_ENVIRONMENT` baked in.
3. **Lifecycle.** Built on first `up`, or on `docker compose -f compose/services-local.yml build`.
   **Not rebuilt on subsequent `up`** unless the image is absent or `--build` is passed `[compose]` —
   so a developer who edits a service and re-runs `up` gets the previous build.
4. **Invariants & enforcement.** Each `../../Pacco.X` must resolve from `compose/` (§3.2), and each
   target repository must have a root `Dockerfile`. Enforced only by build failure.
5. **Extension procedure.** A new service adds one `build:` entry here and one `image:` entry in
   `compose/services.yml`; the two files must otherwise stay line-for-line equivalent — at the base ref
   their only differences are the eleven `image`/`build` lines and the one `NTRADA_CONFIG` value
   (§3.16), which makes any third difference a review signal.
6. **Failure modes.**
   1. Full builds of eleven .NET images with no shared cache layer beyond what Docker infers; the
      first `up` is very slow, and there is no `--parallel` guidance in `README.md`.
   2. Because the built images are unnamed here, Compose names them
      `<project>_<service>` `[compose]`; they do **not** overwrite the `devmentors/*` images, so
      switching back to `services.yml` silently switches back to whatever was last pulled.

### 3.15 Start ordering — the single `depends_on` block

1. **Definition.** The platform's only declared ordering constraint.
2. **Representation & storage.** `compose/services.yml:59-67` (and identically
   `compose/services-local.yml:59-67`): **`operations-service`** declares `depends_on` over eight
   application services — availability, customers, deliveries, identity, orders, ordermaker, parcels,
   vehicles. Notably it does **not** list `pricing-service`, and no service depends on `api-gateway`.
   No other service in either stack has a `depends_on` key, and **no stack declares a dependency on
   any backing service** — nothing waits for Mongo, RabbitMQ, Redis or Consul.
3. **Lifecycle.** Evaluated once, at `up`, to order container *creation and start*. In the short
   (v2/v3 list) form it does **not** wait for readiness — only for the container to have been started
   `[compose]`.
4. **Invariants & enforcement.** The intent is presumably that Operations, which consumes events from
   every other service, starts last. The mechanism cannot deliver that intent: a started container is
   not a ready service, and the real dependency (RabbitMQ) is not listed at all.
5. **Extension procedure.** If ordering genuinely matters, the correct mechanism is
   `depends_on: <service>: condition: service_healthy` together with a `healthcheck` on the depended-on
   service `[compose]` — neither exists today (§3.42). If it does not matter (the likelier reading,
   since every service reconnects to RabbitMQ on its own), the block should be **deleted**, because it
   currently implies a guarantee that does not hold. This is Q-3 (§8.3).
6. **Failure modes.**
   1. Operations starts after eight containers have been *created*, none of which need be listening —
      so the ordering buys nothing and costs a false sense of safety.
   2. The list is a hand-maintained roster that omits `pricing-service`; adding a twelfth service will
      not update it, and nothing will notice.
   3. **Cross-stack ordering is unexpressible**: `depends_on` cannot reference a service in a different
      Compose file, so the application stack can never wait for the infrastructure stack. The operator
      is the only sequencing mechanism the platform has (§4.1).

### 3.16 Gateway mode selection via `NTRADA_CONFIG`

1. **Definition.** The single application-level configuration value this repository sets — and it is
   set to **two different things** in the two service stacks.
2. **Representation & storage.**
   1. `compose/services.yml:9` sets it to the asynchronous gateway configuration file name (the
      `ntrada-async.docker` variant, with a `.yml` suffix).
   2. `compose/services-local.yml:9` sets it to the synchronous one (`ntrada.docker`, **without** a
      suffix).
   The consuming end is `hianshul100_Pacco.APIGateway`, whose `Dockerfile` bakes a default of
   `ntrada.docker`; the resolution of the name to a file — and whether the `.yml` suffix is required,
   optional or fatal — belongs to Ntrada and is modelled in `component-internals/api-gateway.md` §3.5.
   **`Unverifiable — Missing Source Evidence`** within this repository: Ntrada's file-name resolution
   rule is not derivable here, so it cannot be settled from this component whether the two spellings
   differ only in suffix convention or in effect.
3. **Lifecycle.** Read by the gateway process at start-up; changing it requires recreating the
   container.
4. **Invariants & enforcement.** There is no invariant — the two stacks deliberately or accidentally
   diverge, and nothing reconciles them.
5. **Extension procedure.** Decide which mode is canonical, set both stacks to it, and record the
   other as a documented opt-in override. Until then, treat the gateway's request/response semantics
   as **stack-dependent**.
6. **Failure modes.**
   1. **The published-image stack and the local-build stack expose different gateway behaviour.** A
      developer verifying a change against `services-local.yml` (synchronous — the request waits for a
      result) and then deploying against `services.yml` (asynchronous — the request is accepted and a
      correlation identity is returned) sees different responses for identical requests. This is a
      real, reproducible behavioural divergence between two files that differ, otherwise, only in
      where their images come from. Recorded as B-3 (§8.2).
   2. A typo in the value is not detected here; the gateway either falls back to its baked default or
      fails at start-up, depending on Ntrada's resolution rule (unverifiable, above).

### 3.17 Container supervision (`restart: unless-stopped`)

1. **Definition.** The platform's entire failure-recovery policy in compose mode.
2. **Representation & storage.** `restart: unless-stopped` on **every** service in all seven stacks —
   ten in `compose/infrastructure.yml`, ten in `compose/host-infrastructure.yml`, three in
   `compose/mongo-rabbit-redis.yml`, three in `compose/consul-fabio-vault.yml`, four in
   `compose/grafana-seq-jaeger-prometheus.yml`, eleven in each service stack. There is no variation
   and no `restart` policy on any other setting.
3. **Lifecycle.** Enforced by the Docker daemon: a container that exits for any reason is restarted,
   including across daemon restarts, unless it was explicitly stopped `[compose]`.
4. **Invariants & enforcement.** Uniformity is the invariant, and it holds exactly at the base ref.
5. **Extension procedure.** Any new service should carry the same key; if a one-shot initialisation
   container is ever added (there is none today), it must use `restart: "no"` or it will loop forever.
6. **Failure modes.**
   1. A container that crash-loops on a configuration error restarts indefinitely with no backoff cap
      and no alert — the only signal is the container's own log.
   2. `unless-stopped` survives a host reboot, so a machine that once ran Pacco keeps running it; there
      is no `docker compose down` in `README.md`, so an operator following the documentation never
      learns how to stop the platform.

### 3.18 The environment-variable injection surface

1. **Definition.** The complete set of environment variables this repository injects — five distinct
   keys across all seven stacks.
2. **Representation & storage.**

   | Variable | Set on | Where | Purpose |
   | --- | --- | --- | --- |
   | `NTRADA_CONFIG` | `api-gateway` | `compose/services.yml:9`, `compose/services-local.yml:9` | gateway mode (§3.16) |
   | `FABIO_REGISTRY_CONSUL_ADDR` | `fabio` | `compose/infrastructure.yml:20`, `compose/consul-fabio-vault.yml:20`, `compose/host-infrastructure.yml:17` | where Fabio finds Consul (§3.20) |
   | `ACCEPT_EULA` | `seq` | `compose/infrastructure.yml:107`, `compose/grafana-seq-jaeger-prometheus.yml:46`, `compose/host-infrastructure.yml:74` | Seq licence acceptance |
   | `VAULT_ADDR`, `VAULT_DEV_ROOT_TOKEN_ID` | `vault` | `compose/infrastructure.yml:120-121`, `compose/consul-fabio-vault.yml:36-37`, `compose/host-infrastructure.yml:84-85` | dev-mode Vault (§3.27) |

   Two further pairs exist **commented out**: `MONGO_INITDB_ROOT_USERNAME` /
   `MONGO_INITDB_ROOT_PASSWORD` at `compose/infrastructure.yml:57-59` and
   `compose/host-infrastructure.yml:38-40`. Their presence in comment form is the evidence that
   **MongoDB runs with authentication disabled** (§3.21, §3.41).
3. **Lifecycle.** Injected at container creation; changing one requires recreating the container.
4. **Invariants & enforcement.** No variable is templated, defaulted or read from a `.env` file —
   every value is a literal. There is therefore no per-environment variation mechanism at all.
5. **Extension procedure.** Adding a per-environment value means introducing `${VAR}` interpolation
   plus a `.env` convention, and documenting it in `README.md`; today an operator's only lever is to
   edit the tracked YAML, which shows up as a repository modification.
6. **Failure modes.**
   1. Because there is no interpolation, every environment-specific change is a source edit, which
      developers routinely make and never commit — the running platform drifts from the repository
      invisibly.
   2. The Vault dev token is a literal in three tracked files (§3.41).

### 3.19 Consul container

1. **Definition.** The service registry. It holds the platform's only dynamic map from logical service
   name to address.
2. **Representation & storage.** `compose/infrastructure.yml:4-13` — image `consul`, container name
   `consul`, port `8500:8500`, on `pacco`. The volume that would persist `/consul/data` is present but
   **commented out** (lines 12-13). Repeated verbatim at `compose/consul-fabio-vault.yml:4-13` and,
   with `network_mode: host` and no ports, at `compose/host-infrastructure.yml:4-10`.
3. **Lifecycle.** Starts with no arguments, so the image's default entrypoint decides whether it runs
   as a dev-mode single-node agent — `[image]`, not determinable here. Registrations arrive from the
   services themselves at their start-up and are removed on deregistration or health-check failure
   `[convey]`.
4. **Invariants & enforcement.**
   1. The container name `consul` must match the `consul.url` host in every
      `appsettings.docker.json` (§3.11) and the Fabio environment variable (§3.20).
   2. Port 8500 must match the port in those same files.
   3. **All registry state is in-memory**: with the volume commented out, a container restart empties
      the registry. Services re-register on their own start-up, so a Consul restart *without* a service
      restart leaves the registry empty and Fabio routing to nothing until the services next register.
5. **Extension procedure.** To persist the registry, uncomment lines 12-13 **and** the matching
   `consul:` volume at `compose/infrastructure.yml:134-135` — the two are separate edits in the same
   file and both are required. To seed KV configuration, add a volume mount or an init container;
   neither exists today.
6. **Failure modes.**
   1. Registry loss on restart, as above — silent, and it looks like a routing bug.
   2. Consul is published on all interfaces with no ACL configuration and no TLS (§3.44).
   3. Nothing in this repository verifies that Consul is up before the services start (§3.15), and a
      service that fails to register does so with a message in its own log only.

### 3.20 Fabio container and its Consul binding

1. **Definition.** The registry-driven HTTP router that sits between services for their internal
   HTTP calls.
2. **Representation & storage.** `compose/infrastructure.yml:15-25` — image `fabiolb/fabio`, container
   name `fabio`, environment `FABIO_REGISTRY_CONSUL_ADDR` pointing at `consul:8500` (line 20), ports
   `9998:9998` (admin UI) and `9999:9999` (proxy). Duplicated at
   `compose/consul-fabio-vault.yml:15-25`. In `compose/host-infrastructure.yml:12-18` the same variable
   points at `localhost:8500` instead — the one place where the host variant is genuinely *adapted*
   rather than merely re-networked.
3. **Lifecycle.** Fabio watches Consul continuously and rebuilds its routing table from service tags;
   the tags are set by each service, not here `[convey]`.
4. **Invariants & enforcement.**
   1. `FABIO_REGISTRY_CONSUL_ADDR` must name the Consul container and port — it does, in all three
      declarations.
   2. Every service's `appsettings.docker.json` addresses Fabio at `http://fabio:9999`, so the
      container name and the **proxy** port (not the admin port) are load-bearing.
   3. Fabio holds **no route configuration of its own** in this repository — no routes file, no
      `-proxy.addr`, no TLS. The routing table is entirely derived from Consul tags.
5. **Extension procedure.** Routing changes belong in the *service* repositories (their Fabio/Consul
   tags), never here. The only changes that belong here are the Consul address and the published
   ports.
6. **Failure modes.**
   1. If Consul's registry is empty (§3.19), Fabio returns 404 for every route while appearing
      perfectly healthy on 9998.
   2. The admin interface on 9998 is published on all interfaces with no authentication `[image]`.

### 3.21 MongoDB container

1. **Definition.** The platform's primary datastore — one shared MongoDB instance for every service
   that persists documents.
2. **Representation & storage.** `compose/infrastructure.yml:53-65` — image `mongo`, container name
   `mongo`, port `27017:27017`, and a named volume `mongo:/data/db` (lines 64-65) backed by the
   `mongo` volume declaration at lines 138-139. The root-credential environment variables are present
   but **commented out** (lines 57-59). Identical service definition at
   `compose/mongo-rabbit-redis.yml:4-13`; host variant at `compose/host-infrastructure.yml:34-43`.
3. **Lifecycle.** The volume is created on first `up` and survives `docker compose down`; it is removed
   only by `down -v` or an explicit `docker volume rm` `[compose]`.
4. **Invariants & enforcement.**
   1. **Authentication is disabled.** Because `MONGO_INITDB_ROOT_USERNAME`/`_PASSWORD` are commented
      out, the image initialises without a root user, and the connection strings in every
      `appsettings.docker.json` carry no credentials — the two ends agree, which is precisely why the
      insecure state is stable and invisible.
   2. One instance, one port, one volume: there is no replica set, so **MongoDB transactions and
      change streams are unavailable** in this topology.
   3. Database *names* are chosen per service in the sibling repositories, so a single `mongo` process
      hosts every service's database; isolation is by database name only.
5. **Extension procedure.** To enable authentication: uncomment lines 57-59 with values sourced from
   an interpolated variable rather than literals (§3.18), then update the connection string in **every**
   service's `appsettings.docker.json` — a twelve-repository change. To enable a replica set the
   container needs a `command` with `--replSet` and a one-time initiation step; neither exists.
6. **Failure modes.**
   1. Port 27017 with no authentication is published on all interfaces (§3.44).
   2. The volume is named `mongo` within the Compose *project*, so a bring-up from a differently named
      directory creates a **different** volume and appears to lose all data `[compose]`.
   3. `compose/mongo-rabbit-redis.yml:4-13` omits the commented credential block entirely, so an
      operator who enables authentication in `infrastructure.yml` and later starts the split stack
      silently reverts to an unauthenticated Mongo.

### 3.22 RabbitMQ container and the custom plugin image

1. **Definition.** The message broker — the backbone of the platform's event-driven communication —
   and the one backing service whose image this repository builds rather than pulls.
2. **Representation & storage.**
   1. `compose/infrastructure.yml:78-89` — `build: ./rabbitmq`, container name `rabbitmq`, ports
      `5672` (AMQP), `15672` (management UI), `15692` (Prometheus). Volume commented out (lines 88-89).
   2. `compose/rabbitmq/Dockerfile` (2 lines) — `FROM rabbitmq:management`, then a copy of `./plugins`
      to `/etc/rabbitmq/enabled_plugins`.
   3. `compose/rabbitmq/plugins` (1 line, no trailing newline) — an Erlang term list enabling the
      management plugin **and** the Prometheus plugin. This single file is the entire reason the
      custom image exists: the stock `rabbitmq:management` image does not expose 15692.
3. **Lifecycle.** Built once on first `up`; the enabled-plugins file is read by the broker at start.
   All broker state (exchanges, queues, bindings, messages) is in the container's writable layer
   because the volume is commented out — **a `docker compose down` destroys every queue and every
   unconsumed message**.
4. **Invariants & enforcement.**
   1. The plugin file must enable `rabbitmq_prometheus`, or the `rabbitmq` scrape job
      (`compose/prometheus/prometheus.yml:54-56`) is permanently down. Enforced by nothing.
   2. Port 15692 must be published for the host-mode scrape to work; in bridge mode Prometheus reaches
      it on the shared network regardless.
   3. Credentials are the image defaults (§3.41); every service's base `appsettings.json` uses the
      same default user, password and vhost.
5. **Extension procedure.** To add a plugin, extend the single-line Erlang term in
   `compose/rabbitmq/plugins` — the file must remain a valid Erlang term list ending in a period. To
   persist broker state, uncomment `compose/infrastructure.yml:88-89` and add a `rabbitmq:` volume at
   lines 142-143.
6. **Failure modes.**
   1. **`compose/mongo-rabbit-redis.yml:16` uses stock `rabbitmq:3-management` and publishes only 5672
      and 15672.** An operator who chooses the split stack therefore loses the Prometheus plugin and
      the 15692 port — and the RabbitMQ scrape job fails silently. Two files in the same directory
      declare "RabbitMQ" and mean two materially different brokers. Recorded as B-4 (§8.2).
   2. Because state is ephemeral, an outbox or a durable subscription that survives a broker restart
      in production does not survive one here — a class of bug that reproduces only locally.
   3. `rabbitmq:management` is an unpinned tag; a major broker version can arrive on a cold cache
      (§3.13).

### 3.23 Redis container

1. **Definition.** The shared cache.
2. **Representation & storage.** `compose/infrastructure.yml:91-100` — image `redis`, container name
   `redis`, port `6379:6379`, volume `redis:/data` (lines 99-100) declared at lines 144-145.
   Identical at `compose/mongo-rabbit-redis.yml:27-36`; host variant at
   `compose/host-infrastructure.yml:61-67`.
3. **Lifecycle.** Started with no `command`, so persistence behaviour is the image's default `[image]`;
   the `/data` volume exists either way and survives `down`.
4. **Invariants & enforcement.** No password (`requirepass` is never set) and no `command` override;
   the container name and port must match the services' `redis` configuration (§3.11). Redis is one of
   only **two** backing services with a live volume (§3.36).
5. **Extension procedure.** To require a password, add a `command` with `--requirepass` and update
   every service's Redis connection string; there is no per-service Redis database separation today.
6. **Failure modes.**
   1. An unauthenticated Redis published on all interfaces (§3.44).
   2. A live volume for a *cache* while the broker and the registry have none — the durability choices
      are inverted relative to the value of the data (§3.36).

### 3.24 Prometheus container and its custom image

1. **Definition.** The metrics collector, and the second image this repository builds.
2. **Representation & storage.**
   1. `compose/infrastructure.yml:67-76` — `build: ./prometheus`, container name `prometheus`, port
      `9090:9090`; TSDB volume commented out (lines 75-76). Repeated at
      `compose/grafana-seq-jaeger-prometheus.yml:15-24` (with the port quoted, the only quoted port
      value in the repository — cosmetic).
   2. `compose/prometheus/Dockerfile` (3 lines) — `FROM prom/prometheus`, a `WORKDIR /app` that is
      **inert** (the copy that follows uses an absolute destination and the image's entrypoint is
      unaffected `[image]`), and a copy of the adjacent `prometheus.yml` to
      `/etc/prometheus/prometheus.yml` — the path the upstream image reads by default.
3. **Lifecycle.** **The scrape configuration is baked into the image, not mounted.** Changing a scrape
   target therefore requires a `docker compose build` (or `up --build`), not a restart — a genuinely
   surprising property, and the most common way for a Prometheus edit in this repository to appear to
   have no effect.
4. **Invariants & enforcement.** The copy destination must remain `/etc/prometheus/prometheus.yml`;
   the job/target list must track the service roster (§3.25). Nothing enforces either.
5. **Extension procedure.** Prefer converting the build to a bind mount
   (`volumes: ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml`) so that edits take effect on
   restart; that change would also let both stacks share one Prometheus definition. If the build is
   kept, document the rebuild requirement in `README.md`.
6. **Failure modes.**
   1. An engineer edits `compose/prometheus/prometheus.yml`, runs `docker compose up -d`, and observes
      no change, because the image is cached (§3.13, §3.14).
   2. With the TSDB volume commented out, **all metric history is lost on every container recreation**
      — so Prometheus is usable for live inspection only, and any Grafana dashboard showing a
      multi-day range is empty after a restart.
   3. There is no `rule_files` and no alerting configuration in either scrape config: **the platform
      collects metrics but cannot alert on them** (§3.44).

### 3.25 The Prometheus scrape model

1. **Definition.** The scrape configuration is the closest thing the repository has to a machine-
   readable roster of the platform, and it is worth reading as such.
2. **Representation & storage.** `compose/prometheus/prometheus.yml` — `global.scrape_interval` and
   `global.evaluation_interval` both 5s (lines 2-3), then **thirteen jobs** (lines 6-56): `prometheus`
   itself at `prometheus:9090`, the eleven applications, and `rabbitmq` at `rabbitmq:15692`.
   1. The eleven application targets are **bare container names with no port** (lines 12, 16, 20, 24,
      28, 32, 36, 40, 44, 48, 52), so Prometheus applies the scheme default — port 80, which is
      exactly the port every service image listens on (§3.12). The omission is correct, but only by
      coincidence of the default matching.
   2. **No `metrics_path` is set on any job**, so every job uses the default `/metrics`. The metrics
      endpoint on the service side is enabled per service in its own `appsettings.json` `[convey]`.
   3. The job names match the container names exactly for all eleven applications, so the `job` label
      in every metric equals the container name — a property the host variant breaks (§3.26).
3. **Lifecycle.** Baked into the image (§3.24); reloaded only by rebuilding.
4. **Invariants & enforcement.**
   1. The job list must equal the application roster. It does, at the base ref: all eleven services in
      `compose/services.yml` appear, including `ordermaker-service` (lines 38-40).
   2. The 5s interval is aggressive for thirteen targets but harmless at this scale; the inline
      comments on lines 2-3 still say "15 seconds", i.e. **the comments contradict the values they
      annotate** — cosmetic, but a reader-trap.
5. **Extension procedure.** A new service needs one three-line job here **and** one in
   `compose/host-prometheus/prometheus.yml` with a `localhost:<port>` target. Because this file is the
   only place that enumerates all eleven applications *and* the broker, §7.2 nominates it as the
   canonical roster to check a change against.
6. **Failure modes.**
   1. A service whose metrics are disabled in its own configuration appears as a permanently-down
      target with no explanation on this side.
   2. `ordermaker-service` is scraped here but is absent from both PM2 manifests (§3.33), so in PM2
      mode the host variant's `localhost:5015` job can never come up.
   3. Adding a service to the compose stack and forgetting the scrape job produces a service that is
      invisible to monitoring while looking entirely healthy.

### 3.26 Host-networking Prometheus variant

1. **Definition.** A parallel scrape configuration for the mode where services run as host processes.
2. **Representation & storage.** `compose/host-prometheus/prometheus.yml` — same globals and same
   thirteen jobs, but every target is `localhost:<port>`: 9090 for Prometheus itself, 5000 for the
   gateway, 5001–5009 for the services in the same order as the PM2 manifests, 5015 for ordermaker,
   and 15692 for RabbitMQ. `compose/host-prometheus/Dockerfile` is byte-identical in structure to the
   bridge variant.
3. **Lifecycle.** Built by `compose/host-infrastructure.yml:45-51` (`build: ./host-prometheus`).
4. **Invariants & enforcement.** The port list must match `prod-services.yml` (it does) and the
   `launchSettings.json` ports of the eleven siblings (it does at the base ref).
5. **Extension procedure.** Keep the two files in lockstep: any job added to one must be added to the
   other with the corresponding address form.
6. **Failure modes.**
   1. **The gateway's job is named `api` here (line 10) and `api-gateway` in the bridge variant
      (line 10).** The `job` label therefore changes with the deployment mode, so any Grafana
      dashboard, recording rule or alert that selects `job="api-gateway"` silently returns no data in
      host mode. A one-word divergence with query-level consequences; recorded as B-5 (§8.2).
   2. The two files are 56 lines of near-duplicate configuration maintained by hand, with no generator
      and no test — the class of drift in point 1 is expected to recur.

### 3.27 Vault: dev-mode container versus the production runbook

1. **Definition.** The platform declares a secrets manager twice — as a throwaway container and as a
   documented production system — and uses neither.
2. **Representation & storage.**
   1. `compose/infrastructure.yml:115-127` — image `vault`, container name `vault`, port `8200:8200`,
      `cap_add: IPC_LOCK`, and two environment variables: a self-referential `VAULT_ADDR` and
      `VAULT_DEV_ROOT_TOKEN_ID` set to the literal placeholder `secret`. Setting the dev root token
      variable is what puts the image into **dev mode**: in-memory storage, auto-unsealed, one
      well-known root token. Duplicated at `compose/consul-fabio-vault.yml:27-39` and
      `compose/host-infrastructure.yml:79-88`.
   2. `docker-images.txt:204-321` — a genuinely different system: a `vault server` with a Consul
      storage backend and TLS disabled (line 209), followed by an operator procedure for
      initialisation, unsealing, root login, enabling `userpass`, authoring a `services` policy,
      creating a non-root account, upgrading the KV store from v1 to v2, and then the database
      (dynamic MongoDB credentials, lines 274-290) and PKI (lines 291-321) secret engines.
3. **Lifecycle.** The dev container is ready the moment it starts and loses everything when it stops.
   The production procedure is manual, one-off, and produces unseal shares that must be kept off the
   machine.
4. **Invariants & enforcement.**
   1. **No service in the compose topology uses Vault.** Every service's `appsettings.docker.json`
      sets its Vault section to disabled. The container therefore runs, publishes 8200, and serves
      nobody — verified across the sibling repositories, and the single most surprising fact about the
      observability/secrets estate.
   2. The dev token literal is identical in three tracked files and in the runbook (line 197); it is
      quoted here **because being an unmodified upstream default is the evidence** (folder convention,
      `component-internals/index.md` §3.2). No real secret value is reproduced anywhere in this
      document.
   3. The runbook's production Vault talks to Consul over a Docker-Desktop-specific hostname (line
      209) and disables TLS — it is a *demonstration* of a production topology, not a production
      configuration.
5. **Extension procedure.** To actually adopt Vault: enable the Vault section in each service's
   configuration, replace the dev container with a server configuration and a real storage backend,
   move the unseal procedure out of the repository, and provide credentials by an authentication
   method rather than a root token. That is a twelve-repository change plus an operational process,
   and it is Q-4 (§8.3).
6. **Failure modes.**
   1. A reader concludes the platform manages secrets with Vault; it does not — secrets are literals
      in tracked configuration (§3.41).
   2. Dev-mode Vault loses all data on restart, so anything an operator stores while experimenting is
      gone after a `down`.
   3. `docker-images.txt:219-232` records the **actual output of a real `vault operator init` run** —
      unseal key shares and an initial root token — committed to the repository. Cited by path and
      line only; the values are not reproduced here. See §3.41 and B-6 (§8.2).

### 3.28 Grafana container and the absence of provisioning

1. **Definition.** The metrics UI.
2. **Representation & storage.** `compose/infrastructure.yml:27-36` — image `grafana/grafana`,
   container name `grafana`, port `3000:3000`, volume commented out (lines 35-36). Repeated at
   `compose/grafana-seq-jaeger-prometheus.yml:4-13` and `compose/host-infrastructure.yml:20-26`.
3. **Lifecycle.** Starts empty. With the volume commented out, its SQLite database lives in the
   container layer.
4. **Invariants & enforcement.** **There is no provisioning of any kind** — no datasource file, no
   dashboard JSON, no `GF_*` environment variable, nowhere in the repository. Consequently:
   1. The Prometheus datasource must be added by hand, in the UI, after every recreation.
   2. Every dashboard is hand-built and lost with the container.
   3. The admin credentials are the image defaults `[image]`.
5. **Extension procedure.** Provisioning is the obvious improvement: a `grafana/provisioning/`
   directory with a datasource pointing at `http://prometheus:9090` and dashboards as JSON, mounted
   into `/etc/grafana/provisioning`. This would also give the platform a place to record which metrics
   matter — which nothing currently does.
6. **Failure modes.**
   1. The platform ships a metrics UI that displays nothing until a human configures it, and forgets
      that configuration on every `down`. Grafana is, at the base ref, decorative.
   2. Uncommenting the volume (lines 35-36) also requires uncommenting the `grafana:` volume
      declaration (lines 136-137) — the same two-place edit as §3.19.

### 3.29 Jaeger all-in-one container

1. **Definition.** Distributed tracing collector and UI in a single container.
2. **Representation & storage.** `compose/infrastructure.yml:38-51` — image
   `jaegertracing/all-in-one`, container name `jaeger`, seven published ports: 5775/udp, 5778,
   6831/udp, 6832/udp, 9411, 14268, 16686. Duplicated at
   `compose/grafana-seq-jaeger-prometheus.yml:26-39` and `compose/host-infrastructure.yml:28-32`.
   The services send spans over UDP to host `jaeger` on the Thrift-compact agent port `[convey]`.
3. **Lifecycle.** All-in-one keeps spans **in memory**; there is no storage backend, no volume and no
   retention setting.
4. **Invariants & enforcement.**
   1. Container name `jaeger` must match the services' Jaeger UDP host (§3.11).
   2. The runbook's equivalent command (`docker-images.txt:461`) sets `COLLECTOR_ZIPKIN_HTTP_PORT`,
      enabling the Zipkin-compatible collector on 9411. **The compose definitions publish 9411 but do
      not set that variable** — so the port is exposed while the collector that would listen on it is
      not enabled by this configuration. Whether the image enables it by default is `[image]` and
      **`Unverifiable — Missing Source Evidence`** here.
   3. There is no sampling configuration on this side; sampling is per service `[convey]`.
5. **Extension procedure.** For anything beyond local inspection, replace all-in-one with separate
   collector/query/storage services; today the correct change is to *stop publishing* 9411 or to set
   the variable, so that the declaration matches reality.
6. **Failure modes.**
   1. In-memory spans mean trace history disappears on restart and is bounded by memory; a long-running
      instance can accumulate significant memory usage with no configured limit (§3.44).
   2. Seven published ports on all interfaces, of which the platform demonstrably uses two (the UDP
      agent port and 16686 for the UI).

### 3.30 Seq container and its port inversion

1. **Definition.** The structured-log sink every service writes to.
2. **Representation & storage.** `compose/infrastructure.yml:102-113` — image `datalust/seq`,
   container name `seq`, one environment variable accepting the licence, port mapping **`5341:80`**
   (line 111), volume commented out (lines 112-113). Duplicated at
   `compose/grafana-seq-jaeger-prometheus.yml:41-52`; host variant at
   `compose/host-infrastructure.yml:69-77`.
3. **Lifecycle.** Log events arrive over HTTP from every service; all storage is in the container layer
   because the volume is commented out.
4. **Invariants & enforcement.**
   1. Every service's `appsettings.docker.json` targets `http://seq:5341` — an address **inside** the
      network, where the published mapping does not apply. For that to work, the Seq image's ingestion
      listener must itself be reachable on 5341 in-container. The image's internal port assignment is
      `[image]` and **`Unverifiable — Missing Source Evidence`** from this repository; what *is*
      verifiable here is that `5341:80` maps host 5341 to container **80**, and that the runbook's
      standalone command (`docker-images.txt:391`) does exactly the same thing — so the inversion is
      deliberate and long-standing, not a typo.
   2. The consequence in host mode is unambiguous and is a real defect: see §3.34.
5. **Extension procedure.** Persisting log history requires uncommenting lines 112-113 plus the `seq:`
   volume at lines 146-147. If the in-container port is ever confirmed to be 80, the service
   configurations in the twelve sibling repositories are the files to change, not this one.
6. **Failure modes.**
   1. All log history is lost on container recreation — the platform's logs, traces and metrics are
      **all three** ephemeral at the base ref (§3.36).
   2. Seq is published unauthenticated on all interfaces, and it holds the full structured log of every
      service, including whatever those logs contain (§3.44).

### 3.31 PM2 development manifest and its implicit `launchSettings.json` dependency

1. **Definition.** The manifest that runs the platform as ten host processes, from source, without
   containers.
2. **Representation & storage.** `services.yml` (repository root, 41 lines) — an `apps:` list of ten
   entries, each with exactly three keys: `name` (`api`, `availability`, `customers`, `deliveries`,
   `identity`, `operations`, `orders`, `parcels`, `pricing`, `vehicles`), `script: dotnet run`, and
   `cwd: ../Pacco.X/src/Pacco.X.Api` (the gateway's is `../Pacco.APIGateway/src/Pacco.APIGateway`),
   plus `max_restarts: 3`. **There is no `env:` block and no port anywhere in this file.**
3. **Lifecycle.** PM2 starts each app by running `dotnet run` in the given working directory. Because
   `dotnet run` honours `Properties/launchSettings.json`, the profile named after the project (with
   `commandName: Project`) supplies both the listening address and the environment — for example
   `hianshul100_Pacco.Services.Vehicles/src/Pacco.Services.Vehicles.Api/Properties/launchSettings.json:17-24`
   sets `applicationUrl` to `http://localhost:5009` and `ASPNETCORE_ENVIRONMENT` to `local`.
4. **Invariants & enforcement.**
   1. **This repository's development mode is configured by files in twelve other repositories that
      this manifest never mentions.** The port allocation, the environment name and therefore which
      `appsettings.<env>.json` is layered on top all come from `launchSettings.json`. Nothing here
      records that dependency; it is discoverable only by knowing how `dotnet run` behaves.
   2. The `local` environment those files select disables Consul, Fabio, Jaeger, metrics, the outbox,
      Vault and Seq in the service configurations — so **PM2 development mode needs only Mongo,
      RabbitMQ and Redis**, which is exactly the estate `compose/mongo-rabbit-redis.yml` provides
      (§3.35). The pairing is deliberate, and it is not documented anywhere in this repository.
   3. `cwd` is relative, so the manifest must be launched from the repository root (§3.2).
   4. `max_restarts: 3` is uniform across all ten apps; a service that fails four times stays down and
      only `pm2 status` reveals it.
5. **Extension procedure.** Adding a service means one four-line entry here **and** a
   `launchSettings.json` profile in the new repository with a free port; forgetting the latter yields a
   process that binds ASP.NET Core's default port and collides.
6. **Failure modes.**
   1. **`.gitignore:55` ignores `**/Properties/launchSettings.json`.** The sibling repositories track
      theirs anyway, but the platform repository's own ignore rules declare the very files its
      development mode depends on to be untracked artefacts. A contributor who follows this
      repository's convention in a new service produces a service that PM2 cannot start correctly.
      Recorded as B-7 (§8.2).
   2. `dotnet run` builds on each start, so a PM2 restart of ten apps is a ten-project build.
   3. `ordermaker` is missing (§3.33).

### 3.32 PM2 production manifest, the publish-path literal and the missing environment

1. **Definition.** The manifest that runs the platform as ten host processes from published output.
2. **Representation & storage.** `prod-services.yml` (61 lines) — the same ten app names in the same
   order, but with `script: dotnet <Assembly>.dll`, `cwd:
   ../Pacco.X/src/Pacco.X.Api/bin/release/netcoreapp3.1/publish`, `max_restarts: 3`, and an explicit
   `env` block setting `ASPNETCORE_URLS` to `http://*:5000` … `http://*:5009` (lines 7, 13, 19, 25, 31,
   37, 43, 49, 55, 61).
3. **Lifecycle.** Assumes a prior `dotnet publish -c release` in each repository; PM2 then runs the
   framework-dependent assembly directly.
4. **Invariants & enforcement.**
   1. The path segment `netcoreapp3.1` is a **literal target-framework moniker**. Retargeting any
      service to a newer framework silently breaks its `cwd`, and PM2 reports only that the script was
      not found.
   2. The path segment `release` is lower-case; on a case-sensitive filesystem it must match the
      publish output exactly, which depends on how `dotnet publish -c` was invoked.
   3. **No `ASPNETCORE_ENVIRONMENT` is set on any of the ten apps.** Production therefore runs the
      **base** `appsettings.json` profile — the one whose Consul address is `docker.for.win.localhost`
      (`localhost` for OrderMaker), whose RabbitMQ credentials are the image defaults, and which has
      no environment-specific override at all. The manifest named "prod" runs the least
      production-ready configuration in the platform. This is the same defect recorded per service as
      B-2 in `component-internals/pricing-service.md` and B-3 in
      `component-internals/vehicles-service.md`; **this file is where it originates**, and it is
      recorded here as B-8 (§8.2).
   4. `ASPNETCORE_URLS` uses `http://*:<port>`, binding all interfaces — appropriate for a server,
      and different from the development mode's `http://localhost:<port>`.
5. **Extension procedure.** The minimal correction is to add `ASPNETCORE_ENVIRONMENT` to each `env`
   block (and to create the corresponding profile in each service repository). Replacing the literal
   `netcoreapp3.1` with a publish directory that does not encode the framework — or publishing to a
   fixed `out/` — removes the retargeting trap.
6. **Failure modes.**
   1. A framework upgrade in any one service breaks only that service's entry, and only at start-up.
   2. Because there is no environment, a service that relies on a `production` profile existing gets
      base-profile values pointing at Docker-Desktop hostnames that do not resolve on a server.
   3. `ordermaker` is missing here too (§3.33).

### 3.33 OrderMaker's asymmetric membership

1. **Definition.** One of the eleven applications is present in some topologies and absent from
   others — the clearest single instance of the "four topologies, no reconciliation" problem (§3.1).
2. **Representation & storage.**

   | Topology | `ordermaker` present? | Evidence |
   | --- | --- | --- |
   | `compose/services.yml` | yes, port 5015 | lines 78-85 |
   | `compose/services-local.yml` | yes, port 5015 | lines 78-85 |
   | `compose/services.yml` `depends_on` | yes | line 65 |
   | `compose/prometheus/prometheus.yml` | yes | lines 38-40 |
   | `compose/host-prometheus/prometheus.yml` | yes, `localhost:5015` | lines 38-40 |
   | `services.yml` (PM2 dev) | **no** | ten apps, lines 1-41 |
   | `prod-services.yml` (PM2 prod) | **no** | ten apps, lines 1-61 |
   | `README.md` clone list | yes | line 40 |
   | `Pacco.sln` | yes | project reference present |
   | all five scripts | yes | line 2 of each |

3. **Lifecycle.** OrderMaker was added to the platform after the PM2 manifests were written; its port,
   5015, sits outside the contiguous 5000–5009 block, which is the fingerprint of a late addition.
4. **Invariants & enforcement.** The invariant that the four topologies describe the same platform is
   violated here in the most direct way possible. Nothing detects it.
5. **Extension procedure.** Add a `services.yml` entry with `cwd:
   ../Pacco.Services.OrderMaker/src/Pacco.Services.OrderMaker.Api` and a `prod-services.yml` entry
   with the publish path and `ASPNETCORE_URLS` on 5015. The sibling already provides the matching
   `launchSettings.json` port, so the development entry is a four-line addition with no other change
   required.
6. **Failure modes.**
   1. **In PM2 mode the platform runs without OrderMaker**, while the host-mode Prometheus config
      scrapes `localhost:5015` and reports a permanently-down target — the monitoring correctly
      detects a service the process manifests never intended to start, and an operator reading the
      dashboard sees an unexplained outage.
   2. Any behaviour that depends on OrderMaker works under Docker and not under PM2, which is a
      difficult class of bug to attribute.
   3. `compose/services.yml:65` makes `operations-service` depend on a container that a PM2-mode
      operator does not have, which is harmless only because the two modes are never combined.

### 3.34 Host-networking infrastructure variant

1. **Definition.** A second declaration of the whole backing-service estate that removes container
   networking entirely.
2. **Representation & storage.** `compose/host-infrastructure.yml` (105 lines) — the same ten services,
   each with `network_mode: host`, **no `ports` keys at all** (they would be ignored `[compose]`), and
   **no `networks:` top-level block**. Fabio's Consul address is adapted to `localhost:8500` (line 17);
   Prometheus builds from `./host-prometheus` (line 46). The volumes block (lines 90-104) is identical
   to the bridge variant's, including the same five commented-out entries.
3. **Lifecycle.** Intended for the PM2 modes, where the application processes run on the host and would
   otherwise have to reach containers by published port.
4. **Invariants & enforcement.**
   1. `network_mode: host` is a Linux-only mechanism; on Docker Desktop for macOS or Windows the
      containers join the Linux VM's network namespace, not the developer's host, so the platform's
      most likely development machines are exactly the ones on which this file does not do what its
      name promises `[compose]`.
   2. It cannot be combined with either service stack, both of which require the external
      `pacco-network` this file never creates (§3.10).
5. **Extension procedure.** Any service added to `compose/infrastructure.yml` must be mirrored here
   with `network_mode: host` and its `ports` block dropped.
6. **Failure modes.**
   1. **Seq becomes unreachable at its configured address.** In bridge mode the mapping `5341:80`
      republishes Seq on 5341; under `network_mode: host` there is no mapping, so Seq listens on
      whatever port it uses internally — 80, on the evidence of the mapping's right-hand side — while
      every service is configured to send logs to port 5341. Host mode therefore loses centralised
      logging silently, and additionally binds a service to host port 80. Recorded as B-9 (§8.2).
   2. This file is referenced by **no documentation**: `README.md` names only `infrastructure.yml` and
      `services-local.yml`. Together with `compose/host-prometheus/`, it is a third of the compose
      surface that a reader must discover by listing the directory.
   3. Every backing service binds directly to host ports with no mapping layer, so a port conflict with
      anything already running on the developer's machine is unavoidable rather than remappable.

### 3.35 The three split infrastructure stacks

1. **Definition.** `infrastructure.yml` decomposed into three thematic stacks, so an operator can start
   only what they need.
2. **Representation & storage.**
   1. `compose/mongo-rabbit-redis.yml` (48 lines) — the data plane; **creates** `pacco-network`
      (lines 38-40); live `mongo` and `redis` volumes (lines 42-48).
   2. `compose/consul-fabio-vault.yml` (48 lines) — discovery and secrets; **consumes** the network
      (lines 41-44); all volumes commented out (lines 46-48).
   3. `compose/grafana-seq-jaeger-prometheus.yml` (65 lines) — observability; **consumes** the network
      (lines 54-57); all volumes commented out (lines 59-65).
3. **Lifecycle.** Stack 1 must be started first, because it is the only creator among the three
   (§3.10). That ordering constraint is not documented anywhere.
4. **Invariants & enforcement.**
   1. Union of the three = membership of `infrastructure.yml` — true for all ten services.
   2. Content equality with `infrastructure.yml` — **false**: the RabbitMQ image differs (§3.22), and
      the split stacks carry no commented-out Mongo credential block (§3.21). Two of the three
      differences are silent behaviour changes.
   3. The `pacco-network` name is repeated in all three, so the split stacks interoperate with both
      service stacks.
5. **Extension procedure.** Any change to a backing service must now be applied in up to **three**
   places: `infrastructure.yml`, `host-infrastructure.yml`, and the owning split stack. The cheapest
   structural improvement available to this component is to delete the split stacks and rely on
   `docker compose up <service> …` against `infrastructure.yml`, which achieves the same selectivity
   with one definition `[compose]`.
6. **Failure modes.**
   1. The RabbitMQ divergence (§3.22) — the split path silently disables broker metrics.
   2. Starting stack 2 or 3 first fails on the missing external network; the error names the network,
      not the fix.
   3. Threefold duplication guarantees further drift; the two divergences already present were
      introduced this way.

### 3.36 Volume and data-durability model

1. **Definition.** Which of the platform's ten backing services keep their data across a container
   recreation. The answer is two.
2. **Representation & storage.** `compose/infrastructure.yml:133-147` — a `volumes:` block in which
   `mongo` (lines 138-139) and `redis` (lines 144-145) are **declared**, while `consul` (134-135),
   `grafana` (136-137), `prometheus` (140-141), `rabbitmq` (142-143) and `seq` (146-147) are
   **commented out**. Each commented declaration is paired with a commented mount in the service
   definition, so enabling one requires two edits in the same file.

   | Backing service | Durable? | Consequence of a restart |
   | --- | --- | --- |
   | MongoDB | **yes** | application data survives |
   | Redis | **yes** | cache survives (the least valuable data is the best protected) |
   | RabbitMQ | no | queues, bindings and unconsumed messages are lost |
   | Consul | no | the entire service registry is lost (§3.19) |
   | Prometheus | no | all metric history is lost |
   | Seq | no | all logs are lost |
   | Grafana | no | datasources and dashboards are lost (§3.28) |
   | Jaeger | no | no volume is even contemplated — all-in-one is memory-only (§3.29) |
   | Vault | no | dev mode is memory-only by design (§3.27) |
   | Fabio | n/a | stateless by design |

3. **Lifecycle.** Named volumes are created on first `up`, survive `down`, and are removed by `down -v`.
   Volumes are scoped to the Compose project name, which defaults to the directory name `[compose]`.
4. **Invariants & enforcement.** The pattern "mount commented + declaration commented" is applied
   consistently, so the file is internally coherent; what is not coherent is the *choice*.
5. **Extension procedure.** For each service to be made durable, uncomment both the mount in the
   service block and the declaration in the `volumes:` block, then mirror the change into
   `host-infrastructure.yml` and the owning split stack (§3.35).
6. **Failure modes.**
   1. **The three observability signals — logs, metrics and traces — are all ephemeral.** Any
      post-mortem that outlives a container recreation is impossible. For a platform whose stated
      purpose is to demonstrate distributed-systems practice, this is the most consequential
      durability gap.
   2. Broker durability loss (§3.22) makes outbox and retry behaviour untestable locally.
   3. Running `up` from a differently named directory silently creates a fresh set of volumes and
      presents as total data loss (§3.21).

### 3.37 README and asset drift

1. **Definition.** The repository's documentation refers to a different repository than the one it is
   in.
2. **Representation & storage.**
   1. Four images are referenced by absolute URL against
      `raw.githubusercontent.com/devmentors/Pacco/master/assets/…` — `README.md:1` (logo), `:8`
      (overview), `:19` (infrastructure), `:25` (clean architecture) — while `assets/` holds exactly
      those four PNGs (`pacco_logo.png`, `pacco_overview.png`, `infrastructure.png`,
      `clean_architecture.png`). **The committed assets are referenced by nothing.**
   2. Every repository link in the clone list (`README.md:33-44`) and the two script links
      (`README.md:46`) point at `github.com/devmentors/…`.
   3. `README.md:66` links the sample HTTP scenario file in the `Pacco.APIGateway` repository —
      correctly: that file exists at `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest` (last
      touched in `b4e32c5`, 2020-08-27). See §8.4, where a contrary claim in
      `repo-summary/Pacco.md` is corrected.
3. **Lifecycle.** Written upstream; inherited unchanged by the fork.
4. **Invariants & enforcement.** None — a fork's README is not rewritten by forking.
5. **Extension procedure.** Convert the four image references to repository-relative paths
   (`assets/…`), which renders correctly on any fork and removes the dependency on upstream's raw
   host. Leave the *link* targets alone unless the fork is intended to be self-contained.
6. **Failure modes.**
   1. If upstream renames a file, moves the branch or goes private, this fork's README loses all four
      diagrams while the images sit unused in the repository.
   2. A reader following the README's links contributes to upstream rather than the fork.
   3. The documented bring-up (`README.md:51-61`) covers exactly one of the five viable paths in
      §3.9 — the other four, including both PM2 modes and the entire host-networking variant, are
      undocumented.

### 3.38 `.gitignore` posture in a code-free repository

1. **Definition.** The repository carries a 331-line Visual Studio ignore template and no code.
2. **Representation & storage.** `.gitignore` — the standard `dotnet new gitignore`/GitHub VisualStudio
   template, plus `logs/` appended at line 332. The only rule with any effect on this repository's
   actual content would be one matching YAML, `.sln` or `.txt` — there is none, so the file is inert
   here.
3. **Lifecycle.** Copied in at repository creation; the `logs/` line is the one local addition.
4. **Invariants & enforcement.** None.
5. **Extension procedure.** Either trim it to what this repository needs (a `.env` rule would be the
   valuable addition if §3.18's interpolation is ever introduced) or leave it; the cost is confusion,
   not behaviour.
6. **Failure modes.**
   1. `.gitignore:55` ignores `**/Properties/launchSettings.json`, the exact files the PM2 development
      mode depends on (§3.31). The rule has no effect *here* — there are no such files in this
      repository — but it states a convention that, if followed in a service repository, breaks
      development mode. Because engineers copy the platform repository's ignore file, the rule
      propagates.
   2. `Pacco.sln` is tracked despite the template's build-output rules, which is correct but means the
      file must be curated by hand (§3.3).

### 3.39 `docker-images.txt` as the operator runbook

1. **Definition.** A 461-line plain-text file that is, in practice, the provenance of every compose
   service and the only operational documentation the platform has beyond the README.
2. **Representation & storage.** Sequential sections, each a `docker run` command plus notes:
   RabbitMQ (line 42), MongoDB (61), Mongo Express (74), SQL Server 2017 (89), PostgreSQL (102),
   InfluxDB (115), Redis (131) with a `redis-cli` recipe (144), Consul (160) with a `config.json`
   sample (168), Vault dev (197), production Vault with a Consul backend (209) followed by the full
   initialise/unseal/`userpass`/policy/KV-v2 procedure (211-267), the database secret engine for
   dynamic MongoDB credentials (274-290), PKI (291-321), Grafana (333), Prometheus (346) with a sample
   scrape configuration (348-400), Seq (391), Elasticsearch (418) with `vm.max_map_count` guidance
   (402-417), Kibana (433) with a `kibana.yml` sample (439), Logstash (452) and Jaeger (461).
3. **Lifecycle.** Append-only reference; not executed by anything.
4. **Invariants & enforcement.**
   1. **The compose files are derived from this file.** Ports, environment variables and image names
      match line by line — Seq's `5341:80` (line 391 versus `compose/infrastructure.yml:111`), Vault's
      `IPC_LOCK` and dev token (197 versus 115-127), Jaeger's seven ports (461 versus 38-51), RabbitMQ's
      5672/15672 (42 versus 84-86). Reading this file explains *why* the compose stacks look as they do.
   2. It has drifted in both directions: it documents five services the platform does not run (SQL
      Server, PostgreSQL, InfluxDB, Elasticsearch/Kibana/Logstash, Mongo Express) and it sets Jaeger's
      Zipkin collector variable that the compose files omit (§3.29).
   3. Its Prometheus sample (348-400) is a *different* configuration from the two real ones —
      generic upstream defaults, a `rule_files` section and a scrape job using a non-default metrics
      path. It is a template, not the platform's configuration; §8.4 corrects an existing artifact that
      attributes one of its values to `compose/prometheus/prometheus.yml`.
5. **Extension procedure.** When a backing service is added to the compose stacks, add the standalone
   command here too — it is the only place that records *why* a port or variable was chosen. When one
   is removed, delete its section rather than leaving it, so the file stops accumulating services the
   platform never adopted.
6. **Failure modes.**
   1. A reader treats it as the platform's inventory and concludes Pacco uses Elasticsearch and SQL
      Server; it does not.
   2. It contains real Vault initialisation output (§3.41).
   3. Because it is prose, no tool can check it against the compose files.

### 3.40 Release and versioning: the absence of platform-level CI

1. **Definition.** A declared absence with architectural consequences: nothing anywhere builds,
   validates, tags or releases *the platform*.
2. **Representation & storage.** The 29-file inventory (§3.1) contains no `.travis.yml`, no
   `.github/workflows/`, no `azure-pipelines.yml`, no `Makefile`, no `.env`, no `VERSION` file and no
   lock file of any kind. What exists is per repository: each sibling has its own `.travis.yml` and a
   `scripts/dockerize.sh` that derives its image tag from `TRAVIS_BRANCH` (`master` → `latest`,
   `develop` → `dev`) and pushes to `$DOCKER_USERNAME/pacco.<name>`.
3. **Lifecycle.** Each service releases when its own branch builds. The "platform version" is whatever
   eleven independently released `:latest` images happen to be at pull time (§3.13).
4. **Invariants & enforcement.** None. There is no compatibility matrix, no contract-test gate at the
   platform level (the PACT projects live in the service repositories), and no artefact that names a
   consistent set of image versions.
5. **Extension procedure.** The smallest useful step is a `.env` file plus per-service tag variables in
   `compose/services.yml` (§3.13), committed — that single file would then *be* the platform version.
   A CI job that runs `docker compose config` over all seven stacks would catch the entire class of
   YAML and reference errors this document repeatedly notes as "enforced by nothing".
6. **Failure modes.**
   1. No rollback: there is no previous state to roll back to, because no state was ever recorded.
   2. `scripts/dockerize.sh` in the siblings leaves the tag **empty** when `TRAVIS_BRANCH` is neither
      `master` nor `develop`, producing a malformed image reference — a per-repository defect that
      this repository inherits as unpredictable `:latest` content.
   3. Every consistency invariant in this document is unenforced precisely because there is no place
      to enforce it (§3.1 point 4).

### 3.41 Committed credential material and estate-wide default credentials

1. **Definition.** The security posture of the declared estate, stated plainly.
2. **Representation & storage.** *Values are not reproduced in this document; each item is cited by
   path and line only, per the folder convention in `component-internals/index.md` §3.2.*

   | # | Item | Location | Nature |
   | --- | --- | --- | --- |
   | 1 | Vault unseal key shares and the initial root token from a real `vault operator init` run | `docker-images.txt:219-232` | **committed live secret material** |
   | 2 | SQL Server `SA_PASSWORD` | `docker-images.txt:89` | sample credential in a runbook command |
   | 3 | PostgreSQL superuser password | `docker-images.txt:102` | sample credential |
   | 4 | Vault dev root token | `compose/infrastructure.yml:121`, `compose/consul-fabio-vault.yml:37`, `compose/host-infrastructure.yml:85`, `docker-images.txt:197` | unchanged upstream placeholder |
   | 5 | MongoDB root credentials | `compose/infrastructure.yml:57-59`, `compose/host-infrastructure.yml:38-40` | commented out — i.e. **authentication disabled** |
   | 6 | RabbitMQ credentials | not set here; each service's base `appsettings.json` | image defaults |
   | 7 | Grafana admin credentials | not set anywhere | image defaults |
   | 8 | Consul, Redis, Seq, Prometheus, Jaeger, Fabio | not set anywhere | **no authentication at all** |

3. **Lifecycle.** Item 1 is in git history and remains recoverable from it regardless of any future
   deletion. Items 4–8 are the state of every bring-up.
4. **Invariants & enforcement.** There is no secret-management mechanism in this repository: no
   `.env`, no variable interpolation (§3.18), no external secret reference, and Vault — the tool that
   would provide one — is disabled in every service (§3.27).
5. **Extension procedure.**
   1. Treat the material at `docker-images.txt:219-232` as compromised: any Vault instance ever
      initialised with it must be re-keyed and its root token revoked, independently of whether the
      lines are removed. Removing them from `HEAD` does not remove them from history.
   2. Replace items 2–4 with placeholders that are obviously not values, and introduce `${VAR}`
      interpolation (§3.18) so the compose files stop being a place where credentials can be written.
   3. Enable MongoDB authentication (§3.21) and Redis `requirepass` (§3.23) as a pair with the
      corresponding twelve-repository configuration change.
   4. Bind published ports to `127.0.0.1` (§3.12) so a development estate is not a network estate.
6. **Failure modes.**
   1. A developer machine running `infrastructure.yml` exposes an unauthenticated Mongo, Redis, Consul,
      Vault, Seq and RabbitMQ management UI to its entire network segment.
   2. Because the insecure state is *consistent* — the services' configurations expect exactly these
      defaults — nothing ever fails, and the posture is invisible until it is audited.

### 3.42 Health, readiness and the absence of gating

1. **Definition.** A declared absence: nothing in the platform's orchestration knows whether anything
   is working.
2. **Representation & storage.** There is **no `healthcheck` key in any of the seven stacks** — verified
   across `compose/*.yml`. The only ordering construct is the single list-form `depends_on` (§3.15).
   The services do expose a Consul-facing ping endpoint, configured on their side, and each service's
   base configuration names it `ping` `[convey]` — but nothing in this repository references it.
3. **Lifecycle.** Not applicable; the mechanism does not exist.
4. **Invariants & enforcement.** Docker reports every container as `running` from the moment its
   process starts, which for an ASP.NET Core service is well before it is listening, connected to
   RabbitMQ and registered in Consul.
5. **Extension procedure.** Add a `healthcheck` to each application service that polls its own ping
   endpoint, and convert `compose/services.yml:59-67` to the long-form
   `condition: service_healthy` `[compose]`. Add health checks to Mongo, RabbitMQ and Consul, and make
   every application depend on them — that is the change that would make bring-up deterministic.
6. **Failure modes.**
   1. `docker compose ps` showing eleven healthy-looking containers is not evidence that the platform
      works; there is no command in this repository that produces such evidence.
   2. A service that starts before RabbitMQ retries on its own schedule `[convey]`; a service that
      starts before Consul may never register, and Fabio then routes nothing to it (§3.19, §3.20).
   3. Automated verification of a bring-up is impossible without adding the mechanism first, which is
      why no such verification exists.

### 3.43 Scale-out constraint imposed by `container_name`

1. **Definition.** Every service in every stack sets `container_name`, which fixes the instance count
   at one.
2. **Representation & storage.** All eleven applications (`compose/services.yml:6,17,26,35,44,53,71,80,
   89,98,107`) and all ten backing services (`compose/infrastructure.yml`) carry the key.
3. **Lifecycle.** Enforced by Docker at container creation — names are unique per daemon `[compose]`.
4. **Invariants & enforcement.** `docker compose up --scale <service>=N` fails for any N > 1 while
   `container_name` is set `[compose]`. So the platform that runs Consul and Fabio — machinery whose
   entire purpose is to route across multiple instances — **cannot start a second instance of
   anything**.
5. **Extension procedure.** Removing `container_name` restores scaling, but breaks the DNS contract
   (§3.11): the services' configurations resolve backing services by these exact names. The correct
   change is to remove `container_name` only from the *application* services (whose names are used by
   Consul registration and Prometheus targets, both of which would need rework) while keeping it on
   the backing services. This is a substantial change and is Q-5 (§8.3).
6. **Failure modes.**
   1. An engineer demonstrating load-balancing through Fabio cannot do so with the supplied stacks.
   2. Prometheus targets are single addresses (§3.25), so even if scaling worked, monitoring would
      scrape one instance.

### 3.44 Absent operational concerns

1. **Definition.** What a production-intent orchestration layer would declare and this one does not.
   Each absence below was verified by searching all seven stacks and both PM2 manifests.
2. **Representation & storage.**

   | # | Absent concern | Verified by | Consequence |
   | --- | --- | --- | --- |
   | 1 | Resource limits (`deploy.resources`, `mem_limit`, `cpus`) | no such key in `compose/*.yml` | one runaway container can exhaust the host; Jaeger's in-memory spans are unbounded (§3.29) |
   | 2 | Logging driver / rotation | no `logging:` key | container logs grow without bound |
   | 3 | TLS anywhere | no certificate, no `tls`, and the runbook's production Vault explicitly disables it (`docker-images.txt:209`) | all inter-service traffic, all UIs and the broker are plaintext |
   | 4 | Network segmentation | one flat bridge (§3.10) | any compromised container reaches every datastore |
   | 5 | Backup or restore | no volume backup, no `mongodump` anywhere | the two durable volumes have no recovery path |
   | 6 | Alerting rules | no `rule_files` in either scrape config (§3.24) | metrics are collected and never acted on |
   | 7 | Log/metric retention | no retention flags | bounded only by container lifetime (§3.36) |
   | 8 | Secret management in use | Vault disabled per service (§3.27) | credentials are literals (§3.41) |
   | 9 | Any non-local deployment target | no k8s/Helm/Terraform (§1.2) | the platform has no declared production topology other than `prod-services.yml` (§3.32) |
   | 10 | User/permission control on containers | no `user:` key | containers run as their image default |

3. **Lifecycle.** Not applicable.
4. **Invariants & enforcement.** None of these is a defect *given* the repository's evident purpose —
   a demonstration platform for local use. They are recorded because the repository also contains a
   `prod-services.yml`, a production Vault runbook and a PKI procedure, which together imply an intent
   this declaration set does not deliver.
5. **Extension procedure.** If production is genuinely a target, items 3, 4, 5 and 8 are prerequisites,
   and item 9 supersedes the whole component. If it is not, `prod-services.yml` and the production
   sections of `docker-images.txt` should say so in a sentence.
6. **Failure modes.** The single failure mode is misreading: an engineer sees `prod-services.yml` and
   the PKI runbook and treats the estate as production-intent. §7.6 states the correct reading.

### 3.45 The drift register — where this repository disagrees with itself

1. **Definition.** A consolidated index of every internal inconsistency established above. It exists
   because the individual findings are unremarkable and their *density* is the actual architectural
   fact about this component.
2. **Representation & storage.**

   | # | Disagreement | Files | Section | Effect |
   | --- | --- | --- | --- | --- |
   | 1 | Gateway runs async in one stack, sync in the other | `compose/services.yml:9` vs `compose/services-local.yml:9` | §3.16 | behavioural |
   | 2 | RabbitMQ has the Prometheus plugin in one stack, not the other | `compose/infrastructure.yml:79` vs `compose/mongo-rabbit-redis.yml:16` | §3.22 | silent metric loss |
   | 3 | Gateway's Prometheus job is `api-gateway` in one config, `api` in the other | `compose/prometheus/prometheus.yml:10` vs `compose/host-prometheus/prometheus.yml:10` | §3.26 | queries return no data |
   | 4 | OrderMaker exists in compose and Prometheus, not in either PM2 manifest | `compose/services.yml:78-85` vs `services.yml`, `prod-services.yml` | §3.33 | missing service |
   | 5 | The two pull scripts leave the workspace on different branches | `scripts/git-pull.sh:9` vs `scripts/git-pull-fast.sh:4` | §3.7 | wrong code |
   | 6 | Repository membership declared three different ways | `README.md:33-44`, `scripts/*:2`, the discovery backlog | §3.5 | incomplete workspace |
   | 7 | `prod-services.yml` sets no environment while `services.yml` gets one implicitly | `prod-services.yml` vs `launchSettings.json` | §3.32 | wrong configuration in "prod" |
   | 8 | Mongo credential block present-but-commented in one stack, absent in the other | `compose/infrastructure.yml:57-59` vs `compose/mongo-rabbit-redis.yml` | §3.21 | security regression on switch |
   | 9 | Jaeger's Zipkin port published without the variable that enables it | `compose/infrastructure.yml:49` vs `docker-images.txt:461` | §3.29 | dead port |
   | 10 | Scrape-interval comments say 15s, values say 5s | `compose/prometheus/prometheus.yml:2-3`, `compose/host-prometheus/prometheus.yml:2-3` | §3.25 | reader-trap |
   | 11 | README images point at upstream while `assets/` sits unused | `README.md:1,8,19,25` | §3.37 | fragile docs |
   | 12 | `.gitignore` excludes the files PM2 development mode requires | `.gitignore:55` | §3.31, §3.38 | propagating convention error |
   | 13 | Seq's port mapping is inverted and is lost entirely in host mode | `compose/infrastructure.yml:111` vs `compose/host-infrastructure.yml:69-77` | §3.30, §3.34 | logging lost |
   | 14 | `Pacco.sln` and all five scripts reference a repository that does not exist in scope | `Pacco.sln`, `scripts/*:2` | §3.4 | broken assembly step |
   | 15 | `depends_on` omits `pricing-service` and every backing service | `compose/services.yml:59-67` | §3.15 | meaningless guarantee |

3. **Lifecycle.** Each entry arose the same way: a change was made in one of two-to-four parallel
   declarations of the same thing, and nothing detected the omission.
4. **Invariants & enforcement.** The register *is* the evidence that this component has no enforcement
   mechanism. Fifteen divergences across 29 files is not carelessness; it is the predictable output of
   fourfold manual duplication (§3.1 point 4, §3.40).
5. **Extension procedure.** Before any change to this repository, identify which of the four topologies
   the change belongs to and apply it to **all** of them, or record explicitly why not. §7.1–§7.5
   convert this into ordered checklists.
6. **Failure modes.** Every entry above is itself a failure mode; the meta-failure is that fixing one
   without adding enforcement (§3.40 point 3) leaves the mechanism that produced it intact.

---

## 4. Primary control flows

This component has no runtime, so its "control flows" are the operator-driven sequences the files
encode. Each flow below is traced through the actual declarations, with the step at which it can fail
called out.

### 4.1 Flow A — documented bring-up (the README path)

1. The operator clones twelve repositories into one parent directory (`README.md:31-44`), or copies
   the scripts next to `Pacco` and runs `scripts/git-clone.sh` (`README.md:46`, §3.2, §3.6).
2. The operator changes into `Pacco/compose` (`README.md:51`).
3. `docker-compose -f infrastructure.yml up -d` (`README.md:54`):
   1. Compose reads the top-level `networks:` block (`compose/infrastructure.yml:129-131`) and
      **creates** the bridge network `pacco-network` (§3.10).
   2. It builds two images: `./prometheus` (§3.24) and `./rabbitmq` (§3.22). Both are cache-hits after
      the first run, which is why later edits to the scrape configuration appear to do nothing.
   3. It pulls eight unpinned upstream images (§3.13) and starts ten containers with no ordering
      between them (§3.15) and no readiness gate (§3.42).
   4. Two named volumes are created — `mongo` and `redis` (§3.36).
   5. Twenty-one host ports are bound on all interfaces (§3.12).
4. `docker-compose -f services-local.yml up` (`README.md:60`):
   1. Compose resolves `pacco-network` as **external** (`compose/services-local.yml:114-117`). If step
      3 was skipped, this is where the run fails, loudly and correctly.
   2. It builds eleven .NET images from `../../Pacco.X` (§3.14) — the slow step, and the one that
      requires the §3.2 layout to be exactly right.
   3. `operations-service` is created after the eight containers it names (`…:59-67`), which is
      ordering without readiness (§3.15).
   4. Eleven containers start, each with `ASPNETCORE_ENVIRONMENT=docker` baked into its image, and
      each therefore reading its `appsettings.docker.json` — the file whose hostnames are the
      `container_name` values from step 3 (§3.11).
   5. The gateway additionally receives `NTRADA_CONFIG` for the **synchronous** topology (§3.16).
5. Steady state: eleven application containers on one flat network with ten backing services;
   registration into Consul and routing through Fabio proceed from each service's own configuration
   `[convey]`.

**Where this flow breaks, in order of likelihood:** wrong workspace layout (step 1, §3.2); external
network missing (4.1); stale cached images after a source edit (4.2, §3.14); a service that started
before Consul and never registered (5, §3.42).

### 4.2 Flow B — published-image bring-up

Identical to Flow A except step 4, which uses `compose/services.yml`: no build occurs, eleven
`devmentors/pacco.*:latest` images are pulled or taken from cache (§3.13), and the gateway receives
the **asynchronous** configuration (§3.16). The observable behaviour of the gateway therefore differs
from Flow A for identical requests — the divergence documented as B-3 (§8.2).

### 4.3 Flow C — PM2 development bring-up

1. `docker-compose -f compose/mongo-rabbit-redis.yml up -d` — creates `pacco-network` (§3.10) and
   starts Mongo, stock RabbitMQ and Redis (§3.35). This is the estate the `local` profile needs, and
   nothing in the repository says so (§3.31).
2. `pm2 start services.yml` **from the repository root** — the `cwd` values are relative (§3.2).
3. For each of the ten apps, PM2 runs `dotnet run` in a sibling's API project directory.
4. `dotnet run` reads that project's `Properties/launchSettings.json`, selects the `commandName:
   Project` profile, and applies `applicationUrl` (the port) and `ASPNETCORE_ENVIRONMENT: local`
   (e.g. `hianshul100_Pacco.Services.Vehicles/…/launchSettings.json:17-24`). **This is the step at
   which the platform's development ports and configuration are actually decided, and it happens in a
   repository this manifest never names** (§3.31).
5. The `local` profile disables Consul, Fabio, Jaeger, metrics, the outbox, Vault and Seq, so the
   services need only the three backing services from step 1.
6. `ordermaker` never starts (§3.33).

### 4.4 Flow D — PM2 production bring-up

1. `dotnet publish -c release` in each of the ten repositories, producing
   `src/…/bin/release/netcoreapp3.1/publish` — the literal path `prod-services.yml` expects (§3.32).
2. Backing services are started by some means this repository does not specify;
   `compose/host-infrastructure.yml` is the only variant that suits host processes (§3.34), and it is
   undocumented.
3. `pm2 start prod-services.yml` — each app runs its published assembly with `ASPNETCORE_URLS` on its
   assigned port (§3.12).
4. **No `ASPNETCORE_ENVIRONMENT` is set**, so each service loads base `appsettings.json`: Consul at
   `docker.for.win.localhost`, RabbitMQ with default credentials, metrics enabled (§3.32). On a real
   server the Consul address does not resolve.
5. `ordermaker` never starts (§3.33).

### 4.5 Flow E — the metric path

1. A service exposes `/metrics` because its own configuration enables the metrics middleware
   `[convey]`; this repository sets nothing to make that happen.
2. Prometheus, built from `compose/prometheus/Dockerfile` with the scrape configuration **baked in**
   (§3.24), resolves each target's bare container name over the shared network's DNS (§3.11) and
   scrapes port 80, path `/metrics`, every 5 seconds (§3.25).
3. RabbitMQ is scraped at `rabbitmq:15692`, which exists only because
   `compose/rabbitmq/plugins` enables the Prometheus plugin (§3.22) — and only in stacks that build
   that image (§3.35).
4. Grafana is available on 3000 with **no datasource configured**, so nothing reaches a dashboard until
   a human wires it by hand (§3.28).
5. All series are lost when the Prometheus container is recreated (§3.36).

**Host-mode variant:** the same flow against `localhost:5000-5009,5015` with the gateway's job renamed
to `api` (§3.26) — any saved query keyed on the job label breaks.

### 4.6 Flow F — registration and routing

1. Each service registers itself in Consul at start-up, using its own `consul` configuration section,
   which sets its address to its **own container name** and its port to 80 in the `docker` profile
   `[convey]`.
2. Consul, started by this repository with no persistence (§3.19), holds that registry in memory.
3. Fabio, pointed at `consul:8500` by `FABIO_REGISTRY_CONSUL_ADDR` (§3.20), continuously derives its
   routing table from the registry.
4. A service making an internal HTTP call addresses `http://fabio:9999` with a host header or path
   that Fabio maps back to a registered instance `[convey]`.
5. This repository's contribution to the entire flow is three strings: the container name `consul`,
   the container name `fabio`, and the Consul address in Fabio's environment. Everything else is in
   the service repositories.

**Failure mode of note:** restarting the Consul container empties the registry (§3.19) and every route
disappears until services re-register; nothing in the platform triggers re-registration.

### 4.7 Flow G — image release and consumption

1. A change merges to `master` in a service repository.
2. That repository's Travis build runs its own `scripts/dockerize.sh`, which sets `TAG=latest` for
   `master` and `dev` for `develop`, and pushes `$DOCKER_USERNAME/pacco.<name>:$TAG`.
3. `compose/services.yml` references `devmentors/pacco.<name>` with **no tag** (§3.13), so a subsequent
   `docker compose pull` picks up the new `:latest`.
4. There is no step 4: nothing records which eleven images were pulled, nothing validates that they are
   mutually compatible, and nothing can reproduce the set later (§3.40).

### 4.8 Flow H — workspace refresh

1. `scripts/git-pull.sh` iterates twelve repositories, checking out `develop`, pulling, checking out
   `master`, pulling, and **leaving each on `master`** (§3.7).
2. `scripts/git-pull-fast.sh` does the same twelve in parallel and **leaves each on `develop`**.
3. `Pacco.APIGateway.Ocelot` is attempted by both and does not exist in this workspace's scope (§3.4);
   the sequential script reports a `cd` failure, the parallel one buries it in interleaved output.
4. Neither script touches this repository — the operator must update it separately.

---

## 5. Persistence & schema evolution

This component persists no application data and defines no schema. What it does define is **where the
platform's state lives, how long it lives, and how the topology that produces it evolves**. Both are
modelled here, because for an orchestration component they are the equivalent question.

### 5.1 State owned by this component

| # | State | Storage | Lifetime | Owner |
| --- | --- | --- | --- | --- |
| 1 | The bridge network `pacco-network` | Docker network namespace | until the creating project is torn down with no attached endpoints | `compose/infrastructure.yml:129-131` (§3.10) |
| 2 | Named volume `mongo` | Docker local volume driver | survives `down`; removed by `down -v` | `compose/infrastructure.yml:138-139` (§3.36) |
| 3 | Named volume `redis` | Docker local volume driver | same | `compose/infrastructure.yml:144-145` |
| 4 | The two locally built images | Docker image store | until rebuilt or pruned | `compose/prometheus/`, `compose/rabbitmq/` (§3.22, §3.24) |
| 5 | Container writable layers holding **all** broker, registry, log, metric, trace, dashboard and secret state | container filesystem | destroyed on every container recreation | the five commented-out volume declarations (§3.36) |
| 6 | PM2's own process list | PM2 daemon state, outside the repository | until `pm2 delete` | `services.yml`, `prod-services.yml` |

Row 5 is the important one: **nine of the ten backing services keep their state in row 5, and only two
in rows 2–3.** Any statement about Pacco's durability must start there.

### 5.2 Schemas this component does not own but constrains

1. **MongoDB databases and collections** — created implicitly by each service; this repository fixes
   only that they all share one `mongo` process on one volume (§3.21). There is no migration tool, no
   seed script and no initialisation directory mounted into the container, so **schema evolution is
   whatever each service does on start-up** `[convey]`.
2. **RabbitMQ topology** (exchanges, queues, bindings) — declared by each service at start-up; this
   repository contributes only the broker and its plugins (§3.22). Because the broker has no volume,
   the topology is recreated from scratch on every restart, which conveniently hides any incompatible
   declaration change — a redeclare-conflict that would be fatal against a durable broker passes
   silently here.
3. **Consul KV and the service catalogue** — no seed, no persistence (§3.19).
4. **Prometheus TSDB** — no volume, no retention setting; the "schema" is the label set implied by the
   job names (§3.25), and renaming a job (as the host variant does, §3.26) is a breaking change to
   every query written against it.
5. **Vault storage** — dev mode is memory-only; the runbook's KV v1→v2 upgrade
   (`docker-images.txt:257-261`) is the only schema-migration procedure documented anywhere in the
   repository, and it applies to a Vault the platform does not use (§3.27).

### 5.3 Topology evolution — how this component actually changes over time

The repository's own git history and internal structure show three evolution mechanisms, all manual:

1. **Additive duplication.** A new capability is added by copying an existing block. The three split
   stacks (§3.35) and the host-networking variants (§3.26, §3.34) exist this way, and both carry
   divergences introduced after the copy (§3.45 rows 2, 3, 8).
2. **Late-arrival asymmetry.** A component added after the manifests were written is wired into some
   topologies and not others — `ordermaker-service` at port 5015 (§3.33) is the visible instance;
   nothing prevents the next one.
3. **Abandonment in place.** A retired component is removed from the running topology but left in the
   assembly machinery — `Pacco.APIGateway.Ocelot` (§3.4), the unused `assets/` (§3.37), and the five
   runbook services the platform never adopted (§3.39).

There is **no versioning of the topology itself**: no `.env`, no tag pinning, no compatibility matrix,
no changelog (§3.13, §3.40). A given commit of this repository does not describe a reproducible
platform; it describes a *shape* whose contents are decided at pull time.

### 5.4 Backward and forward compatibility rules

1. **Renaming a `container_name` is a breaking change to twelve repositories** (§3.11). There is no
   deprecation path — Docker network aliases could provide one (`networks: pacco: aliases: [old, new]`)
   but no stack uses aliases today.
2. **Changing a port is breaking in up to five files**: both service stacks, both Prometheus configs,
   the PM2 production manifest, plus the sibling's `launchSettings.json` and base `appsettings.json`
   (§3.12).
3. **Changing an image tag is unobservable** because there are no tags (§3.13); the compatibility
   surface is therefore uncontrolled by construction.
4. **Adding a backing service is additive and safe**; removing one is breaking for every service whose
   configuration references it, and this repository cannot tell which those are.
5. **Prometheus job names are a public interface** to dashboards and rules that live outside this
   repository (there are none in the workspace, but Grafana state is per-installation, §3.28).
   Treat a job rename as breaking (§3.26).
6. **The `netcoreapp3.1` literal in `prod-services.yml` couples this repository to the platform's
   target framework** (§3.32); a framework upgrade in the siblings requires a coordinated edit here.

---

## 6. Surface → internals map

The component's surface is not an API; it is the set of artefacts an operator, a developer or another
tool invokes. Each row names the entry point, the mechanism it triggers, whether it mutates anything,
and the section that models it.

### 6.1 Operator entry points

| # | Entry point | Invocation | Mutating? | Internals |
| --- | --- | --- | --- | --- |
| 1 | `compose/infrastructure.yml` | `docker-compose -f infrastructure.yml up -d` | yes — creates network, volumes, two images, ten containers, binds 21 ports | §3.9, §3.10, §3.19–§3.30, §4.1 |
| 2 | `compose/services.yml` | `docker-compose -f services.yml up` | yes — pulls 11 images, starts 11 containers, binds 11 ports | §3.13, §3.15, §3.16, §4.2 |
| 3 | `compose/services-local.yml` | `docker-compose -f services-local.yml up` | yes — builds 11 images from siblings | §3.14, §3.16, §4.1 |
| 4 | `compose/mongo-rabbit-redis.yml` | `docker-compose -f mongo-rabbit-redis.yml up -d` | yes — creates the network and three containers | §3.35, §4.3 |
| 5 | `compose/consul-fabio-vault.yml` | as above | yes — requires the network to exist | §3.35 |
| 6 | `compose/grafana-seq-jaeger-prometheus.yml` | as above | yes — requires the network to exist | §3.35 |
| 7 | `compose/host-infrastructure.yml` | as above | yes — binds host ports directly, no network | §3.34 |
| 8 | `services.yml` | `pm2 start services.yml` (from the repository root) | yes — starts 10 host processes via `dotnet run` | §3.31, §4.3 |
| 9 | `prod-services.yml` | `pm2 start prod-services.yml` | yes — starts 10 published assemblies | §3.32, §4.4 |
| 10 | `scripts/git-clone.sh` / `.ps1` | run from the **parent** directory | yes — clones 12 repositories from `devmentors` | §3.6, §3.8 |
| 11 | `scripts/git-clone-fast.sh` | as above, 12 in parallel | yes | §3.6 |
| 12 | `scripts/git-pull.sh` | as above | yes — leaves every repository on `master` | §3.7, §4.8 |
| 13 | `scripts/git-pull-fast.sh` | as above | yes — leaves every repository on `develop` | §3.7, §4.8 |

### 6.2 Developer and tooling entry points

| # | Entry point | Consumer | Mutating? | Internals |
| --- | --- | --- | --- | --- |
| 1 | `Pacco.sln` | Visual Studio / Rider / `dotnet` | no — nothing builds through it | §3.3, §3.4 |
| 2 | `compose/prometheus/prometheus.yml` | baked into the Prometheus image at build | no by itself | §3.24, §3.25 |
| 3 | `compose/host-prometheus/prometheus.yml` | same, for the host variant | no by itself | §3.26 |
| 4 | `compose/rabbitmq/plugins` | copied into the broker image at build | no by itself | §3.22 |
| 5 | `README.md` | humans | no | §3.37 |
| 6 | `docker-images.txt` | humans; the provenance of §6.1 rows 1–7 | no | §3.39, §3.41 |
| 7 | `assets/` | referenced by nothing | no | §3.37 |
| 8 | `.gitignore`, `LICENSE` | git; humans | no | §3.38 |

### 6.3 Contracts this component asserts on others

| # | Assertion | Asserted by | Other end | Breaks if |
| --- | --- | --- | --- | --- |
| 1 | Ten backing-service hostnames resolve on `pacco-network` | `container_name` in `compose/infrastructure.yml` | every `appsettings.docker.json` | a container is renamed, or host mode is used (§3.34) |
| 2 | Eleven application hostnames resolve | `container_name` in `compose/services.yml` | Consul registrations, Fabio routes, Prometheus targets | a container is renamed (§3.11) |
| 3 | Each application listens on port 80 in-container | the `X:80` port mappings | `ENV ASPNETCORE_URLS http://*:80` in each sibling `Dockerfile` | a service changes its listener |
| 4 | The gateway's Ntrada configuration name | `NTRADA_CONFIG` (§3.16) | `hianshul100_Pacco.APIGateway` | the file is renamed in the gateway repository |
| 5 | Each PM2 dev app's project directory exists | `cwd` in `services.yml` | sibling repository layout | a project is renamed or moved |
| 6 | Each PM2 prod app publishes to `bin/release/netcoreapp3.1/publish` | `cwd` in `prod-services.yml` | sibling build configuration | the target framework changes (§3.32) |
| 7 | Eleven images exist at `devmentors/pacco.*:latest` | `image` in `compose/services.yml` | each sibling's `scripts/dockerize.sh` | a repository's CI stops publishing |
| 8 | Eleven services expose `/metrics` on port 80 | `compose/prometheus/prometheus.yml` | each service's metrics configuration | a service disables metrics |
| 9 | Twelve repositories exist at `github.com/devmentors/*` | `scripts/*` | GitHub | a repository is renamed (§3.4) |
| 10 | Forty-one project files exist at fixed relative paths | `Pacco.sln` | sibling repository layout | a project moves; **already false for one** (§3.4) |

---

## 7. Change/extension guide

### 7.1 Adding a new microservice to the platform

Ordered; every step is required unless marked optional. The count of files is the point — a new
service touches **nine** files in this repository.

1. Choose a port. `5000`–`5009` and `5015` are taken (§3.12); use `5010`.
2. `compose/services.yml` — add a service block with `image`, `container_name`, `restart:
   unless-stopped`, `ports: <port>:80`, `networks: [pacco]`. Keep alphabetical placement consistent
   with the existing ordering.
3. `compose/services-local.yml` — add the identical block with `build: ../../Pacco.X` in place of
   `image` (§3.14). The two files must differ only in that line.
4. `compose/services.yml:59-67` — add the service to `operations-service`'s `depends_on` **only if**
   Operations consumes its events; note the block's limitations first (§3.15).
5. `compose/prometheus/prometheus.yml` — add a three-line job with the bare container name as target
   (§3.25).
6. `compose/host-prometheus/prometheus.yml` — add the matching job with `localhost:<port>` (§3.26).
7. `services.yml` — add a PM2 development entry (`name`, `script: dotnet run`, `cwd`,
   `max_restarts: 3`) (§3.31).
8. `prod-services.yml` — add a PM2 production entry with the publish path and `ASPNETCORE_URLS`
   (§3.32).
9. `Pacco.sln` — add the project references and a solution folder (optional but conventional, §3.3).
10. `scripts/git-clone.sh`, `git-clone-fast.sh`, `git-clone.ps1`, `git-pull.sh`, `git-pull-fast.sh` —
    add the repository name to the array on line 2 of each (five identical edits, §3.2).
11. `README.md:33-44` — add the clone-list entry.
12. In the **service** repository: a root `Dockerfile` with `ASPNETCORE_URLS http://*:80` and
    `ASPNETCORE_ENVIRONMENT docker`; an `appsettings.docker.json` whose hostnames match §6.3 row 1;
    a `launchSettings.json` `Project` profile on the chosen port (§3.31); and a `scripts/dockerize.sh`
    publishing to `devmentors/pacco.<name>`.
13. Update this document (§7.9) and `component-internals/index.md`.

**Verification:** there is no automated check. The closest available is
`docker compose -f compose/services.yml config` and `-f compose/services-local.yml config`, which at
least parse and resolve; then confirm the new Prometheus target reports `up` at `localhost:9090`.

### 7.2 Checking a change against the roster

`compose/prometheus/prometheus.yml` is the only file that enumerates all eleven applications plus the
broker in one place (§3.25). Use it as the checklist when auditing any of the four topologies. Note
that it is *not* authoritative for the PM2 modes, where `ordermaker` is absent by construction
(§3.33).

### 7.3 Adding a backing service

1. `compose/infrastructure.yml` — service block plus, if stateful, a mount **and** a matching entry in
   the `volumes:` block (both, §3.36).
2. `compose/host-infrastructure.yml` — the same block with `network_mode: host` and no `ports`
   (§3.34).
3. The owning split stack — `mongo-rabbit-redis.yml`, `consul-fabio-vault.yml` or
   `grafana-seq-jaeger-prometheus.yml` (§3.35). Copy the block **verbatim** from step 1; the two
   existing divergences (§3.45 rows 2 and 8) were introduced by not doing this.
4. `docker-images.txt` — add the standalone `docker run` equivalent, which is where the port and
   environment choices are explained (§3.39).
5. If it exposes metrics, add a Prometheus job in both configs (§3.25, §3.26).
6. If services must address it, agree the hostname first: it becomes a hard contract across twelve
   repositories (§3.11, §6.3 row 1).

### 7.4 Renaming a container (do not do this casually)

1. `compose/services.yml` and `compose/services-local.yml` — the `container_name` value.
2. `compose/services.yml:59-67` — the `depends_on` entry, if listed.
3. `compose/prometheus/prometheus.yml` — the target **and** the job name (the job name is a public
   interface, §5.4 point 5).
4. `compose/host-prometheus/prometheus.yml` — the job name.
5. The owning repository's `appsettings.docker.json` — its Consul service address.
6. Every consumer that addresses it through Fabio `[convey]`.
7. Prefer adding a network alias for the old name during a transition; no stack uses aliases today
   (§5.4 point 1).

### 7.5 Changing the gateway mode

1. Decide which of the two Ntrada configurations is canonical (§3.16).
2. Set `NTRADA_CONFIG` identically in `compose/services.yml:9` and `compose/services-local.yml:9`.
3. Record the decision in `README.md`, because the gateway's request semantics change with it.
4. Re-verify any HTTP scenario file against the chosen mode — the sample scenarios live in
   `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest` (§3.37 point 3).

### 7.6 Reading `prod-services.yml` correctly

Before treating this repository as production tooling, note: no environment is set (§3.32), no TLS
exists (§3.44 row 3), credentials are literals or defaults (§3.41), and backing services have no
declared production topology. The correct reading is that `prod-services.yml` runs *release builds*,
not that it runs a *production platform*. Any change that increases production intent should start
with §3.44 rows 3, 4, 5 and 8.

### 7.7 Making a change reproducible

The single highest-value change available to this component, in the order the steps depend on each
other:

1. Introduce a committed `.env` with one tag variable per application image.
2. Replace the eleven `image:` values in `compose/services.yml` with `${…:-latest}` interpolations
   (§3.13).
3. Pin the ten backing-service images to explicit tags.
4. Add a CI job that runs `docker compose config` over all seven stacks (§3.40 point 3) — this is what
   converts §3.45's fifteen divergences from "found by reading" to "caught by tooling".

### 7.8 Things that look changeable and are not

1. The `WORKDIR /app` in both Prometheus Dockerfiles is inert (§3.24) — removing it changes nothing,
   and neither does editing it.
2. The `x64`/`x86` solution platforms in `Pacco.sln` all map to `Any CPU` (§3.3) — selecting one has no
   effect.
3. `cd $REPOSITORY && cd ..` in `scripts/git-clone.sh:11` is a no-op (§3.6).
4. The `depends_on` block does not wait for readiness, so reordering or extending it does not make
   bring-up more reliable (§3.15).
5. Editing `compose/prometheus/prometheus.yml` without rebuilding has no effect (§3.24).

### 7.9 Maintenance contract for this document

Any change to this component must update this model **in the same change**.

| # | If you change… | Update here |
| --- | --- | --- |
| 1 | any file in `compose/` | §3.9–§3.36 as applicable, §4, §6.1, §3.45 if a divergence is created or removed |
| 2 | `services.yml` or `prod-services.yml` | §1.3, §3.31, §3.32, §3.33, §4.3, §4.4, §6.1 |
| 3 | a `container_name` or a port | §3.11, §3.12, §6.3, §7.4 |
| 4 | an image reference or tag | §3.13, §4.7, §5.4, §7.7 |
| 5 | `scripts/*` | §3.2, §3.5–§3.8, §4.8, §6.1 |
| 6 | `Pacco.sln` | §3.3, §3.4, §6.2 |
| 7 | `README.md` or `assets/` | §3.37, §4.1 |
| 8 | `docker-images.txt` | §3.39, §3.41 |
| 9 | anything that resolves an entry in §8.2 or §8.3 | remove the entry and state the resolution |
| 10 | the set of modelled components | `component-internals/index.md` §1 and §2 |

---

## 8. Assumptions, Blockers & Open Questions

### 8.1 Assumptions

| # | Assumption | Basis | Impact if wrong |
| --- | --- | --- | --- |
| A-1 | Docker Compose semantics behave as documented for `external: true` networks, list-form `depends_on`, untagged image resolution, `container_name` uniqueness and `network_mode: host` | tagged `[compose]` throughout; not verifiable inside this repository | §3.10, §3.13, §3.15, §3.34 and §3.43 would need re-derivation against the actual engine version in use |
| A-2 | The upstream images behave as their published defaults describe — Consul's dev agent, Vault's dev mode, Seq's in-container listener, Jaeger all-in-one's collectors, Grafana's default admin | tagged `[image]`; no Dockerfile for any of them is in this workspace | §3.19, §3.27, §3.29 and §3.30 are affected; §3.30's port inversion in particular is stated as *unverifiable*, not as fact |
| A-3 | Each service reads the configuration profile named by `ASPNETCORE_ENVIRONMENT` in the standard ASP.NET Core layering order | confirmed for every sibling by the presence of `appsettings.json`, `.local.json` and `.docker.json` and by each `Dockerfile`'s `ENV ASPNETCORE_ENVIRONMENT docker` | §1.3, §3.31, §3.32 and Flows C/D |
| A-4 | `dotnet run` honours `Properties/launchSettings.json` and selects the `commandName: Project` profile | framework behaviour; the profiles exist in all eleven sibling repositories with matching ports | §3.31 and §4.3 — this is the mechanism by which PM2 development ports are chosen |
| A-5 | The eleven `devmentors/pacco.*` images on Docker Hub correspond to the sibling repositories' `master` builds | each sibling's `scripts/dockerize.sh` maps `master` → `latest` | §3.13, §4.7 — if wrong, Flow B runs unknown code |
| A-6 | The workspace's `hianshul100_*` directories are faithful clones of the `hianshul100` forks named in the discovery backlog | directory names and `git log` in each clone | all cross-repository evidence in §3.11, §3.31, §6.3 |
| A-7 | Convey's Consul, Fabio, Jaeger, Seq, metrics and Vault integrations consume the configuration sections in the way the sibling models describe | tagged `[convey]`; modelled per service in the twelve sibling documents | §4.5, §4.6 — this document deliberately does not re-derive them |
| A-8 | The `assets/` PNGs are the same four images the README references upstream | identical file names, identical count | §3.37 only; cosmetic |

### 8.2 Blockers

| # | Blocker | Evidence | Consequence | Suggested resolution |
| --- | --- | --- | --- | --- |
| B-1 | **No bring-up could be executed to validate this model.** The workspace clones are prefixed (`hianshul100_Pacco.Services.Vehicles`), so every `../Pacco.X` and `../../Pacco.X` path in this repository misses; no container runtime is available here either | §3.2 point 6; directory listing of the workspace | Every claim in this document is derived from declarations, not from observed behaviour. Claims about Docker or image behaviour are tagged and assumed (A-1, A-2) | Validate the four topologies once in a correctly named workspace with a Docker daemon; record the result in §4 |
| B-2 | **The platform is not reproducible.** Eleven application images and ten backing-service images are referenced with no tag and no digest; there is no `.env`, no lock file and no version artefact | §3.13, §3.40; absence of `${` in `compose/` | "Which Pacco is running" is unanswerable, and rollback is impossible | §7.7 — a committed `.env` with per-image tags |
| B-3 | **The two service stacks give the gateway different behaviour.** `compose/services.yml:9` selects the asynchronous Ntrada configuration; `compose/services-local.yml:9` selects the synchronous one | the diff between the two files is exactly eleven `image`/`build` lines plus this one value | A change verified locally can behave differently when run from published images — the request/response contract at the edge changes with the stack | §7.5 — pick one, set both, document it |
| B-4 | **`compose/mongo-rabbit-redis.yml:16` runs stock `rabbitmq:3-management`**, without the Prometheus plugin and without publishing 15692, while `compose/infrastructure.yml:79` builds the custom image that has both | §3.22, §3.35 | Choosing the split data stack silently disables broker metrics; the `rabbitmq` scrape job is permanently down with no explanation | Use `build: ./rabbitmq` in the split stack, or delete the split stacks (§7.3 step 3) |
| B-5 | **The gateway's Prometheus job is named `api` in the host config and `api-gateway` in the bridge config** | `compose/host-prometheus/prometheus.yml:10` vs `compose/prometheus/prometheus.yml:10` | Any query, rule or dashboard selecting `job="api-gateway"` returns no data in host mode | Rename to `api-gateway` in the host config |
| B-6 | **Real Vault initialisation output — unseal key shares and an initial root token — is committed** | `docker-images.txt:219-232` (cited by path and line only; values not reproduced) | The material is in git history and is not removable by deleting the lines | Re-key and revoke on any Vault ever initialised with it; replace the lines with a redacted transcript; treat history as exposed |
| B-7 | **`.gitignore:55` ignores `**/Properties/launchSettings.json`, the files PM2 development mode depends on** | §3.31, §3.38 | Inert in this repository, but it states a convention that, if copied into a new service repository, breaks Flow C for that service | Remove the rule here, or add a comment explaining that service repositories must track these files |
| B-8 | **`prod-services.yml` sets no `ASPNETCORE_ENVIRONMENT`**, so the production manifest runs the base configuration profile — Docker-Desktop hostnames, default broker credentials | `prod-services.yml:1-61`; base `appsettings.json` in every sibling | A server started from this manifest cannot resolve `docker.for.win.localhost` and uses default credentials. This is the origin of the per-service finding recorded as B-2 in `pricing-service.md` and B-3 in `vehicles-service.md` | Add the variable to each `env` block and create the matching profile in each service repository |
| B-9 | **Host-networking mode loses Seq entirely.** The bridge stacks republish Seq as `5341:80`; `compose/host-infrastructure.yml:69-77` has no mapping, so Seq listens on its in-container port while every service is configured for 5341 | §3.30, §3.34 | Centralised logging silently stops in the one mode intended for host processes, and Seq binds host port 80 | Confirm Seq's in-container port, then either configure Seq's listener explicitly or change the services' Seq address for host mode |

### 8.3 Open questions

| # | Question | Why it matters | Who can answer |
| --- | --- | --- | --- |
| Q-1 | Should `Pacco.APIGateway.Ocelot` be retired from `Pacco.sln` and all five scripts, or restored to the platform's scope? | It is referenced in six files, exists in neither the README nor the discovery scope, and causes a failure in every workspace-assembly run (§3.4) | Platform owner |
| Q-2 | Which branch should a refreshed workspace end on — `master` or `develop`? | `scripts/git-pull.sh` and `scripts/git-pull-fast.sh` disagree (§3.7), so the answer depends on which script an engineer happened to run | Platform owner |
| Q-3 | Is `operations-service`'s `depends_on` block intended to guarantee anything? | As written it orders creation, not readiness, and omits both `pricing-service` and every backing service (§3.15). If nothing depends on it, deleting it removes a false guarantee | Operations service owner |
| Q-4 | Is Vault intended to be adopted, or is the container and its runbook vestigial? | The container runs in three stacks, the runbook documents PKI and dynamic Mongo credentials, and **every service has Vault disabled** (§3.27). The gap between intent and configuration is the largest in the component | Platform owner / security |
| Q-5 | Should application services keep `container_name`, given that it makes `--scale` impossible on a platform built around Consul and Fabio? | Removing it enables the discovery machinery to do what it exists for, at the cost of the DNS and Prometheus contracts (§3.43) | Platform owner |
| Q-6 | Which of the seven compose stacks is supported, and which are variants? | Only two are documented (§3.37 point 3); the other five — including both host-networking files and all three split stacks — are discoverable only by listing the directory, and two of them change behaviour (B-3, B-4). This restates Q1 of `repo-summary/Pacco.md` with the specific behavioural differences now identified | Platform owner |
| Q-7 | Is `Pacco.Web` in scope for the platform? | It appears in the discovery backlog and in `repo-summary/Pacco.Web.md`, but in none of this repository's twelve-repository lists (§3.5), and it is not modelled in `component-internals/` (see `index.md` §2) | Platform owner |

### 8.4 Cross-references, related patterns and baseline reconciliation

#### 8.4.1 Related patterns

1. [[composable-per-concern-environment-stacks]] — this component **is** the pattern's implementation;
   §3.9, §3.35 and §3.45 record where the composition is inconsistent.
2. [[independent-per-repository-release]] — §3.13, §3.40 and §4.7 give this component's end: eleven
   independently released images consumed by untagged references.
3. [[registry-mediated-discovery-and-routing]] — §3.19, §3.20 and §4.6 model the Consul/Fabio
   containers; the per-service registration end is in the sibling models.
4. [[vault-issued-dynamic-credentials-and-service-pki]] — §3.27 shows the pattern is **documented and
   not adopted**: the runbook describes PKI and dynamic Mongo credentials while every service has
   Vault disabled.
5. [[declarative-configuration-driven-api-gateway]] — §3.16 is where the gateway's configuration file
   is selected, and where the two stacks diverge.
6. [[dual-mode-edge-write]] — the sync/async gateway divergence in §3.16 is what makes this platform's
   edge mode a *deployment* choice rather than a per-route one.
7. [[correlation-and-span-propagation]] and [[structured-logging-with-property-redaction]] — §3.29 and
   §3.30 declare the Jaeger and Seq endpoints those patterns emit to, and record that both are
   ephemeral (§3.36).

#### 8.4.2 Component cross-references

1. `component-internals/api-gateway.md` — owns the consuming end of `NTRADA_CONFIG` (§3.16, §6.3 row 4).
2. `component-internals/operations-service.md` — owns the service named in the only `depends_on`
   (§3.15).
3. `component-internals/ordermaker-saga-service.md` — the service missing from both PM2 manifests
   (§3.33).
4. `component-internals/pricing-service.md` §8.2 B-2 and `component-internals/vehicles-service.md`
   §8.2 B-3 — the per-service statements of the missing production environment; **this document's B-8
   is the origin** (§3.32).
5. `component-internals/index.md` §1 and §2 — the registry updated by this batch.
6. `repo-summary/Pacco.md` — the surface catalogue of the same repository; reconciled below.
7. `baselines/architecture-baseline.md` — platform-level baseline; nothing in this model contradicts
   it.

#### 8.4.3 Reconciliation with `repo-summary/Pacco.md`

Four statements in the existing surface summary are corrected here, with evidence. The summary
remains a valid catalogue; these are the points where a reader would be misled.

| # | Existing statement | Correction | Evidence |
| --- | --- | --- | --- |
| 1 | `repo-summary/Pacco.md:34-35` — the pointer to `Pacco-sample-scenario.rest` "in the APIGateway repo … the file was not located in `Pacco.APIGateway`. **Stale doc — needs validation.**" (repeated as Q4 at line 275) | **The file exists.** It is at `hianshul100_Pacco.APIGateway/Pacco-sample-scenario.rest`, last modified in commit `b4e32c5` ("Updated rest", 2020-08-27). `README.md:66` is accurate, and Q4 can be closed | file present in the workspace; `git log` on that path |
| 2 | `repo-summary/Pacco.md:144` and `:187` — attribute the scrape target `docker.for.mac.localhost:5000/metrics-text` to the platform's Prometheus configuration and to "the gateway's" metrics endpoint | **Misattributed.** Neither string occurs in `compose/prometheus/prometheus.yml` or `compose/host-prometheus/prometheus.yml`. Both occur only in the *sample* configuration inside the runbook, at `docker-images.txt:375` (`metrics_path`) and `:377` (the target). The platform's real scrape configs use bare container names with no `metrics_path` (§3.25) and `localhost:<port>` (§3.26) | `grep` over the whole repository returns exactly those two lines |
| 3 | `repo-summary/Pacco.md:158` — ordermaker "is listed under `api-gateway`'s `depends_on` (`compose/services.yml:65`)" | **Wrong owner.** Line 65 is inside **`operations-service`**'s `depends_on`, which spans lines 59-67. `api-gateway` (lines 4-13) has no `depends_on`, and `operations-service` is the only service in either stack that has one (§3.15) | `compose/services.yml:51-67` |
| 4 | `repo-summary/Pacco.md:41`, `:94` and `:272` — "eight compose entry points" / "8 stack definitions" | **There are seven.** `compose/` contains `infrastructure.yml`, `host-infrastructure.yml`, `services.yml`, `services-local.yml`, `mongo-rabbit-redis.yml`, `consul-fabio-vault.yml` and `grafana-seq-jaeger-prometheus.yml`, plus the `prometheus/`, `host-prometheus/` and `rabbitmq/` build contexts (§3.9). Q1 at line 272 should read "seven" | directory listing of `compose/` |
| 5 | `repo-summary/Pacco.md:25` — "Clone list of twelve repositories — matches `scripts/git-clone.sh`" | **Imprecise.** Both lists have twelve entries, but they are not the same twelve: the README includes `Pacco` and omits `Pacco.APIGateway.Ocelot`; the scripts do the reverse (§3.5) | `README.md:33-44` vs `scripts/git-clone.sh:2` |

`repo-summary/Pacco.md:53` ("the README's clone list omits `Pacco.Web`") is **confirmed** by this
model and restated as Q-7.

#### 8.4.4 Explicitly unverifiable within this component

The following are marked **`Unverifiable — Missing Source Evidence`** at their point of use and are
listed here so they are not mistaken for gaps in the analysis:

1. Ntrada's resolution of the `NTRADA_CONFIG` value to a file, and whether the `.yml` suffix is
   significant (§3.16).
2. The port Seq's ingestion listener uses inside its container, and therefore whether
   `http://seq:5341` reaches it in bridge mode (§3.30).
3. Whether `jaegertracing/all-in-one` enables its Zipkin collector on 9411 without
   `COLLECTOR_ZIPKIN_HTTP_PORT` (§3.29).
4. Whether the `consul` image's default entrypoint runs a dev agent or a server (§3.19).
5. Fabio's routing behaviour given a particular Consul tag set (§3.20) — owned by the service
   repositories.
6. The runtime behaviour of any of the four topologies, since none could be executed here (B-1).

---

*Component-internals model for `platform-infrastructure-orchestration`, batch 7 of 7. Derived from
`hianshul100_Pacco` at `182c054` and from the twelve sibling clones as read-only evidence. Governed by
the maintenance contract in §7.9 and the folder conventions in `component-internals/index.md` §3.*
