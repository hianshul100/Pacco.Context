# Component internals — `operations-grpc-client`

| | |
| --- | --- |
| **Component** | `operations-grpc-client` (project `Pacco.Services.Operations.GrpcClient`) |
| **Source repository** | `hianshul100_Pacco.Services.Operations` (read-only clone; inspected, never modified) |
| **Scoped path** | `src/Pacco.Services.Operations.GrpcClient` |
| **Base ref** | `feature/12998/aidlc` |
| **Batch** | 4 of 7 |
| **Status** | New artifact — no prior `component-internals/operations-grpc-client.md` existed in this repository at the time of writing, so nothing was adopted or superseded. `component-internals/operations-service.md` §3.27, §3.30 and §3.31 already describe this project *from the server's point of view*; that material remains valid and is **complemented, not replaced**, by this document, which models the client's own internals. `baselines/service-summaries.md` §2.5 (`Pacco.Services.Operations.GrpcClient`) and `baselines/api-inventory.md` §5 remain valid. **Write-target note:** the workspace metadata for the run that produced this file marked *every* clone read-only, including `hianshul100_Pacco.Context` itself. Writing here follows the stage instruction naming this the writable architecture/ADR repository, and the precedent of the prior component-internals commits on this branch — it is not a constraint violation. `hianshul100_Pacco.Services.Operations` was read only, never modified. |
| **Grounding** | Every load-bearing claim cites a file and, where relevant, a line range. Statements that cannot be settled from source in this workspace are marked **`Unverifiable — Missing Source Evidence`**. |

> **Scope of verifiability.** The scoped path contains the component in full: **three tracked
> files** — `Program.cs` (**151 lines**), `Operations.proto` (24 lines) and
> `Pacco.Services.Operations.GrpcClient.csproj` (20 lines). Line counts are `awk 'END{print NR}'`
> counts; note that `Operations.proto` has **no trailing newline**, so `wc -l` reports 23 for a file
> whose last line (`}`) is line 24. There is no configuration file, no
> `appsettings.json`, no `Dockerfile`, no test project and no README of its own. Everything this
> component does is in those three files, so the model below is **complete by exhaustion** rather
> than by sampling.
>
> Two dependency families supply mechanisms whose source is *not* in this workspace and are marked
> accordingly: `[grpc]` — `Grpc.Net.Client` 2.28.0, `Grpc.Core` (the `AsyncServerStreamingCall` /
> `IAsyncStreamReader` types), `Grpc.Tools` 2.28.1 and `Google.Protobuf` 3.11.4; `[framework]` —
> `System.Net.Http.HttpClientHandler` and the .NET Core 3.1 console host. `[newtonsoft]` marks
> `Newtonsoft.Json` 12.0.3. Where a mechanism from those packages changes a conclusion, the
> conclusion is flagged.
>
> The **server half** of every contract described here lives in
> `src/Pacco.Services.Operations.Api` and is modelled in
> [`operations-service.md`](operations-service.md); this document does not re-derive it, it cites
> it.

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

`operations-grpc-client` is an **interactive console driver for the `operations-service` gRPC
surface**. It is the only gRPC *consumer* anywhere in the fourteen cloned repositories, and its
entire job is to let a human at a terminal exercise the two RPCs declared in `Operations.proto`:

1. **`GetOperation`** — a unary call, driven by option `1` (`Program.cs:26,116-133`).
2. **`SubscribeOperations`** — a server-streaming call, driven by option `2`
   (`Program.cs:27,135-146`).

It is a *driver*, not a library and not a service. It has:

- `<OutputType>Exe</OutputType>` (`Pacco.Services.Operations.GrpcClient.csproj:6`) — it is a second
  **deployable-shaped** artifact of the Operations repository;
- a `Main` that blocks on `Console.ReadLine()` in a loop (`Program.cs:78-114`) — it cannot run
  unattended;
- no configuration surface beyond two positional command-line arguments (`Program.cs:51-76`).

Its architectural significance is disproportionate to its size: **it is the only executable
specification of how the `operations-service` gRPC contract is meant to be consumed.** Three
behaviours of that contract are stated nowhere else in the workspace and are only discoverable by
reading this client — the default endpoint (`https://localhost:50050`, §3.7), the not-found
sentinel (a blank `id`, §3.13) and the fact that TLS validation must be bypassed to talk to the
service locally (§3.8).

### 1.2 What it explicitly is not

| It is not | Evidence |
| --- | --- |
| A deployed platform component | `Dockerfile:4` publishes **only** `src/Pacco.Services.Operations.Api`. It has no `container_name` in `hianshul100_Pacco/compose/services.yml`, no image in `scripts/dockerize.sh` (which tags exactly one repository, `$DOCKER_USERNAME/pacco.services.operations`), and no PM2 entry in `hianshul100_Pacco/services.yml` or `prod-services.yml`. |
| A reusable client library | It exposes no public type of its own. `Program` is `internal` by default (`Program.cs:14`, no accessibility modifier) and every member is `private static`. Consumers would have to take the generated stub, not this project. |
| A NuGet-publishable contract package | The `.proto` is **copied**, not referenced: `Operations.proto` exists twice, at `src/Pacco.Services.Operations.Api/Operations.proto` and `src/Pacco.Services.Operations.GrpcClient/Operations.proto`, byte-comparable and independently compiled (§3.3). |
| A monitoring or alerting tool | It writes to `Console` only (`Program.cs:47,88,97,112,120,127,131,137,142,149`). There is no logger, no metric, no exit code other than the default `0`, and no persistence of anything it receives. |
| Authenticated | No `CallCredentials`, no `ChannelCredentials` other than the default TLS channel, no `Metadata` header and no token appear anywhere in `Program.cs`. See §3.17 — and note the server does not require any (`operations-service.md` §1.2). |
| Correlation- or trace-aware | No correlation id, no `span_context`, no `traceparent`. See §3.18. |
| Under test | The repository has **no `tests/` directory and no `*.Tests.csproj`**, yet `scripts/test.sh` runs `dotnet test` and `.travis.yml:12-14` invokes it. See §3.20. |

### 1.3 Position in the platform

| Relationship | Detail | Evidence |
| --- | --- | --- |
| Calls | `operations-service` gRPC, methods `GetOperation` and `SubscribeOperations` | `Program.cs:121,138`; server at `Infrastructure/GrpcServiceHost.cs:24,32` |
| Called by | **nothing** | no project in the workspace references `Pacco.Services.Operations.GrpcClient.csproj`; it is referenced only as a solution member (`Pacco.Services.Operations.sln:10`) |
| Shares a contract with | `src/Pacco.Services.Operations.Api/Operations.proto` — by **copy**, not by reference | §3.3 |
| Built by CI | yes — `scripts/build.sh` is `dotnet build -c release` at the repository root, which builds the whole solution including this project (`.travis.yml:12`) | §3.22 |
| Packaged by CI | **no** | `scripts/dockerize.sh`, `Dockerfile:4` |
| Deployed | **no** — see §1.2 | |
| Reachable target in Docker | **no** — the container binds only `http://*:80` (`Dockerfile:9`) and compose maps `5005:80`; port `50050` exists only in `src/Pacco.Services.Operations.Api/Properties/launchSettings.json:20`, i.e. `dotnet run` on a developer machine | §3.7, `operations-service.md` §3.44 |

The last row is the single most consequential fact about this component: **its default target does
not exist in any containerised environment.** It works against `dotnet run`, and only against
`dotnet run`.

---

## 2. Core concepts (exhaustive)

Every significant concept this component defines or owns. Nothing is omitted for brevity; the
component is small enough that this list is exhaustive rather than representative.

| § | Concept | Kind | Primary evidence |
| --- | --- | --- | --- |
| 3.1 | The console executable and its entry point | host | `Program.cs:30-49`, `.csproj:6` |
| 3.2 | Target framework and dependency set | build | `.csproj:4,9-14` |
| 3.3 | The duplicated `.proto` contract | contract | `Operations.proto`, `../Pacco.Services.Operations.Api/Operations.proto` |
| 3.4 | Build-time protobuf code generation | build | `.csproj:16-18`, `scripts/proto/*.sh` |
| 3.5 | The generated client stub `GrpcOperationsServiceClient` | contract | `Program.cs:16,46` |
| 3.6 | `_client` — the static, process-wide stub instance | state | `Program.cs:16,46` |
| 3.7 | Address resolution (`GetAddress`) | config | `Program.cs:32,51-76` |
| 3.8 | The TLS-validation bypass | security | `Program.cs:34-40` |
| 3.9 | Channel construction and lifetime | transport | `Program.cs:42-45` |
| 3.10 | `Actions` — the option dispatch table | control | `Program.cs:24-28` |
| 3.11 | `InitAsync` — the REPL | control | `Program.cs:78-114` |
| 3.12 | The unary `GetOperation` call | operation | `Program.cs:116-133` |
| 3.13 | The blank-`id` not-found sentinel | protocol convention | `Program.cs:125-129` |
| 3.14 | The server-streaming `SubscribeOperations` call | operation | `Program.cs:135-146` |
| 3.15 | `DisplayOperation` and the JSON render settings | presentation | `Program.cs:18-22,148-149` |
| 3.16 | The `state` field's wire encoding | contract semantics | `Operations.proto:21`, server `GrpcServiceHost.cs:53` |
| 3.17 | Absence of authentication and call credentials | security | whole of `Program.cs` |
| 3.18 | Absence of correlation, tracing and logging | observability | whole of `Program.cs` |
| 3.19 | Absence of configuration and secrets | config | no `appsettings*.json` in the scoped path |
| 3.20 | Absence of tests, and the CI step that pretends otherwise | quality | `scripts/test.sh`, `.travis.yml:12-14` |
| 3.21 | Deployment posture — built, never packaged | deployment | `Dockerfile:4`, `scripts/dockerize.sh` |
| 3.22 | Solution membership and build inclusion | build | `Pacco.Services.Operations.sln:10,37-48` |
| 3.23 | Concurrency and cancellation model | runtime | `Program.cs:30,102,138-145` |
| 3.24 | Error handling — the catch-all around every action | resilience | `Program.cs:100-107` |
| 3.25 | Termination semantics and undisposed resources | runtime | `Program.cs:85-113` |

