# RFC: dop Basic Command Specifications

- **RFC Name:** `dop_basic_command_specifications`
- **Start Date:** 2026-04-15
- **Author(s):** Leon
- **Status:** Proposed
- **Translations:** [简体中文](./README.zh.md)
- **RFC PR:** [dorafun-io/rfcs#0002](https://github.com/dorafun-io/rfcs/pull/2)
- **Tracking Issue:** [dorafun-io/dop#2](https://github.com/dorafun-io/dop/issues/2)

---

## 1. Summary

> *(Provide a concise, high-level abstract of the proposal.)*

Following the establishment of the `$DOP_HOME` sandboxing logic in RFC-0001, this proposal outlines the Input/Output (I/O) contracts and expected behaviors for the 16 foundational commands of the Dora Operating Platform (`dop`). These commands cover scaffolding, authentication, basic lifecycle management, and operational observability, ensuring a consistent developer experience across heterogeneous hardware.

## 2. Motivation

> *(Why are we proposing this change?)*

- **Predictability:** Developers need a deterministic mental model when interacting with the CLI. Ambiguous command behaviors lead to frustration.
- **Automation:** CI/CD pipelines and the DoraFun platform rely on standard exit codes and structured terminal outputs (e.g., JSON flags) to automate workflows.
- **Cargo Alignment:** Adopting familiar paradigms from the Rust ecosystem (like token-based login) lowers the barrier to entry for core contributors.

## 3. Detailed Design

### 3.1 Scaffold Array (创世阵列)

#### 3.1.1 `dop init`
- **Signature:** `dop init [--force]`
- **Design Rationale (Why):** Embodied AI development is notoriously fragile due to global C++/Python library conflicts. `init` acts as the "Big Bang" for the deterministic `$DOP_HOME` sandbox, ensuring that `dop` never relies on the host OS's unpredictable global state.
- **Implementation Logic (How):**
  1. **Path Resolution:** Evaluate `$DOP_HOME`. If absent, default to `~/.dop`.
  2. **Directory Allocation:** `mkdir -p` for `bin`, `cache/cargo`, `cache/uv`, `models`, and `logs` with strict `0755` permissions.
  3. **Binary Vendoring:** Spawn concurrent asynchronous HTTP GET requests to fetch pinned pre-compiled binaries (e.g., `uv`, `dora-cli`, `zenohd`) from the DoraFun CDN.
  4. **Symlinking/Execution:** Drop binaries into `$DOP_HOME/bin` and apply `chmod +x`.
- **Edge Cases & Failure Modes:** - Network timeout during fetch: Must rollback partial downloads (Fail-safe).
  - Directory already exists: Exit gracefully with code `0` unless `--force` is passed (Idempotency).

#### 3.1.2 `dop new`
- **Signature:** `dop new <project_name> [--template <registry/name>]`
- **Design Rationale (Why):** Writing boilerplate `dataflow.yaml` and IPC shared-memory code is error-prone. We need a zero-friction Day 1 experience.
- **Implementation Logic (How):**
  1. **Template Fetch:** If `--template` is omitted, default to the embedded `basic-rust-python` template. If provided, fetch the tarball from `github.com/dorafun-io/templates`.
  2. **Hydration Engine:** Use a lightweight templating engine (like `tera` or `minijinja` in Rust) to replace `{{project_name}}`, `{{uuid}}`, and `{{dop_version}}` across the scaffolded files.
  3. **Git Initialization:** Automatically run `git init` and inject a standard `.gitignore` specifically tuned for DoraFun (ignoring `mcap` files and local `target/` dirs).

### 3.2 Cloud & Metadata Array (云端阵列)

#### 3.2.1 `dop login`
- **Signature:** `dop login [--token <jwt>]`
- **Design Rationale (Why):** Headless edge devices (e.g., Raspberry Pi, Jetson) cannot launch browsers. We mandate a Cargo-style personal access token (PAT) workflow for maximum CLI compatibility.
- **Implementation Logic (How):**
  1. **Interactive Prompt:** If `--token` is missing, use a secure standard input reader (e.g., `dialoguer::Password`) to hide keystrokes.
  2. **Verification:** Send a `GET /v1/auth/verify` to the DoraFun API with the Bearer token.
  3. **Secure Storage:** If HTTP 200, write the token to `$DOP_HOME/config/credentials.toml`. **Critical:** Explicitly set OS file permissions to `0600` (read/write by owner only) to prevent privilege escalation leaks.

#### 3.2.2 `dop logout`
- **Signature:** `dop logout`
- **Implementation Logic:** Perform a secure overwrite of `$DOP_HOME/config/credentials.toml` before unlinking the file, ensuring tokens cannot be recovered from disk sectors.

#### 3.2.3 `dop config`
- **Signature:** `dop config <get|set> <key> [value]`
- **Design Rationale:** Users need a centralized way to mutate behavior without manually editing TOML files, which often leads to syntax errors.
- **Implementation Logic (How):**
  1. **AST Parsing:** Use a comment-preserving parser (e.g., `toml_edit` in Rust). Do NOT deserialize to a struct and serialize back, as this destroys user comments.
  2. **Validation:** Implement a registry of allowed keys (e.g., `registry.mirror`, `telemetry.opt_out`). Reject unknown keys with exit code `1`.
  3. **Mutation:** Modify the AST and flush to `$DOP_HOME/config/config.toml`.

#### 3.2.4 `dop info`
- **Signature:** `dop info [--json]`
- **Design Rationale:** Provides immediate observability of the static project state before runtime execution. Crucial for debugging CI/CD pipelines.
- **Implementation Logic:**
  1. **Graph Resolution:** Parse the local `dataflow.yaml`.
  2. **System Probe:** Detect the host architecture (`uname -m`), OS, and available accelerators (checking `/dev/nvidia0` or `/dev/davinci0` for Ascend CANN).
  3. **Output Formatting:** Print a human-readable summary. If `--json` is provided, output a strictly typed JSON schema to `stdout` for jq/automation tools to consume.

### 3.3 Lifecycle Array (生命周期)

#### 3.3.1 `dop build`
- **Signature:** `dop build [--release]`
- **Design Rationale (Why):** Embodied AI projects are inherently polyglot. A single `dataflow.yaml` may define a C++ sensor driver, a Rust core router, and a Python neural network node. Asking developers to manually invoke `cmake`, `cargo`, and `uv` is an anti-pattern.
- **Implementation Logic (How):**
  1. **Manifest Parsing:** Parse `dataflow.yaml` to build a Directed Acyclic Graph (DAG) of node dependencies.
  2. **Isolated Compilation:**
     - For Rust nodes: Dispatch to `cargo build`, injecting `--target-dir` pointing to `$DOP_HOME/cache/cargo`.
     - For Python nodes: Dispatch to `uv build` or `uv pip sync`, locking dependencies within `$DOP_HOME/envs/<uuid>`.
  3. **Artifact Linking:** Symlink the compiled binaries into a `.dop/target/` directory local to the project workspace, maintaining a clean `git status` via the global `.gitignore`.

#### 3.3.2 `dop run`
- **Signature:** `dop run [node_id]`
- **Design Rationale (Why):** The primary entry point for foreground debugging. It must guarantee absolute state reproducibility. If a robot crashes during `dop run`, we must know exactly what code was executing, even if it was uncommitted.
- **Implementation Logic (How):**
  1. **The Gitoxide Shadow Snapshot (Crucial):** Before spawning any process, `dop` uses the embedded `gitoxide` engine to scan the working directory. If it detects uncommitted changes (dirty state), it instantly constructs a Tree and a Commit object directly in `.git/objects` in-memory, without modifying the index. It tags this detached commit as `refs/tags/dop/run-<timestamp>`. This creates a zero-cost "black box flight recorder" for the runtime.
  2. **Context Hijack:** Mutate the environment variables of the spawned child process:
     - Prepend `$DOP_HOME/bin` to `$PATH`.
     - Set `$PYTHONPATH` to the isolated virtual environment.
     - Inject `$DORA_ROOT` for zero-copy memory mapping.
  3. **Foreground Execution & IPC:** Launch the local Zenoh router (if required) and the specified nodes. Forward `stdout/stderr` directly to the terminal, color-coded by `node_id`. Block until `SIGINT` (Ctrl+C).

#### 3.3.3 `dop start`
- **Signature:** `dop start [node_id]`
- **Design Rationale (Why):** Designed for headless edge devices (e.g., Jetson, Ascend CANN) and production deployments where processes must outlive the SSH session.
- **Implementation Logic (How):**
  1. **Snapshot & Build:** Inherits the exact same `gitoxide` shadow commit and build logic as `dop run`.
  2. **Daemonization:** Fork the process and detach it from the controlling TTY (e.g., using `nix::unistd::daemon` in Rust).
  3. **I/O Redirection:** `stdout/stderr` are strictly redirected to `$DOP_HOME/logs/<project_uuid>/<node_id>.log` using a rotating file appender (rolling by size, e.g., 50MB).
  4. **State Tracking:** Write the master PID and node PIDs into `$DOP_HOME/run/<project_uuid>.pid`. Exits immediately with code `0`, returning control to the user.

#### 3.3.4 `dop stop`
- **Signature:** `dop stop [node_id] [--force]`
- **Design Rationale (Why):** Embodied AI nodes often hold exclusive locks on physical hardware (e.g., `/dev/video0` or CAN bus interfaces). Hard-killing them leaves hardware in a zombie state. We must enforce a graceful teardown contract.
- **Implementation Logic (How):**
  1. **PID Resolution:** Read the target PIDs from `$DOP_HOME/run/<project_uuid>.pid`.
  2. **Graceful Teardown (SIGTERM):** Send `SIGTERM` (or `CTRL_C_EVENT` on Windows) to the processes. Nodes are given a strict 5.0-second window to flush memory mappings and release hardware locks.
  3. **Escalation (SIGKILL):** If the 5.0-second timeout expires and processes remain active, escalate to `SIGKILL` (`TerminateProcess`).
  4. **Zero-copy Garbage Collection:** Invoke a cleanup routine to scrub any orphaned POSIX shared memory segments (`/dev/shm/dora_*`) that might have leaked during a hard crash.

### 3.4 Ops & Observability Array (运维阵列)

#### 3.4.1 `dop status`
- **Signature:** `dop status`
- **Design Rationale (Why):** In Embodied AI, CPU/GPU spikes or Zero-copy IPC bottlenecks directly translate to physical latency (e.g., a robot arm reacting 200ms too late). Developers need a real-time, terminal-native dashboard without spinning up heavy web servers.
- **Implementation Logic (How):**
  1. **TUI Rendering:** Utilize a terminal UI library (e.g., `ratatui` in Rust) to draw a non-blocking interface.
  2. **Metrics Gathering:**
     - Probe host OS `procfs` for CPU/Memory per `node_id` PID.
     - Interrogate the local Zenoh router for pub/sub message throughput and IPC shared-memory latency.
  3. **Visual Cues:** Highlight nodes in RED if their message queue backpressure exceeds a threshold (e.g., dropping frames from a 60FPS camera node).

#### 3.4.2 `dop logs`
- **Signature:** `dop logs [node_id] [-f/--follow]`
- **Design Rationale (Why):** Raw `stdout` from polyglot nodes is a chaotic wall of text. We need structured, queryable logs.
- **Implementation Logic (How):**
  1. **Log Aggregation:** Read the JSON-line structured logs generated by `dop start` from `$DOP_HOME/logs/<project_uuid>/`.
  2. **Terminal Formatting:** Deserialize the JSON and format it into a human-readable stream, color-coding log levels (ERROR=Red, WARN=Yellow) and `node_id` prefixes.
  3. **Tailing:** If `-f` is passed, use a file-system watcher (e.g., `notify` crate) to stream new lines instantly.

#### 3.4.3 `dop doctor`
- **Signature:** `dop doctor`
- **Design Rationale (Why):** Hardware and OS misconfigurations account for 80% of Day-1 failures. `dop doctor` acts as the definitive environmental sanity check.
- **Implementation Logic (How):**
  1. **Device Permissions:** Check if the current user belongs to the `video` and `dialout` groups (Linux) to ensure access to USB cameras and serial ports (`/dev/ttyUSB0`).
  2. **Accelerator Probing:** Execute NVML calls for NVIDIA GPUs or `npu-smi` for Huawei Ascend to verify driver integrity.
  3. **Shared Memory Limit:** Inspect `/dev/shm` size and system limits (`ulimit -l` / `memlock`). Warn the user if the limit is too low for zero-copy tensor passing.
  4. **Output:** Print a checklist with `[PASS]`, `[WARN]`, and `[FAIL]`, providing copy-pasteable remediation commands (e.g., `sudo usermod -aG video $USER`).

#### 3.4.4 `dop shell`
- **Signature:** `dop shell`
- **Design Rationale:** Sometimes developers need to run raw `python` or `cargo` commands inside the exact environment that `dop run` uses to debug tricky dependency issues.
- **Implementation Logic:**
  1. Resolve the user's default shell from `$SHELL`.
  2. Spawn a subprocess of that shell with mutated variables: `$PATH` injected with `$DOP_HOME/bin`, and `$PYTHONPATH` targeting the project's virtual env.
  3. Modify the shell prompt (`$PS1`) to prefix `(dop) ` so the user knows they are inside the sandbox.

#### 3.4.5 `dop clean`
- **Signature:** `dop clean [--all]`
- **Design Rationale (Why):** Embodied AI accumulates massive artifacts: compiled C++ binaries, cached HuggingFace models, and our own `gitoxide` shadow commits. Aggressive garbage collection is required.
- **Implementation Logic (How):**
  1. **Standard GC:** Delete `$DOP_HOME/cache/cargo/target` and `$DOP_HOME/logs/` older than 7 days.
  2. **Deep GC (`--all`):**
     - **Model Pruning:** Scan `$DOP_HOME/models`. Delete any weights that are no longer referenced by a `dop.lock` file in known projects.
     - **Shadow Commit Pruning (Crucial):** Call `gitoxide` to find all `refs/tags/dop/*` tags older than 30 days. Delete the tags and run a git garbage collection (`git gc`) to purge the unreferenced blob/tree objects from `.git/objects`, freeing up disk space.

#### 3.4.6 `dop self-update`
- **Signature:** `dop self-update`
- **Implementation Logic:**
  1. Fetch the latest release manifest from the DoraFun CDN.
  2. Download the new `dop` binary to a temporary file (`dop.tmp`).
  3. Perform an **atomic rename** (`std::fs::rename` in Rust). Overwriting the currently running binary directly will cause an OS text-file-busy error (ETXTBSY). Atomic renaming sidesteps this, ensuring the CLI is never bricked even during a power loss.


## 4. Trade-offs & Technical Debt

- **Command Surface Area:** 16 commands is a significant baseline. However, the CLI architecture leverages subcommand parsing (e.g., `clap` in Rust) to provide extensive `--help` documentation, mitigating the learning curve.
- **Background Daemons (`start`/`stop`):** Implementing robust cross-platform daemons (especially on Windows) is technically complex. We accept this debt as edge deployments demand headless execution.

## 5. Unresolved Questions

- [ ] Should `dop status` output strict JSON when piped to another command, or should we introduce a `--json` flag?
- [ ] What is the exact timeout threshold before `dop stop` escalates from SIGTERM to SIGKILL?