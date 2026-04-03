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

**Rechargeable devices:** `rechargeable: true` in the `models` block of `device_catalog.json`. The `Build Notification` node branches on this flag — message says "plug in to charge" rather than "replace N× type".

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

**Device Reconciliation** — Two MQTT in nodes (`highland/status/config/loaded` and `highland/command/config/reload/device_registry`, both QoS 1) → `Remove Stale Devices` function (terminal — no output). Prunes `flow.battery_states` of any key not present in `global.config.device_registry.devices`, cancelling active critical timers for pruned entries.

**Device Recovery** — On Startup inject → Initializer Latch → `Resume Critical Timers` → MQTT out (notify if overdue). Startup-only, one-shot. Restores in-memory critical timers from disk-backed `flow.battery_states`.

**Error Handling** — flow-wide catch → debug

---

## Device Catalog Integration

Battery type/quantity is model-level knowledge stored in `device_catalog.json`, keyed by `model_id`. Config Loader populates `global.config.device_catalog` on startup.

`Extract Battery` looks up device and battery info at runtime:

```javascript
const registry = global.get('config.device_registry') || {};
const deviceEntry = registry.devices?.[deviceKey];
const friendlyName = deviceEntry?.friendly_name || derivedName;
const modelId = deviceEntry?.model_id;
const catalog = global.get('config.device_catalog') || {};
const batterySpec = catalog.models?.[modelId]?.battery || null;
```

`device_catalog.json` is the sole source for battery chemistry. `device_registry.json` no longer contains a `models` block.

---

## Friendly Name Derivation

Registry absence is never a processing gate. Friendly names are derived automatically if the device isn't in the registry:

```javascript
const friendlyName = deviceKey
    .replace(/_/g, ' ')
    .replace(/\b\w/g, c => c.toUpperCase());
// "office_desk_presence" → "Office Desk Presence"
```

If the device is in the registry, `deviceEntry.friendly_name` is used directly — which for Z2M devices is the Z2M friendly name (our device key convention), or the HA `name_by_user` override if set.

---

## Graceful Degradation

| Condition | Behavior |
|-----------|----------|
| Device not in registry | Process with derived friendly name; no battery spec |
| Model not in `device_catalog.json` | WARN log, process without battery spec detail |
| No battery spec | Notification omits type/quantity detail |
| `battery` field absent from payload | Message filtered in `Extract Battery`, no processing |

---

## Startup Recovery

On startup, `Resume Critical Timers` reads `flow.battery_states` (disk-backed) and restores in-memory timers for any device still in `critical` state:
- If `last_notified_critical` > 24hrs ago → notify immediately, start fresh 24hr timer
- If `last_notified_critical` < 24hrs ago → start timer for the remaining window

This guarantees no double-notification within a 24hr window and no silent drops for overdue devices. This group is deliberately separate from Device Reconciliation — `Resume Critical Timers` is not idempotent (it creates timers) and must only run once on startup.

---

## Context Storage

| Key | Store | Content |
|-----|-------|---------|
| `flow.battery_states` | default (disk) | `{ device_key: { state, level, last_reported, last_notified_critical } }` |
| `flow.battery_timers` | volatile (memory) | `{ device_key: timer_handle }` — lost on restart, recovered by `Resume Critical Timers` |

> **Note:** `flow.device_models` has been removed. Device-to-model mapping is now provided by `global.config.device_registry` via the `Utility: Device Registry` flow. The Device Discovery group that previously subscribed to `zigbee2mqtt/bridge/devices` to build this map has been eliminated.

---

*Last Updated: 2026-04-03*
