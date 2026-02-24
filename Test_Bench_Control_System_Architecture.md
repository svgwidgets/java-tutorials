# Test Bench Control System — Production Architecture

**Version:** 0.1.0-DRAFT  
**Date:** 2026-02-24  
**Classification:** Internal Engineering

---

## 1. High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              OPERATORS                                    │
│                     (Browser — Vue 3 SPA)                                │
└──────────────┬─────────────────────────────────────┬──────────────────────┘
               │ HTTPS / WSS                         │ SSE (live data fan-out)
               ▼                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         NGINX / TRAEFIK                                  │
│              (TLS termination, static files, path routing)               │
└──────────────┬─────────────────────────────────────┬──────────────────────┘
               │                                     │
       REST + WS│                             SSE    │
               ▼                                     ▼
┌──────────────────────────┐          ┌────────────────────────────────────┐
│   COMMAND / CONFIG API   │          │     LIVE-DATA STREAMING SERVICE    │
│   (FastAPI, sync ops)    │          │     (FastAPI/Starlette, async)     │
│                          │          │                                    │
│  • Auth / RBAC           │          │  • Fan-out to N browser clients    │
│  • Bench config CRUD     │          │  • Throttle / downsample per sub   │
│  • Sequence CRUD         │          │  • Heartbeat / reconnect token     │
│  • Command dispatch      │          │                                    │
│  • Journal queries       │          └──────────┬─────────────────────────┘
│  • Redline config        │                     │ subscribes
│  • Recording mgmt        │                     ▼
└────────┬─────────────────┘          ┌────────────────────────────────────┐
         │ enqueues                   │         REDIS (Pub/Sub + Streams)  │
         ▼                            │  • Channel: bench.<item>.<signal>  │
┌──────────────────────────┐          │  • Channel: events.*              │
│     COMMAND GATEWAY      │          │  • Stream: commands (with ACK)     │
│  (async worker / queue)  │◄─────── │  • Stream: journal                 │
│                          │ publish  └──────────┬─────────────────────────┘
│  • Validate ARMED gate   │                     │ publish
│  • Two-step confirm      │                     │
│  • Timeout + retry       │          ┌──────────┴─────────────────────────┐
│  • Read-back verify      │          │       PLC DATA INGESTOR            │
└────────┬─────────────────┘          │  (Python async, single instance)   │
         │ gRPC                       │                                    │
         ▼                            │  • gRPC stream subscribe           │
┌──────────────────────────┐          │  • Normalize → Redis publish       │
│    PLC gRPC GATEWAY      │          │  • Change-detect (deadband)        │
│  (owned by Industrial SE)│◄─────────│  • Write to TimescaleDB            │
│                          │ gRPC     └────────────────────────────────────┘
│  • ReadTag(channel)      │
│  • WriteTag(channel,val) │          ┌────────────────────────────────────┐
│  • Subscribe(channels)   │          │      SEQUENCE ENGINE               │
│  • Ack stream            │          │  (Python, dedicated process)       │
│  • GetBenchState()       │          │                                    │
│  • SetBenchArmed(bool)   │          │  • Interprets sequence definition  │
└──────────┬───────────────┘          │  • Drives commands via Gateway     │
           │ ADS.NET                  │  • Evaluates redlines continuously │
           ▼                          │  • Manages ARMED lifecycle         │
┌──────────────────────────┐          │  • Emits progress events           │
│    BECKHOFF PLC          │          │  • Safe-abort orchestration        │
│  (TwinCAT / EtherCAT)   │          └────────────────────────────────────┘
└──────────────────────────┘
                                      ┌────────────────────────────────────┐
                                      │       RECORDER / ARCHIVER          │
                                      │  (Python worker)                   │
                                      │                                    │
                                      │  • Start/stop recording sessions   │
                                      │  • High-freq burst capture         │
                                      │  • Package run → HDF5/Parquet      │
                                      │  • Upload to object storage        │
                                      │  • Low-freq continuous logging     │
                                      └────────────────────────────────────┘

STORAGE
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │ TimescaleDB  │  │    MinIO      │  │    Redis     │
│              │  │              │  │  (S3-compat)  │  │              │
│ • Users/RBAC │  │ • Time-series│  │ • Run archives│  │ • Pub/Sub    │
│ • Bench cfg  │  │ • Sensor data│  │ • HDF5 packs  │  │ • Cmd queue  │
│ • Sequences  │  │ • Continuous │  │ • Config snaps│  │ • Caching    │
│ • Journal    │  │   logging    │  │               │  │ • Sessions   │
│ • Redlines   │  │              │  │               │  │              │
│ • Runs meta  │  │              │  │               │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Why this shape

**Two API surfaces.** The command/config API handles stateful CRUD and authentication — latency-tolerant, transactional. The live-data streaming service is a separate async process optimized for fan-out of high-frequency signals over SSE (simpler than WebSocket for unidirectional data push, auto-reconnect built into the browser `EventSource` API, works through corporate proxies). Commands still flow over WebSocket on the command API for bidirectional ACK.

**Redis as the internal bus.** Every backend service publishes/subscribes through Redis Streams and Pub/Sub. This decouples the PLC ingestor (which must never block) from the API layer, the sequence engine, the recorder, and the live-data fan-out. Redis Streams give us at-least-once delivery with consumer groups, which we need for command acknowledgement and journaling.

**Sequence engine as a separate process.** A running test sequence must survive API restarts. It holds the ARMED lock, drives the redline evaluator, and issues commands through the same command gateway the UI uses. If the process crashes, the PLC's own watchdog (owned by Industrial SE) handles hardware safety; the sequence engine handles *application-level* safe-abort.

---

## 2. Backend Service Decomposition

