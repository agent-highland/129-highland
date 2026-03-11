# MQTT Topics — Authoritative Reference

## Purpose & Scope

This document is the authoritative registry of all `highland/` MQTT topics. It defines what exists, who owns it, whether it's retained, and what its payload looks like.

**Relationship to other docs:**
- **EVENT_ARCHITECTURE.md** — philosophy, patterns, design rationale. Read that first.
- **NODERED_PATTERNS.md** — flow implementation patterns, including how flows consume topics.
- **This document** — the reference. What topics actually exist, settled and locked.

Where this document conflicts with EVENT_ARCHITECTURE.md, **this document wins**. EVENT_ARCHITECTURE.md will be reconciled in a future pass.

---

## Namespace Summary

| Namespace | Purpose | Retained? |
|-----------|---------|-----------|
| `highland/event/` | Point-in-time facts. Something happened. | No (except where noted) |
| `highland/state/` | Current operational truth. What is true right now. | **Always** |
| `highland/status/` | Service health and liveness. Infra concerns only. | No (heartbeats); Yes (health snapshots) |
| `highland/command/` | Imperative instructions to a service. | No |
| `highland/ack/` | Acknowledgment infrastructure. | No |

### Key Distinctions

**`highland/event/` vs `highland/state/`**

An event is something that *happened* — it fires and it's gone. Flows that miss it miss it. State is *what is currently true* — retained, always available, immediately meaningful to a restarting flow.

- Precipitation started → `event/` (it happened at a moment)
- Current synthesized weather conditions → `state/` (it's just true right now)
- Scheduler period transitioned to evening → `event/` (the transition happened)
- What period the house is currently in → `state/` (the house is in this period)

**`highland/state/` vs `highland/status/`**

State is operational — automations read it to make decisions. Status is infrastructure health — monitoring flows and dashboards read it. Don't put device sensor data under `status/`. Don't put heartbeats under `state/`.

---

## `highland/state/` Contract

This namespace is the primary integration surface between Node-RED and Home Assistant. Its rules are strict:

1. **Always retained** — that's the point. A restarting consumer gets current truth immediately.
2. **Always JSON** — structured objects, never raw scalars. HA uses `value_template` to extract fields.
3. **Single writer** — one flow owns each topic. No two flows publish to the same state topic.
4. **Source-neutral** — payload never indicates where the data came from. Consumers don't know or care whether a value originated from a physical sensor, a third-party API, or a calculated derivation.
5. **Atomic** — publish the full object on every update. Never partial patches.

### How Home Assistant Consumes `highland/state/`

Node-RED uses **MQTT Discovery** to register sensor entities with HA. HA auto-creates entities and tracks them — zero manual HA config required.

**Discovery topic pattern:**
```
homeassistant/{component}/{node_id}/{object_id}/config
```

- `{component}` — HA component type: `sensor`, `binary_sensor`, `switch`, etc.
- `{node_id}` — logical device grouping (groups related entities into one HA device)
- `{object_id}` — unique entity identifier within that device

**Example — outdoor temperature from weather conditions:**

Discovery config (published by Node-RED on startup, retained):
```
Topic: homeassistant/sensor/highland_weather/outdoor_temperature/config
```
```json
{
  "name": "Outdoor Temperature",
  "unique_id": "highland_weather_outdoor_temperature",
  "state_topic": "highland/state/weather/conditions",
  "value_template": "{{ value_json.temperature }}",
  "device_class": "temperature",
  "unit_of_measurement": "°F",
  "state_class": "measurement",
  "device": {
    "identifiers": ["highland_weather"],
    "name": "Highland Weather",
    "model": "Synthesized",
    "manufacturer": "Highland"
  }
}
```

State topic (the actual value, retained, updated on each synthesis cycle):
```
Topic: highland/state/weather/conditions
```
```json
{
  "temperature": 72.3,
  "humidity": 64,
  "wind_speed": 8.2,
  ...
}
```

HA creates a `sensor.outdoor_temperature` entity, watches `highland/state/weather/conditions`, and applies `value_template` to get `72.3`. It knows this is a temperature in °F because the discovery config says so. When HA restarts, it re-reads the retained discovery config and retained state — it wakes up fully current.

**Key discovery config fields:**

| Field | Purpose |
|-------|---------|
| `device_class` | Semantic type (`temperature`, `humidity`, `pressure`, `wind_speed`, `precipitation`, `motion`, etc.) — drives icon, graph style, voice assistant behavior |
| `unit_of_measurement` | Scale (`°F`, `%`, `inHg`, `mph`, `in`, etc.) — HA can do unit conversion if the user prefers different units |
| `state_class` | History behavior: `measurement` (current value), `total_increasing` (accumulators like daily precip) |
| `value_template` | Jinja2 expression to extract scalar from JSON payload |

**Discovery is idempotent.** Publishing the same config repeatedly (e.g., on every Node-RED startup) is safe — HA ignores duplicate registrations with the same `unique_id`. Re-publish freely.

**Where discovery configs live:** A dedicated Discovery Registration flow (or the Config Loader flow) publishes all discovery configs on startup. Sensor semantic definitions live there as data — not scattered across flows.

---

## Standard Payload Envelope

All `highland/event/` and `highland/state/` payloads include:

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "{flow_name}",
  ...domain-specific fields...
}
```

`highland/state/` payloads additionally always include all fields (full object on every publish — no partial updates).

---

## Topic Registry

### Scheduler

**Purpose:** Defines the house's daily rhythm and fires time-based task triggers.

---

#### Period State

**`highland/state/scheduler/period`** ← RETAINED

Current period the house is in. Updated when period transitions.

| | |
|--|--|
| **Publisher** | Scheduler flow |
| **Consumers** | Any flow with time-of-day behavior |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T17:35:00Z",
  "source": "scheduler",
  "period": "evening"
}
```