---

## 3. Per concept

### 3.1 The console executable and its entry point

**Definition.** A single-class .NET Core console application. `Program.Main` is the whole
composition root: it resolves an address, builds an `HttpClient`, builds a `GrpcChannel`, builds
the generated stub, prints a banner and hands control to the REPL.

```csharp
static async Task Main(string[] args)
{
    var address = GetAddress(args);
    var httpClientHandler = new HttpClientHandler { ServerCertificateCustomValidationCallback = … };
    var httpClient = new HttpClient(httpClientHandler);
    var channel = GrpcChannel.ForAddress(address, new GrpcChannelOptions { HttpClient = httpClient });
    _client = new GrpcOperationsService.GrpcOperationsServiceClient(channel);
    Console.WriteLine($"Created a GRPC client for an address: '{address}'");
    await InitAsync();
}
```
(`Program.cs:30-49`)

**Representation & storage.** Compiled to `Pacco.Services.Operations.GrpcClient.dll` with
`OutputType=Exe` (`.csproj:6`). No state is persisted anywhere — the process holds everything in
memory and forgets it on exit (§5).

**Lifecycle.** Created by `dotnet run` / `dotnet <dll>`; there is **no registration step of any
kind** — no DI container, no `IHostBuilder`, no `Startup`. Compare `operations-service`, whose
entry point is a `WebHost` with a full Convey service graph
(`../Pacco.Services.Operations.Api/Program.cs:21-52`). This project deliberately has none of it.

**Invariants & enforcement.**

- `async Task Main` requires C# 7.1+; the project sets `<LangVersion>latest</LangVersion>`
  (`.csproj:5`), so this is satisfied. A downgrade of `LangVersion` **fails the build loudly**.
