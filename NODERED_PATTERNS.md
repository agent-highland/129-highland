# Node-RED Patterns & Conventions

## Overview

Design patterns and conventions for Node-RED flows in the Highland home automation system. These patterns prioritize readability, maintainability, and alignment with the event-driven architecture.

---

## Core Principles

1. **Visibility over abstraction** — Keep logic visible in flows; don't hide complexity in subflows unless truly reusable
2. **Horizontal scrolling is the enemy** — Use link nodes and groups to keep flows compact and readable
3. **Pub/sub for inter-flow communication** — Flows talk via MQTT events, not direct dependencies
4. **Centralized error handling** — Flow-wide catch with targeted overrides
5. **Configurable logging** — Per-flow log levels for flexible debugging

---

## Node-RED Environment Configuration

### Using Node.js Modules in Function Nodes

`require()` is not available directly in function nodes in Node-RED 3.x. Modules must be declared in the function node's **Setup tab** and are injected as named variables.

**To use `fs` and `path` (or any other module):**
1. Open the function node
2. Go to the **Setup** tab
3. Add entries under **Modules**:
   - Name: `fs` / Module: `fs`
   - Name: `path` / Module: `path`
4. In the function body, use `fs` and `path` directly — do **not** call `require()`

```javascript
// WRONG — will throw "require is not defined"
const fs = require('fs');

// CORRECT — declared in Setup tab, available as plain variable
const raw = fs.readFileSync(filepath, 'utf8');
```

This applies to any Node.js built-in or npm module used in function nodes.

### Context Storage (settings.js)

Node-RED context must be configured for disk persistence to survive restarts/deploys:

```javascript
// In settings.js
contextStorage: {
    default: {
        module: "localfilesystem"
    }
}
```

This enables:
- Flow context persistence (`flow.set()` / `flow.get()`)
- Global context persistence (`global.set()` / `global.get()`)
- Survives Node-RED restarts and deploys

### Home Assistant Integration

**Primary method:** `node-red-contrib-home-assistant-websocket`

Provides:
- HA entity state access
- Service calls (notifications, backups, etc.)
- Event subscription
- WebSocket connection to HA

**Configuration:**
- Base URL: `http://{ha_ip}:8123`
- Access Token: Long-lived access token from HA (stored in Node-RED credentials, not in config files)

**Use cases:**
| Action | Method |
|--------|--------|
| Trigger HA backup | Service call: `backup.create` |
| Send notification via Companion App | Service call: `notify.mobile_app_*` |
| Check HA entity state | HA API node or WebSocket |
| React to HA events | HA events node |

*Note: Most device control goes through MQTT directly (Z2M, Z-Wave JS). HA integration is primarily for HA-specific features (backups, notifications, entity state that only exists in HA).*

### Watchdog Script

External process that monitors Node-RED and pings Healthchecks.io. Runs on the Node-RED host.

**Location:** `/usr/local/bin/highland-watchdog.sh`

**Implementation:**
```bash
#!/bin/bash
# Highland Watchdog - Monitors Node-RED heartbeat via MQTT
# Pings Healthchecks.io on success, fails silent on miss

MQTT_HOST="hub.local"
MQTT_USER="highland"
MQTT_PASS="from-env-or-file"
HEARTBEAT_TOPIC="highland/status/node_red/heartbeat"
HC_PING_URL="https://hc-ping.com/your-uuid-here"
TIMEOUT=90  # seconds to wait for heartbeat

# Subscribe and wait for one message with timeout
mosquitto_sub -h "$MQTT_HOST" -u "$MQTT_USER" -P "$MQTT_PASS" \
    -t "$HEARTBEAT_TOPIC" -C 1 -W "$TIMEOUT" > /dev/null 2>&1

if [ $? -eq 0 ]; then
    # Heartbeat received, ping Healthchecks.io
    curl -fsS -m 10 --retry 3 "$HC_PING_URL" > /dev/null
else
    # No heartbeat, don't ping (Healthchecks.io will alert after grace period)
    logger "highland-watchdog: No heartbeat from Node-RED"
fi
```

**Cron entry (`/etc/cron.d/highland-watchdog`):**
```
* * * * * root /usr/local/bin/highland-watchdog.sh
```

**Flow side (Health Monitor):**
- Publishes heartbeat to `highland/status/node_red/heartbeat` every 30 seconds
- Watchdog expects at least one heartbeat per 90-second window

**Dependencies:**
- `mosquitto-clients` package (for `mosquitto_sub`)
- `curl` for Healthchecks.io pings

---

## Flow Organization

### Flow Types

| Type | Purpose | Examples |
|------|---------|----------|
| **Area flows** | Own devices and automations for a physical area | `Garage`, `Living Room`, `Front Porch` |
| **Utility flows** | Cross-cutting concerns that transcend areas | `Scheduler`, `Security`, `Notifications`, `Logging`, `Backup`, `ACK Tracker`, `Battery Monitor`, `Health Monitor`, `Config Loader`, `Daily Digest` |

### Naming Convention

Flows are named by their area or utility function:
- `Garage`
- `Living Room`
- `Scheduler`
- `Notifications`

*No prefixes or suffixes needed — the flow list in Node-RED is the organizing structure.*

---

## Link Nodes & Groups

### Preferred Over Subflows For:
- Keeping logic visible within a flow
- Breaking up long horizontal chains
- Creating logical sections within a flow

### Pattern: Grouped Logic with Link Nodes

```
┌─────────────────────────────────────────────────────────────────┐
│ Group: Handle Motion Event                                      │
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────┐  │
│  │ Link In │───►│ Process │───►│ Decide  │───►│ Link Out    │  │
│  │ motion  │    │ payload │    │ action  │    │ to lights   │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group: Control Lights                                           │
│                                                                 │
│  ┌─────────────┐    ┌─────────┐    ┌─────────┐                 │
│  │ Link In     │───►│ Set     │───►│ MQTT    │                 │
│  │ from motion │    │ payload │    │ Out     │                 │
│  └─────────────┘    └─────────┘    └─────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Each group is a logical unit with a clear purpose
- Link nodes connect groups without spaghetti wires
- Flow reads top-to-bottom or left-to-right in sections
- Minimizes horizontal scrolling

---

## Subflows

### Use Sparingly, For Truly Reusable Components

**Good candidates for subflows:**
- Pub/sub wrappers (MQTT In/Out with context awareness)
- Standardized notification dispatch
- Common transformations used across many flows

**Not good candidates:**
- Flow-specific logic (keep it visible)
- One-off utilities (just use a function node)
- Anything that hides important business logic

### Pub/Sub Subflow Pattern

Wrap MQTT In/Out nodes with context awareness:

**Inbound subflow:**
- Subscribes to topic
- Checks if this flow is an intended recipient (based on topic structure or payload)
- Passes through or filters

**Outbound subflow:**
- Accepts message from flow
- Adds standard envelope (timestamp, source, message_id)
- Publishes to appropriate topic
- Passes message through (not terminal) for chaining

---

## Flow Registration

### Purpose

Each area flow self-registers its identity and owned devices. This creates a queryable global registry that enables:
- Targeting messages by area
- Looking up devices by capability
- Knowing which area owns which device

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          On Startup / Deploy                        │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │   Foyer Flow    │──► global.flowRegistry['foyer'] = {...}        │
│  └─────────────────┘                                                │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │   Garage Flow   │──► global.flowRegistry['garage'] = {...}       │
│  └─────────────────┘                                                │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │  Living Room    │──► global.flowRegistry['living_room'] = {...}  │
│  └─────────────────┘                                                │
│                                                                     │
│  Each flow overwrites its own key. No global purge, no timing issues│
└─────────────────────────────────────────────────────────────────────┘
```

### Storage