`period` values: `"day"` | `"evening"` | `"overnight"`

---

#### Period Transition Events

Point-in-time triggers — fire at the moment of transition. Not retained (use state topic to query current period).

**`highland/event/scheduler/day`**
**`highland/event/scheduler/evening`**
**`highland/event/scheduler/overnight`**

| | |
|--|--|
| **Publisher** | Scheduler flow (schedex) |
| **Consumers** | Flows that need to react at the moment of transition |
| **Retained** | No |

```json
{
  "timestamp": "2026-03-09T17:35:00Z",
  "source": "scheduler"
}
```

> **Note:** EVENT_ARCHITECTURE.md marks period events as retained. That is superseded here. The retained state lives at `highland/state/scheduler/period`. The event topics are transition triggers only.

---

#### Task Events

Bespoke point-in-time triggers for specific scheduled jobs.

**`highland/event/scheduler/midnight`**
Daily boundary trigger. Fires at 00:00:00.

> **Renamed from:** `digest_daily` (previously in EVENT_ARCHITECTURE.md). Publishers don't name events after their consumers. Both the Daily Digest flow and the LoRaWAN mailbox flow subscribe to this event.

**`highland/event/scheduler/backup_daily`**
Triggers the backup orchestration flow.

| | |
|--|--|
| **Publisher** | Scheduler flow |
| **Consumers** | Single-purpose (named in topic) |
| **Retained** | No |

All task events carry the minimal payload:
```json
{
  "timestamp": "2026-03-09T00:00:00Z",
  "source": "scheduler"
}
```

---

### Weather

**Architecture:** The Weather flow is a black box. All data sources — Tempest weather station (webhook), Pirate Weather API (polled), NWS/NOAA alerts, and any future sources — are private ingestion paths. Source attribution never appears in published topics. The rest of the system sees a single curated weather service.

**Internal-only (never on the bus):**
- Tempest raw webhook payloads
- Pirate Weather API responses
- NWS/NOAA raw data
- Polling state machine state (`POLL_DORMANT`, `POLL_ACTIVE`, etc.)
- Per-source intermediate calculations

---

#### State Topics (Retained)

