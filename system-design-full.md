# Test Bench Control & Monitoring System — Part 1 (Sections 1–5)

**Version:** 1.0.0 · **Date:** February 2026
**Classification:** Internal Engineering Document

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              OPERATOR STATIONS                              │
│                        (Large curved monitors, Chrome)                      │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        Vue 3 Frontend                                │  │
│  │                                                                      │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │  │
│  │  │  P&ID    │ │Dashboard │ │Sequence  │ │ Command  │ │ Journal  │  │  │
│  │  │  Canvas  │ │  Panels  │ │  Runner  │ │  Panel   │ │   View   │  │  │
│  │  │(SVG/Vue) │ │(Vuetify) │ │  + Plot  │ │ 2-step   │ │  + Audit │  │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │  │
│  │       │             │            │             │            │         │  │
│  │  ┌────┴─────────────┴────────────┴─────────────┴────────────┴─────┐  │  │
│  │  │                  Pinia Stores (domain state)                    │  │  │
│  │  │  auth │ bench │ liveData │ commands │ sequences │ redlines     │  │  │
│  │  └───────────────────────┬────────────────────────────────────────┘  │  │
│  │                          │                                           │  │
│  │  ┌───────────────────────┴────────────────────────────────────────┐  │  │
│  │  │              Transport Layer (composables)                     │  │  │
│  │  │  REST (axios) │ WebSocket (native) │ SSE (EventSource)        │  │  │
│  │  └───────────────────────┬────────────────────────────────────────┘  │  │
│  └──────────────────────────┼───────────────────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │ HTTPS / WSS
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BACKEND (Python)                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    API Gateway (FastAPI + Uvicorn)                   │   │
│  │                                                                     │   │
│  │  ┌──────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐  │   │
│  │  │ Auth │ │  Bench   │ │ Command  │ │ Sequence │ │  Live Data  │  │   │
│  │  │ RBAC │ │  Config  │ │ Service  │ │  Engine  │ │  Streamer   │  │   │
│  │  └──┬───┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬──────┘  │   │
│  │     │          │            │             │              │          │   │
│  │  ┌──┴──────────┴────────────┴─────────────┴──────────────┴───────┐ │   │
│  │  │              Internal Event Bus (in-process asyncio)           │ │   │
│  │  └──┬──────────┬────────────┬─────────────┬──────────────┬───────┘ │   │
│  │     │          │            │             │              │          │   │
│  │  ┌──┴───┐ ┌────┴─────┐ ┌───┴──────┐ ┌───┴──────┐ ┌─────┴───────┐ │   │
│  │  │Redline│ │ Recorder │ │ Journal  │ │ Archive  │ │  Diagram    │ │   │
│  │  │Engine │ │ Service  │ │ Service  │ │ Service  │ │  Service    │ │   │
│  │  └──────┘ └──────────┘ └──────────┘ └──────────┘ └─────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                             │
│  ┌───────────────────────────┴──────────────────────────────────────────┐  │
│  │                    PLC Gateway Client (gRPC)                         │  │
│  └──────────────────────────┬───────────────────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │ gRPC (TLS)
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PLC GATEWAY (Industrial SE — separate team)              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  gRPC Server (.NET/C#)                                              │   │
│  │  ReadTag │ WriteTag │ SubscribeTags │ CommandAcks │ BenchState      │   │
│  │                    Beckhoff ADS.NET Driver                           │   │
│  └─────────────────────────────┬───────────────────────────────────────┘   │
└────────────────────────────────┼────────────────────────────────────────────┘
                                 │ ADS Protocol
                                 ▼
                      ┌──────────────────────┐
                      │   Beckhoff PLC(s)     │
                      │   TwinCAT Runtime     │
                      └──────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                            STORAGE LAYER                                    │
│  ┌──────────────┐  ┌──────────────────┐  ┌─────────────────────────────┐  │
│  │  PostgreSQL   │  │  TimescaleDB     │  │  MinIO (S3-compatible)     │  │
│  │  Config/Auth  │  │  Time-series     │  │  Run archives              │  │
│  │  Bench/Runs   │  │  sensor_data     │  │  Sequence files            │  │
│  │  Sequences    │  │  journal_events  │  │  Config backups            │  │
│  │  Redlines     │  │  redline_evals   │  │                            │  │
│  └──────────────┘  └──────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

| Flow | Path | Protocol |
|------|------|----------|
| Live sensor data | PLC → gRPC Gateway → Backend → WebSocket → Frontend | gRPC stream → WS |
| Operator command | Frontend → REST → Backend → gRPC Gateway → PLC | REST → gRPC |
| Command ack | PLC → gRPC → Backend → WS → Frontend | gRPC → WS push |
| Config CRUD | Frontend ↔ REST ↔ Backend ↔ PostgreSQL | REST |
| Time-series query | Frontend → REST → Backend → TimescaleDB | REST |
| Recording | Backend → TimescaleDB (batch insert) | Internal |
| Journal events | Any service → Journal Service → TimescaleDB | Internal bus |
| Run archive | Backend → Package → MinIO | Internal |

---

## 2. Backend Decomposition

All services run in a **single FastAPI process**. They communicate via an in-process event bus (asyncio queues). This avoids the operational complexity of microservices for a system with 1-5 concurrent users on a single test bench.

### Module Map

| Module | Responsibility |
|--------|---------------|
| `auth/` | JWT login, RBAC enforcement (admin, operator, viewer) |
| `bench/` | Bench config CRUD, versioning (every change = new version), zone management |
| `diagram/` | P&ID diagram JSON storage, export/import |
| `commands/` | Two-step command protocol, ack tracking, timeout monitoring |
| `sequences/` | Sequence CRUD + version control, step definition, file upload |
| `sequences/engine.py` | Async state machine: runs steps, sends commands, checks conditions |
| `sequences/routines.py` | Built-in Fill/Drain/Chilldown parameterized sequences |
| `redlines/` | Redline config CRUD, continuous evaluator (async loop), go/no-go checks |
| `live/` | gRPC subscription → event bus → WebSocket fan-out, per-client throttling |
| `recorder/` | TimescaleDB batch inserts, configurable frequency, auto-start with sequences |
| `journal/` | Append-only event log (DEBUG/INFO/WARN/ERROR), filterable query |
| `plc/` | gRPC client wrapper with auto-reconnect, health monitoring |
| `events/` | In-process async event bus (publish/subscribe) |
| `db/` | SQLAlchemy async session factory, Alembic migrations |

---

## 3. Data Architecture

### Database Selection

| Store | Technology | Rationale |
|-------|-----------|-----------|
| Config, auth, runs | **PostgreSQL 16** | ACID transactions, JSON columns, referential integrity |
| Time-series | **TimescaleDB** (Postgres extension) | Same SQL, same connection, hypertables, auto-compression. No separate InfluxDB to operate. |
| Object storage | **MinIO** (self-hosted S3) | Run archives (GBs), S3 API for portability |

### Key Tables

**PostgreSQL (configuration):**

```
users              — id, username, password_hash, role, is_active
benches            — id, name, current_version_id
bench_versions     — id, bench_id, version_number, config_snapshot (JSONB)
bench_items        — id, bench_id, name, item_type, component_type, zone, channels (JSONB)
diagrams           — id, bench_id, name, diagram_json (JSONB), schema_version
sequences          — id, bench_id, name, sequence_type, current_version_id
sequence_versions  — id, sequence_id, version_number, definition (JSONB), source
runs               — id, bench_id, sequence_version_id, bench_version_id, status, archive_url
redlines           — id, bench_id, name, channel_id, threshold_type, threshold_value, evaluation_mode, action_on_trigger
commands           — id, run_id, bench_item_id, channel, command_type, setpoint_value, confirmed_value, status, timestamps
```

**TimescaleDB (time-series, hypertables):**

```
sensor_data          — time, channel, value, quality, run_id
journal_events       — time, level, source, event_type, message, user_id, run_id, metadata (JSONB)
redline_evaluations  — time, redline_id, channel, measured_value, threshold_value, result, run_id
```

### Run Archive Format

```
run_2026-02-24_T-001.tar.gz
├── metadata.json           # Run ID, status, operator, timestamps
├── bench_config.json       # Frozen bench config at run start
├── sequence.json           # Sequence definition used
├── redlines.json           # Active redline config
├── sensor_data/*.csv       # Per-channel time-series
├── commands.json           # All commands issued
├── journal.json            # Journal slice (run start → end)
└── redline_evals.json      # All evaluations during run
```

---

## 4. API Design

### REST Endpoints

```
Auth:           POST /api/auth/login, /refresh, GET /me
Admin:          POST/PATCH/GET /api/admin/users
Bench:          GET /api/benches, GET/POST/PATCH .../items, .../versions, .../zones
Diagrams:       GET/POST/PUT /api/diagrams, .../export
Commands:       POST /api/commands/prepare, POST .../send, GET .../status
Sequences:      GET/POST /api/sequences, POST .../upload, GET .../versions, .../graph
Runs:           POST /api/runs/start, POST .../abort, GET .../status, .../steps
Redlines:       GET/POST/PATCH /api/redlines, GET .../status
Recorder:       POST /api/recorder/start, /stop, PATCH /frequency, GET /status
Journal:        GET /api/journal (paginated, filterable), GET .../export
Data:           GET /api/data/channels/{channel}/latest, /history, /bulk
Archives:       GET /api/archives, GET .../download
```

### WebSocket (single multiplexed connection)

```
WS /ws/live
  Client → { "subscribe": ["PT-001", "V-001.state"] }
  Client → { "unsubscribe": ["PT-001"] }
  Server → { "type": "data",             "channel": "PT-001", "value": 125.3 }
  Server → { "type": "bench_state",      "armed": true, "recording": true }
  Server → { "type": "command_ack",      "command_id": "...", "status": "applied" }
  Server → { "type": "redline_event",    "redline_id": "...", "result": "fail" }
  Server → { "type": "sequence_progress", "step": 5, "total": 12 }
  Server → { "type": "journal",          "level": "WARN", "message": "..." }
  Server → { "type": "zone_deactivation", "zone": "cryo-3", "active": false }
```

### gRPC Contract (Required from Industrial SE)

```protobuf
service PLCGateway {
  rpc ReadTag(ReadTagRequest) returns (TagValue);
  rpc ReadTags(ReadTagsRequest) returns (TagValues);
  rpc WriteTag(WriteTagRequest) returns (WriteResult);
  rpc SubscribeTags(SubscribeRequest) returns (stream TagUpdate);
  rpc GetBenchState(BenchStateRequest) returns (BenchState);
  rpc SubscribeCommandAcks(AckSubscribeRequest) returns (stream CommandAck);
}

message WriteTagRequest {
  string tag_name = 1;
  oneof value { bool bool_value = 2; double double_value = 3; int64 int_value = 4; }
  string command_id = 5;      // Our UUID for tracking acks
}

message CommandAck {
  string command_id = 1;
  string status = 2;          // "received", "applied", "rejected"
  TagValue read_back = 3;     // Actual PLC value after write
  int64 timestamp_ns = 4;
}
```

---

## 5. Domain Models

### Entity Relationships

```
Bench ─────< BenchVersion (config snapshots)
  │
  ├────< BenchItem ────< Channel (PLC tags)
  │         │
  │         └──── targeted by ──── Command
  │
  ├────< Diagram (P&ID JSON)
  │
  ├────< Sequence ─────< SequenceVersion ──── executed as ──── Run
  │                                              │
  │                                              ├──< Command
  │                                              └──< RunStep
  │
  ├────< Redline ────< RedlineEvaluation
  │
  └────< JournalEvent
```

### Key Models (Pydantic)

```python
class BenchItem:
    id: UUID
    name: str                  # "V-001"
    item_type: str             # "valve", "pump", "sensor"
    component_type: str        # Maps to Vue: "manual-valve", "centrifugal-pump"
    zone: str | None
    channels: list[ChannelConfig]

class ChannelConfig:
    channel_id: str            # "plc.valves.V001.state"
    data_type: str             # "bool", "float", "int"
    direction: str             # "read", "write", "read_write"
    plc_tag: str               # Actual PLC tag for gRPC
    units: str | None
    range: Range | None

class Command:
    id: UUID
    channel: str
    command_type: str          # "set_state", "set_setpoint"
    setpoint_value: Any
    confirmed_value: Any | None
    status: str                # prepared → sent → acknowledged → applied/failed/timeout

class Redline:
    id: UUID
    channel_id: str
    threshold_type: str        # "high", "low", "high_high", "low_low", "deviation"
    threshold_value: float
    evaluation_mode: str       # "go_no_go", "continuous", "always_active"
    action_on_trigger: str     # "safe_abort", "warn", "log_only"
    activation_delay_ms: int

class Run:
    id: UUID
    sequence_version_id: UUID
    bench_version_id: UUID
    status: str                # pending, armed, running, aborting, aborted, completed, failed
    current_step: int | None

class SequenceStep:
    step_number: int
    step_type: str             # "command", "wait", "condition", "parallel", "routine"
    target_item: str | None
    target_value: Any | None
    duration_ms: int | None
    condition_channel: str | None
    condition_operator: str | None
    condition_value: float | None
    routine_type: str | None   # "fill", "drain", "chilldown"
    routine_params: dict | None
```

---

*Sections 6–9 continue in Part 2: Safety & Correctness, Non-Functional Requirements, Repository Structure, Decision Rationale.*
# Test Bench System Architecture — Part 2 (Sections 6–9)

Continuation from Part 1 (Sections 1–5: High-Level Architecture, Backend Decomposition, Data Architecture, API Design, Domain Models).

---

## 6. Safety & Correctness

### 6.1 ARMED Mode

ARMED is a system-wide gate that prevents unsafe mutations during a test run. It is NOT a frontend toggle — it is enforced on the backend. The frontend merely reflects the state.

```
State Machine:

  IDLE ──[operator arms]──→ ARMED ──[go/no-go pass]──→ RUNNING ──[complete]──→ IDLE
    ↑                         │                           │
    │                         │ [go/no-go fail]           │ [abort]
    │                         ↓                           ↓
    │                       IDLE                      ABORTING ──→ IDLE
    │                                                     │
    └─────────────────────────────────────────────────────┘
```

**What ARMED blocks:**

| Operation | IDLE | ARMED | RUNNING | ABORTING |
|-----------|------|-------|---------|----------|
| Modify bench config | ✅ | ❌ | ❌ | ❌ |
| Modify active redlines | ✅ | ❌ | ❌ | ❌ |
| Modify active sequence | ✅ | ❌ | ❌ | ❌ |
| Start another run | ✅ | ❌ | ❌ | ❌ |
| Send manual commands | ✅ | ✅ | ✅ | ❌* |
| Abort run | — | ✅ | ✅ | ❌ |
| View all data | ✅ | ✅ | ✅ | ✅ |
| Edit diagram (layout only) | ✅ | ✅ | ✅ | ✅ |
| Modify non-active redlines (for next run) | ✅ | ✅ | ✅ | ✅ |
| Change recording frequency | ✅ | ❌ | ❌ | ❌ |

*During ABORTING, only the safe-abort sequence commands are permitted. All manual commands are blocked to prevent interference.

**Implementation — Backend Middleware Guard:**

```python
# app/middleware/armed_guard.py

from fastapi import Request, HTTPException

BLOCKED_WHEN_ARMED = {
    ("POST", "/api/benches/{id}/items"),
    ("PATCH", "/api/benches/{id}/items/{itemId}"),
    ("PATCH", "/api/redlines/{id}"),        # Only if redline is active
    ("POST", "/api/runs/start"),            # Can't start second run
    ("PATCH", "/api/recorder/frequency"),
}

async def armed_guard_middleware(request: Request, call_next):
    """
    Single enforcement point. Every state-modifying endpoint
    passes through here. No per-endpoint checks needed.
    """
    run_service = request.app.state.run_service
    active_run = await run_service.get_active_run()

    if active_run and active_run.status in ("armed", "running", "aborting"):
        route_key = (request.method, request.url.path)
        # Pattern match against blocked routes
        if is_blocked(route_key, BLOCKED_WHEN_ARMED):
            raise HTTPException(
                status_code=409,
                detail={
                    "error": "ARMED_MODE_ACTIVE",
                    "message": f"Cannot perform this action while run {active_run.id} is {active_run.status}",
                    "run_id": str(active_run.id),
                    "run_status": active_run.status,
                }
            )

    return await call_next(request)
```

**Why a middleware and not per-endpoint checks:** A single guard is impossible to forget when adding new endpoints. Per-endpoint `if armed: reject` checks get missed. The middleware operates at the routing level — even a new developer who doesn't know about ARMED mode can't accidentally bypass it.

### 6.2 Two-Step Command Protocol

This prevents accidental actuator commands. An operator must explicitly prepare, review, and confirm before any command reaches the PLC.

```
Timeline:

  Operator            Frontend              Backend              PLC Gateway           PLC
     │                    │                    │                     │                   │
     │  "Open V-001"      │                    │                     │                   │
     ├───────────────────→│                    │                     │                   │
     │                    │  POST /commands/   │                     │                   │
     │                    │  prepare           │                     │                   │
     │                    ├───────────────────→│                     │                   │
     │                    │                    │  validate:           │                   │
     │                    │                    │  - item exists       │                   │
     │                    │                    │  - channel writable  │                   │
     │                    │                    │  - ARMED allows it   │                   │
     │                    │                    │  - RBAC permits      │                   │
     │                    │                    │                     │                   │
     │                    │  ← 201 Created     │                     │                   │
     │                    │  { id: "cmd-123",  │                     │                   │
     │                    │    status:          │                     │                   │
     │                    │    "prepared" }     │                     │                   │
     │                    │                    │                     │                   │
     │ UI shows:          │                    │                     │                   │
     │ "Open V-001?       │                    │                     │                   │
     │  [Confirm] [Cancel]│                    │                     │                   │
     │                    │                    │                     │                   │
     │  Clicks [Confirm]  │                    │                     │                   │
     ├───────────────────→│                    │                     │                   │
     │                    │  POST /commands/   │                     │                   │
     │                    │  cmd-123/send      │                     │                   │
     │                    ├───────────────────→│                     │                   │
     │                    │                    │  gRPC WriteTag       │                   │
     │                    │                    ├────────────────────→│   ADS Write        │
     │                    │                    │                     ├──────────────────→│
     │                    │                    │                     │                   │
     │                    │  ← { status:       │                     │                   │
     │                    │    "sent" }         │                     │                   │
     │                    │                    │                     │                   │
     │ UI shows:          │                    │                     │                   │
     │ "SENT ⏳ Waiting   │                    │                     │                   │
     │  for ack..."       │                    │                     │                   │
     │                    │                    │                     │   ADS confirms     │
     │                    │                    │  ← CommandAck        │←──────────────────│
     │                    │                    │  { status: "applied",│                   │
     │                    │                    │    read_back: true } │                   │
     │                    │                    │                     │                   │
     │                    │  WS: command_ack   │                     │                   │
     │                    │←───────────────────│                     │                   │
     │                    │                    │                     │                   │
     │ UI shows:          │                    │                     │                   │
     │ "V-001: OPEN ✅    │                    │                     │                   │
     │  Applied"          │                    │                     │                   │
```

**Command States:**

| Status | Meaning |
|--------|---------|
| `prepared` | Validated and stored. Awaiting operator confirmation. |
| `sent` | Transmitted to PLC via gRPC. |
| `acknowledged` | PLC Gateway received the write. (gRPC WriteResult.success = true) |
| `applied` | PLC confirmed the value changed. (CommandAck with read-back matching setpoint) |
| `failed` | PLC rejected the write, or read-back doesn't match. |
| `timeout` | No ack received within `timeout_ms` (default 5000ms). |

**Command Confirmation Strategy — Read-Back vs. PLC Event:**

We use **read-back verification**, not trusting the write-success alone. The PLC Gateway's `CommandAck` stream includes a `read_back` field — the actual tag value after the write. The backend compares:

```python
async def process_command_ack(self, ack: CommandAck):
    cmd = await self.repo.get(ack.command_id)
    if not cmd or cmd.status not in ("sent", "acknowledged"):
        return

    cmd.acked_at = datetime.utcnow()

    if ack.status == "applied":
        # Verify read-back matches setpoint
        if values_match(cmd.setpoint_value, ack.read_back.value):
            cmd.status = "applied"
            cmd.confirmed_value = ack.read_back.value
        else:
            cmd.status = "failed"
            cmd.error_message = (
                f"Read-back mismatch: expected {cmd.setpoint_value}, "
                f"got {ack.read_back.value}"
            )
    elif ack.status == "rejected":
        cmd.status = "failed"
        cmd.error_message = f"PLC rejected: {ack.read_back}"
    else:
        cmd.status = "acknowledged"

    await self.repo.update(cmd)
    await self.event_bus.publish(CommandAckEvent(command=cmd))
```

**Timeout Handling:**

A background task monitors sent commands:

```python
async def command_timeout_monitor(self):
    """Runs as an asyncio task. Checks every 500ms for timed-out commands."""
    while True:
        await asyncio.sleep(0.5)
        stale = await self.repo.find_stale_commands(
            status="sent",
            older_than=timedelta(milliseconds=5000)
        )
        for cmd in stale:
            cmd.status = "timeout"
            cmd.error_message = "No acknowledgement received within timeout"
            await self.repo.update(cmd)
            await self.event_bus.publish(CommandTimeoutEvent(command=cmd))
            await self.journal.log(
                level="WARN",
                event_type="command_timeout",
                message=f"Command {cmd.id} to {cmd.channel} timed out",
                metadata={"command_id": str(cmd.id)}
            )
```

**Idempotency:** Each command has a UUID. The PLC Gateway uses this ID to deduplicate — if the same command ID is sent twice (e.g., network retry), the PLC processes it only once. The backend enforces that a `prepared` command can only be `sent` once (state transition guard).

### 6.3 Safe Abort / Soft Safe Pattern

When a redline triggers or an operator presses abort, the system doesn't hard-stop everything instantly (that could damage hardware — e.g., slamming a cryogenic valve shut). Instead, it executes a **safe-abort sequence**.

```
Abort Trigger
  (redline fail / operator abort / timeout)
          │
          ▼
  ┌───────────────────────────────────┐
  │      ABORTING State               │
  │                                   │
  │  1. Stop current sequence step    │
  │  2. Block all manual commands     │
  │  3. Execute safe-abort sequence:  │
  │     a. Set all valves to SAFE pos │
  │     b. Stop all pumps             │
  │     c. Wait for confirmations     │
  │     d. Verify safe state          │
  │  4. Stop recording (after flush)  │
  │  5. Archive run data              │
  │  6. Transition to IDLE            │
  └───────────────────────────────────┘
```

**Safe-abort sequence is defined per bench.** It is a special sequence stored in `bench_config` that specifies exactly which commands to send and in what order to bring the bench to a safe state. It is NOT hardcoded — different benches have different safe states.

```python
# Example safe-abort sequence definition in bench config
{
    "safe_abort_sequence": [
        {"action": "close_valve", "target": "V-001", "priority": 1},
        {"action": "close_valve", "target": "V-002", "priority": 1},
        {"action": "stop_pump", "target": "P-001", "priority": 2},
        {"action": "wait", "duration_ms": 2000},
        {"action": "verify_state", "targets": [
            {"item": "V-001", "expected": "closed"},
            {"item": "V-002", "expected": "closed"},
            {"item": "P-001", "expected": "stopped"}
        ]}
    ]
}
```

**Soft Safe vs. Hard Safe:**

| Mode | Trigger | Behavior |
|------|---------|----------|
| Soft Safe | Redline continuous violation, operator abort | Execute safe-abort sequence (orderly shutdown) |
| Hard Safe | Emergency stop (hardware), PLC connection lost | PLC handles autonomously via its own safety program. Backend detects disconnect, marks run as `failed`, logs event. |

**Critical design decision:** The backend never attempts to implement hard safety. The PLC has its own safety PLC (TwinSAFE on Beckhoff). If gRPC connection drops, the PLC falls back to its internal safe state. Our backend only handles soft safety (orderly shutdown when everything is still communicating).

### 6.4 Redline Evaluation Engine

```python
class RedlineEngine:
    """
    Runs as a persistent asyncio task.
    Subscribes to live data via the internal event bus.
    Evaluates all active redlines on every data tick.
    """

    async def start(self):
        self.active_redlines = await self.repo.get_active_redlines()
        await self.event_bus.subscribe("sensor_data", self.on_data)
        await self.event_bus.subscribe("run_state_change", self.on_run_state)

    async def on_data(self, event: SensorDataEvent):
        channel = event.channel
        value = event.value
        now = event.timestamp

        for rl in self.active_redlines:
            if rl.channel_id != channel:
                continue

            # Check activation delay
            if rl.activation_delay_ms > 0 and self.run_armed_at:
                elapsed = (now - self.run_armed_at).total_seconds() * 1000
                if elapsed < rl.activation_delay_ms:
                    continue  # Not yet active

            result = self.evaluate(rl, value)

            # Log evaluation
            await self.record_evaluation(rl, channel, value, result, now)

            if result == "fail":
                await self.handle_redline_failure(rl, channel, value)
            elif result == "warn":
                await self.handle_yellowline_warning(rl, channel, value)

    def evaluate(self, rl: Redline, value: float) -> str:
        """Returns 'pass', 'warn', or 'fail'."""
        match rl.threshold_type:
            case "high":
                return "fail" if value > rl.threshold_value else "pass"
            case "low":
                return "fail" if value < rl.threshold_value else "pass"
            case "high_high":
                return "fail" if value > rl.threshold_value else "pass"
            case "low_low":
                return "fail" if value < rl.threshold_value else "pass"
            case "deviation":
                # Deviation from a setpoint (stored in metadata)
                return "fail" if abs(value - rl.metadata.get("setpoint", 0)) > rl.threshold_value else "pass"

    async def handle_redline_failure(self, rl: Redline, channel: str, value: float):
        match rl.action_on_trigger:
            case "safe_abort":
                await self.journal.log(
                    level="ERROR",
                    event_type="redline_triggered",
                    message=f"REDLINE {rl.name}: {channel} = {value} exceeds {rl.threshold_value}. Initiating safe abort.",
                )
                await self.event_bus.publish(RedlineTriggeredEvent(
                    redline=rl, channel=channel, value=value, action="safe_abort"
                ))
                # Sequence engine subscribes to this and initiates abort
            case "warn":
                await self.journal.log(level="WARN", ...)
                await self.event_bus.publish(RedlineTriggeredEvent(..., action="warn"))
            case "log_only":
                await self.journal.log(level="INFO", ...)

    async def on_run_state(self, event: RunStateChangeEvent):
        """Reload redlines when run state changes (e.g., new run armed)."""
        if event.new_status == "armed":
            self.run_armed_at = datetime.utcnow()
            self.active_redlines = await self.repo.get_active_redlines()
```

**Evaluation modes summarized:**

| Mode | When Evaluated | Blocks Run Start? | Triggers Abort? |
|------|----------------|-------------------|-----------------|
| `go_no_go` | Once, at ARMED → RUNNING transition | Yes, if fails | No (run never starts) |
| `continuous` | Every data cycle during RUNNING | No | Yes, if action is `safe_abort` |
| `always_active` (yellowlines) | At all times, regardless of run state | No | No (warn only) |

---

## 7. Non-Functional Requirements

### 7.1 Performance — High-Frequency Signals

**Problem:** A Beckhoff PLC can produce data at 1-10kHz per channel. Hundreds of channels × 1kHz = 100k+ messages/second. Frontend browsers cannot render this.

**Strategy: Three-tier throttling.**

```
Tier 1: PLC → gRPC Gateway
  └── Industrial SE controls this. We request min_interval_ms in SubscribeRequest.
      Typical: 10ms (100Hz) for fast signals, 100ms (10Hz) for slow signals.

Tier 2: gRPC Gateway → Backend
  └── Backend receives at full subscribed rate.
      Inserts into TimescaleDB at recording frequency (configurable: 100Hz, 10Hz, 1Hz).
      Redline engine evaluates at full rate (safety-critical — no data loss).

Tier 3: Backend → WebSocket → Frontend
  └── Per-client downsampling based on what they're viewing:
      - Dashboard overview: 1Hz (1 update/second per channel)
      - Focused chart (zoomed in): 10Hz
      - P&ID live values: 2Hz
      - Client can request higher via WS message
```

**Implementation — Tier 3 Throttle:**

```python
# app/live/throttle.py

class ChannelThrottle:
    """Per-channel, per-client throttle. Drops messages between intervals."""

    def __init__(self, default_interval_ms: int = 500):
        self.default_interval_ms = default_interval_ms
        self.client_channels: dict[str, dict[str, float]] = {}  # client_id → {channel → last_sent_ts}

    def should_send(self, client_id: str, channel: str, timestamp: float) -> bool:
        last = self.client_channels.setdefault(client_id, {}).get(channel, 0)
        interval = self.default_interval_ms / 1000
        if timestamp - last >= interval:
            self.client_channels[client_id][channel] = timestamp
            return True
        return False

    def set_client_rate(self, client_id: str, channel: str, interval_ms: int):
        """Client requests a specific rate for a channel."""
        # Store per-client-channel override
        ...
```

**TimescaleDB insert batching:**

Don't insert one row at a time. Batch inserts in 100ms windows:

```python
class DataRecorder:
    def __init__(self):
        self.buffer: list[SensorRow] = []
        self.flush_interval = 0.1  # 100ms

    async def ingest(self, channel: str, value: float, time: datetime, quality: str):
        self.buffer.append(SensorRow(time=time, channel=channel, value=value, quality=quality))

    async def flush_loop(self):
        while True:
            await asyncio.sleep(self.flush_interval)
            if self.buffer:
                batch = self.buffer[:]
                self.buffer.clear()
                await self.db.copy_from(batch)  # PostgreSQL COPY protocol — bulk insert
```

Using `COPY` protocol instead of `INSERT` gets 10-50x throughput improvement for time-series inserts.

### 7.2 Observability

| Layer | Tool | What |
|-------|------|------|
| **Structured logging** | `structlog` (Python) | JSON logs with context (run_id, user_id, correlation_id). Every log line is parseable. |
| **Metrics** | `prometheus-fastapi-instrumentator` | HTTP latency, WS connection count, command latency, redline evaluation rate, buffer sizes. |
| **Tracing** | OpenTelemetry (OTLP) → Jaeger or Grafana Tempo | Trace a command from frontend click → REST → gRPC → PLC ack → WS push. Critical for debugging latency. |
| **Dashboards** | Grafana | Prometheus metrics + TimescaleDB queries. Separate from the app UI — this is for system operators, not test operators. |
| **Alerting** | Grafana Alerting or Alertmanager | PLC disconnect > 5s, command timeout rate > threshold, recording buffer full. |

**Structured log example:**

```python
import structlog
logger = structlog.get_logger()

logger.info(
    "command_sent",
    command_id=str(cmd.id),
    channel=cmd.channel,
    setpoint=cmd.setpoint_value,
    user_id=str(user.id),
    run_id=str(run.id) if run else None,
    latency_ms=elapsed_ms,
)
```

Output: `{"event": "command_sent", "command_id": "abc-123", "channel": "plc.valves.V001.state", "setpoint": true, "user_id": "usr-456", "run_id": "run-789", "latency_ms": 12, "timestamp": "2026-02-24T14:30:00Z", "level": "info"}`

### 7.3 Reliability

**gRPC Disconnect Handling:**

```python
class PLCClient:
    async def connect(self):
        self.channel = grpc.aio.insecure_channel(self.address)
        await self.channel.channel_ready()  # Blocks until connected
        self.connected = True

    async def subscribe_with_reconnect(self, tags: list[str]):
        """Auto-reconnect with exponential backoff."""
        backoff = 1
        while True:
            try:
                async for update in self.stub.SubscribeTags(SubscribeRequest(tag_names=tags)):
                    backoff = 1  # Reset on success
                    await self.event_bus.publish(SensorDataEvent.from_proto(update))
            except grpc.aio.AioRpcError as e:
                self.connected = False
                await self.journal.log(
                    level="ERROR",
                    event_type="plc_disconnect",
                    message=f"PLC connection lost: {e.code()}. Reconnecting in {backoff}s",
                )
                await self.event_bus.publish(PLCDisconnectEvent())
                await asyncio.sleep(backoff)
                backoff = min(backoff * 2, 30)  # Max 30s
                await self.connect()
```

**Frontend Degraded Mode:**

When the WebSocket drops, the frontend:
1. Shows a banner: "Connection lost. Reconnecting..." (auto-retry with backoff)
2. Freezes all live values at their last known state (with a "stale" indicator)
3. Disables command buttons (cannot send commands without connection)
4. Disables sequence start (cannot start without backend confirmation)
5. Keeps the P&ID diagram visible (static rendering continues)

This is handled by a `useConnectionHealth` composable in the frontend:

```typescript
// composables/useConnectionHealth.ts
export function useConnectionHealth() {
  const status = ref<'connected' | 'reconnecting' | 'disconnected'>('disconnected')
  const lastConnected = ref<Date | null>(null)
  const canSendCommands = computed(() => status.value === 'connected')
  const canStartRun = computed(() => status.value === 'connected')

  // Reconnect with exponential backoff...
  return { status, lastConnected, canSendCommands, canStartRun }
}
```

### 7.4 Security

| Concern | Approach |
|---------|----------|
| **Authentication** | JWT with short-lived access tokens (15 min) + refresh tokens (24h). Tokens include `user_id`, `role`, `permissions[]`. |
| **Authorization (RBAC)** | Three roles: `admin` (full access), `operator` (run sequences, send commands, configure redlines), `viewer` (read-only). Enforced in middleware, not per-endpoint. |
| **Least privilege** | `viewer` cannot send commands or modify config. `operator` cannot manage users. Service accounts for automated testing have a dedicated role. |
| **Audit trail** | Every command, config change, run start/abort, and login is journaled with user_id, timestamp, and before/after values. Journal is append-only. |
| **Transport** | HTTPS (TLS 1.3) for all frontend ↔ backend. gRPC with TLS for backend ↔ PLC Gateway. |
| **Input validation** | Pydantic on every endpoint. Command values validated against channel data type and range. |
| **Rate limiting** | Commands limited to 10/second per user (prevent script flooding). Login limited to 5 attempts per minute. |
| **Session management** | Tokens stored in httpOnly cookies (not localStorage). Refresh rotation — old refresh token invalidated on use. |

---

## 8. Repository Structure & CI/CD

### 8.1 Monorepo Structure

A monorepo is chosen because the frontend and backend are tightly coupled (shared domain concepts, API types). Separate repos would mean maintaining API contracts in two places. The monorepo keeps them synchronized with a shared CI pipeline.

```
test-bench/
├── .github/
│   └── workflows/
│       ├── ci.yml                     # Lint + test on PR
│       ├── deploy-staging.yml         # Auto-deploy on merge to main
│       └── deploy-prod.yml            # Manual trigger for production
│
├── frontend/                          # Vue 3 + TypeScript + Vuetify
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── .eslintrc.cjs
│   ├── src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   │
│   │   ├── domain/                    # Pure TS models (mirrors backend schemas)
│   │   │   ├── models.ts             # Bench, Node, Edge, Command, Run, Redline...
│   │   │   ├── defaults.ts           # Factories
│   │   │   └── geometry.ts           # Point math for canvas
│   │   │
│   │   ├── stores/                    # Pinia
│   │   │   ├── useAuthStore.ts
│   │   │   ├── useBenchStore.ts
│   │   │   ├── useLiveDataStore.ts
│   │   │   ├── useCommandStore.ts
│   │   │   ├── useSequenceStore.ts
│   │   │   ├── useRedlineStore.ts
│   │   │   ├── useRunStore.ts
│   │   │   └── useDiagramStore.ts
│   │   │
│   │   ├── composables/               # Reusable interaction logic
│   │   │   ├── useCanvas.ts           # Pan, zoom, coordinate transforms
│   │   │   ├── useNodeDrag.ts
│   │   │   ├── usePipeDraw.ts
│   │   │   ├── useOrthoRouter.ts
│   │   │   ├── useConnectionHealth.ts # WS reconnect, degraded mode
│   │   │   ├── useCommandFlow.ts      # Two-step command UI logic
│   │   │   ├── useRunControl.ts       # Start, arm, abort, progress
│   │   │   └── useSerializer.ts       # Diagram JSON export/import
│   │   │
│   │   ├── api/                        # HTTP + WS transport
│   │   │   ├── http.ts                # Axios instance with JWT interceptor
│   │   │   ├── ws.ts                  # WebSocket client with auto-reconnect
│   │   │   ├── auth.api.ts
│   │   │   ├── bench.api.ts
│   │   │   ├── commands.api.ts
│   │   │   ├── sequences.api.ts
│   │   │   ├── runs.api.ts
│   │   │   ├── redlines.api.ts
│   │   │   ├── data.api.ts
│   │   │   └── journal.api.ts
│   │   │
│   │   ├── components/
│   │   │   ├── canvas/                 # P&ID diagram renderer/editor
│   │   │   │   ├── DiagramCanvas.vue
│   │   │   │   ├── NodeRenderer.vue
│   │   │   │   ├── EdgeRenderer.vue
│   │   │   │   ├── JunctionNode.vue
│   │   │   │   ├── CanvasGrid.vue
│   │   │   │   ├── DrawingPreview.vue
│   │   │   │   └── SelectionBox.vue
│   │   │   │
│   │   │   ├── layout/                 # App shell
│   │   │   │   ├── AppToolbar.vue
│   │   │   │   ├── AppStatusBar.vue
│   │   │   │   ├── PalettePanel.vue
│   │   │   │   └── ArmedBanner.vue     # Prominent banner when ARMED
│   │   │   │
│   │   │   ├── commands/               # Command UI
│   │   │   │   ├── CommandPrepareDialog.vue
│   │   │   │   ├── CommandStatusBadge.vue
│   │   │   │   └── CommandHistory.vue
│   │   │   │
│   │   │   ├── sequences/              # Sequence runner UI
│   │   │   │   ├── SequenceEditor.vue
│   │   │   │   ├── SequenceGraph.vue   # Time plot with valve states
│   │   │   │   ├── SequenceProgress.vue
│   │   │   │   └── RoutineConfig.vue   # Fill/Drain/Chilldown params
│   │   │   │
│   │   │   ├── redlines/               # Redline config + status
│   │   │   │   ├── RedlineEditor.vue
│   │   │   │   ├── RedlineStatusGrid.vue
│   │   │   │   └── GoNoGoPanel.vue
│   │   │   │
│   │   │   ├── dashboard/              # Data visualization panels
│   │   │   │   ├── DashboardGrid.vue   # Configurable grid layout
│   │   │   │   ├── ValueCard.vue       # Single value display
│   │   │   │   ├── LiveChart.vue       # On-demand time plot
│   │   │   │   └── BenchItemDetail.vue # Pinned item detail panel
│   │   │   │
│   │   │   ├── journal/                # Journal / audit log viewer
│   │   │   │   ├── JournalTable.vue
│   │   │   │   └── JournalFilters.vue
│   │   │   │
│   │   │   └── auth/
│   │   │       ├── LoginPage.vue
│   │   │       └── UserManagement.vue
│   │   │
│   │   ├── lib/                         # 🏭 P&ID Component Library
│   │   │   ├── components/PID/          # ManualValve, GeneralPump, VerticalTank, AnalogDisplay, etc.
│   │   │   ├── components/common/       # PortIndicator, ComponentLabel
│   │   │   ├── composables/             # usePortEvents
│   │   │   ├── constants/
│   │   │   ├── types/
│   │   │   └── utils/
│   │   │
│   │   ├── pages/
│   │   │   ├── DesignerPage.vue        # P&ID editor/viewer
│   │   │   ├── DashboardPage.vue       # Live monitoring panels
│   │   │   ├── SequencesPage.vue       # Sequence config + runner
│   │   │   ├── RedlinesPage.vue        # Redline config
│   │   │   ├── JournalPage.vue         # Log viewer
│   │   │   ├── AdminPage.vue           # User management
│   │   │   └── ArchivesPage.vue        # Historical runs
│   │   │
│   │   └── router/
│   │       └── index.ts                 # Route definitions + auth guards
│   │
│   └── tests/
│       ├── unit/
│       └── e2e/
│
├── backend/                            # Python (FastAPI)
│   ├── pyproject.toml                  # uv/poetry project
│   ├── alembic.ini
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── deps.py
│   │   ├── auth/
│   │   ├── bench/
│   │   ├── diagram/
│   │   ├── commands/
│   │   ├── sequences/
│   │   ├── redlines/
│   │   ├── live/
│   │   ├── recorder/
│   │   ├── journal/
│   │   ├── plc/
│   │   ├── events/
│   │   └── db/
│   │       ├── session.py
│   │       ├── base.py
│   │       └── migrations/
│   │
│   └── tests/
│       ├── conftest.py
│       ├── test_auth/
│       ├── test_commands/
│       ├── test_sequences/
│       ├── test_redlines/
│       └── test_plc/
│
├── proto/                              # Shared gRPC definitions
│   └── plc_gateway.proto
│
├── docker/
│   ├── Dockerfile.frontend
│   ├── Dockerfile.backend
│   └── docker-compose.yml             # Local dev: Postgres + TimescaleDB + MinIO + backend + frontend
│
├── docs/
│   ├── architecture.md                 # This document
│   ├── api.md                          # OpenAPI reference (auto-generated link)
│   ├── deployment.md
│   └── runbook.md                      # Operational procedures
│
└── README.md
```

### 8.2 Recommended Libraries

**Backend (Python):**

| Library | Purpose | Why This One |
|---------|---------|--------------|
| `fastapi` | Web framework | Async-native, Pydantic integration, auto-docs |
| `uvicorn` | ASGI server | Fast, production-grade, used with FastAPI |
| `sqlalchemy[asyncio]` | ORM | Mature, async support, good for complex queries |
| `alembic` | Migrations | Standard for SQLAlchemy |
| `asyncpg` | Postgres driver | Fastest async Postgres driver (used by SQLAlchemy async) |
| `grpcio` + `grpcio-tools` | gRPC client | Standard Python gRPC implementation |
| `pydantic-settings` | Configuration | Type-safe env var loading |
| `python-jose[cryptography]` | JWT | JWT encoding/decoding |
| `passlib[bcrypt]` | Password hashing | Industry standard |
| `structlog` | Structured logging | JSON logs with context binding |
| `prometheus-fastapi-instrumentator` | Metrics | Auto-instruments FastAPI endpoints |
| `opentelemetry-*` | Tracing | Distributed tracing (OTLP export) |
| `boto3` or `minio` | S3 client | For MinIO/S3 archive uploads |
| `orjson` | JSON serialization | 10x faster than stdlib json for large payloads |
| `pytest` + `pytest-asyncio` | Testing | Async test support |
| `httpx` | Test client | Async HTTP client for integration tests |

**Frontend (Vue):**

| Library | Purpose | Why |
|---------|---------|-----|
| `vue` 3.4+ | Framework | Composition API, performance |
| `vuetify` 3.5+ | UI components | Dark theme, comprehensive, reduces custom CSS |
| `pinia` | State management | Justified by the number of cross-cutting stores |
| `vue-router` | Routing | Multiple pages (designer, dashboard, sequences, etc.) |
| `axios` | HTTP | Interceptors for JWT, retry logic |
| `chart.js` or `uplot` | Charts | uPlot for high-frequency (100k+ points), Chart.js for simple panels |
| `vue-grid-layout` | Dashboard grid | Drag-to-rearrange dashboard panels |

**What we deliberately DON'T use:**

| Avoided | Why |
|---------|-----|
| Redux/Vuex | Pinia is the successor, simpler |
| Socket.IO | Adds unnecessary abstraction over native WebSocket. We control both ends. |
| GraphQL | REST is simpler for this domain. We don't have deeply nested queries. |
| Kafka/RabbitMQ | Overkill for single-process backend. In-process event bus suffices. |
| Redis | Only needed if we scale to multiple backend instances (unlikely for a test bench). |
| Next.js/Nuxt | SSR is unnecessary for an internal operator tool. |

### 8.3 CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:latest-pg16
        env:
          POSTGRES_DB: testbench_test
          POSTGRES_PASSWORD: test
        ports: [5432:5432]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v1
      - run: cd backend && uv sync
      - run: cd backend && uv run ruff check .          # Lint
      - run: cd backend && uv run ruff format --check .  # Format check
      - run: cd backend && uv run mypy app/              # Type check
      - run: cd backend && uv run pytest --cov=app       # Test + coverage

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: cd frontend && npm ci
      - run: cd frontend && npm run lint                  # ESLint
      - run: cd frontend && npm run type-check            # vue-tsc
      - run: cd frontend && npm run test:unit             # Vitest
      - run: cd frontend && npm run build                 # Verify build succeeds

  proto:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install grpcio-tools
      - run: python -m grpc_tools.protoc -I=proto --python_out=backend/app/plc/proto --grpc_python_out=backend/app/plc/proto proto/plc_gateway.proto
      - run: echo "Proto compilation verified"
```

**Deployment:**

| Environment | Trigger | Strategy |
|-------------|---------|----------|
| Dev/Local | `docker compose up` | Everything in containers, hot-reload |
| Staging | Merge to `main` | Auto-deploy, runs against simulated PLC |
| Production | Manual tag + approval | Blue-green deploy, with rollback |

```yaml
# docker/docker-compose.yml (local dev)
services:
  db:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_DB: testbench
      POSTGRES_PASSWORD: devpass
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  backend:
    build: { context: ., dockerfile: docker/Dockerfile.backend }
    ports: ["8000:8000"]
    depends_on: [db, minio]
    environment:
      DATABASE_URL: postgresql+asyncpg://postgres:devpass@db:5432/testbench
      MINIO_ENDPOINT: minio:9000
      PLC_GATEWAY_ADDRESS: host.docker.internal:50051  # or simulated

  frontend:
    build: { context: ., dockerfile: docker/Dockerfile.frontend }
    ports: ["5173:5173"]
    depends_on: [backend]

volumes:
  pgdata:
  miniodata:
```

---

## 9. Decision Rationale

### Why These Specific Choices

**Why FastAPI over Django/Flask?**

Django brings an opinionated ORM, admin panel, and template system we don't need. Our frontend is Vue — we never render HTML server-side. Django's synchronous ORM would require `sync_to_async` wrappers for every database call in our streaming-heavy backend. Flask is lighter but lacks native async — we'd need Quart or similar, losing the Flask ecosystem. FastAPI gives us async-native request handling (critical for WebSocket + gRPC streams running concurrently), Pydantic validation on every endpoint (free), and auto-generated OpenAPI docs (the integration team can see the API without reading code).

**Why PostgreSQL + TimescaleDB and not separate InfluxDB?**

InfluxDB would require us to run, maintain, and query two completely different database systems with different query languages (SQL vs Flux/InfluxQL). TimescaleDB is a PostgreSQL extension — same connection, same SQL, same backup tools, same Alembic migrations. Configuration tables and time-series tables can JOIN in a single query (e.g., "show me all sensor readings for run X where the bench config had zone Z active"). InfluxDB can't do this without application-level joining.

**Why single-process backend, not microservices?**

This is a test bench control system, not a multi-tenant SaaS. There is ONE bench, ONE set of operators, typically 1-5 concurrent users. The entire state fits in memory. Microservices would mean: network calls between services (adding latency to safety-critical paths), distributed transactions (sequence engine needs to atomically read redline state and command state), multiple deployment targets, and service discovery. None of this is justified. If we need to scale later, the modular structure (each service in its own directory with clear interfaces) makes extraction straightforward.

**Why in-process event bus, not Kafka/Redis Streams?**

Same reasoning. The event bus handles ~1000 events/second (sensor data × channels). An in-process asyncio queue handles this trivially. Kafka adds operational complexity (ZooKeeper, brokers, topics, partitions, consumer groups) for a system with a single producer and a handful of consumers. If we need multi-process scaling (e.g., separate recorder process to isolate I/O), we'd introduce Redis Streams (simpler than Kafka, good enough for our throughput).

**Why WebSocket and not SSE for live data?**

SSE is unidirectional (server → client). We need bidirectional: the client sends subscribe/unsubscribe messages and rate-change requests. We could use SSE + REST (SSE for data, REST for subscribe control), but that's two connections where one WebSocket handles both. WebSocket also supports binary frames, which matters if we ever need to stream raw signal data efficiently.

**Why JWT and not session cookies?**

The system may have multiple frontend instances (operator stations). JWT allows stateless authentication — any backend instance can validate the token without shared session storage. Tokens are stored in httpOnly cookies (not localStorage) to prevent XSS access. Refresh rotation prevents token theft.

**Why Pinia stores map 1:1 to backend services?**

Each Pinia store (`useCommandStore`, `useRunStore`, `useRedlineStore`, etc.) mirrors a backend service boundary. This means: API calls are organized by domain (not scattered across components), store actions map directly to REST endpoints (predictable), and when the WebSocket receives a typed message (`type: "command_ack"`), the WS handler routes it to the correct store's handler. No ambiguity about where state lives.

**Why version everything (bench config, sequences, redlines)?**

A test run must be reproducible. If a redline threshold was 200 PSI when the run started, and someone changed it to 180 PSI afterward, the run record must show 200. By snapshotting the bench config version and sequence version at run start, we have a complete frozen record. The run archive includes these snapshots. This is a compliance and safety requirement, not a nice-to-have.

**Why "prepared → sent" command states, not just "send"?**

In aerospace test systems, accidental commands can damage hardware worth millions. The two-step protocol forces the operator to see what they're about to do and confirm. It also creates a clear audit trail: the journal shows who prepared the command (and when), who confirmed it (and when), and what the PLC reported. If something goes wrong, the investigation has full traceability.

**Why safe-abort is a sequence, not a hardcoded behavior?**

Different benches have different safe states. A cryogenic bench might need to vent before closing valves (to prevent pressure buildup). A fuel bench might need to purge before stopping pumps. Hardcoding "close all valves, stop all pumps" would be dangerous. The safe-abort sequence is defined per bench in configuration, reviewed by the test engineer, and versioned with the bench config.

**Why the PLC Gateway is a separate gRPC service owned by Industrial SE, not our backend talking ADS directly?**

Separation of expertise and fault isolation. Industrial SE owns the PLC program and the ADS protocol specifics. They know which tags are safe to write, what the PLC expects, and how the safety PLC behaves. Our backend only speaks gRPC — a clean, versioned contract. If the PLC program changes, Industrial SE updates the gateway; our backend doesn't change. If our backend crashes, the PLC gateway (and the PLC's safety program) continues to operate independently. This is a critical safety boundary.