| Service | Responsibility | Runtime | Scaling |
|---|---|---|---|
| **Auth / RBAC Service** | Login (OAuth2 / LDAP), JWT issuance, role enforcement middleware | FastAPI (command API) | Stateless, horizontal |
| **Bench Config Service** | CRUD for bench definitions, versioned with Postgres `jsonb` + `version` column | FastAPI | Stateless |
| **P&ID Diagram Service** | Store/retrieve diagram layouts (canvas objects, positions, links) | FastAPI | Stateless |
| **Command Service** | Validate → enqueue → dispatch → track ACK → timeout | FastAPI + Redis Streams | Single-leader per bench |
| **Sequence Engine** | Parse sequence def → step execution → redline eval → ARMED management → abort | Dedicated Python process | **One per bench** (singleton) |
| **Redline Engine** | Evaluate redlines/yellowlines against live data, emit violations | Embedded in Sequence Engine + standalone for always-active yellowlines | Co-located |
| **Live Data Service** | Subscribe Redis channels → SSE fan-out to browsers | Starlette async | Horizontal (stateless) |
| **PLC Data Ingestor** | gRPC subscribe → normalize → Redis publish → TimescaleDB write | Python asyncio | **One per PLC** (singleton) |
| **Recorder / Archiver** | Session recording, burst capture, packaging, upload | Python worker | Single-leader per bench |
| **Journal Service** | Append-only event log, query API | FastAPI + Postgres | Stateless reads |

### Service interaction summary

```
Browser ──REST──► Auth ──JWT──► Command Service ──Redis Stream──► Command Gateway ──gRPC──► PLC
Browser ──WS────► Command Service (bidirectional ACK)
Browser ──SSE───► Live Data Service ◄──Redis Pub/Sub──◄ PLC Ingestor ◄──gRPC stream──◄ PLC
Sequence Engine ──Redis Stream──► Command Gateway (same path as UI commands)
Sequence Engine ──Redis Pub/Sub──► Journal, Live Data, Recorder
```

---

## 3. Data Architecture

### 3.1 PostgreSQL (relational, config, audit)

```sql
-- Core identity
users (id, email, password_hash, display_name, created_at, updated_at)
roles (id, name, permissions jsonb)          -- e.g. "operator", "engineer", "admin"
user_roles (user_id FK, role_id FK)

-- Bench configuration (versioned)
benches (id, name, description, created_at)
bench_versions (
    id, bench_id FK, version int,
    config jsonb,                             -- full bench definition snapshot
    created_by FK, created_at,
    is_active boolean DEFAULT false,
    UNIQUE(bench_id, version)
)

-- Bench items (sensors, actuators, zones)
bench_items (
    id, bench_version_id FK,
    tag varchar(64),                          -- "PT-001", "V-001"
    item_type varchar(32),                    -- "pressure_sensor", "gate_valve", ...
    io_type varchar(8),                       -- "digital" | "analog"
    channel varchar(128),                     -- PLC address: "plc.valves.V001.state"
    config jsonb,                             -- range, units, alarms, presets
    zone_id FK nullable,
    display_order int
)

-- P&ID diagrams
diagrams (
    id, bench_version_id FK,
    name, canvas_data jsonb,                  -- positions, pipes, labels, layers
    created_by FK, updated_at
)

-- Sequences (versioned)
sequences (id, bench_id FK, name, description, routine_type varchar(32) nullable)
sequence_versions (
    id, sequence_id FK, version int,
    definition jsonb,                         -- steps, timings, valve states, setpoints
    redline_config jsonb,                     -- associated redlines for this sequence
    uploaded_file_path varchar nullable,       -- if imported from file
    created_by FK, created_at,
    UNIQUE(sequence_id, version)
)

-- Runs
runs (
    id, bench_id FK, sequence_version_id FK,
    status varchar(16),                       -- "pending","armed","running","completed","aborted","faulted"
    started_at, ended_at,
    started_by FK,
    abort_reason text nullable,
    archive_path varchar nullable,            -- S3/MinIO key
    metadata jsonb                            -- summary stats, final outcomes
)

-- Redline definitions
redlines (
    id, sequence_version_id FK nullable,      -- null = global / always-active
    bench_item_id FK,
    level varchar(10),                        -- "redline" | "yellowline"
    evaluation_mode varchar(16),              -- "go_nogo" | "continuous" | "always_active"
    condition jsonb,                          -- { operator: ">", threshold: 150, unit: "bar" }
    activation_delay_ms int DEFAULT 0,
    action varchar(16),                       -- "warn" | "safe_abort" | "block_start"
    is_enabled boolean DEFAULT true,
    created_by FK, updated_at
)

-- Commands (persistent log)
commands (
    id uuid, run_id FK nullable,
    bench_item_id FK,
    command_type varchar(16),                 -- "set_digital", "set_analog", "set_setpoint"
    requested_value jsonb,
    status varchar(16),                       -- "pending","sent","ack_plc","applied","timeout","rejected"
    requested_by FK,
    requested_at timestamptz,
    sent_at timestamptz nullable,
    ack_at timestamptz nullable,
    applied_at timestamptz nullable,
    plc_readback jsonb nullable,
    error_message text nullable
)

-- Journal / audit log (append-only)
journal_events (
    id bigserial,
    bench_id FK,
    run_id FK nullable,
    timestamp timestamptz DEFAULT now(),
    level varchar(8),                         -- DEBUG, INFO, WARN, ERROR
    category varchar(32),                     -- "command", "sequence", "redline", "config", "auth", "system"
    source varchar(64),                       -- service name
    message text,
    details jsonb,                            -- structured payload
    user_id FK nullable
)
CREATE INDEX idx_journal_bench_ts ON journal_events (bench_id, timestamp DESC);
CREATE INDEX idx_journal_run ON journal_events (run_id) WHERE run_id IS NOT NULL;

-- Zones
zones (id, bench_version_id FK, name, description, is_active boolean DEFAULT true)
```

### 3.2 TimescaleDB (time-series, same Postgres instance or dedicated)