| Storage | Persistence | Purpose |
|---------|-------------|---------|
| `flow.identity` | Disk | This flow's identity and devices |
| `global.flowRegistry` | Disk | All flows' registrations |
| `global.config.deviceRegistry` | Disk | Device details (single source of truth for capabilities) |

**Note:** Node-RED context storage is configured for disk persistence. Survives restarts.

### Flow Registry Structure

```json
{
  "foyer": {
    "devices": ["foyer_entry_door", "foyer_environment"]
  },
  "garage": {
    "devices": ["garage_entry_door", "garage_carriage_left", "garage_carriage_right", "garage_motion_sensor", "garage_environment"]
  },
  "living_room": {
    "devices": ["living_room_overhead", "living_room_environment"]
  }
}
```

### Registration Boilerplate

Every area flow includes this pattern:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Flow: Foyer                                                        │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Group: Flow Registration                                   │   │
│  │                                                             │   │
│  │  ┌──────────────────────┐    ┌───────────────────────┐   │   │
│  │  │ Inject               │───►│ Register Flow           │   │   │
│  │  │ • On startup         │    │                         │   │   │
│  │  │ • On deploy          │    │ • Set flow.identity     │   │   │
│  │  │ • Manual trigger     │    │ • Update flowRegistry   │   │   │
│  │  └──────────────────────┘    └───────────────────────┘   │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ... rest of flow logic ...                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Register Flow function node:**

```javascript
const flowIdentity = {
  area: 'foyer',
  devices: ['foyer_entry_door', 'foyer_environment']
};

// Set flow-level identity
flow.set('identity', flowIdentity);

// Update global registry (overwrite this flow's section only)
const registry = global.get('flowRegistry') || {};
registry[flowIdentity.area] = {
  devices: flowIdentity.devices
};
global.set('flowRegistry', registry);

node.status({ fill: 'green', shape: 'dot', text: `Registered: ${flowIdentity.devices.length} devices` });

return msg;
```

### Capability Lookup at Runtime

Device capabilities are NOT stored in the flow registry. They live in the Device Registry (single source of truth). Query at runtime:

```javascript
// "Find all areas with locks"
function getAreasByCapability(capability) {
  const flowRegistry = global.get('flowRegistry');
  const deviceRegistry = global.get('config.deviceRegistry');
  const result = {};
  
  for (const [area, areaData] of Object.entries(flowRegistry)) {
    const matchingDevices = areaData.devices.filter(deviceId => {
      const device = deviceRegistry[deviceId];
      return device && device.capabilities.includes(capability);
    });
    
    if (matchingDevices.length > 0) {
      result[area] = matchingDevices;
    }
  }
  
  return result;
}

// Usage:
getAreasByCapability('lock');
// → { "foyer": ["foyer_entry_door"], "garage": ["garage_entry_door"] }
```

### Message Targeting Pattern

**Security flow wants to lock all locks:**

```javascript
// 1. Find all areas with lock capability
const locksByArea = getAreasByCapability('lock');
// → { "foyer": ["foyer_entry_door"], "garage": ["garage_entry_door"] }

// 2. Target areas in message
const targetAreas = Object.keys(locksByArea); // ["foyer", "garage"]

// 3. Register expected ACKs at device level
const expectedAcks = Object.values(locksByArea).flat(); 
// → ["foyer_entry_door", "garage_entry_door"]

// 4. Publish lockdown with area targets
msg.payload = {
  message_id: 'lock_123',
  source: 'security',
  recipients: targetAreas,
  request_ack: true
};
// Publish to: highland/event/security/lockdown

// 5. Register with ACK Tracker
msg.ackRegistration = {
  correlation_id: 'lock_123',
  expected_sources: expectedAcks,
  timeout_seconds: 30
};
// Publish to: highland/ack/register
```

**Area flow receives and responds:**

```javascript
// Foyer flow receives lockdown message
// recipients: ["foyer", "garage"]
// flow.identity.area: "foyer" → matches, process message

// Command the lock
// ... (via Command Dispatcher)

// Send ACK at device level
msg.payload = {
  ack_correlation_id: 'lock_123',
  source: 'foyer_entry_door',  // Device, not area
  timestamp: new Date().toISOString()
};
// Publish to: highland/ack
```

### Staleness Handling

**On device removal:**
1. Update the flow's registration boilerplate to remove device
2. Deploy flow
3. Flow overwrites its registry entry, device is gone