**`highland/state/weather/conditions`** ← RETAINED

Synthesized current conditions. Combines Tempest observations, model data, and derived calculations into a single authoritative snapshot. Updated on each synthesis cycle.

| | |
|--|--|
| **Publisher** | Weather flow |
| **Consumers** | HA (via MQTT Discovery), any flow needing current conditions |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "weather",
  "temperature": 72.3,
  "feels_like": 70.1,
  "humidity": 64,
  "dew_point": 58.2,
  "pressure": 29.92,
  "wind_speed": 8.2,
  "wind_gust": 14.1,
  "wind_bearing": 225,
  "uv_index": 4,
  "solar_radiation": 312,
  "visibility": 10.0,
  "cloud_cover": 0.35,
  "precipitation_active": false,
  "precipitation_type": "none",
  "precipitation_rate": 0.0,
  "lightning_last_strike_distance": null,
  "lightning_last_strike_at": null
}
```

*Field list is illustrative; finalize against Tempest API and Pirate Weather v2 field set during implementation. All `-999` sentinel values from Pirate Weather must be null-coerced before inclusion.*

---

**`highland/state/weather/forecast`** ← RETAINED

Current forecast summary. Updated on each Pirate Weather poll cycle.

| | |
|--|--|
| **Publisher** | Weather flow |
| **Consumers** | HA (via MQTT Discovery), Daily Digest flow |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "weather",
  "today": {
    "summary": "Partly cloudy with a chance of afternoon showers",
    "high": 76,
    "low": 58,
    "precipitation_probability": 0.45,
    "precipitation_type": "rain",
    "precipitation_accumulation": 0.15
  },
  "tonight": {
    "summary": "Clearing skies",
    "low": 52,
    "precipitation_probability": 0.10
  },
  "alerts": []
}
```

---

**`highland/state/weather/precipitation`** ← RETAINED

Current precipitation event state. Distinct from conditions — this is the active event lifecycle, not point-in-time intensity readings.

| | |
|--|--|
| **Publisher** | Weather flow |
| **Consumers** | HA (via MQTT Discovery), automation flows |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "weather",
  "state": "active",
  "type": "rain",
  "intensity": 0.08,
  "accumulation_today": 0.22,
  "event_started_at": "2026-03-09T13:45:00Z",
  "event_duration_minutes": 45
}
```

`state` values: `"none"` | `"imminent"` | `"active"` | `"tapering"` | `"done"`

---

#### Event Topics (Not Retained)

**`highland/event/weather/precipitation_start`**

```json
{
  "timestamp": "2026-03-09T13:45:00Z",
  "source": "weather",
  "type": "rain",
  "intensity": 0.04
}
```

---

**`highland/event/weather/precipitation_end`**

```json
{
  "timestamp": "2026-03-09T15:30:00Z",
  "source": "weather",
  "type": "rain",
  "duration_minutes": 105,
  "accumulation": 0.31
}
```

---

**`highland/event/weather/precipitation_type_change`**

Fires when precipitation transitions between types during an active event (e.g., rain → freezing rain).

```json
{
  "timestamp": "2026-03-09T14:00:00Z",
  "source": "weather",
  "previous_type": "rain",
  "new_type": "ice"
}
```

---

**`highland/event/weather/lightning_detected`**

```json
{
  "timestamp": "2026-03-09T16:22:00Z",
  "source": "weather",
  "distance_miles": 4.2,
  "energy": 12500
}
```

---

**`highland/event/weather/wind_gust`**

Fires when a wind gust crosses a configured threshold. Threshold defined in `thresholds.json`.

```json
{
  "timestamp": "2026-03-09T14:15:00Z",
  "source": "weather",
  "gust_speed": 38.2,
  "threshold_crossed": 35.0
}
```

---

**`highland/event/weather/alert`**

NWS / NOAA advisory or warning.

```json
{
  "timestamp": "2026-03-09T12:00:00Z",
  "source": "weather",
  "alert_id": "NWS-IDP-PROD-...",
  "title": "Winter Storm Warning",
  "severity": "Extreme",
  "urgency": "Expected",
  "description": "...",
  "effective": "2026-03-09T18:00:00Z",
  "expires": "2026-03-10T12:00:00Z"
}
```

> **Removed:** `highland/event/weather/period_change` (previously in WEATHER_FLOW.md). Period transitions are owned by Scheduler, not Weather. The Weather flow *subscribes* to `highland/event/scheduler/+` to adjust behavior; it does not republish period events.

---

### Calendar

**`highland/state/calendar/camera_suppression`** ← RETAINED

Authoritative snapshot of currently active camera suppression events. Updated on every Calendar Bridge poll cycle (complete re-derivation — not incremental).

| | |
|--|--|
| **Publisher** | Calendar Bridge flow |
| **Consumers** | Camera pipeline flow (stateful consumer — reads retained state on startup) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T16:05:00Z",
  "source": "calendar_bridge",
  "active": [
    {
      "event_id": "google_event_id",
      "title": "Backyard BBQ",
      "start": "2026-03-09T16:00:00Z",
      "end": "2026-03-09T22:00:00Z",
      "cameras": ["rear_yard", "rear_patio"]
    }
  ]
}
```