```sql
CREATE TABLE sensor_data (
    time        TIMESTAMPTZ NOT NULL,
    bench_id    UUID,
    channel     VARCHAR(128),
    value       DOUBLE PRECISION,
    quality     VARCHAR(12) DEFAULT 'good'
);
SELECT create_hypertable('sensor_data', 'time');

-- Continuous aggregate for dashboard overview (1-second rollup)
CREATE MATERIALIZED VIEW sensor_data_1s
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 second', time) AS bucket,
    bench_id, channel,
    avg(value) AS avg_val,
    min(value) AS min_val,
    max(value) AS max_val,
    last(value, time) AS last_val
FROM sensor_data
GROUP BY bucket, bench_id, channel;

-- Retention policy: raw data 90 days, 1s rollup 1 year
SELECT add_retention_policy('sensor_data', INTERVAL '90 days');
SELECT add_retention_policy('sensor_data_1s', INTERVAL '1 year');
```

**Ingestion rate estimate:** 50 channels × 10 Hz = 500 rows/sec — well within single-node TimescaleDB capacity. For burst recording at 100 Hz across 50 channels (5,000 rows/sec), we batch-insert via the `COPY` protocol through the ingestor.

### 3.3 MinIO (S3-compatible object storage)

```
minio/
├── run-archives/
│   └── {bench_id}/{run_id}/
│       ├── run_manifest.json        # metadata, sequence version, redline outcomes
│       ├── sensor_data.parquet      # all channels, full resolution
│       ├── commands.json            # all commands issued
│       ├── journal.json             # filtered journal for this run
│       ├── redline_events.json      # violations and evaluations
│       └── config_snapshot.json     # bench + sequence config at run time
├── config-snapshots/
│   └── {bench_id}/{version}/bench_config.json
└── sequence-uploads/
    └── {sequence_id}/{filename}
```

### 3.4 Run packaging format

Each completed or aborted run produces an archive bundle:

```json
// run_manifest.json
{
  "run_id": "uuid",
  "bench_id": "uuid",
  "sequence": { "id": "uuid", "version": 3, "name": "Chilldown A" },
  "status": "completed",                    // or "aborted"
  "started_at": "2026-02-24T14:00:00Z",
  "ended_at": "2026-02-24T14:23:17Z",
  "started_by": "operator-01",
  "abort_reason": null,
  "channels_recorded": ["PT-001", "TT-001", "V-001", ...],
  "acquisition_hz": 10,
  "total_samples": 138_000,
  "redline_summary": {
    "violations": 0,
    "yellowline_warnings": 2
  },
  "archive_format_version": "1.0"
}
```

Sensor data is stored as **Parquet** (columnar, compressible, readable by pandas/polars/Spark). For teams that need HDF5, a conversion utility is provided.

---

## 4. API Design

### 4.1 REST API (FastAPI)

All endpoints prefixed `/api/v1/`. JWT Bearer auth on every request.

**Auth**
```
POST   /auth/login                    → { access_token, refresh_token }
POST   /auth/refresh                  → { access_token }
GET    /auth/me                       → User + roles
```

**Bench Configuration**
```
GET    /benches                       → [Bench]
GET    /benches/{id}                  → Bench + active version
POST   /benches                       → Bench  (admin)
GET    /benches/{id}/versions         → [BenchVersion]
POST   /benches/{id}/versions         → BenchVersion  (engineer+)
PUT    /benches/{id}/versions/{v}/activate → 200  (engineer+)
GET    /benches/{id}/items            → [BenchItem]  (from active version)
```

**Diagrams**
```
GET    /benches/{id}/diagrams         → [Diagram]
GET    /diagrams/{id}                 → Diagram + canvas_data
PUT    /diagrams/{id}                 → Diagram  (engineer+)
POST   /benches/{id}/diagrams         → Diagram  (engineer+)
```

**Sequences**
```
GET    /benches/{id}/sequences        → [Sequence]
POST   /benches/{id}/sequences        → Sequence
GET    /sequences/{id}/versions       → [SequenceVersion]
POST   /sequences/{id}/versions       → SequenceVersion  (file upload or JSON)
GET    /sequence-versions/{id}        → SequenceVersion + definition + redlines
```

**Runs**
```
POST   /benches/{id}/runs             → Run { status: "pending" }  (operator+)
POST   /runs/{id}/arm                 → Run { status: "armed" }    (operator+, validates redlines)
POST   /runs/{id}/start               → Run { status: "running" }  (operator+, requires armed)
POST   /runs/{id}/abort               → Run { status: "aborted" }  (operator+, always allowed)
GET    /runs/{id}                     → Run + metadata
GET    /runs/{id}/progress            → { current_step, elapsed, remaining_estimate }
GET    /benches/{id}/runs             → [Run]  (filterable by status, date)
GET    /runs/{id}/archive-url         → { presigned_url, expires_at }
```

**Commands**
```
POST   /commands                      → Command { status: "pending" }
GET    /commands/{id}                 → Command (with full lifecycle timestamps)
GET    /runs/{id}/commands            → [Command]
```

**Redlines**
```
GET    /benches/{id}/redlines         → [Redline]  (global + sequence-specific)
POST   /redlines                      → Redline  (engineer+)
PUT    /redlines/{id}                 → Redline  (engineer+, logged to journal)
DELETE /redlines/{id}                 → 204      (engineer+, logged to journal)
```

**Journal**
```
GET    /benches/{id}/journal          → [JournalEvent]  (paginated, filterable by level/category/timerange)
GET    /runs/{id}/journal             → [JournalEvent]  (scoped to run)
```

**Recording**
```
POST   /benches/{id}/recording/start  → { session_id }  (operator+)
POST   /benches/{id}/recording/stop   → { session_id, duration }
PUT    /benches/{id}/recording/frequency → { hz: 100 }  (logged)
GET    /benches/{id}/recording/status → { active, session_id, hz, started_at }
```

**Zones**
```
GET    /benches/{id}/zones            → [Zone]
PUT    /zones/{id}                    → Zone  (activate/deactivate, operator+)
```

