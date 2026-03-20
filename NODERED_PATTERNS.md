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

Node-RED context is configured with three named stores:

```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    },
    initializers: {
        module: "memory"
    },
    volatile: {
        module: "memory"
    }
}
```

**`default` (localfilesystem):** Persists to disk. Used for flow state, config cache, and any value that must survive a Node-RED restart. This is the store used when no store name is specified.

**`initializers` (memory):** In-memory only. Used exclusively for runtime utilities populated by `Utility: Initializers` at startup — functions, helpers, and other values that cannot be JSON-serialized and therefore cannot use `localfilesystem`. These are re-populated on every restart.

**`volatile` (memory):** In-memory only. Used for transient, non-serializable runtime values that must not be persisted to disk — timer handles, open connection references, or anything that would cause a circular reference error if Node-RED attempted to serialize it. Values here are intentionally lost on restart. Seeing `'volatile'` as the third argument signals that the value is transient by design.

**Usage convention:**

```javascript
// Utility: Initializers — storing a helper function
global.set('utils.formatStatus', function(text) { ... }, 'initializers');

// Any function node — retrieving it
const formatStatus = global.get('utils.formatStatus', 'initializers');

// Default store — no store name needed
global.set('config', configObject);
const config = global.get('config');

// Volatile store — timer handles, non-serializable values
flow.set('my_timer', timerHandle, 'volatile');
const timer = flow.get('my_timer', 'volatile');
```

The store name in `global.get` / `global.set` is what makes the naming self-documenting — seeing `'volatile'` tells you the value is transient, `'initializers'` tells you where it was defined.

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

### `api-call-service` Node: Dynamic Action

When building HA service calls dynamically:

- Pass the full `domain.service` string as `msg.payload.action` in the function node
- Leave the **Action** field on the `api-call-service` node **blank** — it reads `msg.payload.action` implicitly
- Set the **Data** field to JSONata `payload.data`
- Do **not** use mustache templates (e.g. `notify.{{payload.service}}`) — the node detects `service` in the expression and generates deprecation warnings

```javascript
// Correct pattern:
msg.payload = {
    action: 'notify.mobile_app_joseph_galaxy_s23',
    data: { title: 'Hello', message: 'World', data: { channel: 'highland_low' } }
};
```

### Healthchecks.io Pinging

Node-RED pings Healthchecks.io **directly via outbound HTTP** from the Health Monitor flow. This is the correct model because:

- Node-RED can make outbound HTTP calls independently of MQTT state
- Using MQTT as an intermediary (e.g. a watchdog script listening for a heartbeat topic) conflates two failure modes — if MQTT goes down but Node-RED is up, the watchdog sees silence and falsely reports Node-RED as unhealthy
- Direct HTTP from Node-RED proves Node-RED liveness independent of any other service

Each service check that Node-RED is responsible for pings its corresponding Healthchecks.io URL on success. Ping URLs are stored in `config.secrets.healthchecks_io`.

**Node-RED's Healthchecks.io ping** (proving Node-RED itself is alive) is sent on a fixed interval from the Health Monitor flow — not via any external script.

> **Watchdog script:** The original watchdog design (subscribing to a Node-RED MQTT heartbeat) is superseded by direct HTTP pinging. Whether a watchdog script has a remaining role (e.g. monitoring something Node-RED genuinely cannot monitor itself) will be determined as each service check is designed. The watchdog script in the runbook Post-Build section should be considered a placeholder pending that analysis.

---

## Flow Organization

### Flow Types

| Type | Purpose | Examples |
|------|---------|----------|
| **Area flows** | Own devices and automations for a physical area | `Garage`, `Living Room`, `Front Porch` |
| **Utility flows** | Cross-cutting concerns that transcend areas | `Scheduler`, `Security`, `Notifications`, `Logging`, `Backup`, `ACK Tracker`, `Battery Monitor`, `Health Monitor`, `Config Loader`, `Daily Digest` |

### Naming Convention

Tab names use a prefix to indicate type:
- Area tabs: `Area: Garage`, `Area: Living Room`
- Utility tabs: `Utility: Connections`, `Utility: Notifications`

