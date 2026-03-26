# Node-RED — Utility: Battery Monitor

## Purpose

Tracks battery levels across all Zigbee devices, detects state transitions, notifies appropriately, and persists state across restarts.

---

## Battery States

| State | Threshold | Notification |
|-------|-----------|-------------|
| `normal` | > 35% | None (or silent recovery from critical) |
| `low` | 15–35% | Once, medium priority |
| `critical` | < 15% | Immediately, high priority; repeats every 24hrs until recovered |

**Threshold rationale:** Some devices (e.g., Sonoff) report in 10% increments. These thresholds provide 7 levels of normal, 2 levels of low, 1 level of critical for coarse reporters.

**Rechargeable devices:** `rechargeable: true` in the model catalog entry. The `Build Notification` node branches on this flag — message says "plug in to charge" rather than "replace N× type".

---

## Notification Behavior

| Transition | Action |
|------------|--------|
| normal → low | Notify once, medium priority |
| low → critical | Notify immediately, high priority; start 24hr repeat timer |
| normal → critical | Notify immediately, high priority; start 24hr repeat timer |
| critical → low | Cancel repeat; notify recovery (medium priority) |
| critical → normal | Cancel repeat; notify recovery (medium priority) |
| low → normal | No notification (silent recovery) |

**Hysteresis:** If a battery level bounces back above a threshold, the device automatically recovers to the appropriate state. No manual intervention required.

---

## MQTT Topics

**Published events (not retained):**

| Topic | Trigger |
|-------|---------|
| `highland/event/battery/low` | Device crossed into low state |
| `highland/event/battery/critical` | Device crossed into critical state |
| `highland/event/battery/recovered` | Device returned to normal or low from critical |

**Published state (retained):**

`highland/state/battery/states` — Full battery state map for all tracked devices. Published on every state transition and unconditionally on startup.

See `standards/MQTT_TOPICS.md` for full payload schemas.

---

## Flow Groups

**Sinks** — `zigbee2mqtt/#` MQTT in → Initializer Latch → `Extract Battery` → Link Out

**Battery State Pipeline** — Link In → `Evaluate State` (Output 1: state changed; Output 2: no change/drop) → `Build Event` → MQTT out (battery event) + `Build Notification` → MQTT out (notify)

**Device Discovery** — `zigbee2mqtt/bridge/devices` MQTT in → `Build Device Map` (populates `flow.device_models`)

**Device Recovery** — On Startup inject → `Recover Critical State` → MQTT out (notify if overdue)

**Error Handling** — flow-wide catch → debug

---

## Device Catalog Pattern

Battery type/quantity is model-level knowledge, not device-instance knowledge. Stored in `device_catalog.json`:

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

**Maintenance:** Add one entry per model the first time a device of that model is paired.

---

## Friendly Name Derivation

Registry absence is never a processing gate. Friendly names are derived automatically:

```javascript
const friendlyName = deviceKey
    .replace(/_/g, ' ')
    .replace(/\b\w/g, c => c.toUpperCase());
// "office_desk_presence" → "Office Desk Presence"
```

`overrides` in `device_catalog.json` provide explicit overrides when the derived name isn't sufficient.

---

## Build Device Map

Subscribes to `zigbee2mqtt/bridge/devices` (retained). Fires on startup and on every Z2M restart. Builds `flow.device_models` map: `{ device_key → model_id }`. No latch needed — pure data population, no `global.config` dependency.

---

## Graceful Degradation

| Condition | Behavior |
|-----------|----------|
| Device not in `device_models` | Process with no model info; no battery spec |
| Model not in catalog | WARN log, process without battery spec detail |
| No battery spec | Notification omits type/quantity detail |
| `battery` field absent from payload | Message filtered in `Extract Battery`, no processing |

---

## Startup Recovery

On startup, `Recover Critical State` reads `flow.battery_states` (disk-backed) and resumes timers for any device still in `critical` state:
- If `last_notified_critical` > 24hrs ago → notify immediately, start fresh 24hr timer
- If `last_notified_critical` < 24hrs ago → start timer for the remaining window

This guarantees no double-notification within a 24hr window and no silent drops for overdue devices.

---

## Context Storage

| Key | Store | Content |
|-----|-------|---------|
| `flow.battery_states` | default (disk) | `{ device_key: { state, level, last_notified_critical } }` |
| `flow.device_models` | default (disk) | `{ device_key: model_id }` — rebuilt from Z2M on startup |
| `flow.battery_timers` | volatile (memory) | `{ device_key: timer_handle }` — lost on restart, recovered by `Recover Critical State` |

---

*Last Updated: 2026-03-26*
