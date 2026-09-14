# Mirage Developer Onboarding (Chronos Framework)

**Naming convention:** Repository codename is **Apate**. Product/idea name is **Mirage**. Framework implementation name is **Chronos**.

**Program phases:**
- **Phase 1 (6 months):** Core systems and validation (complete)
- **Phase 2 (6 months):** Ubuntu-only AI artifact generation under strict constraints

---

## 0. Five-Minute Explanation (Problem → Solution → Why It Matters) 

### The Core Problem

Most honeypots fail for one reason: **state inconsistency under adversarial interaction**.

Attackers do not just run one command. They run chains of dependent actions and test whether the environment has memory:

1. `touch /tmp/.x`
2. `ls /tmp`
3. `cat /tmp/.x`
4. `stat /tmp/.x`

If any step contradicts a prior step, the deception is exposed. Traditional script-based honeypots often return plausible text, but they do not maintain a coherent filesystem state graph. LLM-only systems improve linguistic realism, but hallucinate state when context windows expire.

**The bottleneck is not language quality — it is transactional truth.**

### The Chronos Solution

Chronos treats deception as a systems problem, not a prompt problem.

- **FUSE-backed interface**: all attacker interactions become real filesystem syscalls.
- **State Hypervisor (Redis + Lua)**: mutations are handled atomically, persisted, and auditable.
- **Artifact Generation (Ollama)**: missing file content is generated locally under strict Ubuntu constraints, then committed to Redis — never re-generated.
- **Audit-first design**: all interactions are captured as structured session events.

The key design move: *generate once, constrain strictly, persist, and reuse consistently.*

### 30-Second Version (for slides)

> Existing honeypots fail when attackers test continuity. Chronos solves continuity at the syscall layer using FUSE + Redis atomic state, then uses a local LLM only to fill missing Ubuntu file content that is immediately persisted. Real-time simulation plugins generate background activity (cron jobs, service updates, user sessions, entropy), and an Overseer dashboard monitors all interactions. Result: a consistent, living Ubuntu environment that is easy to validate and hard for attackers to identify as synthetic.

---

## 1. The Big Picture 

**Chronos emulates exactly one system: an Ubuntu server.**

The machine type (web server, database host, development box) is not declared as a "persona" — it is implied by which packages are installed and which services are running. This is defined in `config/ubuntu.yaml`.

Two config files drive the entire AI layer:

| File | Defines |
|---|---|
| `config/ubuntu.yaml` | What the machine **is** (state) — packages, services, users, kernel |
| `config/generation_policy.yaml` | How artifacts are **generated** (behavior) — categories, constraints, model routing |

State and behavior are deliberately kept separate.

---

## 2. The Architecture 

**Two Main Processes** (Docker services):

### **Core Engine** (`src/chronos/core/main.py` → docker service `core-engine`)
```
SSH Gateway (src/chronos/gateway/ssh_server.py)
  │ accepts SSH connections → generates session_id → session_worker process
  ▼
ChronosFUSE (src/chronos/interface/fuse.py)
  │ every syscall is session-aware via fd-table[fd].session_id
  ▼
State Hypervisor (src/chronos/core/state.py)
  │ Redis + atomic Lua scripts — sole source of filesystem truth
  │
  ├── cache hit → return blob immediately (fast path)
  │
  └── cache miss → GenerationOrchestrator (src/chronos/intelligence/orchestrator.py)
             │
             ├── ArtifactPolicyEngine  assigns file class + category
             ├── PromptBuilder     builds constraint-first prompt
             ├── InferenceRuntime    sends to local Ollama
             └── SemanticValidator   validates vs. MachineState
                  │
                  └── persists blob + provenance to Redis

Watcher Pipeline (runs in core-engine)
  │ AuditLogStreamer → PostgreSQL event stream
  ├── EvidenceCollector → generates forensic checksums
  └── SkillDetector → MITRE ATT&CK pattern detection
```