`active` is an empty array (not absent) when no events are active.

---

**`highland/event/calendar/camera_suppression/start`**

Ceremony event — fires once per calendar event when suppression begins. Requires persistent MQTT sessions on consuming flows.

```json
{
  "timestamp": "2026-03-09T16:00:00Z",
  "source": "calendar_bridge",
  "event_id": "google_event_id",
  "title": "Backyard BBQ",
  "start": "2026-03-09T16:00:00Z",
  "end": "2026-03-09T22:00:00Z",
  "cameras": ["rear_yard", "rear_patio"],
  "already_active": false
}
```

---

**`highland/event/calendar/camera_suppression/end`**

```json
{
  "timestamp": "2026-03-09T22:00:00Z",
  "source": "calendar_bridge",
  "event_id": "google_event_id",
  "title": "Backyard BBQ",
  "cameras": ["rear_yard", "rear_patio"]
}
```

---

### Driveway Bins

**Architecture:** Two independent bin sensors (trash, recycling) tracked via LoRaWAN EM320-TILT tilt sensors. Zone detection uses RSSI/SNR from gateway uplink metadata — no additional hardware. State machines are independent; it is normal for one bin to be `AWAY` while the other is `HOME`. Source-level detail (raw RSSI values, SNR) is included in state payloads as diagnostic data but automations react to `state` field only.

---

**`highland/state/driveway/trash_bin`** ← RETAINED
**`highland/state/driveway/recycling_bin`** ← RETAINED

Current state of each bin. Updated on every tilt sensor uplink.

