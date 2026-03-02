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
