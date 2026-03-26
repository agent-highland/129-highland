# Node-RED — Logging Framework

## Concept

Logging answers: *"How important is this for troubleshooting/audit?"*

Logging is separate from notifications. They intersect (a CRITICAL log may auto-generate a notification), but serve different purposes. A CRITICAL log fires a notification; an INFO log is written to disk only.

---

## Log Storage

**Format:** JSONL (JSON Lines) — one JSON object per line

**Location:** `/var/log/highland/` on the Workflow host (volume-mounted into Node-RED container)

**Rotation:** Daily files, 30-day retention

```
/var/log/highland/
├── highland-2026-03-24.jsonl
├── highland-2026-03-25.jsonl
└── highland-2026-03-26.jsonl  (current)
```

---

## Unified Log

A single daily log file contains entries from ALL systems — Node-RED, Z2M, Z-Wave JS, HA, watchdog, etc. This provides a unified view similar to Windows Event Viewer.

**Log entry structure:**

| Field | Purpose | Examples |
|-------|---------|----------|
| `timestamp` | When it happened | `2026-03-26T10:00:00Z` |
| `system` | Which system generated the log | `node_red`, `ha`, `z2m`, `zwave_js`, `watchdog` |
| `source` | Component within that system | `garage`, `scheduler`, `coordinator` |
| `level` | Severity | `VERBOSE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | Human-readable description | `Failed to turn on carriage lights` |
| `context` | Structured additional data | `{"device": "...", "error": "..."}` |

**Example entries:**
```json
{"timestamp":"2026-03-26T10:00:00Z","system":"node_red","source":"garage","level":"ERROR","message":"Failed to turn on carriage lights","context":{"device":"light.garage_carriage","error":"MQTT timeout"}}
{"timestamp":"2026-03-26T10:00:05Z","system":"z2m","source":"coordinator","level":"WARN","message":"Device interview failed","context":{"device":"garage_motion_sensor"}}
```

---

## Log Levels

| Level | Value | When to Use |
|-------|-------|-------------|
| `VERBOSE` | 0 | Granular trace; active debugging only |
| `DEBUG` | 1 | Detailed info useful for troubleshooting |
| `INFO` | 2 | Normal operational events worth recording |
| `WARN` | 3 | Something unexpected but not broken |
| `ERROR` | 4 | Something failed but flow continues |
| `CRITICAL` | 5 | Catastrophic failure; intervention needed |

---

## Per-Flow Log Level Threshold

Each flow has a configured minimum log level stored in flow context:

```javascript
flow.set('logLevel', 'WARN');  // This flow only emits WARN and above
```

When a flow emits a log message:
- If message level ≥ flow threshold → emit to logging utility
- If message level < flow threshold → suppress

**Use case:** Set a flow to `DEBUG` while developing, `WARN` in steady state.

---

## MQTT Log Topic

**Single topic — all systems publish here:**

```
highland/event/log
```

QoS 2 — guaranteed delivery. See `standards/MQTT_TOPICS.md` for full payload schema.

---

## How Systems Log

| System | Mechanism |
|--------|-----------|
| **Node-RED** | Flows publish to `highland/event/log`; Logging utility writes to file |
| **Z2M / Z-Wave JS** | Publish to `highland/event/log` (if configurable), or sidecar script |
| **Home Assistant** | Publish to `highland/event/log` via automation, or sidecar script |

Node-RED's Logging utility flow subscribes to `highland/event/log` and writes ALL entries to the unified JSONL file, regardless of `system`.

---

## Utility: Logging Flow

**Critical note:** `Utility: Logging` does **not** use the Initializer Latch. It inlines all required helpers directly so it remains functional even if `Utility: Initializers` fails. If logging required the latch, a failure in Initializers would silence logging precisely when you need it most.

**Flow:**
```
highland/event/log ──► Logging Utility ──► Append to JSONL
                              │
                              │ (if CRITICAL)
                              ▼
                       highland/event/notify
```

---

## Auto-Notify Behavior

**Only CRITICAL logs auto-notify.** ERROR and below do not.

| Level | Auto-Notify | Rationale |
|-------|-------------|-----------|
| CRITICAL | Yes | System health, potential data loss, immediate intervention |
| ERROR | No | Something failed but system continues |
| WARN and below | No | Informational |

**CRITICAL examples:** Database size exceeded, disk usage critical, core service unresponsive.

**ERROR examples (no auto-notify):** API timeout (data stale but functional), device command failed (retry later).

### Escalation is Flow Responsibility

If a flow wants to notify after repeated ERRORs, *that flow* decides:

```
API call fails → log ERROR
      │
      ▼
Increment failure counter (flow context)
      │
      ▼
Counter > threshold? ──► YES ──► Publish to highland/event/notify
      │
      NO → continue, try again next cycle
```

The logging framework doesn't escalate. Flows own their escalation logic.

---

## Querying Logs

JSONL + `jq` provides powerful ad-hoc querying:

```bash
# All errors from any system
jq 'select(.level == "ERROR")' highland-2026-03-26.jsonl

# All Node-RED entries
jq 'select(.system == "node_red")' highland-2026-03-26.jsonl

# All entries from garage (regardless of system)
jq 'select(.source == "garage")' highland-2026-03-26.jsonl

# Z2M warnings and above
jq 'select(.system == "z2m" and (.level == "WARN" or .level == "ERROR" or .level == "CRITICAL"))' highland-2026-03-26.jsonl

# Last 10 entries
tail -10 highland-2026-03-26.jsonl | jq '.'
```

---

## Log Rotation

Logrotate configuration on the Workflow host (`/etc/logrotate.d/highland`):

```
/var/log/highland/*.jsonl {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 644 highland highland
}
```

---

## Future: Log Shipping (Deferred)

When NAS is available or if cloud aggregation is desired, JSONL is compatible with most aggregators (Loki, Elastic, Datadog). Details TBD when infrastructure supports it.

---

*Last Updated: 2026-03-26*