### 4.2 WebSocket (Command Channel)

Single WS connection per browser session on `wss://.../ws/commands?token=<jwt>`.

```
Client → Server:
{
  "type": "command",
  "request_id": "uuid-client-generated",
  "bench_item_tag": "V-001",
  "action": "set_digital",
  "value": true
}

Server → Client (ACK lifecycle):
{ "type": "command_status", "request_id": "...", "status": "pending",  "ts": "..." }
{ "type": "command_status", "request_id": "...", "status": "sent",     "ts": "..." }
{ "type": "command_status", "request_id": "...", "status": "applied",  "ts": "...", "readback": true }
-- or --
{ "type": "command_status", "request_id": "...", "status": "timeout",  "ts": "...", "error": "No PLC ACK within 3s" }

Server → Client (ARMED state changes):
{ "type": "armed_state", "bench_id": "...", "armed": true, "run_id": "..." }

Server → Client (redline/yellowline violations):
{ "type": "violation", "level": "redline", "bench_item_tag": "PT-001", "message": "Pressure > 150 bar", "ts": "..." }
```

### 4.3 SSE (Live Data)

Endpoint: `GET /api/v1/stream/bench/{bench_id}?channels=PT-001,TT-001,V-001&hz=10`

```
event: data
data: {"ts":"2026-02-24T14:00:00.100Z","values":{"PT-001":127.3,"TT-001":22.1,"V-001":true}}

event: bench_state
data: {"armed":true,"run_id":"...","run_status":"running","step":3,"step_name":"Pressurize"}

event: alarm
data: {"tag":"PT-001","level":"yellowline","message":"Pressure approaching limit","value":145.2}

event: zone_update
data: {"zone_id":"...","name":"Zone A","is_active":false}

event: heartbeat
data: {"ts":"..."}
```

The `hz` parameter lets the client request a lower update rate (the server throttles per-subscription). Default 10 Hz for dashboard, 1 Hz for overview panels. The browser can open multiple SSE connections to different channel sets if needed (e.g., one for the P&ID view, one for a trend chart).

### 4.4 gRPC Contracts (Industrial SE provides)

```protobuf
syntax = "proto3";
package plc_gateway;

service PlcGateway {
  // Single reads
  rpc ReadTag(ReadTagRequest) returns (TagValue);
  rpc ReadTags(ReadTagsRequest) returns (TagValues);
  
  // Writes with acknowledgement
  rpc WriteTag(WriteTagRequest) returns (WriteTagResponse);
  
  // Streaming subscription (server-streaming)
  rpc Subscribe(SubscribeRequest) returns (stream TagUpdate);
  
  // Bench-level operations
  rpc GetBenchState(BenchStateRequest) returns (BenchState);
  rpc SetBenchArmed(SetArmedRequest) returns (SetArmedResponse);
  
  // Health / watchdog
  rpc Ping(PingRequest) returns (PingResponse);
}

message TagValue {
  string channel = 1;
  oneof value {
    bool bool_value = 2;
    double double_value = 3;
    int64 int_value = 4;
  }
  int64 timestamp_ns = 5;
  Quality quality = 6;
}

enum Quality {
  GOOD = 0;
  BAD = 1;
  UNCERTAIN = 2;
}

message WriteTagRequest {
  string channel = 1;
  oneof value {
    bool bool_value = 2;
    double double_value = 3;
  }
  string request_id = 4;     // for correlation
  string requested_by = 5;   // operator ID
}

message WriteTagResponse {
  string request_id = 1;
  bool accepted = 2;         // PLC accepted the write
  string error = 3;          // if not accepted
  int64 timestamp_ns = 4;
}

message SubscribeRequest {
  repeated string channels = 1;
  uint32 deadband_ms = 2;    // minimum interval between updates per channel
}

message TagUpdate {
  string channel = 1;
  oneof value {
    bool bool_value = 2;
    double double_value = 3;
  }
  int64 timestamp_ns = 4;
  Quality quality = 5;
}

message BenchState {
  bool armed = 1;
  bool emergency_stop = 2;
  repeated ZoneState zones = 3;
}

message ZoneState {
  string zone_id = 1;
  bool active = 2;
}

message SetArmedRequest {
  bool armed = 1;
  string run_id = 2;
  string operator_id = 3;
}

message SetArmedResponse {
  bool success = 1;
  string error = 2;
}
```

**Key contract points for Industrial SE:**
- `Subscribe` must support deadband filtering to avoid flooding (we request 100ms deadband = 10 Hz max per channel).
- `WriteTag` must return a definitive `accepted` within 500ms or the backend treats it as timeout.
- `BenchState` is polled every 1s as a safety cross-check against our application state.
- The PLC must implement its own hardware watchdog independent of our software — if our gRPC connection drops for >5s, the PLC should initiate its own safe state.

---

## 5. Key Domain Models

```
Bench ──1:N──► BenchVersion ──1:N──► BenchItem
   │                │                    │
   │                ├──1:N──► Diagram    ├── channel (PLC address)
   │                │                    ├── io_type (digital|analog)
   │                └──1:N──► Zone       └── config (range, alarms, ...)
   │
   ├──1:N──► Sequence ──1:N──► SequenceVersion
   │                                │
   │                                ├── definition (steps, timings)
   │                                └── redline_config
   │
   ├──1:N──► Run
   │          │
   │          ├── sequence_version_id
   │          ├── status (pending → armed → running → completed|aborted|faulted)
   │          ├── archive_path (MinIO)
   │          └──1:N──► Command
   │                      │
   │                      ├── bench_item_id
   │                      ├── status lifecycle (pending → sent → ack → applied | timeout)
   │                      └── readback value
   │
   ├──1:N──► Redline
   │          ├── bench_item_id
   │          ├── level (redline | yellowline)
   │          ├── evaluation_mode (go_nogo | continuous | always_active)
   │          └── action (warn | safe_abort | block_start)
   │
   └──1:N──► JournalEvent
              ├── level (DEBUG|INFO|WARN|ERROR)
              ├── category (command|sequence|redline|config|auth|system)
              └── details (jsonb)
```