**On flow removal:**
1. Flow's registry entry persists (stale)
2. Acceptable: stale entry causes no harm (messages to deleted flow just aren't received)
3. Optional: Manual cleanup or periodic audit

*Details TBD during implementation.*

---

## Startup Sequencing

### The Problem

On Node-RED startup or deploy, two things happen simultaneously that can conflict:

1. MQTT subscriptions are established and the broker **immediately** delivers all retained messages for those topics
2. The flow's own initialization (Config Loader, flow registration, context restoration from disk) is still in progress

A retained message — say, `highland/state/scheduler/period` — can arrive and trigger handler logic before `global.config` is loaded, before flow context is restored, before the flow has registered itself. Depending on what the handler does, this ranges from harmless to quietly wrong.

Additionally, there is no native MQTT signal for "end of retained messages." The broker delivers retained messages as fast as it can after subscription with no marker indicating when retained delivery is complete and real-time messages begin.

### The Init Probe Pattern

The solution is an **echo probe** combined with an initialization gate.

**How it works:**

Within a single MQTT connection, message delivery is ordered. All retained messages for subscribed topics are already in-flight before any new publish can be queued. By publishing a probe message to a topic the flow is also subscribed to, the probe is guaranteed to arrive *after* all retained messages for that connection.

When the flow receives its own probe back, it knows retained delivery is complete.

```
On startup:
  1. Set flow.initialized = false  (gate closed)
  2. Subscribe to all topics (including retained state topics)
  3. Publish probe to highland/command/nodered/init_probe/{flow_name} (non-retained)
  4. Retained messages begin arriving → buffer or discard (gate is closed)
  5. Own probe arrives → all retained messages have been received
  6. Run initialization work (Config Loader complete, flow registration, etc.)
  7. Set flow.initialized = true  (gate opens)
  8. Process buffered retained state
  9. Normal operation begins
```

**Flow diagram:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  Group: Startup Sequencing                                          │
│                                                                     │
│  ┌──────────────────┐    ┌──────────────────────────────────┐    │
│  │ Inject           │───►│ Init                               │    │
│  │ • On startup     │    │ • flow.set('initialized', false)   │    │
│  │ • On deploy      │    │ • flow.set('state_buffer', [])     │    │
│  └──────────────────┘    │ • Publish init_probe               │    │
│                           └──────────────────────────────────┘    │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ MQTT-in: highland/state/#  (retained topics)                 │  │
│  │          ──► gate check ──► if closed: buffer msg            │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ MQTT-in: highland/command/nodered/init_probe/{flow_name}     │  │
│  │          ──► own probe received                              │  │
│  │          ──► run init work (config loaded, flow registered)  │  │
│  │          ──► flow.set('initialized', true)                   │  │
│  │          ──► process buffered state                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Gate Pattern

Every handler that depends on initialized state checks the gate before proceeding:

```javascript
// At the top of any handler that reads flow or global config
if (!flow.get('initialized')) {
    // Buffer if this is retained state we'll need later
    const buffer = flow.get('state_buffer') || [];
    buffer.push(msg);
    flow.set('state_buffer', buffer);
    return null;  // Stop processing
}

// Gate is open — safe to proceed
const config = global.get('config.deviceRegistry');
// ... rest of handler logic
```

### State vs Event Handling During Init

**Retained state messages** (from `highland/state/#`) — buffer these. They represent current truth and will be needed once the gate opens.

**Point-in-time events** (from `highland/event/#`) — discard these. If a real-time event fires during the brief init window (typically < 1 second), it is genuinely gone and cannot be recovered. This is acceptable — the window is very short and events are by definition momentary.

### Processing the Buffer

Once the gate opens, process buffered state in order:

```javascript
// In the init_probe handler, after setting initialized = true
const buffer = flow.get('state_buffer') || [];
flow.set('state_buffer', []);  // Clear buffer

for (const bufferedMsg of buffer) {
    // Re-inject each buffered message into the handler chain
    node.send(bufferedMsg);
}
```

### Reacting to State vs Reacting to Events

This pattern clarifies an important distinction in how flows handle scheduler (and other) state:

**Two entry points, one handler:**

```
highland/state/scheduler/period  ──┐  (retained — arrives during init, buffered,
  (startup recovery path)          │   processed after gate opens)
                                   ├──► mutate flow.current_period ──► period logic
highland/event/scheduler/evening ──┘  (real-time — arrives during normal operation,
  (real-time transition path)         gate already open)
```

The mutation (`flow.set('current_period', value)`) is a waypoint in the chain, not the end of it. The message continues downstream to the handler logic in the same pipeline pass. There is no observer or watch mechanism on context variables — the MQTT message is always the trigger, and context is the shared memory that downstream logic reads from.

**Why this matters:** Any logic in the flow that needs to know the current period calls `flow.get('current_period')`. It never re-queries MQTT. The state topic is the recovery mechanism; local context is the working copy.

### Probe Topic Convention

```
highland/command/nodered/init_probe/{flow_name}
```

Each flow uses its own probe topic to avoid cross-flow interference. The `highland/command/` namespace is appropriate — this is an imperative trigger, not an event or state.

The probe message is non-retained and carries no meaningful payload beyond the standard envelope. It exists solely to act as a position marker in the message queue.

### Notes

- The init window is typically well under one second — the probe round-trip over local MQTT is fast. The gate is a safety net, not a performance concern.
- This pattern applies to **every flow** that subscribes to retained state topics. Area flows, utility flows, all of them.
- The Config Loader flow itself bootstraps before other flows need it — it has no retained state dependencies of its own, so its init is simpler (no probe needed, just load files and set globals).
- If the MQTT broker itself is unavailable on startup, the probe never returns. The flow should have a startup timeout (e.g., 10 seconds) after which it logs an error and enters a degraded state rather than waiting indefinitely.

---

## Error Handling

### Two-Tier Approach

1. **Targeted handlers** — Catch errors in specific groups where you need custom handling
2. **Flow-wide catch-all** — Single Error node per flow catches anything unhandled

```
┌─────────────────────────────────────────────────────────────────┐
│ Flow: Garage                                                    │
│                                                                 │
│  ┌─────────────────────────────────────┐                       │
│  │ Group: Critical Operation           │                       │
│  │                                     │                       │
│  │  [nodes] ───► [targeted error] ─────├──► (custom handling)  │
│  │                                     │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  ┌─────────────────────────────────────┐                       │
│  │ Group: Normal Operation             │                       │
│  │                                     │                       │
│  │  [nodes] ───────────────────────►│ (errors bubble up)    │
│  │                                     │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Flow-wide Error Node                                    │   │
│  │ Catches all unhandled errors → dispatches to logging    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Logging Framework

### Concept

Logging answers: *"How important is this for troubleshooting/audit?"*

Logging is separate from notifications. They intersect (a CRITICAL log may auto-generate a notification), but serve different purposes.

### Log Storage

**Format:** JSONL (JSON Lines) — one JSON object per line

**Location:** `/var/log/highland/` (or equivalent on Node-RED host)

**Rotation:** Daily files

```
/var/log/highland/
├── highland-2025-02-22.jsonl
├── highland-2025-02-23.jsonl
└── highland-2025-02-24.jsonl  (current)
```

**Retention:** Keep N days, delete older (scheduled cleanup task)

### Unified Log

A single daily log file contains entries from ALL systems — Node-RED, Z2M, Z-Wave JS, HA, watchdog, etc. This provides a unified view similar to Windows Event Viewer.

**Log entry structure:**

| Field | Purpose | Examples |
|-------|---------|----------|
| `timestamp` | When it happened | `2025-02-24T10:00:00Z` |
| `system` | Which system generated the log | `node_red`, `ha`, `z2m`, `zwave_js`, `watchdog` |
| `source` | Component within that system | `garage`, `scheduler`, `coordinator` |
| `level` | Severity | `VERBOSE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | Human-readable description | `Failed to turn on carriage lights` |
| `context` | Structured additional data | `{"device": "...", "error": "..."}` |

**Example entries:**

```json
{"timestamp":"2025-02-24T10:00:00Z","system":"node_red","source":"garage","level":"ERROR","message":"Failed to turn on carriage lights","context":{"device":"light.garage_carriage","error":"MQTT timeout"}}
{"timestamp":"2025-02-24T10:00:05Z","system":"z2m","source":"coordinator","level":"WARN","message":"Device interview failed","context":{"device":"garage_motion_sensor"}}
{"timestamp":"2025-02-24T10:00:10Z","system":"ha","source":"recorder","level":"INFO","message":"Database purge completed","context":{"rows_deleted":15000}}
{"timestamp":"2025-02-24T10:00:15Z","system":"watchdog","source":"node_red_monitor","level":"INFO","message":"Heartbeat received","context":{}}
```

### Log Levels

| Level | Value | When to Use |
|-------|-------|-------------|
| `VERBOSE` | 0 | Granular trace; active debugging only |
| `DEBUG` | 1 | Detailed info useful for troubleshooting |
| `INFO` | 2 | Normal operational events worth recording |
| `WARN` | 3 | Something unexpected but not broken |
| `ERROR` | 4 | Something failed but flow continues |
| `CRITICAL` | 5 | Catastrophic failure; intervention needed |

### Per-Flow Log Level Threshold

Each flow has a configured minimum log level (stored in flow context):

```javascript
// Flow context
flow.set('logLevel', 'WARN');  // This flow only emits WARN and above
```

When a flow emits a log message:
- If message level >= flow threshold → emit to logging utility
- If message level < flow threshold → suppress

**Use case:** Set a flow to `DEBUG` while developing, `WARN` in steady state.

### Log Event Topic

Single topic — all systems publish here:

```
highland/event/log
```

### Log Event Payload (MQTT)

```json
{
  "timestamp": "2025-02-24T14:30:00Z",
  "system": "node_red",
  "source": "garage",
  "level": "ERROR",
  "message": "Failed to turn on carriage lights",
  "context": {
    "device": "light.garage_carriage",
    "error": "MQTT timeout"
  }
}
```

### How Systems Log

| System | Mechanism |
|--------|-----------|
| **Node-RED** | Flows publish to `highland/event/log`; Logging utility writes to file |
| **Z2M / Z-Wave JS** | Publish to `highland/event/log` (if configurable), or sidecar script |
| **Home Assistant** | Publish to `highland/event/log` via automation, or sidecar script |
| **Watchdog** | Publish to `highland/event/log` |

Node-RED's Logging utility flow subscribes to `highland/event/log` and writes ALL entries to the unified JSONL file, regardless of `system`.

### Logging Utility Flow

Centralized flow that:
1. Subscribes to `highland/event/log`
2. Appends to today's JSONL file
3. If level = `CRITICAL` → auto-dispatch to Notification Utility

```
highland/event/log ──► Logging Utility ──► Append to JSONL
                              │
                              │ (if CRITICAL)
                              ▼
                       highland/event/notify
```

### Querying Logs

JSONL + `jq` provides powerful ad-hoc querying:

```bash
# All errors from any system
jq 'select(.level == "ERROR")' highland-2025-02-24.jsonl

# All Node-RED entries
jq 'select(.system == "node_red")' highland-2025-02-24.jsonl

# All entries from garage (regardless of system)
jq 'select(.source == "garage")' highland-2025-02-24.jsonl

# Z2M warnings and above
jq 'select(.system == "z2m" and (.level == "WARN" or .level == "ERROR" or .level == "CRITICAL"))' highland-2025-02-24.jsonl

# Last 10 entries
tail -10 highland-2025-02-24.jsonl | jq '.'
```

### Future: Log Shipping (Deferred)

When NAS is available or if cloud aggregation is desired:
- Ship JSONL files to central location
- JSONL is compatible with most aggregators (Loki, Elastic, Datadog)
- Could also stream via MQTT to external subscriber

*Details TBD when infrastructure supports it.*

### Auto-Notify Behavior

**Only CRITICAL logs auto-notify.** ERROR and below do not.

| Level | Auto-Notify | Rationale |
|-------|-------------|-----------|
| CRITICAL | Yes | System health, potential data loss, immediate intervention |
| ERROR | No | Something failed but system continues; log and move on |
| WARN and below | No | Informational |

**CRITICAL examples:**
- Database size threshold exceeded
- Disk usage critical
- Sustained abnormal CPU spikes
- Core service unresponsive

**ERROR examples (no auto-notify):**
- API timeout (data stale but system functional)
- Device command failed (retry later)
- Automation couldn't complete non-critical path

### Escalation is Flow Responsibility

If a flow wants to notify after repeated ERRORs, *that flow* decides:

```
┌───────────────────────────────────────────────────────────────┐
│  Example: Weather Flow                                      │
│                                                             │
│  API call fails → log ERROR                                 │
│       │                                                     │
│       ▼                                                     │
│  Increment failure counter (flow context)                   │
│       │                                                     │
│       ▼                                                     │
│  Counter > threshold? ──► YES ──► Publish to notify         │
│       │                           (deliberate choice)       │
│       ▼                                                     │
│      NO → continue, try again next cycle                    │
└───────────────────────────────────────────────────────────────┘
```

The logging framework doesn't escalate. Flows own their escalation logic.

---

## Device Registry

### Purpose

Centralized knowledge about devices — protocol, topic structure, capabilities, and metadata. Abstracts the differences between Z2M and Z-Wave JS UI so flows don't need to know protocol details.

### Registry Structure

```json
{
  "garage_carriage_left": {
    "friendly_name": "Garage Carriage Light (Left)",
    "protocol": "zigbee",
    "topic": "zigbee2mqtt/garage_carriage_left",
    "area": "garage",
    "capabilities": ["on_off", "brightness"],
    "battery": null
  },
  "garage_motion_sensor": {
    "friendly_name": "Garage Motion Sensor",
    "protocol": "zigbee",
    "topic": "zigbee2mqtt/garage_motion_sensor",
    "area": "garage",
    "capabilities": ["motion", "battery"],
    "battery": {
      "type": "CR2032",
      "quantity": 1
    }
  },
  "foyer_entry_door": {
    "friendly_name": "Front Door Lock",
    "protocol": "zwave",
    "topic": "zwave/foyer_entry_door",
    "area": "foyer",
    "capabilities": ["lock", "battery"],
    "battery": {
      "type": "AA",
      "quantity": 4
    }
  }
}
```

**Fields:**

| Field | Purpose |
|-------|---------|
| `friendly_name` | User-facing name for notifications, dashboards |
| `protocol` | `zigbee` or `zwave` — determines command formatting |
| `topic` | Base MQTT topic for this device |
| `area` | Physical area (for grouping, context) |
| `capabilities` | What actions this device supports |
| `battery` | Battery metadata (null if mains-powered) |

**Note:** The registry key (e.g., `foyer_entry_door`) is used for internal references and ACK correlation. `friendly_name` is used for user-facing output.

### Storage

Device registry is stored as an external JSON file, loaded into global context at startup. See **Configuration Management** section for full details.

**File:** `/home/nodered/config/device_registry.json`

**Access:** `global.get('config.deviceRegistry')`

### Population

**Manual with validation:**
- Maintain the JSON file directly (IDE, version control)
- Validation flow checks actual devices against registry
- Reports discrepancies (log/notify), does not block commands

---

## Configuration Management

### Overview

Centralized configuration using external JSON files. Separation of version-controllable config from secrets.

### File Structure

```
/home/nodered/config/
├── device_registry.json        ← git: yes
├── flow_registry.json          ← git: yes (area→device mappings, if persisted)
├── notifications.json          ← git: yes (recipient mappings, channels)
├── thresholds.json             ← git: yes (battery, health, etc.)
├── healthchecks.json           ← git: yes (service config)
├── secrets.json                ← git: NO (.gitignore)
└── README.md                   ← git: yes (documents config structure)
```

*Note: Scheduler configuration (periods, sunrise/sunset) lives in schedex nodes within the Scheduler flow, not external config.*

### Config Categories

| Category | Examples | Version Control |
|----------|----------|-----------------|
| **Structural** | Device registry, flow registry, notification recipients | Yes |
| **Tunable** | Thresholds, scheduler times, timeouts | Yes |
| **Secrets** | API keys, credentials, tokens, passwords | **No** |

### Example: secrets.json

```json
{
  "mqtt": {
    "username": "highland",
    "password": "..."
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "user": "...",
    "password": "..."
  },
  "weather_api_key": "abc123...",
  "google_calendar_api_key": "...",
  "healthchecks_io": {
    "mqtt": "https://hc-ping.com/uuid-1",
    "z2m": "https://hc-ping.com/uuid-2",
    "zwave": "https://hc-ping.com/uuid-3",
    "ha": "https://hc-ping.com/uuid-4",
    "node_red": "https://hc-ping.com/uuid-5"
  },
  "ai_providers": {
    "openai_api_key": "sk-...",
    "anthropic_api_key": "sk-ant-..."
  }
}
```

### Example: thresholds.json

```json
{
  "battery": {
    "warning": 35,
    "critical": 15
  },
  "health": {
    "disk_warning": 70,
    "disk_critical": 90,
    "cpu_warning": 80,
    "cpu_critical": 95,
    "memory_warning": 80,
    "memory_critical": 95,
    "devices_offline_critical_percent": 20
  },
  "ack": {
    "default_timeout_seconds": 30
  }
}
```

### Example: notifications.json

```json
{
  "recipients": {
    "mobile_joseph": {
      "type": "ha_companion",
      "service": "notify.mobile_app_joseph_phone",
      "admin": true
    },
    "mobile_spouse": {
      "type": "ha_companion",
      "service": "notify.mobile_app_spouse_phone",
      "admin": false
    }
  },
  "channels": {
    "highland_low": { "importance": "low", "dnd_override": false },
    "highland_default": { "importance": "default", "dnd_override": false },
    "highland_high": { "importance": "high", "dnd_override": true },
    "highland_critical": { "importance": "high", "dnd_override": true }
  },
  "daily_digest": {
    "recipients": ["joseph@example.com"],
    "enabled": true
  },
  "defaults": {
    "admin_only": ["mobile_joseph"],
    "all": ["mobile_joseph", "mobile_spouse"]
  }
}
```

**Recipient targeting:**
- `admin: true` — Receives administrative notifications (system health, backups, etc.)
- `admin: false` — Receives household notifications only (security, weather, etc.)
- `defaults.admin_only` — Default recipient list for admin-type notifications
- `defaults.all` — Default recipient list for household notifications

### Config Loader Utility Flow

Loads all config files into global context at startup.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Config Loader (Utility Flow)                                       │
│                                                                     │
│  Triggers:                                                          │
│    • Node-RED startup                                               │
│    • Node-RED deploy                                                │
│    • Manual inject                                                  │
│    • MQTT: highland/command/config/reload                           │
│    • MQTT: highland/command/config/reload/{config_name}             │
│                                                                     │
│  Actions:                                                           │
│    1. Read each JSON file from /home/nodered/config/                │
│    2. Validate JSON structure                                       │
│    3. Store in global.config namespace:                             │
│         global.config.deviceRegistry                                │
│         global.config.flowRegistry                                  │
│         global.config.notifications                                 │
│         global.config.thresholds                                    │
│         global.config.healthchecks                                  │
│         global.config.secrets                                       │
│    4. Log: "Config loaded: {list}"                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Accessing Config in Flows

```javascript
// Device info
const device = global.get('config.deviceRegistry.foyer_entry_door');
const friendlyName = device.friendly_name;

// Thresholds
const batteryWarn = global.get('config.thresholds.battery.warning');
const batteryCrit = global.get('config.thresholds.battery.critical');

// Secrets
const apiKey = global.get('config.secrets.weather_api_key');
const mqttUser = global.get('config.secrets.mqtt.username');

// Notification recipients
const josephDevice = global.get('config.notifications.recipients.mobile_joseph');
const adminRecipients = global.get('config.notifications.defaults.admin_only');
```

### Structural Validation

On load, validate each config file:
- JSON parses correctly
- Required fields present for each entry type
- Log errors, don't crash Node-RED

### Discovery Validation

Periodic or on-demand check comparing device registry against actual Z2M/Z-Wave device lists:
- Devices in Z2M/Z-Wave but not in registry → log/notify (unregistered)
- Devices in registry but not in Z2M/Z-Wave → log/notify (stale or offline)
- Does **not** block commands to unregistered devices

---

## Command Dispatcher

### Purpose

Translate high-level commands ("turn on garage_carriage_left") into protocol-specific MQTT messages. Flows say *what* they want; the dispatcher knows *how*.

### Common Actions (v1)

| Action | Applies To | Notes |
|--------|------------|-------|
| `on` | lights, switches | Turn on |
| `off` | lights, switches | Turn off |
| `toggle` | lights, switches | Toggle state |
| `brightness` | dimmable lights | Set brightness (0-255 or 0-100, normalized) |
| `lock` | locks | Engage lock |
| `unlock` | locks | Disengage lock |
| `raw` | any | Passthrough for unsupported actions |

### Subflow Interface

**Input:**
```json
{
  "entity": "garage_carriage_left",
  "action": "on"
}
```

```json
{
  "entity": "garage_carriage_left",
  "action": "brightness",
  "value": 50
}
```

```json
{
  "entity": "some_device",
  "action": "raw",
  "payload": { "custom": "data" }
}
```

**Behavior:**
1. Lookup entity in Device Registry
2. Validate action against capabilities (optional, could warn/error)
3. Format payload based on protocol + action
4. Publish to appropriate topic

### Protocol Translation

**Zigbee (Z2M):**
```
Topic: zigbee2mqtt/{device}/set
Payload: {"state": "ON"} / {"brightness": 255}
```

**Z-Wave (Z-Wave JS UI MQTT gateway):**
```
Topic: zwave/{node}/set (configurable)
Payload: Protocol-specific, may differ
```

The dispatcher handles this translation internally.

### Extending Actions

| Scenario | Approach |
|----------|----------|
| New device, existing capability | Add to registry only |
| One-off command | Use `raw` passthrough |
| Repeated new capability | Add to common actions |

### Usage in Flows

```
┌──────────────────┐    ┌────────────────────┐
│ Evening period   │───►│ Command Dispatcher │
│ event arrives    │    │                    │
│                  │    │ entity: garage_    │
│                  │    │   carriage_left    │
│                  │    │ action: on         │
└──────────────────┘    └────────────────────┘
```

Flow doesn't know or care about Zigbee topics or payload formats.

---

## ACK Tracker Utility Flow

### Purpose

Centralized tracking of acknowledgment requests. Flows that need confirmation of actions register their expectations, the tracker collects ACKs, and reports results on timeout. Keeps ACK bookkeeping out of individual flows.

### Topics

| Topic | Purpose | Publisher |
|-------|---------|-----------|
| `highland/ack/register` | Register expectation for ACKs | Requesting flow |
| `highland/ack` | ACK responses | Responding flows |
| `highland/ack/result` | Outcome after timeout | ACK Tracker |

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ Flow A (Request Originator)                                         │
│                                                                     │
│  1. Generate message_id                                             │
│  2. Register with ACK Tracker via highland/ack/register             │
│  3. Publish event with message_id, request_ack: true                │
│  4. Subscribe to highland/ack/result, filter by correlation_id      │
│  5. Handle success/failure                                          │
└─────────────────────────────────────────────────────────────────────┘
         │                                          │
         │ highland/event/...                       │ highland/ack/register
         ▼                                          ▼
┌──────────────────────┐                 ┌─────────────────────────────┐
│ Flow B (Subscriber)  │                 │ ACK Tracker Utility Flow    │
│                      │                 │                             │
│  • Receives event    │                 │  • Receives registrations   │
│  • Does work         │                 │  • Starts timeout timer     │
│  • Publishes ACK     │────────────────►│  • Collects ACKs by         │
│                      │ highland/ack    │    correlation_id           │
└──────────────────────┘                 │  • On timeout: publish      │
                                         │    result                   │
                                         └──────────────├──────────────┘
                                                        │
                                                        │ highland/ack/result
                                                        ▼
                                         ┌─────────────────────────────┐
                                         │ Flow A (handles result)     │
                                         │                             │
                                         │  • Lookup missing sources   │
                                         │    in Device Registry       │
                                         │  • Resolve friendly names   │
                                         │  • Notify / escalate        │
                                         └─────────────────────────────┘
```

### Payloads

**Registration (Flow A → Tracker):**
```json
{
  "correlation_id": "abc123",
  "expected_sources": ["foyer_entry_door", "garage_entry_door"],
  "timeout_seconds": 30,
  "source": "security"
}
```

**ACK (Flow B → Tracker):**
```json
{
  "ack_correlation_id": "abc123",
  "source": "foyer_entry_door",
  "timestamp": "2025-02-24T22:00:05Z"
}
```

**Result (Tracker → Flow A):**
```json
{
  "correlation_id": "abc123",
  "expected": 2,
  "received": 1,
  "sources": ["foyer_entry_door"],
  "missing": ["garage_entry_door"],
  "success": false
}
```

### Standard Event Properties

All events that may request ACKs include these optional properties:

```json
{
  "message_id": "uuid-or-similar",
  "timestamp": "...",
  "source": "...",
  "request_ack": true
}
```

When `request_ack: false` (or absent), no ACK infrastructure is engaged.

### Timeout-Only Failure Detection

ACK responses are positive only. If a responder fails to complete its work, it simply doesn't send an ACK. The tracker detects this as a missing response on timeout.

No explicit failure ACKs — keeps the pattern simple. If future use cases require explicit failure reporting, we can extend.

### Separation of Concerns

| Component | Responsibility |
|-----------|----------------|
| **ACK Tracker** | Count ACKs, track by correlation_id, report results (raw keys only) |
| **Requesting Flow** | Register expectations, handle success/failure, decide escalation |
| **Notification Flow** | Resolve friendly names from Device Registry, format user-facing messages |

The tracker doesn't know what a "lock" is or what "foyer_entry_door" means. It just counts and reports.

### Friendly Name Resolution

The `source` in ACK payloads and `missing`/`sources` in results are keys into the Device Registry. Consuming flows (or the Notification flow) perform lookups to get `friendly_name` for user-facing messages.

```
missing: ["garage_entry_door"]
    │
    ▼
Device Registry lookup
    │
    ▼
friendly_name: "Garage Door Lock"
    │
    ▼
Notification: "Lockdown failed: Garage Door Lock did not respond"
```

### Example: Lockdown Flow

```
Security Flow (lockdown):
  │
  ├─► Publish: highland/ack/register
  │   { correlation_id: "lock_123", 
  │     expected_sources: ["foyer_entry_door", "garage_entry_door"], 
  │     timeout_seconds: 30 }
  │
  ├─► Publish: highland/event/security/lockdown
  │   { message_id: "lock_123", request_ack: true }
  │
  └─► Subscribe: highland/ack/result (filter: correlation_id == "lock_123")

Front Door Flow:
  │
  ├─► Receives: highland/event/security/lockdown
  ├─► Commands lock via Command Dispatcher
  └─► Publishes: highland/ack
      { ack_correlation_id: "lock_123", source: "foyer_entry_door" }

Garage Door Flow:
  │
  ├─► Receives: highland/event/security/lockdown
  ├─► Commands lock... (but lock jams, command times out)
  └─► (No ACK sent)

ACK Tracker:
  │
  └─► After 30s, publishes: highland/ack/result
      { correlation_id: "lock_123", expected: 2, received: 1, 
        sources: ["foyer_entry_door"], missing: ["garage_entry_door"], 
        success: false }

Security Flow (receives result):
  │
  ├─► Looks up "garage_entry_door" in Device Registry → "Garage Door Lock"
  └─► Publishes: highland/event/notify
      { severity: "high", title: "Lockdown Failed", 
        message: "Garage Door Lock did not respond", ... }
```

---

## Battery Monitor Utility Flow

### Purpose

Track battery-powered device levels, flag devices needing replacement, notify appropriately.

### Battery States

| State | Threshold | Notification | Notes |
|-------|-----------|--------------|-------|
| `normal` | > 35% | None | Healthy |
| `low` | 35% - 15% | Normal priority | Doesn't break DND; sent once when threshold crossed |
| `critical` | < 15% | High priority | Breaks DND; repeats every 24 hours until recovered |

**Threshold rationale:** Some devices (e.g., Sonoff) report in 10% increments. These thresholds provide 7 levels of normal, 2 levels of low, 1 level of critical for coarse reporters.

### Device Registry Extension

Battery-powered devices include battery metadata:

```json
{
  "garage_motion_sensor": {
    "friendly_name": "Garage Motion Sensor",
    "protocol": "zigbee",
    "topic": "zigbee2mqtt/garage_motion_sensor",
    "area": "garage",
    "capabilities": ["motion", "battery"],
    "battery": {
      "type": "CR2032",
      "quantity": 1
    }
  },
  "foyer_entry_door": {
    "friendly_name": "Front Door Lock",
    "protocol": "zwave",
    "topic": "zwave/foyer_entry_door",
    "area": "foyer",
    "capabilities": ["lock", "battery"],
    "battery": {
      "type": "AA",
      "quantity": 4
    }
  }
}
```

### Flow Behavior

```
Raw MQTT (battery level reports)
         │
         ▼
┌───────────────────────────────────────────────────────────────┐
│  Battery Monitor Utility Flow                               │
│                                                             │
│  1. Receive battery level from device                       │
│  2. Lookup device in registry                               │
│  3. Determine state (normal/low/critical)                   │
│  4. Compare to previous state                               │
│  5. If state changed → publish event, notify if appropriate │
│  6. If critical → schedule 24hr repeat reminder             │
│  7. If recovered from critical → cancel repeat reminder     │
└───────────────────────────────────────────────────────────────┘
         │
         ├──► highland/event/battery/low
         ├──► highland/event/battery/critical
         └──► highland/event/battery/recovered
```

### Events

**State transition events:**
```
highland/event/battery/low
highland/event/battery/critical
highland/event/battery/recovered
```

**Payload:**
```json
{
  "timestamp": "2025-02-23T14:30:00Z",
  "source": "battery_monitor",
  "entity": "garage_motion_sensor",
  "level": 32,
  "previous_state": "normal",
  "new_state": "low",
  "battery": {
    "type": "CR2032",
    "quantity": 1
  }
}
```

### Notification Behavior

| Transition | Action |
|------------|--------|
| normal → low | Notify once, normal priority |
| low → critical | Notify immediately, high priority; start 24hr repeat |
| normal → critical | Notify immediately, high priority; start 24hr repeat |
| critical → low | Cancel repeat; notify recovery (normal priority) |
| critical → normal | Cancel repeat; notify recovery (normal priority) |
| low → normal | No notification (silent recovery) |

### Hysteresis

**Re-evaluate, no manual clearing.** If a battery level bounces back above a threshold, the device automatically recovers to the appropriate state. No manual intervention required.

### Open Questions

- [ ] Where to surface "devices needing batteries" data (dashboard widget? periodic summary?)
- [ ] Shopping list aggregation by battery type (deferred)

---

## Notification Framework

### Concept

Notifications answer: *"How urgently does a human need to know about this?"*

Notifications are separate from logging. They intersect (a CRITICAL log may auto-generate a notification), but serve different purposes.

**Example:** NWS issues a Special Weather Statement for patchy fog at 3am.
- Log level: `INFO` at most (not an error, just an event)
- Notification: LOW severity, does NOT break DND (fog lifting by 6am isn't urgent)

### Notification Topic

Single topic — all details in payload:

```
highland/event/notify
```

### Notification Payload (Internal)

```json
{
  "timestamp": "2025-02-24T14:30:00Z",
  "source": "security",
  "severity": "high",
  "title": "Lock Failed to Engage",
  "message": "Front Door Lock did not respond within 30 seconds",
  "recipients": ["mobile_joseph", "mobile_spouse"],
  "dnd_override": true,
  "media": {
    "image": "http://camera.local/snapshot.jpg"
  },
  "actionable": true,
  "actions": [
    { "id": "retry", "label": "Retry Lock" },
    { "id": "dismiss", "label": "Dismiss" }
  ],
  "sticky": true,
  "group": "security_alerts",
  "correlation_id": "lockdown_20250224_2200"
}
```

### Severity Levels

| Severity | DND Override | Use Case |
|----------|--------------|----------|
| `low` | No | Informational; can wait (fog advisory, routine status) |
| `medium` | No | Worth knowing soon, but not urgent |
| `high` | Yes | Needs attention now (lock failure, unexpected motion) |
| `critical` | Yes | Emergency (fire, flood, intrusion) |

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `severity` | Yes | `low`, `medium`, `high`, `critical` |
| `title` | Yes | Short summary |
| `message` | Yes | Full detail |
| `recipients` | No | Target devices; default = all |
| `dnd_override` | No | Derived from severity if not specified |
| `media` | No | Image and/or video URLs |
| `actionable` | No | Can recipient respond?; default = false |
| `actions` | No | Available response actions |
| `sticky` | No | Notification persists until dismissed; default = false |
| `group` | No | Group related notifications together |
| `correlation_id` | No | For linking response back to originating event; also used as notification tag |

### Mobile Implementation: HA Companion App (Android)

Initial implementation uses Home Assistant Companion App. Future channels (Telegram, Signal, etc.) can be added later.

#### Device Targeting

Per-device targeting. Recipients map to HA notify services:

| Recipient | HA Service |
|-----------|------------|
| `mobile_joseph` | `notify.mobile_app_joseph_phone` |
| `mobile_spouse` | `notify.mobile_app_spouse_phone` |

**Use case:** Administrative notifications (system health, backups) go to `mobile_joseph` only. Security alerts go to all devices.

#### Android Notification Channels

Pre-configure channels in HA Companion App for user control over sound/vibration/DND:

| Channel ID | Purpose | DND Override |
|------------|---------|--------------|
| `highland_low` | Informational | No |
| `highland_default` | Standard alerts | No |
| `highland_high` | Urgent alerts | Yes |
| `highland_critical` | Emergency | Yes |

#### Severity → HA Companion Mapping

| Our Severity | HA Priority | Channel | Persistent |
|--------------|-------------|---------|------------|
| `low` | `low` | `highland_low` | No |
| `medium` | `default` | `highland_default` | No |
| `high` | `high` | `highland_high` | No (unless `sticky: true`) |
| `critical` | `high` | `highland_critical` | Yes |

#### HA Companion Service Call

Our payload translated to HA service call:

```yaml
service: notify.mobile_app_joseph_phone
data:
  title: "Lock Failed to Engage"
  message: "Front Door Lock did not respond within 30 seconds"
  data:
    channel: "highland_high"
    importance: "high"
    persistent: true
    image: "http://camera.local/snapshot.jpg"
    tag: "lockdown_20250224_2200"
    group: "security_alerts"
    actions:
      - action: "RETRY_lockdown_20250224_2200"
        title: "Retry Lock"
      - action: "DISMISS_lockdown_20250224_2200"
        title: "Dismiss"
```

#### Clearing Notifications

To programmatically dismiss a notification (e.g., lock succeeded on retry):

```yaml
service: notify.mobile_app_joseph_phone
data:
  message: "clear_notification"
  data:
    tag: "lockdown_20250224_2200"
```

Same `tag`, message `"clear_notification"` — notification disappears from device.

**Use case:** Battery critical notification auto-clears when battery recovers.

#### Notification Grouping

Use `group` field to stack related notifications:

```yaml
data:
  group: "battery_alerts"
```

Multiple battery warnings appear grouped rather than as separate notifications.

### Action Responses

When user taps a notification action, HA fires an event. The Notification Utility flow captures this and publishes to MQTT for consistent handling:

**HA Event:**
```
Event: mobile_app_notification_action
Data: { action: "RETRY_lockdown_20250224_2200" }
```

**Published to MQTT:**
```
Topic: highland/event/notify/action_response
Payload: {
  "timestamp": "2025-02-24T14:32:00Z",
  "source": "notification",
  "action": "retry",
  "correlation_id": "lockdown_20250224_2200",
  "device": "mobile_joseph"
}
```

Originating flow (e.g., Security) subscribes to `highland/event/notify/action_response`, filters by `correlation_id`, and handles accordingly.

### Notification Utility Flow

Centralized flow that:
1. Subscribes to `highland/event/notify`
2. Translates internal payload → HA Companion format
3. Calls appropriate `notify.mobile_app_*` service(s) based on `recipients`
4. Subscribes to HA `mobile_app_notification_action` events
5. Translates action responses → `highland/event/notify/action_response`

```
┌─────────────────────────────────────────────────────────────────────┐
│  Notification Utility Flow                                          │
│                                                                     │
│  highland/event/notify ───► Translate ───► notify.mobile_app_*     │
│                                                                     │
│  HA event: mobile_app_notification_action                           │
│       │                                                             │
│       ▼                                                             │
│  highland/event/notify/action_response                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Future Channels (Deferred)

| Channel | Implementation | Notes |
|---------|----------------|-------|
| `telegram` | Telegram Bot API | Two-way interaction possible |
| `signal` | Signal CLI or API | Privacy-focused |
| `tv` | HA notification to TV entity | WebOS, Android TV, etc. |
| `tts` | Text-to-speech on smart speakers | |
| `email` | SMTP integration | |

Channels can be added to the Notification Utility flow as needed. The internal payload structure remains the same; only the translation layer changes.

---

## Health Monitoring

### Overview

Two-tier monitoring: local detailed tracking via MQTT, external dead-man's-switch via Healthchecks.io (hosted). Node-RED monitors everything except itself; a watchdog script monitors Node-RED.

**Philosophy:** Treat this as a line-of-business application, not a hobbyist project. Degradation detection is as important as outage detection — you're more likely to need to remedy a filling disk than recover from a complete outage.

### Health Dimensions

| Check Type | What it measures | Failure mode |
|------------|------------------|--------------|
| **Responsiveness** | Can we talk to it? | Service down, network issue |
| **Thresholds** | Is it operating within acceptable parameters? | Resource exhaustion, degradation |

### Status Values

| Status | Meaning |
|--------|---------|
| `healthy` | Responding AND all metrics within acceptable ranges |
| `degraded` | Responding BUT one or more metrics in warning territory |
| `unhealthy` | Not responding OR critical threshold exceeded |

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Node-RED (Health Monitor Flow)                                     │
│                                                                     │
│  Monitors: MQTT, Z2M, Z-Wave JS, HA, host resources                 │
│  Publishes: highland/status/{service}/health                        │
│  Notifies: highland/event/notify (on status change)                 │
│  Pings: Healthchecks.io for each monitored service                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Watchdog Script (Communication Hub)                            │
│                                                                     │
│  Monitors: Node-RED (via heartbeat on MQTT)                         │
│  Publishes: highland/status/node_red/health                         │
│  Pings: Healthchecks.io for Node-RED                                │
│  Note: Cannot notify locally if Node-RED is down — that's the point │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Healthchecks.io (Free Tier)                                        │
│                                                                     │
│  Receives pings from Node-RED + Watchdog                            │
│  Only notifies if pings stop (local notification failed/impossible) │
└─────────────────────────────────────────────────────────────────────┘
```

### Threshold Definitions

| Metric | Warning | Critical | Notes |
|--------|---------|----------|-------|
| Disk usage | > 70% | > 90% | Applies to all hosts |
| CPU usage (sustained) | > 80% for 5 min | > 95% for 5 min | Transient spikes ignored |
| Memory usage | > 80% | > 95% | |
| Devices offline (Z2M) | Any (1+) | > 20% of total | Single device offline = degraded |
| Devices offline (Z-Wave) | Any (1+) | > 20% of total | Single device offline = degraded |

### Per-Service Checks

| Service | Responsiveness Check | Threshold Metrics |
|---------|---------------------|-------------------|
| **MQTT broker** | Publish/subscribe test | Host disk, CPU, memory |
| **Zigbee2MQTT** | HTTP API or MQTT bridge topic | Devices online/offline, host resources |
| **Z-Wave JS UI** | WebSocket or HTTP API | Nodes online/dead, host resources |
| **Home Assistant** | HTTP API (`/api/`) | DB size, host resources |
| **Node-RED** | HTTP admin API or heartbeat | Host resources |
| **Communication Hub (host)** | Implicit via services | Disk, CPU, memory |
| **Node-RED host** | Implicit via Node-RED | Disk, CPU, memory |

### Topics

```
highland/status/{service}/heartbeat   → Simple "I'm alive" ping
highland/status/{service}/health      → Detailed health + metrics
```

**Services:**

| Service | Monitored By | Topic |
|---------|--------------|-------|
| MQTT broker | Node-RED | `highland/status/mqtt/health` |
| Zigbee2MQTT | Node-RED | `highland/status/z2m/health` |
| Z-Wave JS UI | Node-RED | `highland/status/zwave/health` |
| Home Assistant | Node-RED | `highland/status/ha/health` |
| Node-RED | Watchdog script | `highland/status/node_red/health` |

### Heartbeat Payload (Simple)

Published by Node-RED to indicate it's alive (consumed by watchdog):

```json
{
  "timestamp": "2025-02-24T10:00:00Z",
  "source": "node_red"
}
```

Topic: `highland/status/node_red/heartbeat`

### Health Payload (Detailed)

```json
{
  "timestamp": "2025-02-24T10:00:00Z",
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

### Alarm Evaluation Logic

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

### Check Frequency

Check frequency is per-service based on criticality:

| Service | Frequency | Rationale |
|---------|-----------|-----------|
| MQTT broker | 1 min | Critical — everything depends on it |
| Z2M | 1 min | Critical — Zigbee devices depend on it |
| Z-Wave JS | 1 min | Critical — Z-Wave devices depend on it |
| Home Assistant | 5 min | Important but less critical (automations run without it) |
| Node-RED heartbeat | 1 min | Critical — runs all automations |

### Healthchecks.io Configuration

**Grace period:** Larger than polling period to account for latency. Suggested: 2x polling period + 30 seconds buffer.

| Check | Poll Frequency | Grace Period |
|-------|----------------|--------------|
| MQTT | 1 min | 2.5 min |
| Z2M | 1 min | 2.5 min |
| Z-Wave JS | 1 min | 2.5 min |
| HA | 5 min | 10.5 min |
| Node-RED | 1 min | 2.5 min |

### Failure Threshold

**Fail fast:** One missed check = unhealthy. No transient failure allowance initially.

Rationale: Internal checks should not fail transiently. If something fails once, it's genuinely down (or restarting). If this proves too noisy, we can add tolerance later.

### Notification Behavior

**Status change notifications:**

| Status Change | Action |
|---------------|--------|
| healthy → degraded | Notify (normal priority) |
| degraded → unhealthy | Notify (high priority) |
| healthy → unhealthy | Notify (high priority) |
| unhealthy → degraded | Notify recovery (normal priority) |
| unhealthy → healthy | Notify recovery (normal priority) |
| degraded → healthy | No notification (silent recovery) |

**External alerting:**

| Scenario | Who Notifies |
|----------|--------------|
| Z2M goes down | Node-RED (via `highland/event/notify`) |
| HA goes down | Node-RED (via `highland/event/notify`) |
| Node-RED goes down | Healthchecks.io (external alert) |
| MQTT goes down | Healthchecks.io (Node-RED can't notify without MQTT) |

### Watchdog Script (Communication Hub)

Simple script running on cron (every minute):

```
1. Check last message on highland/status/node_red/heartbeat
2. If timestamp > threshold (e.g., 90 seconds old):
   - Publish: highland/status/node_red/health { status: "unhealthy" }
   - Do NOT ping Healthchecks.io (absence of ping triggers their alert)
3. If timestamp fresh:
   - Publish: highland/status/node_red/health { status: "healthy" }
   - Ping: Healthchecks.io /ping/uuid-nodered
```

### Future Enhancement

Lightweight MQTT health listener (Python/Go) on Communication Hub:
- Subscribes to `highland/status/#`
- Tracks all service health centrally
- Handles notifications independently of Node-RED
- Services become responsible only for self-reporting heartbeats

This decouples notification logic from monitoring logic, but is not required for initial implementation.

---

## Daily Digest ("State of the Smart Home")

### Purpose

Nightly email summarizing the state of the home. Provides awareness without requiring you to go look.

### Timing

**Trigger:** Midnight (via Scheduler task event)

**Delay:** 5 seconds after midnight to ensure we're on the "new day" — so "today" in the digest refers to the correct date.

### Content Sections

| Section | Source | Content |
|---------|--------|---------|
| **Calendar** | Google Calendar API | Events for next 24-48h (recurring maintenance, one-time appointments) |
| **Weather** | Weather API | Today summary + tonight summary |
| **Battery Status** | Battery Monitor | Devices in `low` or `critical` state |
| **System Health** | Health Monitor | Any services `degraded` or `unhealthy` |

*Additional sections can be added over time (security summary, energy usage, etc.)*

### Calendar Integration

**Source:** Google Calendar (dedicated account, shared with household)

**Query frequency:** Once daily (midnight digest generation)

**Fallback:** If API unavailable, log error, generate digest without calendar section, note "Calendar unavailable" in output.

**Future:** AI assistant integration for adding/modifying events via natural language (MCP or function calling).

### Recipients

Configurable list of email recipients. Stored in config:

```json
// notifications.json
{
  "daily_digest": {
    "recipients": ["joseph@example.com"],
    "enabled": true
  }
}
```

**Use case:** During development/testing, limit recipients to avoid spamming household with test emails.

### Implementation

**Markdown → HTML approach:**
- Build digest content as markdown (easy to write/maintain)
- Convert to HTML using `node-red-node-markdown` (uses `markdown-it`)
- Send as HTML email via SMTP

**Benefits:**
- Markdown is readable in source form
- Same content could be sent to other channels (Telegram, logged, dashboard)
- No manual HTML wrangling

### Flow Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│  Daily Digest Utility Flow                                          │
│                                                                     │
│  Trigger: highland/event/scheduler/digest_daily (midnight)          │
│           + 5 second delay                                          │
│                                                                     │
│  1. Gather data:                                                    │
│     • Query Google Calendar API (next 24-48h events)                │
│     • Query Weather API (today/tonight)                             │
│     • Get battery state from global context or Battery Monitor      │
│     • Get health state from global context or Health Monitor        │
│                                                                     │
│  2. Build markdown content (template)                               │
│                                                                     │
│  3. Convert markdown → HTML (node-red-node-markdown)                │
│                                                                     │
│  4. Send email via SMTP                                             │
│                                                                     │
│  5. Log: "Daily digest sent to {recipients}"                        │
└─────────────────────────────────────────────────────────────────────┘
```

### Markdown Template Example

```markdown
# State of the Smart Home
**{{ date }}**

## 📅 Upcoming Events
{{ #each calendar_events }}
- **{{ this.summary }}** — {{ this.date }}
{{ /each }}
{{ #unless calendar_events }}
No upcoming events.
{{ /unless }}

## 🌤️ Weather
**Today:** {{ weather.today.summary }}, High {{ weather.today.high }}°
**Tonight:** {{ weather.tonight.summary }}, Low {{ weather.tonight.low }}°

## 🔋 Battery Status
{{ #if batteries_needing_attention }}
{{ #each batteries_needing_attention }}
- **{{ this.friendly_name }}** — {{ this.level }}% ({{ this.state }})
{{ /each }}
{{ else }}
All batteries healthy.
{{ /if }}

## 🏠 System Health
{{ #if health_issues }}
{{ #each health_issues }}
- **{{ this.service }}** — {{ this.status }}: {{ this.summary }}
{{ /each }}
{{ else }}
All systems operational.
{{ /if }}
```

*Template syntax is illustrative; actual implementation may use Mustache, Handlebars, or string interpolation in a function node.*

### Email Delivery

**Method:** SMTP (direct)

**Config (secrets.json):**
```json
{
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "user": "...",
    "password": "..."
  }
}
```

### Future Enhancements

- Send to additional channels (Telegram, dashboard widget)
- Weekly summary in addition to daily
- Include security event summary
- Include energy/usage stats when available

---

## Open Questions

- [x] ~~Pub/sub subflow implementation details~~ → **See Flow Registration pattern; area-level targeting, device-level ACKs**
- [x] ~~Logging persistence destination~~ → **JSONL files, daily rotation, unified log with `system` field**
- [x] ~~Mobile notification channel selection~~ → **HA Companion App (Android) for initial implementation; per-device targeting; future channels deferred**
- [x] ~~Should ERROR-level logs also auto-notify, or only CRITICAL?~~ → **CRITICAL only; escalation is flow responsibility**
- [x] ~~Device Registry storage location~~ → **External JSON file, loaded to global.config.deviceRegistry**
- [x] ~~Device Registry population~~ → **Manual with discovery validation (log/notify discrepancies, don't block)**
- [x] ~~Where to surface "devices needing batteries" data~~ → **Daily Digest email + immediate notifications for critical; dashboard deferred**
- [x] ~~ACK pattern design~~ → **Centralized ACK Tracker utility flow**
- [x] ~~Health monitoring approach~~ → **Node-RED monitors services + Watchdog monitors Node-RED + Healthchecks.io for external alerting**

---

*Last Updated: 2026-03-18*