### **World Engine** (`src/chronos/core/world_main.py` → docker service `world-engine`)
```
SimulationOrchestrator (src/chronos/simulation/orchestrator.py)
  │ manages event_bus and coordinates plugins
  ▼
Plugin Suite (runs as background threads):
  ├── MetadataAgingPlugin      → file timestamps drift naturally
  ├── CronServicePlugin        → /etc/crontab tasks fire periodically
  ├── AptServicePlugin         → simulate package manager updates
  ├── UserSimulatorPlugin      → generate user login/logout events
  ├── SystemEntropyPlugin      → inject realistic /proc entropy changes
  │
  ├── AuthLogPlugin            → append to /var/log/auth.log
  ├── SyslogPlugin             → append to /var/log/syslog
  └── JournalPlugin            → simulate systemd journal entries
       │
       └── all events published via EventBus (Redis-backed)
```

### **Overseer Dashboard** (`src/chronos/dashboard/` → Rust egui native app)
```
Real-time monitoring of:
  ├── Active sessions (per-SSH connection)
  ├── File generation events (FUSE cache hits/misses)
  ├── Simulation plugin state (cron, entropy, user activity)
  ├── Metrics export (Prometheus endpoint)
  └── Audit log streaming (PostgreSQL → live events)
```

---

## 3. Directory Map 

```text
src/chronos/
├── core/          # State management + core engine
│  ├── main.py       # Entry point: starts FUSE, SSH gateway, watcher pipeline
│  ├── world_main.py    # Entry point: starts simulation world engine
│  ├── state.py      # State Hypervisor (Redis + Lua)
│  ├── database.py     # Redis & PostgreSQL connections
│  ├── metrics.py      # Prometheus metrics export
│  ├── persistence.py   # PostgreSQL schema + ORM
│  ├── audit_spool.py   # Audit event accumulation
│  ├── data_logger.py   # Structured event logging
│  └── lua/        # Atomic scripts (atomic_create.lua, atomic_write.lua, etc.)
│
├── interface/       # FUSE filesystem layer
│  └── fuse.py       # Syscall handlers (read/write/mkdir/stat/…)
│
├── intelligence/      # AI generation pipeline
│  ├── ubuntu_profile.py  # Loads config/ubuntu.yaml → MachineState
│  ├── artifact_policy.py # File class resolution + artifact category sampling
│  ├── prompt_builder.py  # Constraint-first prompt construction
│  ├── validator.py    # 4-tier semantic validation
│  ├── orchestrator.py   # Non-blocking background generation pool
│  ├── inference.py    # Ollama HTTP client (local inference only)
│  ├── deterministic_renderer.py  # Renders MachineState to text
│  ├── provenance.py    # Tracks generation provenance + versioning
│  └── fallback.py     # Fallback when Ollama is unavailable
│
├── gateway/        # Entry points + containment
│  ├── ssh_server.py    # SSH honeypot (paramiko-based)
│  ├── dispatcher.py    # AST-based command dispatcher
│  ├── shell_parser.py   # Shell syntax parsing
│  └── containment.py   # Process isolation, chroot, resource limits
│
├── simulation/      # World engine + plugins
│  ├── orchestrator.py   # SimulationOrchestrator + event_bus coordination
│  ├── event_bus.py    # Redis-backed event publishing/subscribing
│  ├── logs/        # Log simulation plugins
│  │  ├── auth.py       # AuthLogPlugin (/var/log/auth.log)
│  │  ├── syslog.py     # SyslogPlugin (/var/log/syslog)
│  │  └── journal.py    # JournalPlugin (systemd journal)
│  ├── metadata/      # Metadata + background activity plugins
│  │  ├── aging.py      # MetadataAgingPlugin (file timestamp drift)
│  │  └── entropy.py    # SystemEntropyPlugin (/proc entropy)
│  └── services/      # Background service simulation plugins
│     ├── cron.py       # CronServicePlugin (periodic tasks)
│     ├── apt.py        # AptServicePlugin (package updates)
│     └── user_simulator.py  # UserSimulatorPlugin (login/logout events)
│
├── dashboard/      # Overseer native UI (Rust egui)
│  ├── Cargo.toml      # Rust project definition
│  └── src/        # Main dashboard UI code
│
├── watcher/        # Monitoring + forensics
│  ├── log_streamer.py   # Real-time PostgreSQL audit streaming
│  ├── evidence_collector.py  # Forensic checksum generation
│  └── event_processor.py # Event ingestion + attack pattern detection
│
├── skills/         # Threat intelligence (analysis only, not generation)
│  ├── command_analyzer.py # MITRE ATT&CK detection
│  ├── threat_library.py  # Known attack signatures
│  └── skill_detector.py  # Attacker behavioral profiling
│
├── layer0/         # Rust performance layer
│  ├── Cargo.toml      # Rust project definition
│  └── src/        # Traffic classification, circuit breakers
│
└── utils/         # Shared utilities
```