### Sequence Definition Schema (jsonb)

```json
{
  "format_version": "1.0",
  "name": "Chilldown Sequence A",
  "total_duration_estimate_s": 1200,
  "steps": [
    {
      "id": 1,
      "name": "Pre-check",
      "type": "gate",
      "description": "Verify all valves closed, tank empty",
      "conditions": [
        { "tag": "V-001", "operator": "==", "value": false },
        { "tag": "T-001.level", "operator": "<", "value": 5 }
      ],
      "timeout_s": 30,
      "on_timeout": "abort"
    },
    {
      "id": 2,
      "name": "Open inlet valve",
      "type": "command",
      "commands": [
        { "tag": "V-001", "action": "set_digital", "value": true }
      ],
      "confirm_readback": true,
      "readback_timeout_s": 5
    },
    {
      "id": 3,
      "name": "Fill to 80%",
      "type": "wait_condition",
      "condition": { "tag": "T-001.level", "operator": ">=", "value": 80 },
      "timeout_s": 600,
      "on_timeout": "abort",
      "redlines_active": ["RL-001", "RL-002"]
    },
    {
      "id": 4,
      "name": "Hold pressure",
      "type": "timed_hold",
      "duration_s": 120,
      "redlines_active": ["RL-001", "RL-002", "RL-003"]
    },
    {
      "id": 5,
      "name": "Close inlet",
      "type": "command",
      "commands": [
        { "tag": "V-001", "action": "set_digital", "value": false }
      ]
    }
  ],
  "abort_sequence": [
    { "tag": "V-001", "action": "set_digital", "value": false },
    { "tag": "P-001", "action": "set_digital", "value": false },
    { "tag": "V-002", "action": "set_digital", "value": true, "comment": "open drain" }
  ]
}
```

---

## 6. Safety & Correctness

### 6.1 ARMED Mode Rules

ARMED is a system-wide gate that prevents unintended actions during a test run.

| Rule | Enforcement |
|---|---|
| Only one run can be ARMED per bench at a time | Postgres advisory lock + `runs` table constraint |
| ARMED requires: all redlines configured, bench version locked, operator role | Validated in `POST /runs/{id}/arm` |
| While ARMED: no bench config changes, no sequence edits, no redline modifications | RBAC middleware checks `bench.armed_run_id`; rejects with 409 |
| While ARMED: manual commands are **blocked** unless explicitly whitelisted in sequence config | Command gateway checks armed state; only sequence engine commands pass |
| Abort is **always** allowed regardless of ARMED state | Hardcoded bypass in command gateway |
| ARMED flag is set on **both** the application (Postgres) and the PLC (via `SetBenchArmed` gRPC) | Sequence engine sets both; if PLC set fails, arm is rolled back |
| If PLC connection drops while ARMED, the PLC enters its own safe state; the app logs ERROR and enters "degraded ARMED" | PLC ingestor monitors connection; emits event on disconnect |

### 6.2 Command Confirmation Strategy

Every command follows a 4-phase lifecycle:

```
1. PENDING    – validated, enqueued in Redis Stream
2. SENT       – gRPC WriteTag called on PLC gateway
3. ACK_PLC    – PLC gateway returned accepted=true (PLC received the write)
4. APPLIED    – read-back confirms the actual value matches the requested value
```

**Read-back verification:** After receiving `ACK_PLC`, the command gateway subscribes to the channel's next update from the PLC ingestor. If the read-back value matches within a tolerance (exact for digital, ±deadband for analog), status transitions to `APPLIED`. If no matching read-back arrives within `readback_timeout` (default 3s), status transitions to `TIMEOUT` and a WARN journal event is emitted.

**Idempotency:** Every command carries a client-generated `request_id` (UUID). The command gateway deduplicates on `request_id` with a 60s TTL in Redis. If the same `request_id` arrives twice, the second is returned with the current status of the first.

**Two-step procedure (set setpoint then send):** For analog commands (e.g., control valve position), the UI sends a `set_setpoint` command first (which stages the value locally and shows it in the UI without transmitting to PLC), followed by a `confirm_setpoint` command that actually dispatches to the PLC. This prevents accidental value changes.

### 6.3 Safe Abort / Soft Safe Pattern

```
TRIGGER (any of):
  - Operator presses ABORT button (always available)
  - Redline violation with action="safe_abort"
  - Sequence step timeout with on_timeout="abort"
  - PLC connection loss > 5 seconds
  - Sequence engine crash (detected by watchdog)

SAFE-ABORT PROCEDURE:
  1. Sequence engine stops advancing steps immediately
  2. Execute abort_sequence commands (defined in sequence definition) IN ORDER
     - These are typically: close all valves, stop all pumps, open drain
  3. Each abort command follows the same ACK lifecycle (but with a shorter timeout of 2s)
  4. If any abort command fails:
     a. Retry once
     b. If still failed: log ERROR, continue with remaining abort commands
     c. Emit "PARTIAL_ABORT" event — operator must verify manually
  5. After all abort commands complete:
     a. Disarm bench (SetBenchArmed=false)
     b. Set run status = "aborted"
     c. Stop recording, package archive
     d. Emit journal event with full abort context

SOFT SAFE (yellowline response):
  - Do NOT abort
  - Emit warning to UI (violation event on WS + SSE)
  - Log WARN to journal
  - If yellowline persists > configured duration: escalate to operator attention (flashing UI)
```

### 6.4 Timeout Configuration

| Operation | Default Timeout | Configurable? |
|---|---|---|
| gRPC WriteTag response | 500ms | Per bench config |
| Command read-back | 3s | Per bench item |
| Sequence step timeout | Per step definition | Yes (sequence config) |
| PLC connection watchdog | 5s | Per bench config |
| Abort command timeout | 2s | Fixed |
| SSE heartbeat interval | 15s | Fixed |
| WS ping/pong | 30s | Fixed |
| JWT access token expiry | 15min | Env var |