| | |
|--|--|
| **Publisher** | LoRaWAN Bin Monitor flow |
| **Consumers** | Notification flow, HA (via MQTT Discovery), Daily Digest |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T07:45:00Z",
  "source": "lora_bin_monitor",
  "state": "AWAY",
  "rssi": -108,
  "snr": -4.2,
  "battery_pct": 94
}
```

`state` values: `"HOME"` | `"AWAY_SETTLING"` | `"AWAY"` | `"PICKED_UP"` | `"RETURNED"`

---

**`highland/event/driveway/trash_bin/zone_changed`**
**`highland/event/driveway/recycling_bin/zone_changed`**

Fires when bin transitions between HOME and AWAY zones (after debounce).

```json
{
  "timestamp": "2026-03-09T07:45:00Z",
  "source": "lora_bin_monitor",
  "previous_state": "HOME",
  "new_state": "AWAY"
}
```

---

**`highland/event/driveway/trash_bin/picked_up`**
**`highland/event/driveway/recycling_bin/picked_up`**

Fires when X-axis rotation ≥ 85° threshold is crossed while bin is `AWAY`. Indicates truck has serviced the bin.

```json
{
  "timestamp": "2026-03-09T07:45:00Z",
  "source": "lora_bin_monitor"
}
```

---

**`highland/event/driveway/trash_bin/returned`**
**`highland/event/driveway/recycling_bin/returned`**

Fires when bin transitions from `PICKED_UP` or `AWAY` back to `HOME`.

```json
{
  "timestamp": "2026-03-09T16:30:00Z",
  "source": "lora_bin_monitor",
  "previous_state": "PICKED_UP"
}
```

---

### Mailbox

**Architecture:** LoRaWAN door/contact sensor (Milesight EM300-MCS) on the mailbox door, combined with USPS Informed Delivery email parsing. State machine driven by two independent signals: physical door events and email confirmation. Neither signal alone is sufficient — the state machine correlates both. Midnight calendar boundary (`highland/event/scheduler/midnight`) drives the `DELIVERY_EXCEPTION` check.

---

**`highland/state/mailbox/delivery`** ← RETAINED

Current mailbox delivery state machine state.

| | |
|--|--|
| **Publisher** | LoRaWAN Mailbox flow |
| **Consumers** | Notification flow, HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T14:22:00Z",
  "source": "lora_mailbox",
  "state": "MAIL_WAITING",
  "last_door_event": "2026-03-09T14:20:00Z",
  "advisory_received_at": "2026-03-09T06:03:00Z",
  "confirmation_received_at": "2026-03-09T14:21:00Z"
}
```

`state` values: `"IDLE"` | `"UNCLASSIFIED"` | `"ADVISORY_RECEIVED"` | `"MAIL_WAITING"` | `"DELIVERY_EXCEPTION"` | `"RETRIEVED"`

---

**`highland/event/mailbox/door_activity`**

Any open/close event from the mailbox door sensor. Unclassified at time of publication — the state machine determines significance.

```json
{
  "timestamp": "2026-03-09T14:20:00Z",
  "source": "lora_mailbox",
  "previous_state": "ADVISORY_RECEIVED",
  "new_state": "ADVISORY_RECEIVED"
}
```

---

**`highland/event/mailbox/mail_expected`**

Morning USPS Informed Delivery advisory processed. Delivery expected today.

```json
{
  "timestamp": "2026-03-09T06:03:00Z",
  "source": "lora_mailbox",
  "previous_state": "IDLE",
  "new_state": "ADVISORY_RECEIVED"
}
```

---

**`highland/event/mailbox/mail_delivered`**

Delivery confirmed — email-resolved door event correlated within lag window.

```json
{
  "timestamp": "2026-03-09T14:22:00Z",
  "source": "lora_mailbox",
  "previous_state": "ADVISORY_RECEIVED",
  "new_state": "MAIL_WAITING"
}
```

---

**`highland/event/mailbox/mail_retrieved`**

Door event while state is `MAIL_WAITING` — someone collected the mail.

```json
{
  "timestamp": "2026-03-09T17:45:00Z",
  "source": "lora_mailbox",
  "previous_state": "MAIL_WAITING",
  "new_state": "RETRIEVED"
}
```

---

**`highland/event/mailbox/delivery_exception`**

Advisory was received but no delivery confirmation arrived by midnight. Non-terminal — resolves if confirmation arrives later.

```json
{
  "timestamp": "2026-03-09T00:00:00Z",
  "source": "lora_mailbox",
  "previous_state": "ADVISORY_RECEIVED",
  "new_state": "DELIVERY_EXCEPTION",
  "advisory_received_at": "2026-03-09T06:03:00Z"
}
```

---

### Garage Door

**Architecture:** The Konnected GDO blaQ is integrated via a Node-RED bridge — not the HA native ESPHome integration. The bridge subscribes to the Blaq's local SSE stream (`GET /events`) for push state updates and issues HTTP POSTs to the Blaq REST API for commands. HA never speaks to the device directly. See **GARAGE_DOOR.md** for full integration design.

**Internal-only (never on the bus):**
- Raw Blaq REST API responses
- SSE raw event payloads
- SSE connection state machine (`CONNECTING`, `CONNECTED`, `RECONNECTING`)
- HTTP response codes from command calls