---

## 3.5 Simulation Plugins Deep Dive

**Why Plugins?** Traditional honeypots look dead. Attackers run `ps`, `netstat`, `last`, and notice no background activity. Chronos solves this by having real processes (simulated) that modify the filesystem and log files over time.

**The Event Bus**: All plugins communicate via a Redis-backed EventBus (see `src/chronos/simulation/event_bus.py`). Each plugin:
1. Registers a handler for specific event types.
2. Can publish new events (e.g., "user logged in").
3. Processes events asynchronously in background threads.
4. All mutations go through State Hypervisor (atomicity guaranteed).

**Plugin Categories**:

| Plugin | Function | Example Output |
|--------|----------|-----------------|
| **MetadataAgingPlugin** | Slowly drifts file timestamps, access times. | `stat /etc/passwd` shows realistic time progression |
| **CronServicePlugin** | Fires /etc/cron.d/* jobs periodically. | Lines appended to /var/log/auth.log when cron runs |
| **AptServicePlugin** | Simulates package manager cache updates. | /var/cache/apt/ metadata changes |
| **UserSimulatorPlugin** | Generates login/logout events. | /var/run/utmp entries, /var/log/auth.log "user logged in" |
| **SystemEntropyPlugin** | Updates /proc/sys/kernel/random/entropy_avail. | Entropy value drifts realistically |
| **AuthLogPlugin** | Appends to /var/log/auth.log. | Accumulates login/cron events from other plugins |
| **SyslogPlugin** | Appends to /var/log/syslog. | System messages, service restarts |
| **JournalPlugin** | Simulates systemd journal entries. | /var/log/journal/* binary entries |

**Adding a New Plugin**:

1. Create file: `src/chronos/simulation/services/my_plugin.py`
2. Inherit from base plugin class (define in event_bus.py or create abstract base).
3. Implement `__init__(redis, event_bus)` and event handler methods.
4. Register in `src/chronos/simulation/orchestrator.py::register_plugins()`.
5. Publish events via `event_bus.publish(MyEvent(...))`.
6. Plugin is active immediately on world tick.

---

## 4. Life of a Command 

### Connection Setup

**Command**: SSH connection to `localhost:2222`

1. **SSH Gateway** (paramiko) accepts the connection.
2. Generates a unique `session_id` (UUID).
3. Forks a **session_worker** subprocess (runs in isolation with tight resource limits).
4. Worker registers PID mapping: `session_pid:{pid} → session_id` in Redis (120s TTL).
5. Worker initializes `SessionEnvironment` (shell state, PWD, environment variables).
6. Worker chroots into `/mnt/honeypot` and applies containment (CPU/memory/file limits, AppArmor).
7. Worker enters command loop, awaiting input over a pipe connection.

### Scenario A: Attacker writes a file

**Command**: `echo "test" > /tmp/pwn`

1. **SSH Channel** passes the command to **session_worker**.
2. **ShellParser** parses it into an AST.
3. **ASTDispatcher** routes to file creation handler.
4. **FUSE** intercepts the `create()` syscall.
5. **State Hypervisor** runs a **Lua script** in Redis: checks parent dir, allocates inode, links name → inode.
6. **FUSE** returns inode to OS, then fire-and-forget submits background generation task to `GenerationOrchestrator` (if file content is needed).
7. OS calls `write(inode, data)` → FUSE stores the written bytes as a blob in Redis.
8. Future reads of this file hit the cache immediately.

### Scenario B: Attacker reads a ghost file

**Command**: `cat /etc/nginx/nginx.conf` (inode exists, no content yet)

1. **FUSE** intercepts `open()` + `read()`.
2. **State Hypervisor** checks Redis: inode exists, but `content_hash` is NULL.
3. **FUSE** calls `orchestrator.get_or_generate(inode, path, session_id, machine_state)`.
4. **ArtifactPolicyEngine** resolves: `config_file`, category `valid`, model `llama3:8b`, max 80 lines.
5. **PromptBuilder** builds a constrained prompt with only nginx-relevant MachineState facts (version, installed modules, etc.).
6. **InferenceRuntime** sends to local Ollama.
7. **SemanticValidator** checks result against MachineState (nginx version, no mysql references, Ubuntu conventions).
8. If validation passes: result persisted to Redis with provenance metadata.
9. Future reads of this file hit the cache (fast path).
10. If Ollama times out or validation fails: FUSE returns `EAGAIN`, client retries, generation continues in background. Eventually hits cache.

### Scenario C: World Engine updates background state

**World Tick** (every 60 seconds, from `world_main.py`)

1. **SimulationOrchestrator** publishes `WorldTick` event to EventBus.
2. Registered plugins receive the tick:
   - **MetadataAgingPlugin**: increments file access times slightly.
   - **CronServicePlugin**: checks if any /etc/cron.d/* jobs should fire. If yes, appends to `/var/log/auth.log` (via AuthLogPlugin).
   - **AptServicePlugin**: randomly updates /var/cache/apt metadata.
   - **UserSimulatorPlugin**: occasionally injects login/logout events.
   - **SystemEntropyPlugin**: updates `/proc/sys/kernel/random/entropy_avail`.
   - **AuthLogPlugin**, **SyslogPlugin**, **JournalPlugin**: append accumulated events to their respective log files.
3. All mutations go through State Hypervisor (atomicity guaranteed).
4. Audit log events are published to PostgreSQL (via audit_spool).
5. **Overseer Dashboard** ingests these events in real-time.

---

## 5. Developer Cheatsheet 

**Start the full stack (all Docker services):**
```bash
make up
```
This starts:
- `chronos_redis` (state store)
- `chronos_db` (PostgreSQL audit logs)
- `chronos_ollama` (local LLM inference)
- `chronos_core` (FUSE + SSH gateway)
- `chronos_world` (simulation plugins)

**Launch the Overseer Dashboard (native Rust UI):**
```bash
make ui
```
Connects to the running stack and displays real-time monitoring of:
- Active SSH sessions
- File generation events (cache hit/miss)
- Simulation plugin activity
- Metrics and audit streams

**Watch logs from core engine:**
```bash
make logs
```

**Watch logs from world engine:**
```bash
make logs-world
```

**Connect as attacker:**
```bash
make ssh
# or: ssh -p 2222 ubuntu@localhost  (any password works)
```
This connects to the SSH gateway. Commands are parsed, dispatched, and backed by the FUSE filesystem.

**Run verification tests (Phase 1-4):**
```bash
make verify
```
Tests:
- Phase 1: State Hypervisor & DB
- Phase 2: FUSE interface
- Phase 3: Intelligence layer (generation + validation)
- Phase 4: Gateway, Watcher, Skills

**Run core infrastructure validation:**
```bash
make validate-core
```

**Run attack detection validation:**
```bash
make validate-attacks
```

**Run full end-to-end validation:**
```bash
make validate-full
```

**Run demo scripts (no Docker required):**
```bash
make demo-standalone   # Skills showcase
make demo-integration  # Full system demo
```

**Reset everything (stop stack + remove volumes):**
```bash
make clean
```

**Edit the Ubuntu machine definition:**
```bash
$EDITOR config/ubuntu.yaml
```

**Edit generation behavior (categories, constraints, model routing):**
```bash
$EDITOR config/generation_policy.yaml
```

---

## 5. Overseer Dashboard Guide

The **Overseer Dashboard** is a native Rust egui application that provides real-time monitoring of the Chronos honeypot stack.

**Launch the Dashboard:**
```bash
make ui
```

**What It Shows**:

| Panel | Information |
|-------|-------------|
| **Active Sessions** | Current SSH connections, session IDs, usernames, connected time |
| **File Generation Events** | Real-time log of FUSE cache hits/misses, generated artifacts, model used |
| **Generation Queue** | Pending artifacts waiting to be generated, priority, constraints |
| **Simulation Plugins** | Status of each plugin (enabled/disabled), last tick time, event count |
| **Metrics** | Prometheus-style metrics (session count, generation latency, cache hit rate) |
| **Audit Stream** | Live feed of PostgreSQL audit events, user commands, system activity |
| **Error Log** | Any errors from core engine, world engine, or plugins |

**Dashboard Architecture**:

The dashboard is a standalone Rust program (not Python) that:
1. Connects to the running Chronos stack via HTTP (metrics endpoint) and Redis (event stream).
2. Uses egui for cross-platform native UI (works on macOS, Linux, Windows).
3. Subscribes to Redis EventBus to receive world-tick updates in real-time.
4. Queries PostgreSQL audit logs for historical analysis.
5. Runs entirely separately from the core engine — can be launched/stopped without affecting honeypot operation.

**Typical Workflow**:

1. Start the full stack: `make up`
2. In another terminal, launch dashboard: `make ui`
3. Watch real-time activity as you SSH in and run commands: `make ssh`
4. Dashboard shows:
   - SSH connection established → new session entry
   - Each command execution → file generation events
   - Background plugins → simulation events on world ticks

---

## 7. Pro Tips 

- **Dual Entry Points**: Two separate Python processes run in Docker:
  - `core-engine` (main.py): Handles SSH connections, FUSE, file generation, watcher pipeline.
  - `world-engine` (world_main.py): Runs simulation plugins that generate background activity.
  - Both communicate via Redis (state store) and PostgreSQL (audit logs), never via direct IPC.

- **State Atomicity**: File creation atomicity lives in `src/chronos/core/lua/atomic_create.lua` and `atomic_write.lua`. These Lua scripts run inside Redis with no race conditions. Never bypass them.

- **No Cloud LLMs**: All inference goes to Ollama on `chronos-net` (Docker internal network). There are no OpenAI/Anthropic API keys. Everything runs locally.

- **Ubuntu Only**: `SemanticValidator` will reject any generated content that references Windows, macOS, or non-Ubuntu package managers (yum, dnf, zypper). This is intentional and non-configurable.

- **Session ≠ Process**: FUSE syscalls are session-aware via the PID mapping in Redis (`session_pid:{pid} → session_id`). Never rely on `/proc` PID lookups across the chroot boundary.

- **Simulation Plugins Are Event-Driven**: Plugins communicate via Redis EventBus, not direct function calls. This allows them to be added/removed without restarting the core engine. New plugin? Register it in `src/chronos/simulation/orchestrator.py::register_plugins()`.

- **Skill Detection ≠ Generation Fidelity**: The `SkillDetector` in `src/chronos/skills/` feeds monitoring and logging. It does **not** change which model generates content — that is determined solely by file class in `generation_policy.yaml`.

- **Overseer Dashboard**: The Rust egui dashboard is optional for monitoring but very useful for understanding system behavior during attacks. Run `make ui` to launch it.

- **Resource Limits Are Enforced**: Session workers run with:
  - CPU: 60 seconds max
  - Memory: 512 MB max
  - File size: 1 MB max
  - Open files: 128 max
  These are set in `session_worker()` (ssh_server.py) via `resource.setrlimit()`. Adjust if needed for specific test scenarios.

- **Provenance Tracking**: Every generated file blob is stored with provenance metadata (model, prompt hash, timestamp, validation score). This is critical for forensics. See `src/chronos/intelligence/provenance.py`.

- **Log Files Are Simulated**: Log files like `/var/log/auth.log`, `/var/log/syslog`, etc. are not static files. They are dynamically appended to by simulation plugins on each world tick. Attackers see realistic, time-ordered entries.

---

## 8. Testing & Validation Phases

Chronos has a structured test framework with four phases of verification. Run all of them with:

```bash
make verify
```

### **Phase 1: State Hypervisor & Database**
**File**: `tests/verification/verify_phase1.py`

Tests the foundational layer:
- Redis connectivity and atomic Lua script execution
- PostgreSQL schema and connection pooling
- State persistence across process restarts
- File inode allocation and linking
- Basic filesystem state consistency

**Run individually**:
```bash
PYTHONPATH=src python3 tests/verification/verify_phase1.py
```

**Expected output**: "Phase 1: ✅ 8/8 tests passed"

### **Phase 2: FUSE Interface**
**File**: `tests/verification/verify_phase2.py`

Tests syscall handling:
- FUSE mount point exists and is accessible
- Syscall interception (open, read, write, mkdir, stat, etc.)
- Session-aware file attribution via PID mapping
- Proper error codes and edge cases

**Run individually**:
```bash
PYTHONPATH=src python3 tests/verification/verify_phase2.py
```

**Expected output**: "Phase 2: ✅ 6/6 tests passed"

### **Phase 3: Intelligence Layer**
**File**: `tests/verification/verify_phase3.py`

Tests AI generation pipeline:
- UbuntuProfile loads config/ubuntu.yaml correctly
- ArtifactPolicyEngine resolves file classes and categories
- PromptBuilder constructs valid Ubuntu-constrained prompts
- SemanticValidator rejects non-Ubuntu content
- Inference pipeline integrates with Ollama (if available)

**Run individually**:
```bash
PYTHONPATH=src python3 tests/verification/verify_phase3.py
```

**Expected output**: "Phase 3: ✅ 7/7 tests passed"

### **Phase 4: Gateway & Watcher**
**File**: `tests/verification/verify_phase4.py`

Tests high-level components:
- SSH gateway accepts connections
- ShellParser handles various shell syntax
- ASTDispatcher routes commands correctly
- Watcher pipeline captures audit events
- SkillDetector identifies MITRE ATT&CK patterns

**Run individually**:
```bash
PYTHONPATH=src python3 tests/verification/verify_phase4.py
```

**Expected output**: "Phase 4: ✅ 5/5 tests passed"

### **Real Attack Validation**
**File**: `tests/validation/test_real_attack.py`

Simulates actual attack scenarios:
- Persistence via cron jobs
- Privilege escalation attempts
- Data exfiltration detection
- Attack chain validation (multi-step attacks)

**Run individually**:
```bash
PYTHONPATH=src python3 tests/validation/test_real_attack.py
```

### **Core Infrastructure Validation**
Validates system stability:
```bash
make validate-core
```

### **Full End-to-End Validation**
Runs complete integration tests:
```bash
make validate-full
```

---

## 9. Troubleshooting

**Q: "FUSE mount point not accessible"**  
A: Ensure you're running inside Docker or the mount exists. Check: `mount | grep honeypot`

**Q: "Ollama connection timeout"**  
A: Ollama container may be starting. Wait 30s and retry. Check: `docker logs chronos_ollama`

**Q: "Redis connection refused"**  
A: Redis container may not be healthy. Check: `make ps` and verify `chronos_redis` is running.

**Q: "Generation quality is poor"**  
A: Check `config/generation_policy.yaml` constraints. May need to increase `max_lines` or adjust model temperature in `inference.py`.

**Q: "Session worker crashes with resource limit"**  
A: Adjust `resource.setrlimit()` calls in `src/chronos/gateway/ssh_server.py::session_worker()`.

**Q: "Dashboard won't connect"**  
A: Ensure the stack is running (`make up`). Dashboard expects Redis on `localhost:6379` and metrics on `localhost:9090`.
