# Node-RED — Device Registry & Command Dispatcher

## Device Registry

### Purpose

Centralized knowledge about devices — protocol, topic structure, capabilities, and metadata. Abstracts the differences between Z2M and Z-Wave JS UI so flows don't need to know protocol details.

### Storage

Stored as `/home/nodered/config/device_registry.json`, loaded into global context at startup by the Config Loader.

**Access:** `global.get('config.deviceRegistry')`

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
| `battery` | Battery metadata (`null` if mains-powered) |

The registry key (e.g., `foyer_entry_door`) is used for internal references and ACK correlation. `friendly_name` is used for user-facing output.

### Population

Manual with validation — maintain the JSON file directly (version controlled). A discovery validation flow checks actual devices against registry, reports discrepancies via log/notify, and does not block commands.

### Discovery Validation

Periodic or on-demand check comparing device registry against actual Z2M/Z-Wave device lists:
- Devices in Z2M/Z-Wave but not in registry → log/notify (unregistered)
- Devices in registry but not in Z2M/Z-Wave → log/notify (stale or offline)
- Does **not** block commands to unregistered devices

---

## Device Catalog

Battery-powered devices share battery metadata at the model level, not the device level. The device catalog separates model-level knowledge from device-instance knowledge.

**File:** `/home/nodered/config/device_catalog.json`

```json
{
  "models": {
    "PS-S04D": {
      "battery": { "type": "CR2450", "quantity": 2 }
    },
    "some-rechargeable-model": {
      "rechargeable": true
    }
  },
  "overrides": {
    "office_desk_presence": {
      "friendly_name": "Joseph's Desk Presence"
    }
  }
}
```

**Maintenance:** Add one entry per model the first time a device of that model is paired. All subsequent devices of the same model require no config changes.

**Friendly name derivation:** If a device is not in `overrides`, the friendly name is derived automatically:

```javascript
const friendlyName = deviceKey
    .replace(/_/g, ' ')
    .replace(/\b\w/g, c => c.toUpperCase());
// "office_desk_presence" → "Office Desk Presence"
```

---

## Command Dispatcher

### Purpose

Translate high-level commands ("turn on garage_carriage_left") into protocol-specific MQTT messages. Flows say *what* they want; the dispatcher knows *how*.

### Subflow Interface

**Input:**
```json
{ "entity": "garage_carriage_left", "action": "on" }
```
```json
{ "entity": "garage_carriage_left", "action": "brightness", "value": 50 }
```
```json
{ "entity": "some_device", "action": "raw", "payload": { "custom": "data" } }
```

**Behavior:**
1. Lookup entity in Device Registry
2. Validate action against capabilities (warn on unsupported — don't block)
3. Format payload based on protocol + action
4. Publish to appropriate topic

### Common Actions (v1)

| Action | Applies To |
|--------|------------|
| `on` | lights, switches |
| `off` | lights, switches |
| `toggle` | lights, switches |
| `brightness` | dimmable lights (value: 0–255) |
| `lock` | locks |
| `unlock` | locks |
| `raw` | any — passthrough for unsupported actions |

### Protocol Translation

**Zigbee (Z2M):**
```
Topic: zigbee2mqtt/{device}/set
Payload: {"state": "ON"} / {"brightness": 255}
```

**Z-Wave (Z-Wave JS UI MQTT gateway):**
```
Topic: zwave/{node}/set
Payload: protocol-specific
```

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

### Extending Actions

| Scenario | Approach |
|----------|----------|
| New device, existing capability | Add to registry only |
| One-off command | Use `raw` passthrough |
| Repeated new capability | Add to common actions |

---

## ACK Tracker Utility Flow

### Purpose

Centralized tracking of acknowledgment requests. Flows that need confirmation of actions register their expectations; the tracker collects ACKs and reports results on timeout. Keeps ACK bookkeeping out of individual flows.

### Topics

| Topic | Purpose | Publisher |
|-------|---------|-----------|
| `highland/ack/register` | Register expectation for ACKs | Requesting flow |
| `highland/ack` | ACK responses | Responding flows |
| `highland/ack/result` | Outcome after timeout | ACK Tracker |

### Payloads

**Registration:**
```json
{
  "correlation_id": "abc123",
  "expected_sources": ["foyer_entry_door", "garage_entry_door"],
  "timeout_seconds": 30,
  "source": "security"
}
```

**ACK response:**
```json
{
  "ack_correlation_id": "abc123",
  "source": "foyer_entry_door",
  "timestamp": "..."
}
```

**Result (after timeout):**
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

### Separation of Concerns

| Component | Responsibility |
|-----------|----------------|
| **ACK Tracker** | Count ACKs, track by correlation_id, report results (raw device keys only) |
| **Requesting flow** | Register expectations, handle success/failure, decide escalation |
| **Notification flow** | Resolve friendly names from Device Registry, format user-facing messages |

The tracker doesn't know what a "lock" is. It just counts and reports. The requesting flow handles escalation by looking up friendly names from the Device Registry.

### Friendly Name Resolution

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

### Timeout-Only Failure Detection

ACK responses are positive only. If a responder fails to complete its work, it simply doesn't send an ACK. The tracker detects this as a missing response on timeout — no explicit failure ACKs required.

---

*Last Updated: 2026-03-26*