Groups within a tab have descriptive names. Link nodes are named for what they carry.

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
```

**Benefits:**
- Each group is a logical unit with a clear purpose
- Link nodes connect groups without spaghetti wires
- Flow reads left-to-right in sections
- Minimizes horizontal scrolling

---

## Subflows

### Use Sparingly, For Truly Reusable Components

**Good candidates for subflows:**
- Latches — reusable startup gates
- Gates — connection-aware routing (see Connection Gate)
- Common transformations used identically across many flows

**Not good candidates:**
- Flow-specific logic (keep it visible)
- One-off utilities (just use a function node)
- Anything that hides important business logic

### Initializer Latch

Gates flow execution until `Utility: Initializers` has populated the `initializers` context store. Place at the MQTT ingress of any flow that uses utilities from the `initializers` store or reads `global.config`.

**Environment variables:**
- `RETRY_INTERVAL_MS` — delay between retries in ms (default: 250)
- `MAX_RETRIES` — maximum retry attempts (default: 20)
- `CONTEXT_PREFIX` (UI label: **Scope**) — prefix for flow context keys; required when multiple instances on the same flow tab

Total timeout at defaults: 5 seconds.

**Behavior:**
- Messages are buffered immediately on arrival
- Polls `global.get('initializers.ready', 'initializers')` until true, then drains buffer via Output 1
- On timeout: sets degraded state, discards buffer, shows red ring on subflow instance, emits via Output 2
- Output 2 is optional — the red ring is sufficient visibility for operator-introduced failures

**Degraded state cause:** Always a bug in `Utility: Initializers` introduced by a deploy. Recovery: fix Initializers → redeploy Initializers → redeploy affected flows.

---

## Startup Sequencing

### The Problem

On Node-RED startup or deploy, MQTT subscriptions deliver retained messages immediately while Initializers may not have finished populating the `initializers` store. Node-RED makes no startup ordering guarantees.

### Solution

Place an `Initializer Latch` at the MQTT ingress of every flow that:
- Subscribes to retained state topics, or
- Uses utilities from the `initializers` store (`utils.formatStatus`, etc.), or
- Reads `global.config`

The latch buffers messages until initializers are ready, then drains them in order.

### Bootstrapping Limitation

You cannot use infrastructure to report infrastructure failures. If MQTT is unavailable:
- `node.error()` / `node.warn()` write to Node-RED's internal log — visible via `docker compose logs nodered`
- Node status (red ring) is visible in the editor regardless of MQTT state
- Healthchecks.io receives pings via direct HTTP independently of MQTT

---

## Error Handling

1. **Targeted handlers** — Catch errors in specific groups where custom handling is needed
2. **Flow-wide catch-all** — Single Error node per flow, dispatches to `highland/event/log`

---

## Logging Framework

### Concept

Logging answers: *"How important is this for troubleshooting/audit?"* Separate from notifications, though CRITICAL logs auto-forward to `highland/event/notify`.

### Log Storage

**Format:** JSONL — one JSON object per line
**Location:** `/var/log/highland/highland-YYYY-MM-DD.jsonl`
**Rotation:** Daily via cron, retain 30 days

### Log Entry Structure

| Field | Purpose |
|-------|---------|
| `timestamp` | ISO 8601 |
| `system` | `node_red`, `ha`, `z2m`, `zwave_js` |
| `source` | Component within system |
| `level` | `VERBOSE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | Human-readable description |
| `context` | Structured additional data |

### Log Levels

| Level | When to Use |
|-------|-------------|
| `VERBOSE` | Granular trace; active debugging only |
| `DEBUG` | Detailed troubleshooting info |
| `INFO` | Normal operational events |
| `WARN` | Unexpected but not broken |
| `ERROR` | Something failed but flow continues |
| `CRITICAL` | Catastrophic failure; intervention needed |

**CRITICAL only** auto-notifies. Escalation is the flow's responsibility.

### MQTT/Console Fallback Pattern

Every flow with a logging path uses this pattern to handle MQTT unavailability:

```
Log Event link in → MQTT Available? switch (global.connections.mqtt == 'up')
    ↓ up                          ↓ else
Format Log Message → MQTT out    Log to Console (node.error/warn)
```

Log messages are set on `msg.log_level`, `msg.log_message`, and `msg.log_context` by the emitting node before reaching this group.

---

## Configuration Management

### File Structure

```
/home/nodered/config/
├── device_registry.json        ← git: yes
├── notifications.json          ← git: yes
├── thresholds.json             ← git: yes
├── healthchecks.json           ← git: yes
├── secrets.json                ← git: NO
└── README.md                   ← git: yes
```

### Example: notifications.json

```json
{
  "people": {
    "joseph": {
      "admin": true,
      "channels": {
        "ha_companion": "notify.mobile_app_joseph_phone"
      }
    },
    "spouse": {
      "admin": false,
      "channels": {
        "ha_companion": "notify.mobile_app_spouse_phone"
      }
    }
  },
  "daily_digest": {
    "recipients": ["joseph@example.com"],
    "enabled": true
  },
  "defaults": {
    "admin_only": ["joseph"],
    "all": ["joseph", "spouse"]
  }
}
```

**Person-centric model:** Each person has an `admin` flag and a `channels` map. Adding a new channel means adding an entry to each person's `channels` object.

**`defaults`** — Named recipient lists. Callers copy these values into the payload `recipients` field explicitly — no implicit defaulting.

### Config Loader

Loads all config files into `global.config` at startup and on `highland/command/config/reload`.

---

## Notification Framework

### Concept

Notifications answer: *"How urgently does a human need to know about this?"*

### Notification Payload

```json
{
  "timestamp": "2025-02-24T14:30:00Z",
  "source": "security",
  "channels": ["ha_companion"],
  "recipients": ["joseph", "spouse"],
  "severity": "high",
  "title": "Lock Failed to Engage",
  "message": "Front Door Lock did not respond within 30 seconds",
  "sticky": true,
  "group": "security_alerts",
  "correlation_id": "lockdown_20250224_2200"
}
```

### Required Fields

| Field | Notes |
|-------|-------|
| `channels` | Non-empty array. No defaulting. |
| `recipients` | Non-empty array of person names. No defaulting. |
| `severity` | `low`, `medium`, `high`, `critical` |
| `title` | Short summary |
| `message` | Full detail |

### Channel Selection Philosophy

**Both `channels` and `recipients` are required** — every notification represents a deliberate design-time decision about who receives it and how.

**Multi-channel is intent, not failover.** `["ha_companion", "pushover"]` means deliver via both. `["ha_companion"]` means HA only — the Connection Gate handles availability.

**Resiliency is the caller's responsibility.** If a notification requires guaranteed delivery, the caller specifies multiple channels. The Notification Utility delivers what it can via the channels specified — it does not retry or compensate for a channel being unavailable. A caller that needs resilience against HA being down should include a secondary channel; a caller that specifies only `ha_companion` has made a conscious decision that delivery is best-effort.

**Graceful degradation within a channel.** Each channel adapter extracts what it supports and silently ignores the rest.

**Missing address → WARN log, skip, continue.** Deliver as much as possible.

### Severity → HA Companion Mapping

| Severity | Channel | DND Override | Persistent |
|----------|---------|--------------|------------|
| `low` | `highland_low` | No | No |
| `medium` | `highland_default` | No | No |
| `high` | `highland_high` | Yes | No (unless `sticky`) |
| `critical` | `highland_critical` | Yes | Yes |

### Clearing Notifications

Publish to `highland/command/notify/clear`:

```json
{
  "correlation_id": "lockdown_20250224_2200",
  "recipients": ["joseph"]
}
```

`correlation_id` must match the original delivery. `tag` ≠ `correlation_id`.

---

## Utility: Connections

### Purpose

Tracks live connection state and exposes it via global context. Distinct from Health Checks — Connections exposes state *inward* to flows; Health Checks reports state *outward* to Healthchecks.io.

### Detection Mechanism