---

**`highland/state/garage/door`** ← RETAINED

Current state of the garage door cover.

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery), any flow with garage-aware behavior |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "CLOSED",
  "current_operation": "IDLE"
}
```

`state` values: `"OPEN"` | `"CLOSED"`
`current_operation` values: `"IDLE"` | `"OPENING"` | `"CLOSING"`

HA MQTT cover uses `value_template` on this topic. During travel, `current_operation` drives the OPENING/CLOSING display; `state` reflects the settled position.

---

**`highland/state/garage/light`** ← RETAINED

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "OFF"
}
```

`state` values: `"ON"` | `"OFF"`

---

**`highland/state/garage/remote_lock`** ← RETAINED

State of the Security+ remote control lockout feature. When locked, all physical remotes and keypads are disabled at the opener. This is not a door lock.

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "UNLOCKED"
}
```

`state` values: `"LOCKED"` | `"UNLOCKED"`

---

**`highland/state/garage/obstruction`** ← RETAINED

State of the safety beam sensor. `ON` means the beam is interrupted (obstruction present).

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery), any flow managing door close sequences |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "OFF"
}
```

`state` values: `"ON"` (obstructed) | `"OFF"` (clear)

---

**`highland/state/garage/motion`** ← RETAINED

Motion sensor state. Present only if a compatible motion-sensing LiftMaster wall button is installed on the opener. Entity is registered unconditionally — will remain unavailable if hardware is absent.

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "OFF"
}
```

`state` values: `"ON"` | `"OFF"`

---

**`highland/state/garage/motor`** ← RETAINED

Whether the opener's drive motor is actively running.

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "OFF"
}
```

`state` values: `"ON"` (running) | `"OFF"` (idle)

---

**`highland/state/garage/synced`** ← RETAINED

Whether the Blaq has successfully completed the Security+ protocol handshake with the opener motor. `OFF` indicates a communication problem between the Blaq and the opener.

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "ON"
}
```

`state` values: `"ON"` (synced) | `"OFF"` (not synced)

---

**`highland/state/garage/openings`** ← RETAINED

Cumulative lifetime open/close cycle count reported by the opener motor. Useful for maintenance tracking.

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "count": 1247
}
```

---

**`highland/state/garage/learn`** ← RETAINED

State of the "learn mode" on the opener (used for pairing physical remotes).

| | |
|--|--|
| **Publisher** | Garage Bridge flow |
| **Consumers** | HA (via MQTT Discovery) |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge",
  "state": "OFF"
}
```

`state` values: `"ON"` | `"OFF"`

---

#### Garage Door Events (Not Retained)

**`highland/event/garage/door_opened`**

Fires when the door reaches the fully open state (`current_operation` transitions to `IDLE` with `state: OPEN`).

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge"
}
```

---

**`highland/event/garage/door_closed`**

Fires when the door reaches the fully closed state.

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge"
}
```

---

**`highland/event/garage/obstruction_detected`**

Fires on rising edge of the obstruction sensor.

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge"
}
```

---

**`highland/event/garage/obstruction_cleared`**

Fires when the obstruction sensor returns to clear.

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge"
}
```

---

**`highland/event/garage/motion_detected`**

Fires on rising edge of the motion sensor (if present).

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "garage_bridge"
}
```

---

#### Garage Door Commands

**`highland/command/garage/door`**

Published by HA MQTT cover entity via `command_template`. Node-RED translates to the appropriate Blaq REST endpoint.

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "ha_ui",
  "action": "open"
}
```

`action` values: `"open"` | `"close"` | `"stop"` | `"toggle"`

---

**`highland/command/garage/light`**

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "ha_ui",
  "action": "turn_on"
}
```

`action` values: `"turn_on"` | `"turn_off"` | `"toggle"`

---