---

## 7. Non-Functional Requirements

### 7.1 Performance — High-Frequency Signal Handling

**Ingestion path (PLC → TimescaleDB):**
- PLC ingestor receives gRPC stream updates, applies change-detection deadband (configurable per channel, default 0.1% of range).
- Writes are batched: accumulate 100ms of samples, then `COPY` to TimescaleDB in a single batch. This keeps write latency off the hot path.
- During burst recording (100 Hz), the ingestor switches to a dedicated write buffer with larger batches.

**Fan-out path (Redis → Browser):**
- The live-data service subscribes to Redis Pub/Sub channels.
- Per SSE connection, it maintains a throttle buffer: at the client's requested `hz`, it sends the latest sample (not every intermediate). This means 50 channels at 10 Hz = 500 values/sec per client — roughly 10 KB/s of SSE traffic.
- For trend/chart views that need historical data, the client fetches from REST (`/api/v1/data/{channel}?from=...&to=...&resolution=1s`), which reads from TimescaleDB continuous aggregates.

**Downsampling strategy:**
| Context | Resolution | Source |
|---|---|---|
| Live P&ID display | 10 Hz (latest value) | SSE |
| Dashboard cards | 1 Hz | SSE (throttled) |
| Trend chart (zoomed in, <5min) | Full resolution | TimescaleDB raw |
| Trend chart (1 hour) | 1s aggregates | TimescaleDB continuous aggregate |
| Trend chart (24 hours) | 10s aggregates | TimescaleDB continuous aggregate |
| Archived run playback | Full resolution | Parquet from MinIO |

### 7.2 Observability

**Structured logging (all services):**
- `structlog` with JSON output → shipped to centralized log aggregator (ELK or Loki).
- Every log line includes: `service`, `bench_id`, `run_id` (if applicable), `user_id`, `trace_id`.
- Log levels: DEBUG (development only), INFO (normal operations), WARN (degraded but functional), ERROR (requires attention).

**Metrics (Prometheus):**
- `command_latency_seconds` (histogram, labels: bench, status)
- `plc_connection_state` (gauge: 1=connected, 0=disconnected)
- `live_data_fanout_clients` (gauge)
- `sensor_ingest_rate_hz` (gauge per bench)
- `redline_violations_total` (counter, labels: bench, level)
- `sequence_step_duration_seconds` (histogram)
- `ws_connections_active` (gauge)
- `sse_connections_active` (gauge)

**Distributed tracing (OpenTelemetry):**
- Propagate `trace_id` from browser → REST/WS → command gateway → gRPC → PLC response → read-back.
- This lets you trace a single valve command from button click to PLC application.

**Alerting:**
- PLC connection lost > 10s → PagerDuty
- Command timeout rate > 5% in 1 min → PagerDuty
- Sequence engine process crash → PagerDuty (supervisor auto-restarts, but alert fires)
- TimescaleDB disk usage > 80% → Slack
- API error rate > 1% → Slack

### 7.3 Reliability — Disconnect Handling

| Failure | Detection | Response |
|---|---|---|
| Browser loses SSE | `EventSource` auto-reconnects (built-in) | Client shows "reconnecting" banner; on reconnect, fetches latest state via REST |
| Browser loses WS | `onclose` event, client retry with backoff | Pending commands show "uncertain" status; re-fetched on reconnect |
| Backend loses PLC gRPC | gRPC health check + subscribe stream error | Ingestor emits `plc_disconnected` event; all channels marked `quality=UNCERTAIN`; if ARMED, safe-abort countdown begins |
| Redis down | Connection pool retry | Services degrade: live data stops, commands queue locally with bounded buffer (100 items, 30s), then reject |
| Sequence engine crash | Systemd watchdog (or K8s liveness probe) | Auto-restart within 5s; on restart, check `runs` table for active run; if found with `status=running`, initiate safe-abort (cannot resume mid-sequence safely) |
| TimescaleDB down | Connection error on write | Ingestor buffers to local disk (WAL-style append file, bounded 1GB); retries every 5s; live data unaffected (Redis path is separate) |

### 7.4 Security

- **Transport:** TLS everywhere. No plaintext HTTP/WS in production.
- **Authentication:** JWT with short-lived access tokens (15 min) + refresh tokens (7 days, rotated on use). Tokens include `user_id`, `roles[]`, `bench_permissions[]`.
- **RBAC enforcement:** FastAPI dependency injection middleware. Every endpoint declares required role. Bench-scoped permissions (operator on bench A ≠ operator on bench B).
- **ARMED mode as security gate:** Even admin cannot modify config while a bench is armed. Only abort bypasses.
- **Audit trail:** Every state-changing API call logs to `journal_events` with `user_id`, `details`, and the previous value (for config changes). Journal table is append-only (no UPDATE/DELETE granted to application role).
- **PLC gateway security:** gRPC channel uses mTLS. The PLC gateway only accepts connections from known service certificates.
- **Input validation:** Pydantic models on every endpoint. Channel names validated against bench config (no arbitrary PLC writes). Analog values clamped to configured range before dispatch.
- **Rate limiting:** Commands rate-limited to 10/sec per user per bench (prevents accidental rapid toggling). Admin override available.

---

## 8. Repository Structure & CI/CD

### 8.1 Frontend (Vue 3)

