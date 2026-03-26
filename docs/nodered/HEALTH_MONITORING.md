# Node-RED — Health Monitoring

## Philosophy

Treat this as a line-of-business application. Degradation detection is as important as outage detection.

---

## Single Point of Failure Problem

Node-RED alone as the health reporter creates ambiguity. If Node-RED goes down, all service checks stop pinging simultaneously — making it impossible to distinguish "Node-RED is down" from "everything is down."

**Solution: each service self-reports its own liveness independently of Node-RED.** Node-RED's edge checks prove the *connection* is healthy; each service's own ping proves the *service* is healthy.

**Healthchecks.io naming convention:**
- `{Service}` — the service's own self-report, independent of Node-RED
- `Node-RED / {Service} Edge` — Node-RED's check proving the connection to that service is healthy

**Failure signature matrix:**

| Failure | Service check | Edge check | Node-RED check |
|---------|---------------|------------|----------------|
| Node-RED down | ✅ pinging | ❌ silent | ❌ silent |
| Service down | ❌ silent | ❌ silent | ✅ pinging |
| Network path broken (both up) | ✅ pinging | ❌ silent | ✅ pinging |

Three distinct signatures — completely unambiguous diagnosis.

**Current implementation status:**
- `Node-RED` — Node-RED self-reports via direct HTTP ping ✅
- `Home Assistant` — native HA automation pinging Healthchecks.io directly ✅
- `Node-RED / Home Assistant Edge` — Node-RED's HTTP check to HA API ✅
- MQTT, Z2M, Z-Wave JS self-reports and edge checks — pending

---

## Architecture

```
Node-RED (Health Monitor Flow)
  Monitors: MQTT, Z2M, Z-Wave JS, HA, host resources
  Publishes: highland/status/{service}/health
  Notifies: highland/event/notify (on status change)
  Pings: Healthchecks.io for each monitored service (direct HTTP)

HA (automations/health_monitoring.yaml)
  Pings: Healthchecks.io for Home Assistant self-report
  Pings: Healthchecks.io for HA/Z2M and HA/Z-Wave edge checks
```

Node-RED pings Healthchecks.io **directly via outbound HTTP**, independent of MQTT state. This correctly separates Node-RED liveness from MQTT liveness — if MQTT goes down but Node-RED is up, Healthchecks.io still receives pings.

> **Watchdog script:** The original watchdog design (subscribing to a Node-RED MQTT heartbeat and pinging Healthchecks.io on receipt) is superseded by direct HTTP pinging. Whether a watchdog script has a remaining role for any specific service check is TBD per-service.

---

## Status Values

| Status | Meaning |
|--------|---------|
| `healthy` | Responding AND all metrics within acceptable ranges |
| `degraded` | Responding BUT one or more metrics in warning territory |
| `unhealthy` | Not responding OR critical threshold exceeded |

---

## Per-Service Checks

| Service | Responsiveness Check | Threshold Metrics |
|---------|---------------------|-------------------|
| **MQTT broker** | Publish/subscribe test | Host disk, CPU, memory |
| **Zigbee2MQTT** | HTTP API or MQTT bridge topic | Devices online/offline, host resources |
| **Z-Wave JS UI** | WebSocket or HTTP API | Nodes online/dead, host resources |
| **Home Assistant** | HTTP API (`/api/`) | DB size, host resources |
| **Node-RED** | HTTP admin API | Host resources |

---

## Threshold Definitions

| Metric | Warning | Critical | Notes |
|--------|---------|----------|-------|
| Disk usage | > 70% | > 90% | Applies to all hosts |
| CPU usage (sustained) | > 80% for 5 min | > 95% for 5 min | Transient spikes ignored |
| Memory usage | > 80% | > 95% | |
| Devices offline (Z2M) | Any (1+) | > 20% of total | Single device offline = degraded |
| Devices offline (Z-Wave) | Any (1+) | > 20% of total | |

---

## Check Frequency & Healthchecks.io Configuration

| Service | Check Frequency | Grace Period |
|---------|----------------|--------------|
| Node-RED | 30 sec | 3 min |
| Home Assistant | 30 sec | 3 min |
| MQTT | 1 min | 3 min |
| Z2M | 1 min | 3 min |
| Z-Wave JS | 1 min | 3 min |

**Grace period:** 3 minutes for all checks — whole-minute minimum (Healthchecks.io limitation), sufficient buffer for transient latency without being so long that alerts are meaningless.