**`highland/command/garage/remote_lock`**

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "ha_ui",
  "action": "lock"
}
```

`action` values: `"lock"` | `"unlock"`

---

**`highland/command/garage/learn`**

```json
{
  "timestamp": "2026-03-11T09:15:00Z",
  "source": "ha_ui",
  "action": "turn_on"
}
```

`action` values: `"turn_on"` | `"turn_off"` | `"toggle"`

---

### Security

**`highland/state/security/mode`** ← RETAINED

Current security posture of the house.

| | |
|--|--|
| **Publisher** | Security flow |
| **Consumers** | Any flow with security-aware behavior; HA |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T22:00:00Z",
  "source": "security",
  "mode": "home"
}
```

`mode` values: `"home"` | `"away"` | `"lockdown"`

---

**`highland/event/security/lockdown`**

Lockdown initiated. Carries ACK request for lock confirmation.

```json
{
  "timestamp": "2026-03-09T22:00:00Z",
  "source": "security",
  "message_id": "lockdown_20260309_2200",
  "request_ack": true,
  "recipients": ["foyer", "garage"]
}
```

---

**`highland/event/security/away`**

House set to away mode.

```json
{
  "timestamp": "2026-03-09T09:00:00Z",
  "source": "security"
}
```

---

### Logging

**`highland/event/log`**

All log entries from all systems. Consumed by Logging Utility flow, written to JSONL.

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "system": "node_red",
  "source": "weather",
  "level": "ERROR",
  "message": "Pirate Weather API timeout",
  "context": {
    "attempt": 3,
    "error": "ETIMEDOUT"
  }
}
```

---

### Notifications

**`highland/event/notify`**

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "security",
  "severity": "high",
  "title": "Lock Failed to Engage",
  "message": "Front Door Lock did not respond within 30 seconds",
  "recipients": ["mobile_joseph", "mobile_spouse"],
  "dnd_override": true,
  "actionable": true,
  "actions": [
    { "id": "retry", "label": "Retry Lock" },
    { "id": "dismiss", "label": "Dismiss" }
  ],
  "sticky": true,
  "group": "security_alerts",
  "correlation_id": "lockdown_20260309_2200"
}
```

**`highland/event/notify/action_response`**

User tapped a notification action — normalized from HA Companion App event.

```json
{
  "timestamp": "2026-03-09T14:32:00Z",
  "source": "notification",
  "action": "retry",
  "correlation_id": "lockdown_20260309_2200",
  "device": "mobile_joseph"
}
```

---

### Battery

**`highland/event/battery/low`**
**`highland/event/battery/critical`**
**`highland/event/battery/recovered`**

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
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

---

### Backup

**`highland/event/backup/completed`**
**`highland/event/backup/failed`**

```json
{
  "timestamp": "2026-03-09T03:15:00Z",
  "source": "hub",
  "host": "hub",
  "file": "hub_backup_20260309_031500.tar.gz"
}
```

---

### Status / Health

**`highland/status/{service}/heartbeat`**

Simple liveness ping. Published by the monitored service itself on a regular interval.

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "node_red"
}
```

---

**`highland/status/{service}/health`** ← RETAINED

Detailed health snapshot with threshold metrics.

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "service": "z2m",
  "status": "degraded",
  "checks": {
    "responsive": true,
    "thresholds": {
      "disk_percent": { "value": 45, "status": "ok" },
      "devices_offline": { "value": 2, "status": "warning" }
    }
  },
  "summary": "2 devices offline"
}
```

`status` values: `"healthy"` | `"degraded"` | `"unhealthy"`

**Monitored services:** `mqtt`, `z2m`, `zwave`, `ha`, `node_red`

---

### Commands

**`highland/command/backup/trigger`**
**`highland/command/backup/trigger/{host}`**

**`highland/command/config/reload`**
**`highland/command/config/reload/{config_name}`**

**`highland/command/calendar/reload`**

All command payloads carry minimal envelope:
```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "scheduler"
}
```

---

### ACK Infrastructure

**`highland/ack/register`** — Register ACK expectation (flow → ACK Tracker)