`status` node scoped to a connection-bearing node. Fires immediately on state change — no polling, no extra dependencies.

**Signal mapping:** `'red'` or `'yellow'` → `'down'`; anything else → `'up'`.

### Startup Settling

Connections briefly drop on restart. The flow debounces `'down'` transitions during a configurable settling window:

- `Startup Tasks` → `Establish Cadence` sets `flow.timer_cadence` and starts a `setTimeout` setting `flow.settled = true` (volatile store) after the cadence
- During window: `'down'` starts a debounce timer; `'up'` cancels it silently
- After window: all transitions logged immediately

**Single cadence value** — `flow.timer_cadence` drives both the settling window and debounce timers.

### MQTT Catch-22

When MQTT is down, normal log path is unavailable. `State Change Logging` routes to `Log to Console` when `connections.mqtt !== 'up'`.

### Global Context Keys

| Key | Values |
|-----|--------|
| `connections.home_assistant` | `'up'` / `'down'` |
| `connections.mqtt` | `'up'` / `'down'` |

### Usage

```javascript
const haAvailable = global.get('connections.home_assistant') !== 'down';
const mqttAvailable = global.get('connections.mqtt') !== 'down';
```

`!== 'down'` handles startup case — `undefined !== 'down'` defaults to available.

---

## Connection Gate Subflow

### Purpose

Guards message flow based on live connection state. Handles repeated up/down transitions — distinct from Initializer Latch which is a one-time startup concern.

### Interface

- **1 input** — any message
- **Output 1 (Pass)** — connection up; message delivered immediately or after recovery
- **Output 2 (Fallback)** — connection down and unrecovered; caller handles or discards

### Environment Variables

| Variable | UI Label | Default |
|----------|----------|---------|
| `CONNECTION_TYPE` | Connection | — (`home_assistant` or `mqtt`) |
| `RETENTION_MS` | Retention (ms) | `0` (immediate Output 2) |
| `CONTEXT_PREFIX` | Scope | `''` |

### Behavior

| Scenario | Result |
|----------|--------|
| Connection up | Output 1 immediately |
| Down, `RETENTION_MS` = 0 | Output 2 immediately |
| Down, recovers within window | Output 1 on recovery |
| Down, window expires | Output 2 |

Output 1 = connected (immediately or recovered). Output 2 = unrecovered. No silent drops.

**Latest-only** — new message while polling cancels existing poll, starts fresh.
**Poll interval** internalized at 500ms.
**`RETENTION_MS = 0` is the expected default for notification delivery** — non-zero values are for specific use cases where brief hold-and-retry is appropriate.

### Internal Structure

**Evaluate Gate** — function node (2 outputs), sets `node.status()` for each path.
**Status Monitor** — `status` node scoped to Evaluate Gate → `Set Status` → subflow status output.

### Node Status Values

| Status | Meaning |
|--------|---------|
| Green dot — Passed | Immediate Output 1 |
| Red dot — Fallback | Immediate Output 2 |
| Yellow ring — Waiting... | Polling for recovery |
| Green ring — Recovered | Output 1 after recovery |
| Red ring — Expired | Output 2 after window |

---

## Utility: Notifications

### Purpose

Centralized delivery of notifications to people via configured channels. All notification traffic enters via MQTT — no other flow calls HA notify services directly.

### Topics

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `highland/event/notify` | Inbound | Deliver a notification |
| `highland/command/notify/clear` | Inbound | Dismiss a delivered notification |
| `highland/event/log` | Outbound | Log delivery outcomes |

### Groups

**Receive Notification** — MQTT in (`highland/event/notify`) → Initializer Latch → Validate Payload → Build Targets → `link call` (Deliver, dynamic) → Log Event link out

**HA Companion Delivery** — Link In (`Home Assistant Companion`) → Connection Gate → Build Service Call → HA service call node → `link out` (return mode)

**Clear Notification** — MQTT in (`highland/command/notify/clear`) → Initializer Latch → Build Clear Call → `link call` (Deliver, dynamic) → Log Event link out