**Fail fast:** One missed check = unhealthy. No transient failure tolerance initially. If this proves too noisy, tolerance can be added later.

---

## Health Payload

**`highland/status/{service}/health`** ← RETAINED

```json
{
  "timestamp": "2026-03-26T10:00:00Z",
  "service": "z2m",
  "status": "degraded",
  "checks": {
    "responsive": true,
    "thresholds": {
      "disk_percent": { "value": 45, "status": "ok" },
      "cpu_percent": { "value": 22, "status": "ok" },
      "memory_percent": { "value": 61, "status": "ok" },
      "devices_offline": { "value": 2, "status": "warning" }
    }
  },
  "summary": "2 devices offline"
}
```

**Alarm evaluation logic:**
```
For each threshold metric:
  if value > critical_threshold → metric status = "critical"
  else if value > warning_threshold → metric status = "warning"
  else → metric status = "ok"

Overall status:
  if not responsive → "unhealthy"
  else if any metric = "critical" → "unhealthy"
  else if any metric = "warning" → "degraded"
  else → "healthy"
```

---

## Notification Behavior

| Status Change | Action |
|---------------|--------|
| healthy → degraded | Notify (normal priority) |
| degraded → unhealthy | Notify (high priority) |
| healthy → unhealthy | Notify (high priority) |
| unhealthy → degraded | Notify recovery (normal priority) |
| unhealthy → healthy | Notify recovery (normal priority) |
| degraded → healthy | No notification (silent recovery) |

---

## Utility: Connections Flow

### Purpose

Tracks the live state of external service connections and exposes that state via global context for any flow that needs to make runtime decisions based on it. Distinct from the Health Monitor — Health Monitor *reports* infrastructure health outward (to Healthchecks.io, to logs); Connections *exposes* connection state inward to other flows.

### Detection Mechanism

Uses Node-RED's built-in `status` node scoped to a connection-bearing node. The `status` node fires immediately when the monitored node's connection state changes — no polling, no second connection, no additional palette dependencies.

**Signal mapping (via `msg.status.fill`):**
- `'red'` or `'yellow'` → connection is down → `connections.{key} = 'down'`
- anything else → connection is up → `connections.{key} = 'up'`

### Startup Settling Window

On restart, connections briefly drop before re-establishing. Without mitigation, every restart generates spurious "connection lost" log entries. The flow uses a **startup settling window:**

- `Establish Cadence` function sets `flow.timer_cadence` (regular flow context) and starts a `setTimeout` that sets `flow.settled = true` in the `volatile` store after the cadence elapses
- `Evaluate State` nodes read `flow.get('settled', 'volatile')` to determine whether they are in the startup window

**During startup window:** `'down'` transitions start a debounce timer — if still down when the timer fires, log it. `'up'` transitions cancel any pending timer silently.

**After the window:** All transitions logged immediately as real-time events.

**Timer handles** are stored in the `volatile` context store — they are non-serializable and must not be stored in the default (localfilesystem) store.

### MQTT Availability — The Catch-22

When MQTT goes down, the normal log path (`highland/event/log` via MQTT out) is unavailable. The State Change Logging group handles this explicitly:

```
Log Event link in → MQTT Available? switch (global.connections.mqtt == 'up')
    ↓ up                                   ↓ else
Format Log Message → MQTT out         Log to Console (node.error/warn)
```

### Global Context Keys

| Key | Store | Values | Set by |
|-----|-------|--------|--------|
| `connections.home_assistant` | default | `'up'` / `'down'` | HA Evaluate State |
| `connections.mqtt` | default | `'up'` / `'down'` | MQTT Evaluate State |

### Usage in Other Flows

```javascript
// Check HA availability before routing a notification
const haAvailable = global.get('connections.home_assistant') !== 'down';

// Check MQTT availability before publishing
const mqttAvailable = global.get('connections.mqtt') !== 'down';
```

The `!== 'down'` guard handles the startup case where the flag hasn't been set yet — `undefined !== 'down'` is `true`, defaulting to assuming the connection is available rather than silently dropping messages during the brief init window.

### Flow Context Keys

| Key | Store | Value |
|-----|-------|-------|
| `timer_cadence` | default | ms integer (e.g. 5000) — single cadence drives both settling window and debounce timers |
| `settled` | volatile | `true` / undefined |
| `home_assistant_timer` | volatile | timer handle or null |
| `mqtt_timer` | volatile | timer handle or null |

---

*Last Updated: 2026-03-26*