```
frontend/
├── public/
├── src/
│   ├── api/                          # API client layer
│   │   ├── client.ts                 # Axios/fetch instance, JWT interceptor
│   │   ├── auth.ts
│   │   ├── bench.ts
│   │   ├── commands.ts
│   │   ├── sequences.ts
│   │   ├── runs.ts
│   │   ├── journal.ts
│   │   └── types/                    # Generated from OpenAPI schema
│   │
│   ├── realtime/                     # Live data layer
│   │   ├── sse-client.ts             # SSE connection manager
│   │   ├── ws-client.ts              # WebSocket command channel
│   │   ├── use-live-data.ts          # composable: useLiveData(benchId, channels)
│   │   ├── use-command.ts            # composable: useCommand() → { send, status }
│   │   └── use-bench-state.ts        # composable: useBenchState(benchId)
│   │
│   ├── stores/                       # Pinia stores
│   │   ├── auth.store.ts
│   │   ├── bench.store.ts
│   │   ├── run.store.ts
│   │   └── notifications.store.ts
│   │
│   ├── views/                        # Route-level views
│   │   ├── LoginView.vue
│   │   ├── DashboardView.vue         # Bench list, status overview
│   │   ├── BenchView.vue             # Single bench — tabs below
│   │   ├── bench/
│   │   │   ├── PidDiagramTab.vue     # Canvas P&ID editor/viewer
│   │   │   ├── ControlPanelTab.vue   # Grid of bench items + commands
│   │   │   ├── SequenceTab.vue       # Sequence list, editor, graphical view
│   │   │   ├── RunTab.vue            # Active run monitor, progress, abort
│   │   │   ├── TrendsTab.vue         # On-demand time-series charts
│   │   │   ├── JournalTab.vue        # Filterable event log
│   │   │   ├── ConfigTab.vue         # Bench config, redlines, zones (engineer+)
│   │   │   └── ArchiveTab.vue        # Past runs, download archives
│   │   └── AdminView.vue             # User management (admin)
│   │
│   ├── components/
│   │   ├── pid/                      # P&ID rendering components
│   │   │   ├── PidCanvas.vue         # Canvas wrapper (Konva.js or Fabric.js)
│   │   │   ├── PidGateValve.vue      # From existing library
│   │   │   ├── PidControlValve.vue
│   │   │   ├── PidPump.vue
│   │   │   ├── PidTank.vue
│   │   │   ├── PidSensor.vue
│   │   │   ├── PidPipe.vue
│   │   │   └── PidRenderer.ts        # Reads diagram data → renders on canvas
│   │   │
│   │   ├── bench/                    # Bench item cards, status
│   │   │   ├── BenchItemCard.vue
│   │   │   ├── CommandButton.vue     # Two-step command (set + confirm)
│   │   │   ├── SetpointInput.vue     # Analog setpoint with confirm
│   │   │   ├── StatusBadge.vue
│   │   │   └── ArmButton.vue
│   │   │
│   │   ├── sequence/
│   │   │   ├── SequenceEditor.vue    # Step editor UI
│   │   │   ├── SequenceTimeline.vue  # Graphical time plot (valve states over time)
│   │   │   ├── SequenceProgress.vue  # Live progress during run
│   │   │   └── RedlineConfigPanel.vue
│   │   │
│   │   ├── charts/
│   │   │   ├── TrendChart.vue        # uPlot or Chart.js wrapper
│   │   │   └── LiveGauge.vue
│   │   │
│   │   ├── journal/
│   │   │   └── JournalTable.vue
│   │   │
│   │   └── common/
│   │       ├── ConnectionIndicator.vue
│   │       ├── ArmedBanner.vue
│   │       ├── WarningOverlay.vue    # Redline/yellowline popups
│   │       └── ConfirmDialog.vue
│   │
│   ├── composables/                  # Shared logic
│   │   ├── use-auth.ts
│   │   ├── use-rbac.ts               # hasRole(), canControl()
│   │   ├── use-polling.ts
│   │   └── use-confirmation.ts       # Two-step action pattern
│   │
│   ├── router/
│   │   └── index.ts                  # Route guards for auth + RBAC
│   │
│   ├── lib/                          # Existing P&ID component library
│   │   ├── io/
│   │   ├── visuals/
│   │   ├── animations/
│   │   ├── components/
│   │   └── index.ts
│   │
│   ├── App.vue
│   └── main.ts
│
├── tests/
│   ├── unit/
│   ├── component/                    # Vue Test Utils
│   └── e2e/                          # Playwright
│
├── package.json
├── vite.config.ts
├── tsconfig.json
└── .env.example
```

**Key library choices:**
- **Canvas P&ID:** Konva.js (via `vue-konva`) — performant canvas rendering for draggable/zoomable diagrams with many elements. SVG components from the existing library can be rendered as Konva `Image` nodes or ported to Konva shapes.
- **Charts:** uPlot (60KB, handles 100K+ points at 60fps) for time-series trends. Lightweight alternative: Chart.js.
- **State:** Pinia (official Vue state management).
- **API client:** `ky` or plain `fetch` with typed wrappers generated from OpenAPI.
- **Forms:** VeeValidate + Zod schemas.

### 8.2 Backend (Python)

