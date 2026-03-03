# mirrord: Contributor Onboarding Guide

## 1. What Is mirrord?

mirrord lets you run a program on your laptop as if it were running inside a Kubernetes cluster. When your program opens a file, connects to a database, or listens for HTTP requests, mirrord silently intercepts those calls and executes them on a remote pod instead of locally.

**Why this matters:** Normally, to test code against a real cluster you must build a container image, push it to a registry, update a deployment, and wait. mirrord skips all of that. You run your code locally with a single command and it "sees" the cluster's files, network, environment variables, and DNS -- without deploying anything.

**One-sentence summary:** mirrord is a local-to-remote bridge that tricks your process into thinking it is running inside a Kubernetes pod.

---

## 2. Core Concepts (Glossary)

| Term | Meaning |
|------|---------|
| **Layer** | A shared library (`.so` on Linux, `.dylib` on macOS) injected into your process. It replaces standard C library functions (like `open`, `connect`, `read`) with custom versions that can forward work to the cluster. |
| **Hook / Detour** | The act of replacing a C library function with a custom one. When your app calls `open()`, the hook intercepts it before the OS sees it. |
| **Agent** | A short-lived container that mirrord creates inside your Kubernetes cluster. It performs the real file reads, DNS lookups, and network operations on behalf of your local process. |
| **Intproxy** (Internal Proxy) | A local process that sits between the layer and the agent. It routes messages, manages the connection, and handles reconnections. |
| **Steal mode** | The agent redirects real incoming traffic away from the pod and sends it to your local process instead. Your local process handles the request. |
| **Mirror mode** | The agent copies incoming traffic (without redirecting it). Both your local process and the pod see the same traffic. |
| **Target** | The Kubernetes resource (pod, deployment, etc.) whose context your local process will assume. |
| **Protocol** | A set of message types (`ClientMessage`, `DaemonMessage`) serialized with bincode. This is how the layer and agent talk to each other. |
| **LD_PRELOAD / DYLD_INSERT_LIBRARIES** | OS mechanisms that force a shared library to load before any other when a process starts. This is how the layer gets injected. |
| **iptables** | A Linux firewall tool. The agent uses it to redirect network traffic from the target pod to itself (in steal mode). |
| **Network namespace** | A Linux isolation feature. Each pod has its own network namespace with its own IP addresses, routes, and DNS config. The agent runs network operations inside the target pod's namespace so that DNS and routing behave identically to the real pod. |

---

## 3. How It All Fits Together

### The Three Tiers

```
YOUR LAPTOP                              KUBERNETES CLUSTER
┌─────────────────────────┐              ┌─────────────────────┐
│                         │              │                     │
│  ┌───────────────────┐  │              │  ┌───────────────┐  │
│  │  Your Application │  │              │  │    Agent      │  │
│  │  ┌─────────────┐  │  │              │  │  (ephemeral   │  │
│  │  │   Layer     │  │  │    TCP       │  │   container)  │  │
│  │  │  (injected) │──┼──┼─────────────►│  │               │  │
│  │  └─────────────┘  │  │              │  │  Reads files  │  │
│  └───────────────────┘  │              │  │  Steals traffic│ │
│                         │              │  │  Resolves DNS  │  │
│  ┌───────────────────┐  │              │  └───────┬───────┘  │
│  │     Intproxy      │  │              │          │          │
│  │  (message router) │  │              │  ┌───────▼───────┐  │
│  └───────────────────┘  │              │  │  Target Pod   │  │
│                         │              │  │  (your app's  │  │
│  ┌───────────────────┐  │              │  │   twin in k8s)│  │
│  │       CLI         │  │              │  └───────────────┘  │
│  │  (mirrord exec)   │  │              │                     │
│  └───────────────────┘  │              └─────────────────────┘
└─────────────────────────┘
```

### End-to-End Flow

Here is what happens when you run `mirrord exec --target pod/my-app -- python server.py`:

1. **CLI** parses your command and config file, connects to Kubernetes, and creates an **agent** pod next to your target pod.
2. **CLI** starts the **intproxy** process locally. The intproxy opens a TCP connection to the agent.
3. **CLI** sets environment variables (`LD_PRELOAD` pointing to the layer library, intproxy address, config) and launches `python server.py`.
4. When `python server.py` starts, the OS loads the **layer** library before anything else. The layer replaces libc functions (`open`, `connect`, `bind`, `getaddrinfo`, etc.) with its own versions (hooks/detours).
5. When your Python code calls `open("/etc/config.yaml")`, the **layer** intercepts it, sends a message through the **intproxy** to the **agent**, which opens the file on the target pod's filesystem. The result is sent back, and your code sees the remote file's contents.
6. When your Python code calls `bind("::", 8080)`, the **layer** tells the agent to steal traffic on port 8080. The agent sets up iptables rules so that any traffic hitting the pod on port 8080 gets forwarded to your laptop instead.
7. When the process exits, the agent cleans up iptables rules and the agent pod is deleted.

---

## 4. Repository Map

### Top-Level Directories

| Directory | What's Inside |
|-----------|---------------|
| `mirrord/` | All Rust crates (the actual source code). This is where you will spend most of your time. |
| `tests/` | End-to-end tests that spin up a real Kubernetes cluster, deploy an agent, and test full flows. |
| `test-utils/` | Shared test utilities. |
| `scripts/` | Shell scripts for building, releasing, and CI. |
| `images/` | Dockerfiles and container build contexts. |
| `changelog.d/` | Changelog fragment files. Every PR adds one. |
| `xtask/` | Cargo xtask build automation (preferred over raw scripts). |
| `.github/` | CI workflows, PR templates, issue templates. |

### The Crates (Inside `mirrord/`)

These are ordered by importance for understanding the project:

| Crate | What It Does | Key Entry File |
|-------|-------------|----------------|
| **mirrord-protocol** | Defines every message that flows between layer and agent. Start here to understand what operations exist. | `src/codec.rs` |
| **mirrord-layer** | The injected library. Hooks libc functions and sends requests. This is the largest and most complex crate. | `src/lib.rs`, `src/file/hooks.rs`, `src/socket/hooks.rs` |
| **mirrord-agent** | Runs in-cluster. Handles file I/O, traffic stealing, DNS, outgoing connections. Linux-only. | `src/entrypoint.rs` |
| **mirrord-intproxy** | Local message router between layer and agent. | `src/lib.rs` |
| **mirrord-cli** | The `mirrord` binary users run. Orchestrates everything. | `src/main.rs` |
| **mirrord-config** | Parses JSON/YAML config files and CLI args into a typed config struct. | `src/lib.rs` |
| **mirrord-kube** | Creates agent pods, selects target containers, detects service meshes. | `src/api/container.rs` |
| **mirrord-layer-lib** | Shared layer logic (detour guards, proxy connection, setup). Separated so that the Windows layer can reuse it. | `src/detour.rs`, `src/proxy_connection.rs` |
| **mirrord-operator** | CRD definitions for the paid operator product. You likely won't touch this early on. | `src/lib.rs` |
| **mirrord-sip** | macOS-only: handles System Integrity Protection by re-signing binaries. | `src/lib.rs` |
| **mirrord-console** | Debug logging tool. Run it alongside mirrord to see layer logs in a separate terminal. | `src/lib.rs` |

### Supporting Crates (Less Important for Onboarding)

`mirrord-auth`, `mirrord-tls-util`, `mirrord-analytics`, `mirrord-progress`, `mirrord-macros`, `mirrord-vpn`, `mirrord-agent-env`, `mirrord-agent-iptables`, `mirrord-protocol-io`, `mirrord-intproxy-protocol`, `mirrord-config-derive`, `mirrord-layer-macro`, `mirrord-layer-win`, `str-win`

---

## 5. Deep Dive: Core Mechanisms

### 5.1 Syscall Interception (The Layer)

**What it does:** When your application calls a C library function like `open()` or `connect()`, the layer intercepts it and decides: should this run locally or remotely?

**How hooks work:**

The layer uses a library called **Frida GUM** (a dynamic instrumentation toolkit) to replace functions at runtime. When the layer loads, it:

1. Scans all loaded libraries to find function addresses (e.g., where `libc::open` lives in memory).
2. Replaces each target function with a custom "detour" function.
3. Saves the original function pointer so the detour can call it when needed (bypass).

```
Before hook:                  After hook:
App calls open() ──► libc    App calls open() ──► layer's open_detour()
                                                       │
                                                       ├─► remote? → send to agent
                                                       └─► local?  → call original libc open()
```

The code that does this lives in `mirrord/layer/src/hooks.rs`:

```rust
// Simplified from the actual code
hook_manager.hook_export_or_any("open", open_detour as *mut c_void)?;
```

**The DetourGuard pattern:**

There is a subtle problem: the layer's own code sometimes needs to call `open()` (e.g., to create temporary files). Without protection, this would trigger the hook again, causing infinite recursion. The `DetourGuard` prevents this:

```rust
// Pseudocode of what happens in every hook:
fn open_detour(path, flags) {
    let guard = DetourGuard::new();   // Try to "acquire" the guard
    if guard.is_none() {
        // We're already inside a detour. Call the real libc open.
        return original_open(path, flags);
    }
    // Guard acquired. We're the outermost detour call.
    // Do our interception logic...
    // Guard is automatically released when this function returns.
}
```

The guard is per-thread (uses `thread_local!`), so two threads can be in detours simultaneously without conflict.

**File interception flow (simplified):**

1. App calls `open("/etc/hosts", O_RDONLY)`
2. `open_detour` fires, acquires DetourGuard
3. Layer checks the file filter: should `/etc/hosts` be handled remotely?
   - If the config says "local-only" for this path → call original `open()`, done
   - If remote → continue