```json
{
  "correlation_id": "lockdown_20260309_2200",
  "expected_sources": ["foyer_entry_door", "garage_entry_door"],
  "timeout_seconds": 30,
  "source": "security"
}
```

**`highland/ack`** — ACK response (area flow → ACK Tracker)

```json
{
  "ack_correlation_id": "lockdown_20260309_2200",
  "source": "foyer_entry_door",
  "timestamp": "2026-03-09T22:00:05Z"
}
```

**`highland/ack/result`** — Outcome after timeout (ACK Tracker → requesting flow)

```json
{
  "correlation_id": "lockdown_20260309_2200",
  "expected": 2,
  "received": 1,
  "sources": ["foyer_entry_door"],
  "missing": ["garage_entry_door"],
  "success": false
}
```

---

## Retention Summary

| Topic Pattern | Retained |
|---------------|----------|
| `highland/state/#` | **Always** |
| `highland/status/+/health` | Yes |
| `highland/status/+/heartbeat` | No |
| `highland/event/#` | **No** (except see note below) |
| `highland/command/#` | No |
| `highland/ack/#` | No |

> **Scheduler period events** (`highland/event/scheduler/day|evening|overnight`) were previously documented as retained in EVENT_ARCHITECTURE.md. That behavior is superseded: these events are **not retained**. Current period is available at `highland/state/scheduler/period` (retained). Flows should subscribe to the state topic on startup, not rely on a retained event.

---

## Wildcard Subscription Patterns

| Pattern | Use Case |
|---------|----------|
| `highland/state/#` | All retained operational state |
| `highland/state/weather/#` | All weather state |
| `highland/state/driveway/#` | Both bin states |
| `highland/state/garage/#` | All garage state (door, light, lock, sensors) |
| `highland/event/scheduler/#` | All scheduler events (periods + tasks) |
| `highland/event/driveway/#` | All bin events (both bins, all transition types) |
| `highland/event/driveway/trash_bin/#` | All trash bin events only |
| `highland/event/driveway/recycling_bin/#` | All recycling bin events only |
| `highland/event/mailbox/#` | All mailbox events |
| `highland/event/garage/#` | All garage events |
| `highland/event/+/leak/#` | Any leak in any area |
| `highland/event/+/motion_detected` | Any motion in any area |
| `highland/status/#` | All health and heartbeat |
| `highland/status/+/health` | Health snapshots only |
| `highland/command/backup/#` | All backup commands |
| `highland/command/garage/#` | All garage commands |

---

## Domains Pending Definition

Topics not yet designed. Will be added as each domain is designed.

| Domain | Notes |
|--------|-------|
| **Presence / Occupancy** | FP300 sensors; `highland/state/{area}/occupancy` likely pattern |
| **Video Pipeline** | Kill switch, detection events, triage results |
| **HA Assist / Voice** | Marvin persona events; may not need bus presence |
| **Area sensors** | Zigbee environmental, motion, contact — `highland/state/{area}/environment` |
| **Lighting state** | TBD if lighting state needs bus representation beyond Z2M raw topics |
| **Entry Door Locks** | Z-Wave/Zigbee door locks; state beyond what Z2M/ZWaveJS already exposes. Distinct from garage remote lockout (which is defined above). |

---

## Changelog

| Date | Change |
|------|--------|
| 2026-03-11 | Added Garage Door section (Konnected Blaq bridge). Added garage wildcard patterns. Clarified Locks pending domain as entry door locks, distinct from garage remote lockout. |
| 2026-03-10 | Added Driveway Bins and Mailbox sections from LORA.md. Removed both from Domains Pending. Added driveway and mailbox wildcard patterns. |
| 2026-03-09 | Initial creation. Established `highland/state/` namespace and MQTT Discovery pattern. Corrected: scheduler period events are no longer retained (state lives at `highland/state/scheduler/period`). Renamed `digest_daily` → `midnight`. Removed `weather/period_change`. Moved calendar suppression state from `status/` to `state/`. |

---

*Last Updated: 2026-03-11*