**State Change Logging** — Log Event link in → MQTT Available? switch → Format Log Message → MQTT out / Log to Console

**Test Cases** — Persistent sanity tests; intentionally not removed.

### Initializer Latch Scopes

| Group | `CONTEXT_PREFIX` |
|-------|-----------------|
| Receive Notification | `notify_in-` |
| Clear Notification | `notify_clear-` |

### Validate Payload

Required: `channels`, `recipients`, `severity`, `title`, `message`. Both arrays must be non-empty. Invalid → WARN log, drop.

### Build Targets (Fan Out)

Iterates `channels × recipients`. Looks up `global.config.notifications.people.{person}.channels.{channel}`. For each valid combination, sends one message with `msg.payload._delivery` and `msg.target` set:

```javascript
node.send({
    payload: {
        ...msg.payload,
        _delivery: { channel, recipient, address }
    },
    target: resolveLinkTarget(channel)
});
```

`resolveLinkTarget()` maps channel names to their `Link In` node names:

```javascript
function resolveLinkTarget(channel) {
    switch (channel) {
        case 'ha_companion': return 'Home Assistant Companion';
        default: throw new Error(`Unable to resolve channel: ${channel}`);
    }
}
```

Adding a new channel: add a case here and a new delivery group with a matching `Link In` name. Missing recipient or address → WARN log, skip, continue.

### `link call` Node (Deliver)

Reads `msg.target` dynamically and routes to the matching `Link In` node name. Set to **dynamic** link type, 30 second timeout. Output wires to Log Event link out — logging happens once on the return path after delivery completes. Timeouts handled by a catch node scoped to the `link call` — logs WARN and moves on.

### Connection Gate (HA Companion Delivery)

`CONNECTION_TYPE = home_assistant`, `CONTEXT_PREFIX = ha-`, `RETENTION_MS = 0`. Output 2 unwired — if HA is down the message drops. Resiliency is the caller's responsibility via channel selection.

### Build Service Call

Handles both delivery and clear paths, branched on `_delivery.type`:

```javascript
// Clear path
if (_delivery.type === 'clear') {
    msg.payload = {
        action: _delivery.address,
        data: { message: 'clear_notification', data: { tag: msg.payload.correlation_id } }
    };
    msg.log_message = `Notification cleared for ${_delivery.recipient} via ${_delivery.channel}`;
    return msg;
}

// Delivery path
msg.payload = { action: _delivery.address, data: { title, message, data } };
msg.log_message = `Notification delivered to ${_delivery.recipient} via ${_delivery.channel}`;
return msg;
```

**Severity → HA Companion mapping:**

| Severity | Channel | Importance | Persistent |
|----------|---------|------------|------------|
| `low` | `highland_low` | `low` | No |
| `medium` | `highland_default` | `default` | No |
| `high` | `highland_high` | `high` | No (unless `sticky: true`) |
| `critical` | `highland_critical` | `high` | Yes |

**`api-call-service` node:** Action field blank (reads `msg.payload.action` implicitly); Data field = JSONata `payload.data`.

### Build Clear Call

Mirrors Build Targets — iterates recipients, resolves `ha_companion` address, sets `_delivery.type: 'clear'`, and sends via `link call`:

```javascript
node.send({
    payload: {
        _delivery: { channel: 'ha_companion', recipient, address, type: 'clear' },
        correlation_id
    },
    target: 'Home Assistant Companion'
});
```

`correlation_id` must match the original delivery. `tag` ≠ `correlation_id`.

**Clear payload (MQTT):**
```json
{
  "correlation_id": "lockdown_20250224_2200",
  "recipients": ["joseph"]
}
```

### HA Companion Delivery — Return Path

The last node in the group is a `link out` set to **return** mode. This returns the message to whichever `link call` dispatched it (Receive Notification or Clear Notification), completing the call/return cycle and triggering downstream logging.

### State Change Logging

Same MQTT/console fallback pattern as `Utility: Connections`. `Build Service Call` sets `msg.log_message` on both delivery and clear paths before returning to the caller. `Format Log Message` reads `msg.log_level` (default `INFO`), `msg.log_message`, and `msg.log_context`.