```
backend/
├── src/
│   ├── app/
│   │   ├── main.py                   # FastAPI app factory
│   │   ├── config.py                 # Pydantic Settings (env-based)
│   │   ├── dependencies.py           # DI: get_db, get_redis, get_current_user
│   │   │
│   │   ├── auth/
│   │   │   ├── router.py             # /auth/* endpoints
│   │   │   ├── service.py            # JWT creation, validation, RBAC checks
│   │   │   ├── models.py             # SQLAlchemy: User, Role, UserRole
│   │   │   ├── schemas.py            # Pydantic: LoginRequest, TokenResponse
│   │   │   └── rbac.py               # require_role() dependency
│   │   │
│   │   ├── bench/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   ├── models.py             # Bench, BenchVersion, BenchItem, Zone
│   │   │   └── schemas.py
│   │   │
│   │   ├── diagram/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   ├── models.py             # Diagram
│   │   │   └── schemas.py
│   │   │
│   │   ├── sequence/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   ├── models.py             # Sequence, SequenceVersion
│   │   │   ├── schemas.py
│   │   │   └── validators.py         # Validate sequence definition schema
│   │   │
│   │   ├── run/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   ├── models.py             # Run
│   │   │   └── schemas.py
│   │   │
│   │   ├── command/
│   │   │   ├── router.py
│   │   │   ├── ws_handler.py         # WebSocket endpoint
│   │   │   ├── service.py            # Dispatch, ACK tracking
│   │   │   ├── gateway.py            # Redis Stream → gRPC (command gateway worker)
│   │   │   ├── models.py             # Command
│   │   │   └── schemas.py
│   │   │
│   │   ├── redline/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   ├── engine.py             # Redline evaluation logic
│   │   │   ├── models.py             # Redline
│   │   │   └── schemas.py
│   │   │
│   │   ├── journal/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   ├── models.py             # JournalEvent
│   │   │   └── schemas.py
│   │   │
│   │   ├── recording/
│   │   │   ├── router.py
│   │   │   ├── service.py
│   │   │   └── schemas.py
│   │   │
│   │   ├── streaming/
│   │   │   ├── sse_handler.py        # SSE fan-out endpoint
│   │   │   ├── throttle.py           # Per-subscription rate limiter
│   │   │   └── router.py
│   │   │
│   │   └── shared/
│   │       ├── database.py           # SQLAlchemy async engine + session
│   │       ├── redis.py              # Redis connection pool
│   │       ├── models_base.py        # DeclarativeBase, mixins
│   │       └── exceptions.py         # Domain exceptions → HTTP mapping
│   │
│   ├── workers/                      # Separate processes (not part of FastAPI)
│   │   ├── plc_ingestor.py           # gRPC subscribe → Redis + TimescaleDB
│   │   ├── sequence_engine.py        # Step executor + redline evaluator
│   │   ├── command_gateway.py        # Redis Stream consumer → gRPC writes
│   │   ├── recorder.py              # Recording session manager
│   │   └── archiver.py              # Package run → Parquet → MinIO
│   │
│   └── protos/                       # gRPC proto files + generated stubs
│       ├── plc_gateway.proto
│       └── plc_gateway_pb2.py        # generated
│       └── plc_gateway_pb2_grpc.py   # generated
│
├── tests/
│   ├── conftest.py                   # Fixtures: test DB, Redis mock, gRPC mock
│   ├── unit/
│   │   ├── test_redline_engine.py
│   │   ├── test_command_service.py
│   │   ├── test_sequence_validator.py
│   │   └── test_rbac.py
│   ├── integration/
│   │   ├── test_run_lifecycle.py
│   │   ├── test_command_ack.py
│   │   └── test_armed_mode.py
│   └── e2e/
│       └── test_full_sequence_run.py
│
├── alembic/                          # DB migrations
│   ├── env.py
│   └── versions/
│
├── pyproject.toml                    # uv / poetry
├── Dockerfile
├── docker-compose.yml                # Postgres, TimescaleDB, Redis, MinIO, app
└── .env.example
```

**Key library choices:**
- **Web framework:** FastAPI (async, OpenAPI auto-generation, Pydantic validation)
- **ORM:** SQLAlchemy 2.0 (async) with Alembic migrations
- **Redis:** `redis-py` with async support
- **gRPC:** `grpcio` + `grpcio-tools` for stub generation
- **Time-series writes:** `asyncpg` with `COPY` for batch inserts to TimescaleDB
- **Parquet:** `pyarrow` for archive packaging
- **MinIO:** `boto3` (S3-compatible)
- **Task scheduling:** No Celery — workers are standalone `asyncio` processes managed by systemd or K8s. Simpler, fewer moving parts.
- **Testing:** `pytest` + `pytest-asyncio` + `httpx` (async test client for FastAPI)

### 8.3 CI/CD Outline

```yaml
# .github/workflows/ci.yml (simplified)

name: CI
on: [push, pull_request]

jobs:
  backend-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff mypy
      - run: ruff check backend/
      - run: mypy backend/src/ --strict

  backend-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:latest-pg16
        env: { POSTGRES_PASSWORD: test }
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e "backend/[test]"
      - run: pytest backend/tests/ --cov=backend/src --cov-report=xml
      - uses: codecov/codecov-action@v4

  frontend-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd frontend && npm ci
      - run: cd frontend && npm run lint
      - run: cd frontend && npx vue-tsc --noEmit

  frontend-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd frontend && npm ci
      - run: cd frontend && npm test -- --coverage
      - run: cd frontend && npx playwright install && npm run test:e2e

  build:
    needs: [backend-test, frontend-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t testbench-api backend/
      - run: cd frontend && npm ci && npm run build
      - run: docker build -t testbench-frontend frontend/
      # Push to registry on main branch
```

**Deployment:** Docker Compose for development and single-machine deployment. Kubernetes (Helm chart) for multi-bench production deployments. The sequence engine and PLC ingestor run as `Deployment` with `replicas: 1` and a PodDisruptionBudget (singleton per bench).

---

## Summary of Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Live data transport | SSE (not WebSocket) | Unidirectional, auto-reconnect, proxy-friendly, simpler |
| Command transport | WebSocket | Bidirectional ACK needed |
| Internal bus | Redis Streams + Pub/Sub | Decouples services, at-least-once delivery, lightweight |
| Time-series DB | TimescaleDB | SQL-compatible, continuous aggregates, same Postgres ecosystem |
| Archive format | Parquet | Columnar, compressible, pandas/polars/Spark compatible |
| Sequence engine | Dedicated process (not task queue) | Must survive API restarts, holds ARMED lock, real-time constraints |
| P&ID canvas | Konva.js | Performance at scale, zoom/pan, drag, layer support |
| Auth | JWT (short-lived) + refresh | Stateless API auth, bench-scoped RBAC |
| Config versioning | Immutable version rows | Full audit trail, snapshot at run time, rollback support |
| Safe-abort | Application-level + PLC watchdog | Defense in depth: software handles graceful, PLC handles catastrophic |