- There is no guard that `_client` was assigned before `InitAsync` runs; the ordering at
  `Program.cs:46-48` is the only thing that makes it safe. Reordering those two lines yields a
  `NullReferenceException` on the first option — **loud, but at runtime, not at compile time**
  (`_client` is a nullable reference by C# 7.3 rules; nullable reference types are not enabled).

**Extension procedure.** To add startup behaviour (a config file, a logger, a DI container) you
must add it to `Main` by hand; there is no extension point. To add a *new operation*, see §3.10 —
that is the one extension the design does anticipate.

**Failure modes.** Any exception thrown from `GetAddress`, the handler/channel construction or the
banner escapes `Main` uncaught, and the .NET host prints the stack trace and exits with a non-zero
code. Only exceptions thrown **inside an action** are contained (§3.24).

---

### 3.2 Target framework and dependency set

**Definition.** The build contract of the component.

| Item | Value | Evidence |
| --- | --- | --- |
| SDK | `Microsoft.NET.Sdk` (**not** `.Web`) | `.csproj:1` |
| `TargetFramework` | `netcoreapp3.1` | `.csproj:4` |
| `LangVersion` | `latest` | `.csproj:5` |
| `OutputType` | `Exe` | `.csproj:6` |
| `Google.Protobuf` | 3.11.4 | `.csproj:10` |
| `Grpc.Net.Client` | 2.28.0 | `.csproj:11` |
| `Grpc.Tools` | 2.28.1 | `.csproj:12` |
| `Newtonsoft.Json` | 12.0.3 | `.csproj:13` |

**Representation & storage.** `Pacco.Services.Operations.GrpcClient.csproj`, 20 lines, no
`Directory.Build.props` anywhere in the repository to inherit from (verified: no such file in the
clone).

**Lifecycle.** Changed only by editing the `.csproj`. Versions are **pinned exactly** — unlike the
sibling API project, which floats every Convey package on `0.4.*`
(`../Pacco.Services.Operations.Api/Pacco.Services.Operations.Api.csproj:9-33`). This client is the
more reproducible of the two builds.

**Invariants & enforcement.**

- `Grpc.Net.Client` 2.28.0 targets `netstandard2.1`, satisfied by `netcoreapp3.1`. A retarget to
  `netstandard2.0`/`net472` **fails loudly** at restore.
- `Grpc.Tools` is referenced **without `PrivateAssets="All"`** (`.csproj:12`). For a leaf
  executable this is harmless, but if this project were ever turned into a library the build-time
  code generator would flow transitively to its consumers. This is a latent packaging defect, not
  a current one.
- `Newtonsoft.Json` is used for exactly one thing — pretty-printing responses (§3.15). It is not
  used for any wire format; protobuf handles the wire.
- The client's `Grpc.Net.Client` 2.28.0 and `Grpc.Tools` 2.28.1 do **not** match the manual
  compile scripts, which reference `Grpc.Tools.1.22.0` (`scripts/proto/lin-compile.sh:1-2`) — see
  §3.4.
- `netcoreapp3.1` reached end of support on 2022-12-13 (external fact, not derivable from this
  workspace); no file in the workspace records a migration plan.
  **`Unverifiable — Missing Source Evidence`** as to whether a migration is intended.

**Extension procedure.** Add a `PackageReference`; there is no lock file, no central package
management and no `packages.config` to keep in step.

**Failure modes.** A `Grpc.Net.Client` major upgrade past 2.x changes `GrpcChannel.ForAddress`
options; the compile breaks loudly. A silent risk exists only around `Grpc.Tools` (§3.4).

---

### 3.3 The duplicated `.proto` contract

**Definition.** `Operations.proto` (24 lines) declares the service and three messages this client
consumes:

```proto
syntax = "proto3";
package Services.Operations;

service GrpcOperationsService {
    rpc GetOperation (GetOperationRequest) returns (GetOperationResponse) {}
    rpc SubscribeOperations (Empty) returns (stream GetOperationResponse) {}
}

message Empty {}
message GetOperationRequest { string id = 1; }
message GetOperationResponse {
    string id = 1; string userId = 2; string name = 3;
    string state = 4; string code = 5; string reason = 6;
}
```
(`Operations.proto:1-24`)

**Representation & storage.** A **second physical copy** of the server's IDL. The authoritative
copy is `src/Pacco.Services.Operations.Api/Operations.proto`; the two files are currently
identical in content (same package, same service name, same six fields with the same numbers).
They are compiled independently into two different assemblies, both emitting the CLR namespace
`Services.Operations` (`Program.cs:10` imports it).

**Lifecycle.** Hand-edited. There is **no generation step, no shared package, no submodule and no
copy script** that keeps the two files in step — `scripts/proto/{lin,mac,win}-compile.sh` compile
*each copy separately* (`lin-compile.sh:7-8`), they do not synchronise them.

**Invariants & enforcement.**

| Invariant | Enforced where | Violation behaviour |
| --- | --- | --- |
| Field numbers must match the server's | **nowhere** | **Silent.** protobuf matches on field *number*, not name. Renumbering one copy makes the client read `userId` out of the `name` slot with no error. |
| Field names must match the server's | **nowhere** | **Silent for the wire**, loud for the code: a renamed field breaks `Program.cs` compilation but does not break interop. |
| Package/service names must match | the wire, at call time | **Loud** — a mismatched `package` or `service` produces a gRPC `UNIMPLEMENTED` status, because the method path is `/Services.Operations.GrpcOperationsService/GetOperation`. |
| New fields must use numbers ≥ 7 | **nowhere** | **Silent** misparse if violated. |
| `Empty` is hand-declared, not `google/protobuf/empty.proto` | `Operations.proto:10-11` | A client that imports the well-known `Empty` sends `google.protobuf.Empty`, which is wire-compatible (both are zero-byte messages) but is a *different type name*; since the type name is not on the wire for a unary/stream request body, this happens to work. Documented here because it looks like a bug and is not. |

**Extension procedure.** To add a field to `GetOperationResponse`:

1. Add it to `src/Pacco.Services.Operations.Api/Operations.proto` with the next free number (7+).
2. **Copy the identical change into `src/Pacco.Services.Operations.GrpcClient/Operations.proto`.**
   Nothing does this for you and nothing checks it.
3. Populate it in the server's mapper, `Infrastructure/GrpcServiceHost.cs:46-53`.
4. Rebuild both projects (code generation is automatic, §3.4).
5. Optionally surface it — `DisplayOperation` (§3.15) serialises the whole message reflectively, so
   a new field appears in the console output **without** any client code change.

Step 5 is the one piece of good news in this design: the client's *display* path is
schema-agnostic, so field additions need no client edit beyond the `.proto` copy.

**Failure modes.**

- **Drift.** The two copies are the component's principal structural risk. A server-side field
  addition that is not copied here means this client silently ignores the new field (protobuf
  skips unknown fields); a *renumbering* means it misreads existing ones.
- **Deletion of the client's copy** breaks the build loudly (`.csproj:17` requires the file).

---

### 3.4 Build-time protobuf code generation

**Definition.** How `Operations.proto` becomes C#.

```xml
<ItemGroup>
    <Protobuf Include="Operations.proto" />
</ItemGroup>
```
(`.csproj:16-18`)

**Representation & storage.** `Grpc.Tools` 2.28.1 (`.csproj:12`) hooks the build and emits
`Operations.cs` + `OperationsGrpc.cs` into `obj/Debug|Release/netcoreapp3.1/` — **not** into source
control. No generated file is tracked in the clone (verified: the scoped path contains three files
only).

**Lifecycle.** Regenerated on **every build**. There are also three *manual* scripts, which are
vestigial:

```
PROTOC=tools/Grpc.Tools.1.22.0/linux_x64/protoc
$PROTOC --csharp_out $SERVER --grpc_out $SERVER … $SERVER/$PROTO
$PROTOC --csharp_out $CLIENT --grpc_out $CLIENT … $CLIENT/$PROTO
```
(`scripts/proto/lin-compile.sh:1-8`; identical `mac-compile.sh`, `win-compile.sh`)

These write generated C# **into the source directories**, use `Grpc.Tools` **1.22.0** (six minor
versions behind the `.csproj`'s 2.28.1), and depend on a `tools/` directory that **does not exist
in the clone**. They are dead scripts kept for historical reasons.

**Invariants & enforcement.**

- The `<Protobuf>` item has no `GrpcServices` attribute, so `Grpc.Tools` defaults to
  `GrpcServices="Both"` `[grpc]` — meaning this *client* project also generates
  `GrpcOperationsService.GrpcOperationsServiceBase`, a **server** base class it never implements.
  Harmless dead code, but it explains why the generated output is larger than a client needs. Set
  `GrpcServices="Client"` to trim it.
- If someone runs `scripts/proto/*-compile.sh` successfully, the emitted `.cs` files land *beside*
  the `.proto` and are then **also** compiled by the SDK's default glob, producing duplicate type
  definitions and a **loud** `CS0101` build failure. The scripts are therefore not merely dead —
  running them breaks the build.
- The generated CLR namespace is `Services.Operations`, derived from `package Services.Operations;`
  (`Operations.proto:3`), and is imported at `Program.cs:10`. Changing the `package` line breaks
  the `using` **loudly**.

**Extension procedure.** Add another `.proto` under `<ItemGroup>` alongside the existing
`<Protobuf Include>`; nothing else is required. To stop generating the unused server base, add
`GrpcServices="Client"` to the item.

**Failure modes.** A missing `Grpc.Tools` reference produces no generated types and a wall of
`CS0246` errors — loud. Running the manual scripts produces `CS0101` — loud. The dangerous case is
the one the build *cannot* catch: the `.proto` copies drifting (§3.3).

---

### 3.5 The generated client stub `GrpcOperationsServiceClient`

**Definition.** The strongly-typed proxy generated from the `service` block: a nested class
`GrpcOperationsService.GrpcOperationsServiceClient` `[grpc]`, offering (among the standard
overloads) `GetOperationAsync(GetOperationRequest)` returning an `AsyncUnaryCall<GetOperationResponse>`
and `SubscribeOperations(Empty)` returning an `AsyncServerStreamingCall<GetOperationResponse>`.

**Representation & storage.** Generated into `obj/` (§3.4); referenced at `Program.cs:16` (field
declaration) and `Program.cs:46` (construction).

**Lifecycle.** Constructed **once**, from the channel, at `Program.cs:46`, and never replaced or
disposed. It is not a disposable type — the *channel* is (§3.9).

**Invariants & enforcement.**

- `GetOperationAsync` returns `AsyncUnaryCall<T>`, which is awaitable directly because it defines a
  `GetAwaiter` `[grpc]`; `Program.cs:121` awaits it without calling `.ResponseAsync`. Both forms
  are equivalent; the awaited form discards the response headers, trailers and status.
  **Consequence: this client cannot read a gRPC status code or trailing metadata** on the unary
  path — it only ever sees the message or an exception.
- `SubscribeOperations` returns a **disposable** `AsyncServerStreamingCall<T>`; the code correctly
  wraps it in `using` (`Program.cs:138`).
- The stub is not thread-safe to *replace* but is safe to *call* concurrently `[grpc]`; the REPL
  never calls concurrently anyway (§3.23).

**Extension procedure.** Adding an `rpc` to the `.proto` regenerates the stub with a new method;
then wire it into `Actions` (§3.10).

**Failure modes.** If the channel is unusable (unreachable host, TLS failure), the stub throws
`RpcException` on the *call*, not on construction — so the banner at `Program.cs:47` prints
"Created a GRPC client for an address: …" even when nothing is listening. **The success message is
not evidence of connectivity.**

---

### 3.6 `_client` — the static, process-wide stub instance

**Definition.** `private static GrpcOperationsService.GrpcOperationsServiceClient _client;`
(`Program.cs:16`).

**Representation & storage.** A mutable static field. There is no DI container, no lifetime scope
and no factory; the entire component's dependency graph is this one field plus two static readonly
tables (§3.10, §3.15).

**Lifecycle.**

| Phase | Where |
| --- | --- |
| Declared (null) | `Program.cs:16` |
| Assigned | `Program.cs:46`, after channel construction |
| Read | `Program.cs:121` (unary), `Program.cs:138` (stream) |
| Released | never — the process exits |

**Invariants & enforcement.** "Assign before use" is enforced only by statement order in `Main`
(`Program.cs:42-48`). Nothing declares it, no null-check exists, and nullable reference types are
not enabled (`.csproj` sets no `<Nullable>`), so the compiler cannot warn.

**Extension procedure.** If a second endpoint or a second channel were ever needed (e.g. to talk to
two `operations-service` replicas), the static field becomes the obstacle: it would have to become
a dictionary or be passed as a parameter. This is the design's principal scaling limit and it is
cheap to fix precisely because the component is 151 lines.

**Failure modes.** `NullReferenceException` if the ordering in `Main` is disturbed — loud, at the
first call, not at startup.

---

### 3.7 Address resolution (`GetAddress`)

**Definition.** The component's entire configuration mechanism: two optional positional arguments,
host and port, with hard-coded defaults.

```csharp
private static string GetAddress(string[] args)
{
    var host = string.Empty; var port = 0;
    if (args?.Any() == true && args.Length >= 2)
    {
        host = args[0];
        if (int.TryParse(args[1], out var providedPort)) { port = providedPort; }
    }
    if (string.IsNullOrWhiteSpace(host)) { host = "localhost"; }
    if (port <= 0) { port = 50050; }
    return $"https://{host}:{port}";
}
```
(`Program.cs:51-76`)

**Representation & storage.** Nothing is stored. The resolved value is a local `string` in `Main`
and is embedded in the channel.

**Lifecycle.** Evaluated once, at `Program.cs:32`. There is no re-resolution, no reload and no
`IOptions`-style binding — contrast the sibling API project, which binds five `appsettings*.json`
profiles through Convey.

**Invariants & enforcement.**

| Rule | Enforcement | Violation behaviour |
| --- | --- | --- |
| Both arguments or neither | `args.Length >= 2` (`:56`) | **Silent.** Passing a single argument — e.g. `dotnet run operations-service` — is **ignored entirely**; the host reverts to `localhost`. There is no message, no warning, no usage text. This is the component's most user-hostile behaviour. |
| Port must parse as `int` | `int.TryParse` (`:59`) | **Silent.** A non-numeric port leaves `port = 0`, which then falls through to the default `50050` (`:70-73`). `client foo abc` connects to `https://foo:50050` without comment. |
| Port must be positive | `port <= 0` (`:70`) | **Silent** fallback to `50050`. A deliberate `0` (meaning "any port") is impossible to express. |
| Scheme is always `https` | string interpolation (`:75`), unconditional | **Silent and absolute.** There is **no way to target a plaintext HTTP/2 endpoint** — which is exactly what the deployed container exposes (`Dockerfile:9` binds `http://*:80`). See Failure modes. |

**Extension procedure.** To make this client usable against the containerised service you must
change `GetAddress` — either accept a full URI as `args[0]`, or add a scheme argument, or set
`AppContext.SetSwitch("System.Net.Http.SocketsHttpHandler.Http2UnencryptedSupport", true)` and emit
`http://` `[framework]`. All three are edits to `Program.cs`; there is no configuration path.

**Failure modes.**

- **The default target `https://localhost:50050` only exists under `dotnet run`.** It comes from
  `src/Pacco.Services.Operations.Api/Properties/launchSettings.json:20`
  (`"applicationUrl": "http://localhost:5005;https://localhost:50050"`), which the Docker image does
  not use. Against `docker-compose` the client fails with a connection error.
- Because the scheme is hard-coded to `https`, pointing the client at the compose-mapped
  `localhost:5005` fails with a TLS handshake error rather than an obvious "wrong protocol"
  message.

---

### 3.8 The TLS-validation bypass

**Definition.**

```csharp
// Only for the local development purposes.
var httpClientHandler = new HttpClientHandler
{
    ServerCertificateCustomValidationCallback =
        HttpClientHandler.DangerousAcceptAnyServerCertificateValidator
};
```
(`Program.cs:34-39`)

**Representation & storage.** A `HttpClientHandler` instance, wrapped in an `HttpClient`
(`:40`), handed to the channel (`:42-45`).

**Lifecycle.** Constructed once in `Main`; lives for the process.

**Invariants & enforcement.** There is **no switch, no environment check and no build
configuration** guarding it. The comment at `:34` states the intent ("only for local development");
nothing enforces it. If this binary were ever run against a real endpoint it would accept **any**
certificate, including one presented by a man-in-the-middle.

Why it exists: the ASP.NET Core dev-certificate that `dotnet run` presents on
`https://localhost:50050` is self-signed, and gRPC-over-HTTP/2 in .NET requires TLS unless
unencrypted HTTP/2 is explicitly opted into `[framework]`. The bypass is the shortest path to a
working demo.

**Extension procedure.** To harden: gate the callback behind an argument or
`#if DEBUG`, or trust the dev certificate via `dotnet dev-certs https --trust` and delete the
handler entirely. Deleting it is a one-line change — `GrpcChannel.ForAddress(address)` without
`GrpcChannelOptions` is sufficient.

**Failure modes.** Not a runtime failure — a security posture. It is recorded here because
`component-internals` is the document downstream phases read before reusing code: **this pattern
must not be copied into any component that talks to a non-local endpoint.**

---

### 3.9 Channel construction and lifetime

**Definition.**

```csharp
var channel = GrpcChannel.ForAddress(address, new GrpcChannelOptions { HttpClient = httpClient });
```
(`Program.cs:42-45`)

**Representation & storage.** A local variable in `Main`. It is **not** stored in a field, **not**
disposed, and **not** shut down.

**Lifecycle.** Created before the stub, captured by the stub, and released only when the process
exits.

**Invariants & enforcement.**

- Supplying `HttpClient` via `GrpcChannelOptions` means the channel does **not** own the handler's
  lifetime and will not dispose it `[grpc]`. Neither is disposed here either, so the socket pool is
  released by process exit. For a short-lived console tool this is acceptable; it is called out
  because copying the pattern into a long-lived host leaks connections.
- No `GrpcChannelOptions.MaxReceiveMessageSize`, `Credentials`, `LoggerFactory`,
  `HttpVersion` or retry/`ServiceConfig` is set — every one of them takes its `Grpc.Net.Client`
  2.28.0 default `[grpc]`.
- **No deadline and no retry policy exist anywhere in this component.** A hung server blocks the
  REPL indefinitely on the unary call.

**Extension procedure.** Add options to the `GrpcChannelOptions` initialiser at `:42-45`; add a
`CallOptions`/`deadline` argument at the two call sites (`:121`, `:138`) for per-call control.

**Failure modes.** Unreachable endpoint → `RpcException` with `StatusCode.Unavailable` at the first
call, caught and printed by §3.24. TLS mismatch → the same, wrapped. Neither is distinguished; the
user sees a raw `ToString()` of the exception.

---

### 3.10 `Actions` — the option dispatch table

**Definition.** The component's one genuine extension point.

```csharp
private static readonly IDictionary<string, Func<Task>> Actions = new Dictionary<string, Func<Task>>
{
    ["1"] = GetOperationAsync,
    ["2"] = SubscribeOperationsStreamAsync,
};
```
(`Program.cs:24-28`)

**Representation & storage.** A `static readonly` dictionary from menu key → parameterless async
delegate, initialised at type-load time.

**Lifecycle.** Immutable in practice — declared `readonly` on the field, though the *dictionary
itself* is mutable (`IDictionary<,>`, not `IReadOnlyDictionary<,>`); nothing mutates it.

**Invariants & enforcement.**

| Invariant | Enforced where | Violation behaviour |
| --- | --- | --- |
| Every key must appear in the printed menu | **nowhere** | **Silent.** The menu is a hand-written `const string` (`:80-83`) listing "Options (1-2)". Adding `["3"] = …` to the table without editing the string yields a working-but-invisible option. |
| Every menu entry must have a key | **nowhere** | **Silent** the other way: a menu line with no table entry falls to `"Invalid option: {option}"` (`:112`). |
| Keys are compared with the default string comparer | `Dictionary<string,…>` default | Ordinal, case-sensitive. Irrelevant for `"1"`/`"2"`; it matters the moment a letter key is added. |
| `"q"` must not be a key | **nowhere** | Adding `["q"]` would make the loop invoke it *and then* exit, because `option != "q"` is evaluated at the top of the next iteration (`:86`). |

**Extension procedure — the exact steps to add an operation:**

1. Add the `rpc` to **both** `Operations.proto` copies (§3.3) and implement it on the server
   (`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs`).
2. Write a `private static async Task <Name>Async()` method that reads whatever input it needs from
   `Console` and calls `_client`.
3. Add `["3"] = <Name>Async,` to `Actions` (`Program.cs:24-28`).
4. **Edit the `message` constant at `Program.cs:80-83`** — both the "Options (1-2)" range text and
   the new numbered line. *If you skip this step the option works but nobody can discover it; there
   is no validation that links the two.*

**Failure modes.** Divergence between the table and the menu string, in either direction, is silent
(above). Nothing else about the table can fail.

---

### 3.11 `InitAsync` — the REPL

**Definition.** The read-eval-print loop that is the component's whole user interface.

```csharp
var option = string.Empty;
while (option != "q")
{
    Console.WriteLine(message);
    Console.Write("> ");
    option = Console.ReadLine();
    if (string.IsNullOrWhiteSpace(option)) { Console.WriteLine("Missing option"); continue; }
    Console.WriteLine();
    if (Actions.ContainsKey(option))
    {
        try { await Actions[option](); }
        catch (Exception ex) { Console.WriteLine(ex); }
        continue;
    }
    Console.WriteLine($"Invalid option: {option}");
}
```
(`Program.cs:85-113`)

**Representation & storage.** Pure control flow; the only state is the local `option` string.

**Lifecycle.** Entered once from `Main` (`:48`), exits when the user types exactly `q`.

**Invariants & enforcement.**

- **Exit is exact-match, case-sensitive.** `Q`, `quit`, `exit` and `:q` all fall through to
  `"Invalid option"` (`:112`). `Ctrl+C` is the practical alternative and is unhandled — no
  `CancelKeyPress` handler exists, so the runtime terminates the process abruptly.
- `Console.ReadLine()` returns `null` on EOF (a closed stdin, e.g. `dotnet run < /dev/null`).
  `string.IsNullOrWhiteSpace(null)` is `true`, so the loop prints `"Missing option"` and
  `continue`s — **and then reads `null` again, forever.** Piping input into this component
  produces an **infinite output loop**, not a clean exit. This is a real defect, reachable by any
  attempt to script the tool.
- A blank line is tolerated (`"Missing option"`), an unknown token is tolerated
  (`"Invalid option"`), and an action's exception is tolerated (§3.24). The loop is
  **unbreakable except by `q`, `Ctrl+C`, or a process kill.**
- `Actions.ContainsKey(option)` followed by `Actions[option]()` is a double lookup; behaviourally
  irrelevant, mentioned only so a reader does not assume a `TryGetValue` contract.

**Extension procedure.** To make the tool scriptable — the most likely evolution — the loop needs
(a) an EOF guard (`if (option is null) break;`), (b) a non-interactive mode that runs one action
from `args` and exits, and (c) an exit code. All three live in this method and in `GetAddress`
(§3.7); nothing else in the component would change.

**Failure modes.** The EOF spin above; and the fact that option `2` never returns (§3.14), which
means the REPL is a one-shot menu in practice: choosing "subscribe" ends the session's ability to
do anything else.

---

### 3.12 The unary `GetOperation` call

**Definition.** Option `1`.

```csharp
Console.Write("Type the operation id: ");
var id = Console.ReadLine();
Console.WriteLine("Sending the request...");
var response = await _client.GetOperationAsync(new GetOperationRequest { Id = id });
```
(`Program.cs:116-124`)

**Representation & storage.** Nothing persisted. `id` is a local string; the response is a local
message object printed and discarded.

**Lifecycle.** One call per menu selection.

**Server-side effect.** `GrpcServiceHost.GetOperation` logs the peer and calls
`IOperationsService.GetAsync(request.Id)`, which reads the Redis key `requests:{id}` and
deserialises `OperationDto`
(`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs:24-30`,
`Services/OperationsService.cs:24-29,62`). **The call is read-only** — it mutates nothing, and it
does not extend the sliding expiry, because `GetStringAsync` on `IDistributedCache` refreshes the
sliding window `[framework]` — see `operations-service.md` §3.5 for the full treatment of that
subtlety.

**Invariants & enforcement.**

| Invariant | Enforcement | Violation behaviour |
| --- | --- | --- |
| `id` must be non-null | **none** | `GetOperationRequest.Id = null` throws `ArgumentNullException` from the generated protobuf setter `[grpc]` — `string` fields in proto3 reject null. Pressing *Enter* at the prompt sends `""` (not null) and is accepted; EOF sends `null` and throws, caught by §3.24. |
| `id` must be a GUID | **none, on either side** | **Silent.** The server treats it as an opaque cache-key suffix (`requests:{id}`), so `banana` is a valid lookup that simply misses. |
| `id` must exist | **none** | See §3.13 — a miss is a *successful* RPC with an empty message. |
| Response must be complete | **none** | Awaiting `AsyncUnaryCall<T>` discards status/trailers (§3.5). |

**Extension procedure.** To validate input, parse with `Guid.TryParse` before constructing the
request. To surface the gRPC status, capture the call (`var call = _client.GetOperationAsync(…)`)
and read `call.GetStatus()` after `await call.ResponseAsync`.

**Failure modes.** Server down → `RpcException(Unavailable)`, printed raw. Server throws → the
server's exception becomes `RpcException(Unknown)` with a generic message; note the service has
**no gRPC exception interceptor** (`operations-service.md` §3.30), so no structured error detail
ever reaches this client.

---

### 3.13 The blank-`id` not-found sentinel

**Definition.** The client's — and therefore the contract's — encoding of "not found".

```csharp
if (string.IsNullOrWhiteSpace(response.Id))
{
    Console.WriteLine($"* Operation was not found for id: {id} *");
    return;
}
Console.WriteLine($"* Operation was found for id: {id} *");
```
(`Program.cs:125-132`)

**Representation & storage.** A convention, not a type. It exists because the server maps a `null`
DTO to a **default-constructed** `GetOperationResponse`:

```csharp
private static GetOperationResponse Map(OperationDto operation)
    => operation is null ? new GetOperationResponse() : new GetOperationResponse { … };
```
(`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs:43-54`)

In proto3 every `string` field defaults to `""`, never `null`, so an empty response is
indistinguishable from a real operation whose `id` is empty — which cannot happen, because
`OperationsService.TrySetAsync` always assigns `operation.Id = id`
(`Services/OperationsService.cs:44`).

**Lifecycle.** Evaluated per response; nothing is stored.

**Invariants & enforcement.**

- **The convention is enforced nowhere and documented nowhere.** It is not in the `.proto`, not in
  a comment, and not in the service's README. A second client that checks `response == null` or
  expects `StatusCode.NotFound` will report *every* miss as a found operation with blank fields.
  This is the single most important undocumented fact about the gRPC surface, and this 4-line
  block is its only expression in the workspace.
- The idiomatic alternative — `throw new RpcException(new Status(StatusCode.NotFound, …))` — is
  **not** used; contrast the HTTP surface, which does return 404
  (`../Pacco.Services.Operations.Api/Program.cs:36-40`). **The two surfaces disagree about how
  "not found" is expressed.**

**Extension procedure.** If the sentinel is ever replaced with a proper `NOT_FOUND` status, this
client must be updated in the same change — the `if` at `:125` would then be dead code that
suppresses nothing, and the `RpcException` would surface through §3.24 as a stack trace rather than
a friendly message.

**Failure modes.** Silent misinterpretation by any client that does not implement the check.

---

### 3.14 The server-streaming `SubscribeOperations` call

**Definition.** Option `2`.

```csharp
Console.WriteLine("Subscribing to the operations stream...");
using (var stream = _client.SubscribeOperations(new Empty()))
{
    while (await stream.ResponseStream.MoveNext())
    {
        Console.WriteLine("* Received the data from the operations stream *");
        DisplayOperation(stream.ResponseStream.Current);
    }
}
```
(`Program.cs:135-146`)

**Representation & storage.** Nothing persisted. Each message is printed and dropped.

**Lifecycle.** Starts on selection; **never ends by design**. The server's implementation is an
unconditional `while (true)` that blocks on `BlockingCollection<OperationDto>.Take()`
(`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs:32-41`), so `MoveNext()` never
returns `false` under normal operation. The `using` block's `Dispose` is therefore **unreachable in
practice**; the stream is torn down by process exit.

**What actually flows.** The server pushes an item for every `OperationUpdated` event raised by
`OperationsService.TrySetAsync` (`Services/OperationsService.cs:57`) *in the same process*. Three
consequences, all inherited from the server and all invisible from this file alone:

1. **Not per-user and not filtered.** Every operation for every user of that replica is streamed to
   every subscriber. Contrast the SignalR path, which is scoped to a per-user group
   (`operations-service.md` §3.22).
2. **Not replica-aware.** The `BlockingCollection` is per-process and per-`GrpcServiceHost`
   instance; a client connected to replica A never sees operations processed by replica B.
3. **Competing consumers, not broadcast.** `BlockingCollection.Take()` **removes** the item, so two
   concurrent subscribers *split* the stream rather than each receiving every update
   (`operations-service.md` §3.28). Running two copies of this client halves each one's view.

**Invariants & enforcement.**

| Invariant | Enforcement | Violation behaviour |
| --- | --- | --- |
| The call is read-only | server-side: no write path in `SubscribeOperations` | n/a |
| The loop must be cancellable | **absent** | **Silent lock-up.** `MoveNext()` is called with no `CancellationToken` overload (`:140`), so the REPL is unreachable for the rest of the process's life. `q` can never be typed again. |
| Backpressure | none in this client | Each message is `Console.WriteLine`-d synchronously; a fast producer is throttled by console I/O `[framework]`. |
| One subscription per process | not enforced | Because option `2` never returns (above), a second subscription is unreachable — the invariant holds by accident. |

**Extension procedure.** To make the stream exitable: use the
`MoveNext(CancellationToken)` overload `[grpc]` and cancel it from a `Console.CancelKeyPress`
handler or a background key reader. To make it filterable, the **`.proto` must change** — the
request is `Empty` (`Operations.proto:7,10-11`), so there is no field to carry a user id or an
operation id, and both copies plus the server would have to be updated together (§3.3).

**Failure modes.** Server restart mid-stream → `RpcException`, escaping the `while` into §3.24's
catch, printed, and the REPL resumes. There is **no reconnect** and no resume-from-offset: every
update produced while disconnected is lost permanently, because the server's queue is an in-memory
`BlockingCollection` with no persistence.

---

### 3.15 `DisplayOperation` and the JSON render settings

**Definition.** The single output formatter.

```csharp
private static readonly JsonSerializerSettings JsonSerializerSettings = new JsonSerializerSettings
{
    ContractResolver = new CamelCasePropertyNamesContractResolver(),
    Formatting = Formatting.Indented
};

private static void DisplayOperation(GetOperationResponse response)
    => Console.WriteLine(JsonConvert.SerializeObject(response, JsonSerializerSettings));
```
(`Program.cs:18-22,148-149`)

**Representation & storage.** A `static readonly` settings object; output goes to stdout only.

**Lifecycle.** Initialised at type load; used by both operations (`:132`, `:143`).

**Invariants & enforcement.**

- The formatter is **reflective over the generated CLR type**, so it renders whatever fields the
  generated `GetOperationResponse` has. **Adding a field to the `.proto` therefore surfaces
  automatically** — this is the only part of the component that does not need editing when the
  contract grows (§3.3, step 5).
- `CamelCasePropertyNamesContractResolver` restores the `.proto`'s original casing: protobuf's C#
  generator emits `Id`, `UserId`, `Name`, `State`, `Code`, `Reason` (PascalCase) `[grpc]`, and the
  resolver renders them as `id`, `userId`, … matching `Operations.proto:18-23` and the HTTP
  surface's JSON. Removing the resolver changes the console output casing but nothing functional.
- Newtonsoft has no protobuf awareness: it will also serialise generated infrastructure members if
  any are public. For this message type — six scalar `string` properties — the output is exactly
  the six fields. A message containing `repeated`/`map` fields would render as Newtonsoft's view of
  `RepeatedField<T>`, which is *not* the canonical protobuf-JSON encoding. **If richer message
  shapes are ever added, prefer `Google.Protobuf.JsonFormatter`** (already available via
  `Google.Protobuf` 3.11.4, `.csproj:10`) over Newtonsoft.

**Extension procedure.** Swap `JsonConvert.SerializeObject(...)` for
`JsonFormatter.Default.Format(response)` to get spec-compliant protobuf JSON; no other change is
needed.

**Failure modes.** None observed for the current message shape. The `Newtonsoft.Json` reference
exists solely for this method and can be deleted along with it.

---

### 3.16 The `state` field's wire encoding

**Definition.** `string state = 4;` (`Operations.proto:21`) — a *string*, not a protobuf `enum`.

**Representation & storage.** Produced by the server as
`operation.State.ToString().ToLowerInvariant()`
(`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs:53`), where `State` is the
CLR enum `OperationState { Pending, Completed, Rejected }`
(`../Pacco.Services.Operations.Api/Types/OperationState.cs`). This client never parses it — it
prints it verbatim (§3.15).

**Lifecycle.** Server-side only; the client is a pass-through.

**Invariants & enforcement.**

- The set of legal values is enforced **nowhere on the wire**. A consumer must know the three
  lowercase strings by convention.
- **The gRPC and HTTP surfaces disagree.** `GET /operations/{operationId}` serialises `OperationDto`
  directly (`../Pacco.Services.Operations.Api/Program.cs:42`), so `state` arrives there as the
  enum's **integer ordinal**, while gRPC delivers the lowercase name. Any consumer that treats the
  two channels as interchangeable is wrong. This client only sees the gRPC form.
- Adding an `OperationState` member is safe for gRPC (a new lowercase string appears) and silently
  changes the HTTP ordinals if inserted anywhere but the end.

**Extension procedure.** Changing `state` to a protobuf `enum` requires editing both `.proto`
copies, the server's `Map`, and any consumer that string-compares — this client would keep working
unchanged, because `DisplayOperation` is reflective.

**Failure modes.** Silent mismatch for any consumer that hard-codes the HTTP integer encoding and
then switches transport.

---

### 3.17 Absence of authentication and call credentials

**Definition.** The component sends **no credentials of any kind**.

**Representation & storage.** Verified by exhaustion: `Program.cs` contains no `Metadata`,
no `CallCredentials`, no `ChannelCredentials.Create`, no `Authorization` header, no token and no
certificate. The only security-adjacent code is the *disabling* of certificate validation (§3.8).

**Lifecycle.** n/a.

**Invariants & enforcement.** None needed — **the server requires none.** `operations-service`
calls neither `UseAuthentication()` nor `UseAccessTokenValidator()` in `UseInfrastructure`, and its
gRPC host carries no `[Authorize]`
(`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs:11-55`;
`operations-service.md` §1.2). Anyone who can reach port `50050` can read any operation by id and
subscribe to the whole stream.

**Extension procedure.** If the service ever adds JWT validation to the gRPC endpoint, this client
must attach a token per call:
`_client.GetOperationAsync(request, new Metadata { { "Authorization", $"Bearer {token}" } })`, plus
some way to obtain the token (the platform's issuer is `identity-service`,
`identity-service.md` §6). None of that exists today.

**Failure modes.** Not a failure — an exposure. Recorded so that a downstream phase adding auth to
`operations-service` knows this client breaks and where.

---

### 3.18 Absence of correlation, tracing and logging

**Definition.** The component participates in none of the platform's observability conventions.

| Convention | Present here? | Platform counterpart |
| --- | --- | --- |
| Correlation id (`message_context`) | **no** | every Convey service, e.g. `ordermaker-saga-service.md` §3.16 |
| Span context (`span_context`) | **no** | `patterns/observability/correlation-and-span-propagation.md` |
| Jaeger tracing | **no** | `Convey.Tracing.Jaeger` in most `.csproj`s; absent from `.csproj:9-14` |
| Structured logging (Serilog/Seq) | **no** | `Convey.Logging` + `logger` section in every service's `appsettings.json` |
| Metrics | **no** | `Convey.Metrics.AppMetrics` |
| `ILogger` of any kind | **no** — output is `Console.WriteLine` | |

**Representation & storage.** Absence, verified by the whole of `Program.cs` and by the four-package
dependency list (`.csproj:9-14`).

**Lifecycle.** n/a.

**Invariants & enforcement.** None. The consequence is concrete: **a call made by this client is
untraceable.** The server logs `context.Peer`
(`../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs:27,35`) — an IP:port — and
nothing else, so a gRPC read cannot be correlated with the saga or request that produced the
operation.

**Extension procedure.** Attach a `Metadata` entry per call and have the server read it; there is
no framework wiring to reuse, because the client has no host builder.

**Failure modes.** Diagnostic blindness only.

---

### 3.19 Absence of configuration and secrets

**Definition.** The scoped path contains **no `appsettings.json`, no `appsettings.*.json`, no
`launchSettings.json`, no `.env` and no `certs/`** — confirmed by the three-file inventory.

**Representation & storage.** Configuration is two positional CLI arguments (§3.7). That is the
entire surface.

**Lifecycle.** n/a — nothing to reload.

**Invariants & enforcement.** The absence is itself the invariant worth recording: this component
holds **no secret**, reads **no Vault path** (contrast `operations-service`, which calls
`UseVault()` at `../Pacco.Services.Operations.Api/Program.cs:50`), and needs no
`ASPNETCORE_ENVIRONMENT`. It can be run by anyone with the source and a terminal.

**Extension procedure.** Adding configuration means adding `Microsoft.Extensions.Configuration` and
a builder to `Main`; there is nothing to extend today.

**Failure modes.** The flip side of §3.7: because there is no configuration file, **the endpoint
cannot be changed without either recompiling or remembering the two-argument rule** — and passing
one argument is silently ignored.

---

### 3.20 Absence of tests, and the CI step that pretends otherwise

**Definition.** There is no test project for this component — or for the whole repository. The
repository has no `tests/` directory and no `*.Tests.csproj`; `Pacco.Services.Operations.sln`
declares exactly two projects, the API (`:8`) and this client (`:10`).

**Representation & storage.** `scripts/test.sh` is two lines, `#!/bin/bash` and `dotnet test`;
`.travis.yml:12-14` runs `./scripts/build.sh` then `./scripts/test.sh`.

**Lifecycle.** Runs on every CI build of `master` and `develop` (`.travis.yml:7-9`).

**Invariants & enforcement.** `dotnet test` on a solution with no test project **exits 0** — it
finds nothing to run and reports success. The CI pipeline therefore *reports a passing test stage
that never executed a test.* This is a green signal with no information behind it.

**Extension procedure.** Add a `tests/Pacco.Services.Operations.GrpcClient.Tests` project and
reference it from the `.sln`. Note that the component as written is close to untestable:
`GetAddress` is `private static` (testable only via `InternalsVisibleTo` + reflection), and every
other method reaches directly for `Console` and the static `_client`. Making it testable means
extracting `GetAddress` to an internal static class and injecting the stub — a small refactor with
no behavioural risk.

**Failure modes.** False confidence. A downstream phase reading the CI badge must not treat it as
coverage.

---

### 3.21 Deployment posture — built, never packaged

**Definition.** The component is compiled by CI and then discarded.

| Stage | Includes this project? | Evidence |
| --- | --- | --- |
| `dotnet build -c release` (root) | **yes** — builds the `.sln`, both projects | `scripts/build.sh`, `.travis.yml:12` |
| `dotnet test` | vacuously | §3.20 |
| `docker build` | **no** — `Dockerfile:4` publishes only `src/Pacco.Services.Operations.Api` | `Dockerfile:1-11` |
| `docker push` | **no** — one repository tag, `$DOCKER_USERNAME/pacco.services.operations` | `scripts/dockerize.sh` |
| Compose | **no** entry | `hianshul100_Pacco/compose/services.yml` (the Operations entry maps `5005:80` for the API only) |
| PM2 | **no** entry | `hianshul100_Pacco/services.yml`, `prod-services.yml` |
| Gateway route | n/a — gRPC is not routed by Ntrada | all four `ntrada*.yml` |

**Representation & storage.** No artifact of this project leaves CI.

**Lifecycle.** Built on every push to `master`/`develop`; never versioned, never released.

**Invariants & enforcement.** The **build inclusion is what matters**: because `dotnet build` walks
the solution, a compile error in this client **breaks the CI build of `operations-service`**. That
is the component's only production-affecting property. It is otherwise inert.

**Extension procedure.** To ship it (if it is ever deemed operational — see §8, Q1): add a second
`RUN dotnet publish src/Pacco.Services.Operations.GrpcClient` stage, a second image tag in
`dockerize.sh`, and resolve §3.7 first — the hard-coded `https://localhost:50050` default is useless
in a container. To *retire* it: remove the project from the `.sln` and delete the folder; nothing
references it.

**Failure modes.** The only failure this component can cause today is a red CI build for its
sibling.

---

### 3.22 Solution membership and build inclusion

**Definition.** `Pacco.Services.Operations.sln:10` declares the project with GUID
`{2E81C8DB-4F7B-414E-A0B9-1B7F4F492BA6}`, and `:37-48` maps it to all six
`Debug|Release × AnyCPU|x64|x86` configurations, all resolved to `Any CPU`.

**Representation & storage.** The `.sln` file at the repository root.

**Lifecycle.** Edited by hand or by an IDE.

**Invariants & enforcement.** The six configuration rows must exist or the project is skipped for
that configuration — Visual Studio maintains them; a hand-edit that drops
`Release|Any CPU.Build.0` would silently exclude the project from the CI build, which would *hide*
compile errors rather than surface them. **Silent.**

**Extension procedure.** Use `dotnet sln add`/`remove` rather than hand-editing.

**Failure modes.** As above — a dropped `Build.0` row is invisible until someone wonders why a
broken file compiles.

---

### 3.23 Concurrency and cancellation model

**Definition.** Single-threaded, cooperatively async, entirely uncancellable.

| Property | Value | Evidence |
| --- | --- | --- |
| Threading | one logical flow; `async Task Main` | `Program.cs:30` |
| Synchronisation | none needed — no shared mutable state beyond `_client`, assigned once | §3.6 |
| `ConfigureAwait` | never used | `Program.cs:48,102,121,140` |
| Synchronisation context | none (console apps have no `SynchronizationContext`) `[framework]` | so the missing `ConfigureAwait(false)` is harmless here |
| `CancellationToken` | **zero occurrences in the file** | `Program.cs` |
| Deadlines | none | §3.9 |
| `Ctrl+C` handling | none | §3.11 |

**Lifecycle.** n/a.

**Invariants & enforcement.** "One operation at a time" is enforced by the REPL's `await`
(`:102`) — the loop cannot start a second action until the first completes. Combined with §3.14
(option `2` never completes), this yields the component's real usage model: **either a sequence of
point reads, or exactly one subscription, never both.**

**Extension procedure.** Introducing a `CancellationTokenSource` wired to `Console.CancelKeyPress`,
and passing its token to `MoveNext` and to the unary call's `CallOptions`, is the single highest-
value change available to this component — it fixes §3.11's EOF spin and §3.14's lock-up at once.

**Failure modes.** Lock-up, as described.

---

### 3.24 Error handling — the catch-all around every action

**Definition.**

```csharp
try { await Actions[option](); }
catch (Exception ex) { Console.WriteLine(ex); }
```
(`Program.cs:100-107`)

**Representation & storage.** A `catch (Exception)` printing `ex.ToString()` (the implicit
`Console.WriteLine(object)` overload), i.e. type, message and full stack trace.

**Lifecycle.** Wraps every action invocation; nothing else in the process is guarded.

**Invariants & enforcement.**

- **Every exception is equal.** `RpcException(Unavailable)` (server down),
  `RpcException(Unknown)` (server threw), `ArgumentNullException` (EOF on the id prompt, §3.12) and
  a genuine bug all render identically as a stack trace. There is no `switch` on
  `RpcException.StatusCode`, no user-facing message and no exit code.
- **The catch is inside the loop**, so the REPL survives every failure and immediately re-prints
  the menu. The tool cannot be made to exit by an error.
- **`Main`'s own construction path is unguarded** (§3.1) — a malformed address argument that made
  `GrpcChannel.ForAddress` throw would crash before the menu ever appears.

**Extension procedure.** Catch `RpcException` first and map `StatusCode.Unavailable` /
`Unauthenticated` / `NotFound` to human-readable lines, then keep the general catch as a backstop.

**Failure modes.** Poor diagnosability, and — because the loop continues — a user can be shown the
same stack trace indefinitely without any hint that the server is simply not running.

---

### 3.25 Termination semantics and undisposed resources

**Definition.** How the process ends and what it leaves behind.

**Representation & storage.** `Main` returns when `InitAsync` returns; `InitAsync` returns when
`option == "q"` (`Program.cs:86`). The process then exits with code `0` — the default for a
returning `async Task Main` `[framework]`. There is no explicit exit code anywhere.

**Lifecycle / what is not released.**

| Resource | Created | Disposed |
| --- | --- | --- |
| `HttpClientHandler` | `:35-39` | **never** |
| `HttpClient` | `:40` | **never** |
| `GrpcChannel` | `:42-45` | **never** (no `ShutdownAsync`, no `Dispose`) |
| `AsyncServerStreamingCall` | `:138` | via `using` — but unreachable in practice (§3.14) |

**Invariants & enforcement.** Nothing enforces disposal; the OS reclaims sockets at exit. For a
console tool this is acceptable. It is recorded because **the pattern is unsafe to copy into a
long-lived host**, where an undisposed `GrpcChannel` holds an HTTP/2 connection and its keep-alive
timers indefinitely.

**Extension procedure.** Wrap the channel in `using var channel = …` in `Main`; the `HttpClient`
should be disposed after the channel, or dropped entirely by letting the channel create its own
handler once §3.8 is resolved.

**Failure modes.** None in the current shape. Under a `q`-less termination (`Ctrl+C`), an in-flight
stream is aborted without a `Cancel`, so the **server's `SubscribeOperations` loop is left blocked
on `Take()` and its `OperationUpdated` handler is never unsubscribed** — the server-side leak
documented in `operations-service.md` §3.28. **The client is the trigger for that server-side
leak**, which is the one way this non-deployed component can affect a deployed one.

---

## 4. Primary control flows

### 4.1 Startup

```
dotnet run [host] [port]
  └─ Program.Main(args)                                   Program.cs:30
       ├─ GetAddress(args)                                :32  → "https://{host}:{port}", defaults localhost/50050  (§3.7)
       ├─ new HttpClientHandler { ServerCertificateCustomValidationCallback = Dangerous… }
       │                                                  :35-39  TLS validation disabled            (§3.8)
       ├─ new HttpClient(handler)                         :40
       ├─ GrpcChannel.ForAddress(address, opts)           :42-45  no connection attempt yet          (§3.9)
       ├─ new GrpcOperationsServiceClient(channel)        :46                                        (§3.5)
       ├─ Console.WriteLine("Created a GRPC client …")    :47  ← prints even if nothing is listening
       └─ await InitAsync()                               :48                                        (§3.11)
```

No network I/O occurs during startup. The first packet is sent by the first action.

### 4.2 Point read — `GetOperation` (option `1`)

```
InitAsync loop                                            Program.cs:85-113
  └─ Actions["1"] → GetOperationAsync()                   :26, :116
       ├─ Console.ReadLine()                              :118  ← operation id, unvalidated  (§3.12)
       ├─ _client.GetOperationAsync(new GetOperationRequest { Id = id })
       │                                                  :121-124
       │     ── HTTP/2 POST /Services.Operations.GrpcOperationsService/GetOperation ──▶
       │        server: GrpcServiceHost.GetOperation                    (Api) GrpcServiceHost.cs:24
       │          ├─ _logger.LogInformation(… context.Peer)             :27
       │          ├─ IOperationsService.GetAsync(id)                    Services/OperationsService.cs:24
       │          │    └─ IDistributedCache.GetStringAsync("requests:{id}")   :26,62   → Redis
       │          │         └─ JsonConvert.DeserializeObject<OperationDto>    :28
       │          └─ Map(dto)  → null ⇒ new GetOperationResponse()      :43-54       (§3.13)
       │     ◀── GetOperationResponse ──
       ├─ if (string.IsNullOrWhiteSpace(response.Id)) → "not found"     :125-129     (§3.13)
       └─ DisplayOperation(response) → camelCase indented JSON          :132, :148   (§3.15)
```

**Side effects:** none on the client. On the server: one Redis `GET` and one log line. **Read-only.**

### 4.3 Live stream — `SubscribeOperations` (option `2`)

```
InitAsync loop
  └─ Actions["2"] → SubscribeOperationsStreamAsync()      :27, :135
       └─ using (_client.SubscribeOperations(new Empty())) :138
            │  ── HTTP/2 POST …/SubscribeOperations (server-streaming) ──▶
            │     server: GrpcServiceHost.SubscribeOperations           (Api) GrpcServiceHost.cs:32
            │       ├─ log peer                                          :35
            │       └─ while (true) { _operations.Take(); WriteAsync(Map(op)); }   :36-40
            │            ▲ fed by  _operationsService.OperationUpdated += … TryAdd  :21
            │              raised by OperationsService.TrySetAsync                  Services/OperationsService.cs:57
            │              which is called by the three generic message handlers    Handlers/Generic*Handler.cs
            └─ while (await stream.ResponseStream.MoveNext())            :140   ← never returns false
                 └─ DisplayOperation(current)                            :143
```

**The producer end of this stream is the whole platform.** Every RabbitMQ message named in
`../Pacco.Services.Operations.Api/messages.json` that reaches a generic handler ends in
`TrySetAsync`, raises `OperationUpdated`, and is pushed here. This client is therefore a **live tap
on the platform's entire request-status feed** — unauthenticated (§3.17), unfiltered, and limited
to the single server replica it is connected to (§3.14).

### 4.4 Shutdown

```
user types "q" → loop condition fails                     :86
  └─ InitAsync returns → Main returns → exit code 0        (§3.25)
       channel / HttpClient / handler: never disposed
```

Unreachable while a subscription is active (§3.14, §3.23).

---

## 5. Persistence & schema evolution

### 5.1 Datastores

**The component has none.** There is no database, no cache, no file write, no `%APPDATA%`/`~/.config`
state and no log file. Verified by exhaustion over the three files in the scoped path: the only
`System.IO`-adjacent calls are `Console` reads and writes.

Everything it displays lives in `operations-service`'s Redis, under keys `requests:{id}` with a
300-second sliding expiry (`../Pacco.Services.Operations.Api/Services/OperationsService.cs:50-55`,
`appsettings.json` → `requests.expirySeconds`). That store, its expiry semantics and its eviction
behaviour are modelled in `operations-service.md` §3.4–§3.6 and are **not restated here**.

### 5.2 The only schema this component owns

`Operations.proto` — a **copy** of the server's IDL (§3.3). "Schema evolution" for this component
means protobuf evolution, and the applicable rules are:

| Change | Wire-safe? | What must be done here |
| --- | --- | --- |
| Add a field with a new number ≥ 7 | yes | Copy the line into this project's `.proto`. Display picks it up automatically (§3.15). |
| Rename a field, same number | yes on the wire | Copy the change **and** fix any C# that referenced the old name (`response.Id` at `:125` is the only such reference). |
| Renumber an existing field | **no** | Both copies must change atomically, or this client misreads every response — **silently**. |
| Remove a field | yes if the number is reserved | Copy the change; remove any C# reference. |
| Add an `rpc` | yes | Copy, then §3.10's four-step procedure. |
| Change `state` from `string` to `enum` | **no** | Both copies plus the server's `Map`; this client needs no code change (§3.16). |
| Switch `Empty` to `google.protobuf.Empty` | yes | Add the `import`; both copies. |

### 5.3 How a schema change is *applied*

There is **no migration framework, no seeding, no versioning and no admin path** — the three
concepts this section normally covers do not exist for this component. Applying a schema change is:
edit both `.proto` files → rebuild → the generated C# in `obj/` is replaced. There is no artifact
to migrate because there is no persisted state.

### 5.4 Idempotency and re-runs

Every operation this component performs is a read. Running it twice, or ten times, changes nothing
on the server beyond log lines — with one exception worth naming: each `SubscribeOperations` call
adds an `OperationUpdated` handler on the server that is **never removed**
(`operations-service.md` §3.28), so repeated subscribe-and-kill cycles accumulate server-side
handlers and split the update stream further. **Re-running this client is not free for the server.**

---

## 6. Surface → internals map

### 6.1 What the component exposes to a *human operator*

| Surface (menu option) | Internal mechanism | Mutating? |
| --- | --- | --- |
| Banner: `Created a GRPC client for an address: '…'` | `Program.cs:47`, printed after channel construction — **not** a connectivity check (§3.5) | no |
| `1` — Get the single operation by id | `GetOperationAsync()` → unary RPC → server Redis `GET requests:{id}` (§3.12, §4.2) | **read-only** |
| `2` — Subscribe to the operations stream | `SubscribeOperationsStreamAsync()` → server-streaming RPC → server in-memory `BlockingCollection` (§3.14, §4.3) | **read-only on data**; *does* mutate server-side subscriber state (adds a permanent event handler and consumes items destructively) |
| `q` — quit | loop exit (§3.11, §3.25) | no |
| blank input | `"Missing option"` (`:92-95`); infinite loop on EOF (§3.11) | no |
| any other token | `"Invalid option: …"` (`:112`) | no |
| `args[0] args[1]` | `GetAddress` (§3.7); a **single** argument is silently ignored | no |

### 6.2 What the component *consumes* from `operations-service`

| Remote surface | Internal mechanism it drives on the server | Notes for callers |
| --- | --- | --- |
| `GrpcOperationsService/GetOperation` | `GrpcServiceHost.GetOperation` → `OperationsService.GetAsync` → Redis `requests:{id}` | Absent operation ⇒ **empty message, not a `NOT_FOUND` status** (§3.13). No auth required (§3.17). |
| `GrpcOperationsService/SubscribeOperations` | `GrpcServiceHost.SubscribeOperations` → `BlockingCollection.Take()` fed by `OperationUpdated` | Per-replica, per-process; **competing consumers**, not broadcast; no filter, no replay, no end (§3.14). |
| *(absent)* any write RPC | — | **The gRPC contract exposes no mutation at all.** `Operations.proto` declares two RPCs, both reads. Nothing this component does can change platform state. |
| *(absent)* the HTTP surface `GET /operations/{operationId}` | — | Not used by this component; note it encodes `state` differently (§3.16). |
| *(absent)* the SignalR hub `/pacco` | — | Not used; it is the browser's channel and is the only per-user-scoped one. |

### 6.3 What the component exposes to *other code*

Nothing. No public type, no library output, no API. See §1.2.

---

## 7. Change/extension guide

Each entry states the change, the exact steps, and **what silently rejects the change if a step is
skipped**.

### 7.1 To point the client at a different server

1. Pass **both** arguments: `dotnet run -- <host> <port>`.
2. *Silently rejected if:* you pass only one — `args.Length >= 2` (`Program.cs:56`) discards it and
   you connect to `localhost` with no warning. Passing a non-numeric port silently yields `50050`
   (`:59,70-73`).
3. *Not possible at all:* a plaintext `http://` target. The scheme is hard-coded (`:75`). To reach
   the Docker container (`http://*:80`, `Dockerfile:9`) you must edit `GetAddress` **and** enable
   unencrypted HTTP/2 `[framework]`.

### 7.2 To add a menu operation

1. Add the `rpc` to **both** `.proto` copies (§3.3) and implement it in
   `../Pacco.Services.Operations.Api/Infrastructure/GrpcServiceHost.cs`.
2. Write `private static async Task <Name>Async()` in `Program.cs`.
3. Register it in `Actions` (`Program.cs:24-28`).
4. **Edit the menu string at `Program.cs:80-83`** — both the `(1-2)` range and a new numbered line.
5. *Silently rejected if:* you skip step 4 — the option works but is undiscoverable; or you skip
   step 3 — the menu advertises an option that prints `"Invalid option"` (`:112`). Nothing links the
   two.

### 7.3 To add a field to `GetOperationResponse`

1. Next free number is **7** (`Operations.proto:17-24` uses 1–6).
2. Edit `src/Pacco.Services.Operations.Api/Operations.proto`.
3. **Copy the identical edit to `src/Pacco.Services.Operations.GrpcClient/Operations.proto`.**
4. Populate it in `GrpcServiceHost.Map` (`Api/Infrastructure/GrpcServiceHost.cs:46-53`) and, if it
   comes from operation state, in `DTO/OperationDto.cs` and `OperationsService.TrySetAsync`.
5. Rebuild. No client C# change is needed — `DisplayOperation` is reflective (§3.15).
6. *Silently rejected if:* you skip step 3 — this client ignores the field (protobuf discards
   unknown fields, no error); or you reuse a number 1–6 — this client misparses existing fields with
   no error at all.

### 7.4 To make the tool scriptable / non-interactive

1. Guard EOF in the REPL: `option = Console.ReadLine(); if (option is null) break;`
   (`Program.cs:89`) — without this, piping stdin produces an **infinite** `"Missing option"` loop
   (§3.11).
2. Add a non-interactive branch in `Main`: if a third argument names an action, run it once and
   return.
3. Return a meaningful exit code (today it is always `0`, §3.25).
4. Pass a `CancellationToken` to `MoveNext` (§3.14) so streaming can terminate.

### 7.5 To make the stream usable

1. `stream.ResponseStream.MoveNext(cancellationToken)` plus a `Console.CancelKeyPress` handler
   (§3.23).
2. For filtering, **the contract must change**: the request is `Empty` (`Operations.proto:7`), so
   there is no field for a user id — both `.proto` copies and the server's
   `SubscribeOperations` signature must change together.
3. *Silently rejected if:* you filter client-side instead — the server's `Take()` is destructive, so
   discarding a message locally means **another subscriber never receives it either** (§3.14).

### 7.6 To secure the client

1. Delete the `ServerCertificateCustomValidationCallback` (`Program.cs:35-39`) and trust the dev
   certificate (`dotnet dev-certs https --trust`), or supply a real certificate authority.
2. If/when the server adds authentication, attach `Metadata` per call (§3.17).
3. *Silently rejected if:* you leave the bypass in place — nothing warns, and the client will
   happily complete a TLS handshake with any certificate.

### 7.7 To retire the component

1. `dotnet sln remove src/Pacco.Services.Operations.GrpcClient/Pacco.Services.Operations.GrpcClient.csproj`.
2. Delete the folder.
3. Nothing else references it (§1.3). CI stops building it; no image, compose entry or PM2 entry
   needs touching because none exists (§3.21).
4. *Consider first:* it is the only executable record of §3.7's default address and §3.13's
   not-found sentinel. Deleting it deletes that documentation unless those facts are moved into the
   `.proto` as comments or into `operations-service.md`.

---

## 8. Assumptions, Blockers & Open Questions

> [!IMPORTANT]
> Items below are tagged **[ACTION NOW]** (a human must decide before dependent work proceeds) or
> **[handled later by \<stage\>]** (a named later stage owns it). Nothing here is papered over: each
> entry states what could not be determined from source and what would settle it.

### 8.1 Assumptions

| # | Assumption | Rationale | Impact if wrong | Validation path |
| --- | --- | --- | --- | --- |
| A1 | The two `Operations.proto` copies are byte-identical today | Both files were read in full in this workspace and match field-for-field, number-for-number (`Operations.proto:1-24` in each project) | If they had already drifted, §3.13's and §3.16's conclusions about wire compatibility would be wrong | `diff src/Pacco.Services.Operations.Api/Operations.proto src/Pacco.Services.Operations.GrpcClient/Operations.proto` in CI |
| A2 | `Grpc.Tools` defaults to `GrpcServices="Both"`, so this client also generates an unused server base class (§3.4) | Documented `Grpc.Tools` behaviour; the `.csproj` sets no attribute. **`[grpc]` — package source is not in this workspace** | Only affects generated-code size; no behavioural impact | Inspect `obj/**/OperationsGrpc.cs` after a build |
| A3 | Awaiting `AsyncUnaryCall<T>` directly discards status and trailers (§3.5) | Documented `Grpc.Core` behaviour `[grpc]` | If the awaiter surfaced status, the client would have richer error information than §3.24 describes | Read `AsyncUnaryCall<T>.GetAwaiter` in `Grpc.Core.Api` |
| A4 | `dotnet test` exits 0 in a solution with no test project (§3.20) | Standard .NET CLI behaviour; the repository has no test project and CI is green | If it exited non-zero, CI would be red and someone would have noticed — so the assumption is well-supported | Run `./scripts/test.sh` at the repository root |
| A5 | `netcoreapp3.1` is out of support | External fact; no file in the workspace records a framework lifecycle or migration plan | A migration is more or less urgent than implied | Platform owner |

### 8.2 Blockers

| # | Blocker | Blocks | Owner | Resolution path |
| --- | --- | --- | --- | --- |
| B1 | **[ACTION NOW]** The default target `https://localhost:50050` exists **only** under `dotnet run`. The container binds `http://*:80` (`Dockerfile:9`) and compose maps `5005:80`, so **there is no environment in which this client can reach a deployed `operations-service` without a code change** (§3.7, §1.3) | Any use of this component as an operational tool; any assumption that the gRPC surface is reachable in a deployed environment | Owner of `hianshul100_Pacco.Services.Operations` | Decide whether the gRPC endpoint should be exposed in Docker (add a Kestrel HTTP/2 endpoint and a compose port mapping) or whether both the endpoint and this client are dev-only. Record the decision in the repository. |
| B2 | **[ACTION NOW]** TLS validation is unconditionally disabled (§3.8), guarded only by a comment | Any decision to distribute this binary, and any reuse of the pattern | Security owner for the Pacco platform | Gate the callback behind a debug/`--insecure` flag, or trust the dev certificate and delete it |

### 8.3 Open questions

| # | Question | Why it matters | Proposed answer (if any) | Decision owner |
| --- | --- | --- | --- | --- |
| Q1 | **[ACTION NOW]** Is this project a throwaway demo or an operational tool? | It determines whether B1 must be fixed or the project deleted. It is also the only carrier of two undocumented contract facts (§3.7, §3.13) | Almost certainly a demo: no container, no compose/PM2 entry, no configuration, an interactive-only REPL, and a hard-coded dev address. It is retained in the build only because it is a solution member | Owner of `hianshul100_Pacco.Services.Operations`. Restates `service-summaries.md` G14/Q11 and `capability-baseline.md` Q16 — **still unresolved as of this batch** |
| Q2 | **[handled later by high-level spec]** Should the not-found sentinel (§3.13) be replaced by `StatusCode.NotFound`? | The gRPC and HTTP surfaces currently disagree about "not found" and about `state`'s encoding (§3.16). Any new consumer will get one of them wrong | Yes — align gRPC with the HTTP 404 and make `state` an enum. Both are breaking changes to a contract with exactly one known consumer, which is the cheapest moment to make them | Owner of `hianshul100_Pacco.Services.Operations` |
| Q3 | **[ACTION NOW]** Should the duplicated `.proto` be replaced by a shared contract package? | §3.3 is the component's only silent-corruption risk. A renumbering in one copy misparses responses with no error | At minimum add a CI `diff` (A1). Better: one `.proto` referenced by both projects via a linked file, or a small NuGet contract package | Owner of `hianshul100_Pacco.Services.Operations` |
| Q4 | **[handled later by architecture evolution]** Should `scripts/proto/{lin,mac,win}-compile.sh` be deleted? | They reference a `tools/Grpc.Tools.1.22.0` directory that does not exist, and **running them breaks the build** by emitting duplicate types next to the `.proto` (§3.4) | Delete them; `Grpc.Tools` in both `.csproj`s already does the job on every build | Owner of `hianshul100_Pacco.Services.Operations` |
| Q5 | **[handled later by high-level spec]** Should the streaming endpoint be per-user scoped, like SignalR? | Today any caller receives every user's operations, unauthenticated (§3.14, §3.17) | Yes, if the endpoint survives at all — it needs a request field to scope on, which is a `.proto` change | Security owner, with the service owner |

### 8.4 Explicitly unverifiable

| Claim | Status |
| --- | --- |
| Whether anyone has ever run this client against a non-local endpoint | **`Unverifiable — Missing Source Evidence`** — no logs, no runbook, no README mentions it |
| The exact generated members of `GrpcOperationsServiceClient` | **`Unverifiable — Missing Source Evidence`** in this workspace — `obj/` is not committed; the members used at `Program.cs:121,138` are inferred from the `.proto` and standard `Grpc.Tools` codegen `[grpc]` |
| Whether `Grpc.Net.Client` 2.28.0 applies any default retry/service-config | **`Unverifiable — Missing Source Evidence`** — package source absent. No `ServiceConfig` is set here (§3.9) |
| Whether CAKE (tenant `Q5SCXYFS`) holds any governance for this component | **No.** A graph query scoped to `data_scope IN [$tenant_code, 'global']` returned **0 nodes for the whole tenant**, and a fallback `cake_search` returned content from an unrelated domain. There is **no ADR, decision, constraint or catalog record** for `operations-grpc-client` — consistent with every prior batch in this repository. Recorded as **Case B** in `cake_influence_report.json`, written at the workspace root (the run's working directory), not inside this repository |

---

*End of `operations-grpc-client` component-internals model. Maintenance contract: any later phase
that changes this component's internals — `Program.cs`, `Operations.proto`, or its `.csproj` — must
update this document in the same change.*