### Notes

- `Utility: Notifications` is the only flow that calls HA notify services
- All HA delivery and clear traffic flows through the same HA Companion group — the group is channel-specific, not operation-specific
- Adding a new channel: add a case to `resolveLinkTarget()`, build a new delivery group with a matching `Link In` name, wire the return `link out`
- Action responses deferred until actionable notifications are implemented
- Test Cases group preserved for sanity testing

---

## Health Monitoring

### Philosophy

Treat this as a line-of-business application. Each service self-reports its own liveness independently of Node-RED.

**Healthchecks.io naming:**
- `{Service}` — service's own self-report
- `Node-RED / {Service} Edge` — Node-RED's connection check
- `Home Assistant / {Service} Edge` — HA's connection check

**Current implementation status:**
- `Node-RED` ✅ | `Home Assistant` ✅ | `Communications Hub` ✅
- `Node-RED / Home Assistant Edge` ✅ | `Node-RED / MQTT Edge` ✅
- `Node-RED / Zigbee Edge` ✅ | `Node-RED / Z-Wave Edge` ✅
- `Home Assistant / Zigbee Edge` ✅ | `Home Assistant / Z-Wave Edge` ✅

**Check frequency:** All checks: 1 minute period, 3 minute grace.

---

## Daily Digest

**Trigger:** Midnight + 5 second delay
**Content:** Calendar (next 24–48h), weather, battery status, system health
**Implementation:** Markdown → HTML → SMTP email

---

## Flow Registration

Each area flow self-registers at startup:

```javascript
const flowIdentity = { area: 'foyer', devices: ['foyer_entry_door'] };
flow.set('identity', flowIdentity);
const registry = global.get('flowRegistry') || {};
registry[flowIdentity.area] = { devices: flowIdentity.devices };
global.set('flowRegistry', registry);
```

---

## ACK Tracker

Centralized ACK tracking for flows that need confirmation of actions.

| Topic | Purpose |
|-------|---------|
| `highland/ack/register` | Register expectation |
| `highland/ack` | ACK response |
| `highland/ack/result` | Outcome after timeout |

---

## Open Questions

- [x] ~~Pub/sub subflow implementation details~~ → **Flow Registration pattern**
- [x] ~~Logging persistence destination~~ → **JSONL files, daily rotation**
- [x] ~~Mobile notification channel selection~~ → **HA Companion primary; explicit multi-channel via `channels` field; no implicit failover**
- [x] ~~Should ERROR-level logs also auto-notify?~~ → **CRITICAL only**
- [x] ~~Device Registry storage~~ → **External JSON, global.config.deviceRegistry**
- [x] ~~ACK pattern design~~ → **Centralized ACK Tracker**
- [x] ~~Health monitoring approach~~ → **Each service self-reports + Node-RED edge checks + HA edge checks + Healthchecks.io**
- [x] ~~Startup sequencing / race conditions~~ → **Initializer Latch subflow at MQTT ingress**
- [x] ~~HA connection state detection~~ → **`status` node pattern; `connections.home_assistant` and `connections.mqtt`; startup settling window**
- [x] ~~Notification routing when HA is down~~ → **No implicit failover; sender specifies channels; resiliency is caller's responsibility**
- [x] ~~Connection-aware message routing~~ → **Connection Gate subflow; RETENTION_MS=0 default for notifications**
- [x] ~~Notification recipient/channel model~~ → **Person-centric config; `channels` and `recipients` required; graceful degradation**
- [x] ~~Utility: Notifications~~ → **Built and tested; HA Companion delivery, Connection Gate, person lookup, clear path**
- [x] ~~Fan-out routing pattern~~ → **`link call` with dynamic `msg.target`; `resolveLinkTarget()` maps channel keys to `Link In` node names; delivery groups return via `link out` (return mode); catch node handles timeouts**
- [ ] **Action responses** — deferred until actionable notifications are implemented
- [ ] **Utility: Scheduler** — period transitions and task events

---

*Last Updated: 2026-03-20*