4. Layer sends `FileRequest::Open { path: "/etc/hosts" }` through the intproxy to the agent
5. Agent opens `/proc/<target_pid>/root/etc/hosts` (this gives the target container's view of the file)
6. Agent responds with a remote file descriptor number (say, `42`)
7. Layer creates a local temporary file and gets a local fd (say, `5`)
8. Layer records the mapping: local fd `5` → remote fd `42` in a global hashmap (`OPEN_FILES`)
9. Layer returns fd `5` to the application
10. When the app later calls `read(5, ...)`, the layer sees fd `5` in its map, sends `FileRequest::Read { fd: 42 }` to the agent, and returns the remote data

**Socket interception flow (incoming traffic / steal mode):**

1. App calls `bind(sock, 0.0.0.0:8080)` and `listen(sock)`
2. Layer intercepts `listen()`, sends a "subscribe to port 8080" message to the agent
3. Agent sets up an iptables rule: `PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port <agent_port>`
   - This means: any TCP packet arriving at the pod on port 8080 gets redirected to the agent instead
4. Agent accepts the redirected connections and forwards data to the layer through the intproxy
5. App calls `accept()` → layer returns the forwarded connection
6. From the app's perspective, it received a connection on port 8080, just like it would in the cluster

**Socket interception flow (outgoing traffic):**

1. App calls `connect(sock, "database:3306")`
2. Layer intercepts, sends the connection request to the agent
3. Agent makes the real connection inside the target pod's network namespace
   - This means it uses the pod's DNS, routing tables, and network policies
   - The connection appears to originate from the pod, not your laptop
4. Data is proxied back and forth through the intproxy

**DNS interception:**

1. App calls `getaddrinfo("my-service.default.svc.cluster.local")`
2. Layer intercepts, sends `GetAddrInfoRequest` to agent
3. Agent resolves the name using the target pod's DNS configuration (which knows about Kubernetes service names)
4. Layer returns the resolved addresses to the app

### 5.2 The Protocol

Every interaction between layer and agent is a message. Messages are defined in `mirrord/protocol/src/codec.rs`.

**Two message types:**

- `ClientMessage` — sent from layer to agent (requests)
- `DaemonMessage` — sent from agent to layer (responses)

**Examples of ClientMessage variants:**
- `FileRequest` — open/read/write/stat/readdir a file
- `TcpSteal` — subscribe to a port, acknowledge data
- `TcpOutgoing` — make an outgoing TCP connection
- `UdpOutgoing` — make an outgoing UDP connection
- `GetAddrInfoRequest` — resolve a hostname
- `GetEnvVarsRequest` — fetch environment variables
- `Ping` — keepalive
- `SwitchProtocolVersion` — negotiate which protocol version to use

**Serialization:** Messages use **bincode** (a compact binary format). This is efficient but means you cannot read messages with human-readable tools.

**Version negotiation:** At connection start, the layer sends its protocol version. The agent responds with the minimum of both versions. Feature-gated code checks this version before using newer message variants.

**Exhaustiveness requirement:** Rust's pattern matching forces every component to handle every message variant. When you add a new variant to `ClientMessage`, the compiler will tell you exactly which files need updating. This is a feature, not a bug.

### 5.3 The Agent

The agent runs as a Kubernetes Job (a short-lived pod). Key facts:

- **Parent/child model:** The agent forks. The parent's only job is to clean up iptables rules when the child exits (even if it crashes or runs out of memory).
- **Network namespace:** Network-related tasks (DNS, outgoing connections, traffic stealing) run inside the target pod's network namespace, so they see the same network as the target.
- **File access:** Uses `/proc/<pid>/root/...` paths to access the target container's filesystem without entering its mount namespace.
- **Main loop:** A `tokio::select!` loop in `entrypoint.rs` that simultaneously handles messages from the layer and responses from background tasks (stealer, mirror, DNS, outgoing).

**Background tasks in the agent:**
- `TcpStealerTask` — manages iptables redirect rules and distributes stolen traffic
- `TcpMirrorApi` — sniffs traffic without redirecting (mirror mode)
- `TcpOutgoingApi` / `UdpOutgoingApi` — handle outgoing connections
- `DnsWorker` — resolves DNS queries using the target's resolv.conf

### 5.4 The Intproxy (Internal Proxy)

The intproxy sits between layer and agent on your laptop. Why not connect directly?

- **Multiple layers:** If your process forks, each child has its own layer. The intproxy multiplexes all of them over a single agent connection.
- **Reconnection:** If the agent connection drops, the intproxy handles retries.
- **Message routing:** Different message types go to specialized "proxy tasks" (FilesProxy, IncomingProxy, OutgoingProxy, SimpleProxy), each maintaining its own state for request/response matching.

### 5.5 Configuration

Users configure mirrord via a JSON or YAML file. The root struct is `LayerConfig` in `mirrord/config/src/lib.rs`.

Key sections:
- `target` — what pod/deployment to connect to
- `feature.fs` — filesystem mode: `read` (remote reads, local writes), `write` (remote reads AND writes), `local` (no remote fs), `localwithoverrides` (mostly local, some paths remote)
- `feature.network.incoming` — `steal`, `mirror`, or `off`
- `feature.network.outgoing` — whether to intercept outgoing connections
- `feature.network.dns` — whether to intercept DNS
- `feature.env` — which environment variables to import from the target

---

## 6. Good Starting Points for New Contributors

### 6.1 Read These Files First

In this order:

1. **`mirrord/protocol/src/codec.rs`** — Skim the `ClientMessage` and `DaemonMessage` enums. This gives you a complete picture of every operation mirrord supports. You don't need to understand every variant, just get the shape.

2. **`mirrord/layer/src/file/hooks.rs`** — Read `open_detour` and `read_detour`. These are the simplest hooks to understand and they demonstrate the full pattern: intercept → check filter → send message → receive response → return to app.

3. **`mirrord/agent/src/entrypoint.rs`** — Read `handle_client_message()`. This shows the other side: how the agent receives and processes each request type.

4. **`mirrord/intproxy/src/lib.rs`** — Skim the main event loop to understand message routing.

5. **`mirrord/config/src/lib.rs`** — Read `LayerConfig` to understand what users can configure.

### 6.2 Types of Contributions That Are Good First Tasks

**Bug fixes in the layer** — These are often about a specific libc function not being hooked or behaving incorrectly in edge cases. The pattern is well-established: add a detour function, register the hook, handle the new message in the agent. Look for issues labeled `good first issue` on GitHub.

**Configuration additions** — Adding a new config option involves `mirrord-config` (parsing), the layer or agent (using the new option), and tests. This is well-scoped and teaches you multiple crates.

**Integration tests** — The `mirrord/layer/tests/` directory has many test apps. Writing a new test teaches you how the layer works without needing to modify production code. Tests create a fake agent, so you don't need a cluster.

**Documentation** — Updating docs or adding code comments based on your onboarding experience is valuable and low-risk.

### 6.3 Why These Are Good Starting Points

- The **hook pattern** is highly repetitive. Once you understand one hook (like `open_detour`), you can read or write any of them.
- **Integration tests** don't require a Kubernetes cluster. They simulate the agent in-process, so you can run them with just `cargo test -p mirrord-layer`.
- **Config changes** are isolated. You modify one crate, wire it up, and test. There's little risk of breaking unrelated features.
- Each of these touches a small surface area but teaches you the message flow end-to-end.

---

## 7. Development Workflow Quick Reference

```bash
# Check that code compiles (fast feedback)
cargo check -p mirrord-layer --keep-going
cargo check -p mirrord-protocol --keep-going
# Agent is linux-only:
cargo check -p mirrord-agent --target x86_64-unknown-linux-gnu --keep-going

# Run integration tests (no cluster needed)
cargo test -p mirrord-layer

# Format code (run after every change)
cargo fmt

# Run the CLI
cargo run -p mirrord

# Debug with mirrord-console (separate terminal)
cargo run --bin mirrord-console --features binary
# Then run mirrord with: MIRRORD_CONSOLE_ADDR=127.0.0.1:11233

# Build on macOS (universal binary)
cargo xtask build-cli
```

### PR Checklist

1. Run `cargo fmt`
2. Run `cargo check -p <crate> --keep-going` for each modified crate
3. Add a changelog fragment in `changelog.d/` (see CONTRIBUTING.md for naming)
4. If adding a hook: add both "remote path" and "bypass path" tests
5. All imports at the top of the file, never inside functions
6. Use `to_owned()` not `to_string()` for `&str` → `String`

---

## 8. Key Design Patterns to Recognize

| Pattern | Where | Why |
|---------|-------|-----|
| **DetourGuard** | Every hook function | Prevents infinite recursion when layer code calls hooked functions |
| **Arc\<RemoteFile\>** in OPEN_FILES | `mirrord/layer/src/file/` | Supports `dup()` — multiple local fds can point to the same remote file. Close only fires when the last reference drops. |
| **`tokio::select!`** main loop | Agent's `entrypoint.rs`, intproxy's `lib.rs` | Handles multiple async event sources concurrently |
| **BackgroundTasks** | Agent and intproxy | Manages spawned async tasks with cancellation tokens and mpsc channels |
| **`LazyLock<VersionReq>`** | Protocol crate | Gates features by protocol version so old agents work with new layers (and vice versa) |
| **Detour enum** (`Success` / `Bypass` / `Error`) | Layer hooks | Clean control flow for "did we handle it, skip it, or fail?" |
| **Parent/child process in agent** | `entrypoint.rs` | Parent exists solely to clean up iptables, even if child crashes |

---

## 9. Critical Flows (Code-Level Walkthroughs)

This section traces every important flow through the actual source code. Each flow shows what happens conceptually, which files and functions are involved, what protocol messages fly between components, and how data gets from your app to the cluster and back.

### 9.1 Startup Flow: `mirrord exec`

This is the first flow to understand. Everything starts here. When you type:

```
mirrord exec --target pod/my-app -- python server.py
```

...six things happen in sequence: parse config, create agent in Kubernetes, start the intproxy, launch your app with the layer injected, initialize the layer, and connect everything together.

#### Step 1: CLI parses config and resolves the target

**Where:** `mirrord/cli/src/main.rs` — `exec()` calls `exec_process()`

The CLI loads configuration from three sources (lowest to highest priority): config file (JSON/YAML), environment variables, CLI arguments. The result is a `LayerConfig` struct.

```
exec() → exec_process()
         ├── LayerConfig::resolve()   — merge all config sources
         └── config.verify()          — validate settings
```

#### Step 2: Create the agent pod in Kubernetes

**Where:** `mirrord/cli/src/connection.rs` — `create_and_connect()`; `mirrord/kube/src/api/kubernetes.rs` — `KubernetesAPI::create_agent()`

The CLI connects to your Kubernetes cluster (using your kubeconfig) and creates an agent. The agent is a Kubernetes **Job** — a pod that runs once and exits. It is placed in the same namespace as your target pod and given access to the target's filesystem and network.

```
create_and_connect()
├── KubernetesAPI::new()              — connect to cluster
└── k8s_api.create_agent()            — create agent Job
    ├── create_agent_params()         — get target's container PID, environment
    └── Targeted::create_agent()      — submit Job to K8s API
        └── returns AgentKubernetesConnectInfo { pod_name, agent_port }
```

The agent type depends on config:
- **Targeted + Job** (default): creates a Job pod alongside the target
- **Targeted + Ephemeral**: injects an ephemeral container into the target pod itself
- **Targetless**: creates a standalone Job with no specific target

#### Step 3: Start the intproxy

**Where:** `mirrord/cli/src/execution.rs` — `MirrordExecution::start_internal()`; `mirrord/cli/src/internal_proxy.rs` — `proxy()`

The CLI spawns a child process running `mirrord intproxy`. It passes connection details through environment variables:

| Environment variable | Value | Purpose |
|---------------------|-------|---------|
| `MIRRORD_AGENT_CONNECT_INFO` | JSON | Tells intproxy how to reach the agent (pod name, port, namespace) |
| `MIRRORD_LAYER_RESOLVED_CONFIG` | Base64 | Full config for the layer to read later |

The intproxy process:
1. Reads `MIRRORD_AGENT_CONNECT_INFO` from its environment
2. Connects to the agent via Kubernetes port-forward (a tunnel from your laptop to the agent pod)
3. Sends `ClientMessage::Ping`, waits for `DaemonMessage::Pong` to confirm the connection works
4. Opens a TCP listener on localhost (random port)
5. Prints the listener address (e.g. `127.0.0.1:54321`) to stdout — the CLI reads this

```
proxy()
├── read MIRRORD_AGENT_CONNECT_INFO env var
├── connect_and_ping()
│   ├── AgentConnection::new()     — TCP via K8s port-forward
│   ├── send Ping
│   └── wait for Pong
├── create TCP listener on localhost
├── print address to stdout         — CLI reads this line
└── IntProxy::run()                 — start main event loop
```

**Where the intproxy event loop lives:** `mirrord/intproxy/src/lib.rs` — `IntProxy::run_inner()`. It starts several background tasks:

| Task | What it does |
|------|-------------|
| `LayerInitializer` | Accepts new layer TCP connections |
| `LayerConnection` (one per layer) | Handles messages from each layer instance |
| `AgentConnection` | Manages the single connection to the agent |
| `FilesProxy` | Routes file operation requests/responses |
| `IncomingProxy` | Routes incoming traffic (steal/mirror) messages |
| `OutgoingProxy` | Routes outgoing connection messages |
| `SimpleProxy` | Routes DNS and environment variable messages |
| `PingPong` | Periodic health checks |

#### Step 4: Launch your application with the layer injected

**Where:** `mirrord/cli/src/main.rs` — end of `exec_process()`

The CLI sets three critical environment variables and then replaces itself with your application using `execve` (a Unix system call that swaps the current process for a new one):

| Environment variable | Value | Purpose |
|---------------------|-------|---------|
| `LD_PRELOAD` (Linux) or `DYLD_INSERT_LIBRARIES` (macOS) | Path to `libmirrord_layer.so` / `.dylib` | Forces the OS to load the layer library into your process |
| `MIRRORD_LAYER_INTPROXY_ADDR` | `127.0.0.1:<port>` | Tells the layer where to find the intproxy |
| `MIRRORD_LAYER_RESOLVED_CONFIG` | Base64-encoded config | Full configuration for the layer |

After `execve`, the CLI process is gone. Your application process takes over, with the layer loaded.

#### Step 5: Layer initializes inside your process

**Where:** `mirrord/layer/src/lib.rs` — `mirrord_layer_entry_point()` (marked `#[ctor]`, which means it runs automatically when the library loads, before your app's `main()`)

```
#[ctor] mirrord_layer_entry_point()
└── layer_pre_initialization()
    ├── read config from MIRRORD_LAYER_RESOLVED_CONFIG env var
    ├── decide load type: Full / SIPOnly / Skip
    └── layer_start(config)                    — for Full load:
        ├── init_tracing()                     — set up logging
        ├── init_layer_setup(config)           — prepare file filters, socket config
        ├── enable_hooks()                     — replace libc functions with detours
        │   ├── file::hooks::enable_file_hooks()    — open, read, write, close, stat...
        │   ├── socket::hooks::enable_socket_hooks() — socket, bind, listen, connect...
        │   └── exec_hooks::enable_exec_hooks()      — execve, fork, for child processes
        ├── ProxyConnection::new(intproxy_addr) — TCP connect to intproxy
        └── fetch_env_vars()                   — get remote env vars from target pod
```

The `enable_hooks()` function uses Frida GUM to replace libc functions. After this call, every `open()`, `connect()`, `bind()`, etc. in your process goes through the layer's detour functions first.

#### Step 6: Everything is connected

After initialization, the full chain is in place:

```
Your App ←→ Layer (in-process) ←→ Intproxy (localhost TCP) ←→ Agent (K8s port-forward) ←→ Target Pod
```

Every subsequent syscall flows through this chain. The following sections trace each type of operation.

---

### 9.2 File Open Flow

**Scenario:** Your app calls `open("/etc/hosts", O_RDONLY)`. mirrord intercepts this and opens the file on the target pod's filesystem instead.

#### Conceptual steps

1. Your app calls `open()` — the layer's hook fires instead of libc's
2. Layer decides: should this path be remote or local?
3. If remote: layer sends a request to the agent (via intproxy)
4. Agent opens the file on the target's filesystem
5. Agent returns a remote file descriptor (a number identifying the open file)
6. Layer creates a local placeholder file and maps local fd → remote fd
7. Layer returns the local fd to your app — your app doesn't know anything happened

#### Code trace

**Layer hook** — `mirrord/layer/src/file/hooks.rs`:

```
open_detour(raw_path, open_flags)                    — hooks.rs
├── DetourGuard::new()                                — if None, we're already in a hook → call original libc open
├── open(path, open_options)                          — file/ops.rs
│   ├── common_path_check(path, is_write)             — check file filter, apply path remapping
│   │   └── returns Bypass if path should stay local
│   └── RemoteFile::remote_open(path, open_options)   — file/ops.rs
│       ├── create OpenFileRequest { path, open_options }
│       └── make_proxy_request_with_response(request)  — layer-lib/proxy_connection.rs
│           ├── assign message_id (atomic counter)
│           ├── encode and send via TCP to intproxy
│           └── block waiting for response with matching message_id
```

**Intproxy routing** — `mirrord/intproxy/src/lib.rs`:

```
handle_layer_message(FromLayer { message_id, layer_id, message })
└── match message:
    LayerToProxyMessage::File(req) →
        FilesProxy.send(FilesProxyMessage::FileReq(message_id, layer_id, req))
```

**FilesProxy** — `mirrord/intproxy/src/proxies/files.rs`:

```
FilesProxy::run() receives FileReq
├── store (message_id, layer_id) in request queue     — for matching the response later
└── send ClientMessage::FileRequest(req) to agent via AgentConnection
```

**Agent handler** — `mirrord/agent/src/entrypoint.rs`:

```
handle_client_message(ClientMessage::FileRequest(req))
└── file_manager.handle_message(req)                   — agent/src/file.rs
    └── match FileRequest::Open { path, open_options }:
        FileManager::open(path, open_options)
        ├── resolve_path(&path)                        — prepend /proc/<pid>/root/ to see target's filesystem
        ├── OpenOptions::from(open_options).open(&path) — actually open the file
        ├── allocate remote fd from counter
        ├── store in self.open_files HashMap
        └── return OpenFileResponse { fd }
→ respond(DaemonMessage::File(FileResponse::Open { fd }))
```

**Response flows back:**

```
Agent → (TCP) → Intproxy
    handle_agent_message(DaemonMessage::File(response))
    └── FilesProxy.send(FilesProxyMessage::FileRes(response))
        └── pop (message_id, layer_id) from request queue
        └── send ToLayer { message_id, layer_id, response } → LayerConnection → Layer
```

**Layer receives response** — back in `file/ops.rs`:

```
RemoteFile::remote_open() receives OpenFileResponse { fd: 42 }
├── create_local_fake_file(42)      — create temp file, open it to get a local fd (say, 5)
├── OPEN_FILES.insert(5, Arc::new(RemoteFile { fd: 42, path }))
└── return local fd 5 to the app
```

#### Protocol messages involved

| Direction | Message | Defined in |
|-----------|---------|------------|
| Layer → Agent | `ClientMessage::FileRequest(FileRequest::Open { path, open_options })` | `protocol/src/codec.rs`, `protocol/src/file.rs` |
| Agent → Layer | `DaemonMessage::File(FileResponse::Open { fd })` | `protocol/src/codec.rs`, `protocol/src/file.rs` |

#### Why the local fake file?

Your app expects `open()` to return a real file descriptor (fd) — a number the OS recognizes. The layer can't return the agent's fd number directly because the OS doesn't know about it. So the layer creates a real local file (a temp file), gets its fd, and maintains a mapping. When your app later calls `read(5)`, the layer sees fd 5, looks up remote fd 42, and sends the read to the agent.

---

### 9.3 File Read/Write Flow

**Scenario:** After opening a remote file (fd 5 locally, fd 42 on the agent), your app calls `read(5, buffer, 1024)`.

#### Code trace

**Layer hook** — `mirrord/layer/src/file/hooks.rs`:

```
read_detour(fd=5, buf, count=1024)
├── DetourGuard::new()
├── get_remote_fd(5)                                  — file/ops.rs
│   └── OPEN_FILES.lock().get(&5) → RemoteFile { fd: 42 }
├── RemoteFile::remote_read(42, 1024)
│   ├── create ReadFileRequest { remote_fd: 42, buffer_size: 1024 }
│   └── make_proxy_request_with_response(request)     — blocks until agent responds
```

**Agent** — `mirrord/agent/src/file.rs`:

```
FileManager::read(fd=42, buffer_size=1024)
├── self.open_files.get_mut(&42) → the actual file handle
├── file.read(&mut buffer)                            — read from target's filesystem
└── return ReadFileResponse { bytes, read_amount }
```

**Write works identically** but in reverse — `write_detour` sends `WriteFileRequest { fd, write_bytes }`, agent calls `file.write(&buffer)`, returns `WriteFileResponse { written_amount }`.

#### Protocol messages

| Direction | Message |
|-----------|---------|
| Layer → Agent | `FileRequest::Read { remote_fd, buffer_size }` or `FileRequest::Write { fd, write_bytes }` |
| Agent → Layer | `FileResponse::Read { bytes, read_amount }` or `FileResponse::Write { written_amount }` |

---

### 9.4 File Close Flow

**Scenario:** Your app calls `close(5)` on a remotely-opened file.

This flow uses a Rust pattern: the layer stores remote files as `Arc<RemoteFile>` (a reference-counted smart pointer). When the last reference is dropped, the `Drop` implementation sends the close request. This handles `dup()` correctly — if your app duplicates a file descriptor, both fds point to the same `Arc`. Closing one fd decrements the count; only the last close sends the remote close.

```
close_detour(fd=5)
├── OPEN_FILES.lock().remove(&5) → drops Arc<RemoteFile>
│   └── if last reference (Arc count reaches 0):
│       RemoteFile::drop()
│       └── make_proxy_request_no_response(CloseFileRequest { fd: 42 })
│           → agent removes fd 42 from its open_files HashMap
```

Note: `close` uses `make_proxy_request_no_response` — it sends the message but doesn't wait for a reply. The agent just cleans up silently.

---

### 9.5 Incoming Traffic Flow (Steal Mode)

**Scenario:** Your app is a web server. It calls `bind(sock, 0.0.0.0:8080)` then `listen(sock)`. mirrord redirects real traffic hitting the pod's port 8080 to your local machine.

This is the most complex flow. It involves four stages: bind, listen (subscribe), receive connection, and exchange data.

#### Stage 1: Bind

**Where:** `mirrord/layer/src/socket/hooks.rs` — `bind_detour()`; `mirrord/layer/src/socket/ops.rs` — `bind()`

```
bind_detour(sockfd, raw_address=0.0.0.0:8080, address_length)
├── DetourGuard::new()
├── bind(sockfd, raw_address, address_length)          — socket/ops.rs
│   ├── retrieve socket from SOCKETS HashMap
│   ├── check incoming config: should we handle port 8080?
│   │   └── if port is in the ignore list → call original libc bind, done
│   ├── bind_similar_address(sockfd, address)          — bind locally
│   │   ├── try binding to 0.0.0.0:8080 locally
│   │   ├── if port taken → try port 0 (random)       — OS assigns a free port
│   │   └── get actual bound address with getsockname()
│   └── update SOCKETS: state = Bound { requested_port: 8080, actual_port: <random> }
```

The layer binds locally on a potentially different port. This is fine — the layer knows the mapping.

#### Stage 2: Listen (triggers port subscription)

**Where:** `mirrord/layer/src/socket/ops.rs` — `listen()`

```
listen_detour(sockfd, backlog)
├── retrieve socket from SOCKETS — must be in Bound state
├── call original libc listen(sockfd, backlog)         — start listening locally
├── send PortSubscribe to agent:
│   make_proxy_request_with_response(
│       IncomingRequest::PortSubscribe(Port {
│           port: 8080,           — the port on the target pod
│           listening_on: <local address the layer actually bound to>
│       })
│   )
└── update SOCKETS: state = Listening
```

**Intproxy** routes this to `IncomingProxy`, which forwards to the agent as `ClientMessage::Tcp(LayerTcp::PortSubscribe)` or `ClientMessage::TcpSteal(LayerTcpSteal::PortSubscribe)` depending on the mode.

**Agent** — `mirrord/agent/src/steal/`:

```
TcpStealerTask receives PortSubscribe(port=8080)
└── RedirectorTask sets up iptables rule:
    iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port <agent_port>
```

This rule intercepts all TCP packets destined for port 8080 on the target pod and redirects them to the agent's listening port instead. The real application in the pod stops receiving traffic on that port.

#### Stage 3: Receive connection

When external traffic hits the pod on port 8080:

```
1. iptables redirects the packet to the agent
2. Agent accepts the connection
3. Agent sends DaemonMessage::TcpSteal(DaemonTcp::NewTcpConnection { connection_id, source, destination })
4. Intproxy routes to IncomingProxy
5. IncomingProxy creates a local TCP connection to the layer's listening address
6. Layer's accept() returns this connection to your app
```

Your app sees a new connection on its listening socket. From your app's perspective, a client just connected — it doesn't know the connection was proxied through an agent in a Kubernetes cluster.

#### Stage 4: Data exchange

```
External client → target pod:8080 → (iptables redirect) → agent
Agent sends: DaemonTcp::Data { connection_id, bytes }
→ intproxy IncomingProxy → local TCP → layer → your app reads data

Your app writes response → layer → local TCP → intproxy IncomingProxy
→ ClientMessage::TcpSteal(LayerTcpSteal::Data { connection_id, bytes })
→ agent → external client
```

#### Protocol messages

| Direction | Message | When |
|-----------|---------|------|
| Layer → Agent | `LayerTcpSteal::PortSubscribe(StealType::All(port))` | App calls `listen()` |
| Agent → Layer | `DaemonTcp::SubscribeResult` | Confirms subscription |
| Agent → Layer | `DaemonTcp::NewTcpConnection { connection_id, source, destination }` | External client connects |
| Agent → Layer | `DaemonTcp::Data { connection_id, bytes }` | Client sends data |
| Layer → Agent | `LayerTcpSteal::Data { connection_id, bytes }` | App sends response |
| Agent → Layer | `DaemonTcp::Close { connection_id }` | Connection ends |

---

### 9.6 Outgoing Connection Flow

**Scenario:** Your app calls `connect(sock, "10.0.0.5:3306")` to reach a MySQL database that's only accessible from inside the cluster.

#### Conceptual steps

1. Layer intercepts `connect()`
2. Layer sends the destination address to the agent
3. Agent makes the real TCP connection from inside the target pod's network namespace — so it uses the pod's routing, DNS, and firewall rules
4. Data is proxied: app writes → layer → intproxy → agent → database, and back

#### Code trace

**Layer** — `mirrord/layer/src/socket/hooks.rs` — `connect_detour()`:

```
connect_detour(sockfd, raw_address=10.0.0.5:3306, address_length)
├── DetourGuard::new()
├── connect(sockfd, raw_address, address_length)       — socket/ops.rs
│   ├── check outgoing config: is outgoing interception enabled?
│   │   └── if disabled or address filtered out → call original libc connect, done
│   ├── send OutgoingConnectRequest { remote_address: 10.0.0.5:3306 }
│   │   via make_proxy_request_with_response()
│   └── receive response with local proxy address
│       └── call original libc connect(sockfd, localhost:<proxy_port>)
```

**Intproxy** — `OutgoingProxy` receives the request, forwards to agent as `ClientMessage::TcpOutgoing(LayerTcpOutgoing::Connect)`.

**Agent** — `mirrord/agent/src/outgoing.rs`:

```
TcpOutgoingApi receives LayerConnect { remote_address: 10.0.0.5:3306 }
├── spawn connection task in target's network namespace
│   └── TcpStream::connect("10.0.0.5:3306")           — real connection from the pod's network
└── send DaemonTcpOutgoing::Connect { connection_id } back
```

**Data proxying:** Once connected, the `OutgoingProxy` in the intproxy creates a local TCP socket pair. Your app writes to one end; the intproxy reads from it and sends data to the agent, which writes to the real database connection. Responses flow back the same way.

#### Protocol messages

| Direction | Message |
|-----------|---------|
| Layer → Agent | `ClientMessage::TcpOutgoing(LayerTcpOutgoing::Connect { remote_address })` |
| Agent → Layer | `DaemonMessage::TcpOutgoing(DaemonTcpOutgoing::Connect { connection_id })` |
| Both directions | `TcpOutgoing::Data { connection_id, bytes }` for data transfer |
| Either side | `TcpOutgoing::Close { connection_id }` when connection ends |

---

### 9.7 DNS Resolution Flow

**Scenario:** Your app calls `getaddrinfo("my-service.default.svc.cluster.local", "80", ...)` to resolve a Kubernetes service name. This hostname only exists inside the cluster's DNS — your laptop doesn't know about it.

#### Code trace

**Layer** — `mirrord/layer/src/socket/hooks.rs` — `getaddrinfo_detour()`:

```
getaddrinfo_detour(node="my-service.default.svc.cluster.local", service="80", hints, result)
├── DetourGuard::new()
├── check DNS config: is DNS interception enabled?
├── create GetAddrInfoRequest { node: "my-service.default.svc.cluster.local" }
├── make_proxy_request_with_response(request)
│   → intproxy SimpleProxy → agent
```

**Intproxy** — `SimpleProxy` handles this:

```
SimpleProxy receives AddrInfoReq(msg_id, layer_id, request)
├── push (msg_id, layer_id) onto addr_info_reqs queue
└── send ClientMessage::GetAddrInfoRequest(request) to agent
```

**Agent** — `mirrord/agent/src/dns.rs`:

```
DnsWorker receives GetAddrInfoRequest
├── resolve hostname using target pod's DNS configuration
│   └── reads /etc/resolv.conf from target's network namespace
│       which contains cluster DNS server (e.g. 10.96.0.10)
│       and search domains (e.g. default.svc.cluster.local)
└── return GetAddrInfoResponse { addresses: [10.96.42.7] }
```

**Layer receives response** and allocates `libc::addrinfo` structs that it returns to your app. Your app gets back the cluster-internal IP address `10.96.42.7` for the service, as if it were running inside the cluster.

#### Protocol messages

| Direction | Message |
|-----------|---------|
| Layer → Agent | `ClientMessage::GetAddrInfoRequest { node }` (or V2 with hints) |
| Agent → Layer | `DaemonMessage::GetAddrInfoResponse(Ok(addresses))` |

---

### 9.8 Environment Variables Flow

**Scenario:** Your app reads `os.environ["DATABASE_URL"]`. mirrord imports this variable from the target pod so your app sees the cluster's value.

This flow is different from the others — it happens once during layer initialization, not on every syscall.

#### Code trace

**Layer** — `mirrord/layer/src/lib.rs` — `layer_start()`:

```
layer_start(config)
├── ... (hooks setup, proxy connection) ...
└── if config.feature.env is enabled:
    fetch_env_vars()
    ├── create GetEnvVarsRequest { env_vars_filter, env_vars_select }
    │   — filter/select controls which vars to import (include/exclude lists)
    ├── make_proxy_request_with_response(request)
    └── for each (key, value) in response:
        std::env::set_var(key, value)    — inject into process environment
```

**Agent** — `mirrord/agent/src/entrypoint.rs`:

```
handle_client_message(ClientMessage::GetEnvVarsRequest(req))
├── env::select_env_vars(self.env.clone(), req.env_vars_filter, req.env_vars_select)
│   └── self.env was populated at agent startup by reading the target container's
│       environment via container runtime API (Docker/containerd inspect) or
│       /proc/<pid>/environ
└── respond(DaemonMessage::GetEnvVarsResponse(filtered_env))
```

After this, `std::env::var("DATABASE_URL")` in your app returns the value from the target pod.

#### Protocol messages

| Direction | Message |
|-----------|---------|
| Layer → Agent | `ClientMessage::GetEnvVarsRequest { env_vars_filter, env_vars_select }` |
| Agent → Layer | `DaemonMessage::GetEnvVarsResponse(HashMap<String, String>)` |

---

### 9.9 Message Routing Through the Intproxy

All messages between layer and agent pass through the intproxy. Understanding how the intproxy dispatches messages is essential for working on any flow.

#### Layer → Agent direction

**Where:** `mirrord/intproxy/src/lib.rs` — `handle_layer_message()`

Every message from the layer arrives as a `FromLayer { message_id, layer_id, message }`. The intproxy checks the message type and sends it to the right proxy task:

| Message type | Routed to | Then sent to agent as |
|-------------|-----------|----------------------|
| `LayerToProxyMessage::File(req)` | `FilesProxy` | `ClientMessage::FileRequest(req)` |
| `LayerToProxyMessage::GetAddrInfo(req)` | `SimpleProxy` | `ClientMessage::GetAddrInfoRequest(req)` |
| `LayerToProxyMessage::GetEnv(req)` | `SimpleProxy` | `ClientMessage::GetEnvVarsRequest(req)` |
| `LayerToProxyMessage::Outgoing(req)` | `OutgoingProxy` | `ClientMessage::TcpOutgoing(req)` |
| `LayerToProxyMessage::Incoming(req)` | `IncomingProxy` | `ClientMessage::TcpSteal(req)` |

Each proxy task maintains a **request queue** — a FIFO list of `(message_id, layer_id)` pairs. When a response comes back from the agent, the proxy pops the oldest entry from the queue to figure out which layer sent the original request.

#### Agent → Layer direction

**Where:** `mirrord/intproxy/src/lib.rs` — `handle_agent_message()`

| Message type | Routed to |
|-------------|-----------|
| `DaemonMessage::File(res)` | `FilesProxy` |
| `DaemonMessage::GetAddrInfoResponse(res)` | `SimpleProxy` |
| `DaemonMessage::GetEnvVarsResponse(res)` | `SimpleProxy` |
| `DaemonMessage::TcpOutgoing(msg)` | `OutgoingProxy` |
| `DaemonMessage::UdpOutgoing(msg)` | `OutgoingProxy` |
| `DaemonMessage::Tcp(msg)` | `IncomingProxy` (mirror mode) |
| `DaemonMessage::TcpSteal(msg)` | `IncomingProxy` (steal mode) |
| `DaemonMessage::Pong` | `PingPong` |
| `DaemonMessage::SwitchProtocolVersionResponse(v)` | All proxies (sets protocol version) |

---

### 9.10 Agent Main Loop

**Where:** `mirrord/agent/src/entrypoint.rs` — `ClientConnectionHandler::start()`

The agent uses a `tokio::select!` loop — a Rust construct that waits on multiple async event sources simultaneously and processes whichever one fires first. There are seven branches:

```
loop {
    tokio::select! {
        // 1. Message from the layer (via intproxy)
        msg = connection.receive() => {
            handle_client_message(msg)
            // dispatches to: file_manager, tcp_outgoing_api, dns_api, etc.
        }

        // 2. Mirrored traffic from TcpMirrorApi
        msg = tcp_mirror_api.recv() => respond(msg)

        // 3. Stolen traffic from TcpStealerApi
        msg = tcp_stealer_api.recv() => respond(msg)

        // 4. Outgoing TCP data from TcpOutgoingApi
        msg = tcp_outgoing_api.recv_from_task() => respond(msg)

        // 5. Outgoing UDP data from UdpOutgoingApi
        msg = udp_outgoing_api.recv_from_task() => respond(DaemonMessage::UdpOutgoing(msg))

        // 6. DNS resolution results from DnsWorker
        msg = dns_api.recv() => respond(DaemonMessage::GetAddrInfoResponse(msg))

        // 7. Reverse DNS results
        msg = reverse_dns_api.recv() => respond(DaemonMessage::ReverseDnsLookup(msg))
    }
}
```

`handle_client_message()` (same file) dispatches each `ClientMessage` variant:

| ClientMessage variant | Handler |
|----------------------|---------|
| `FileRequest(req)` | `file_manager.handle_message(req)` — synchronous, responds immediately |
| `TcpOutgoing(msg)` | `tcp_outgoing_api.send_to_task(msg)` — async, response comes via branch 4 |
| `UdpOutgoing(msg)` | `udp_outgoing_api.send_to_task(msg)` — async, response via branch 5 |
| `GetEnvVarsRequest(req)` | Filters `self.env`, responds immediately |
| `GetAddrInfoRequest(req)` | `dns_api.make_request(req)` — async, response via branch 6 |
| `Tcp(msg)` | `tcp_mirror_api.handle_client_message(msg)` — mirror mode subscriptions |
| `TcpSteal(msg)` | `tcp_stealer_api.handle_client_message(msg)` — steal mode subscriptions |
| `Ping` | Responds with `Pong` immediately |
| `SwitchProtocolVersion(v)` | Negotiates version, responds immediately |
| `Close` | Returns `false` to exit the loop |

Note the split: file operations and env vars are **synchronous** (the agent handles them and responds in the same function call). Network and DNS operations are **asynchronous** (dispatched to background tasks, responses come back through separate `select!` branches).
