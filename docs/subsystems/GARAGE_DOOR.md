# Garage Door — Integration Design

## Overview

The Konnected GDO blaQ is integrated via a Node-RED bridge — not the HA native ESPHome integration. The bridge subscribes to the blaQ's local SSE stream for push state updates and issues HTTP POSTs to the blaQ REST API for commands. HA never speaks to the device directly.

**Why a bridge instead of native ESPHome:** The ESPHome integration in HA works, but it routes control through HA, which violates Highland's resiliency principle. The Node-RED bridge gives safety-critical garage door state and control to Node-RED directly — it survives HA restarts independently.

---

## blaQ API Surface

### SSE Stream (Push State — Primary)

```
GET http://{blaq_ip}/events
Content-Type: text/event-stream
```

The blaQ pushes state change events in real time. Node-RED subscribes once at startup and maintains the connection. Reconnection logic handles drops.

**Event types published to the stream (confirmed):**

The SSE stream publishes full ESPHome entity state for every entity on the device as JSON. Each message has the shape:
```json
{"name_id": "cover/Garage Door", "id": "cover-garage_door", "value": 0, "state": "CLOSED", "current_operation": "IDLE", "position": 0}
```

Key confirmed `id` values and their relevant fields:

| `id` | Key fields |
|------|------------|
| `cover-garage_door` | `state`, `current_operation`, `position` |
| `light-garage_light` | `state`, `color_mode` |
| `lock-lock` | `state` |
| `binary_sensor-obstruction` | `state` |
| `binary_sensor-motor` | `state` |
| `sensor-garage_openings` | `value` (count) |

All other entity messages are filtered and dropped by Parse Event.

### REST Commands

All commands are HTTP POST with no request body unless otherwise noted.

**Button IDs:** N/A — door commands use cover endpoints directly (see below)

**Cover endpoints:**
```
POST http://{blaq_ip}/cover/garage_door/open
POST http://{blaq_ip}/cover/garage_door/close
POST http://{blaq_ip}/cover/garage_door/stop
POST http://{blaq_ip}/cover/garage_door/toggle
POST http://{blaq_ip}/cover/garage_door/set?position=0.5   (decimal 0.0–1.0)
```

**Light endpoints:**
```
POST http://{blaq_ip}/light/garage_light/turn_on
POST http://{blaq_ip}/light/garage_light/turn_off
POST http://{blaq_ip}/light/garage_light/toggle
```

**Lock endpoints:**
```
POST http://{blaq_ip}/lock/lock/lock
POST http://{blaq_ip}/lock/lock/unlock
```

**Switch IDs:** `learn` only
```
POST http://{blaq_ip}/switch/learn/turn_on
POST http://{blaq_ip}/switch/learn/turn_off
POST http://{blaq_ip}/switch/learn/toggle
```

---

## Node-RED Bridge Architecture

```
blaQ SSE stream  →  [Node-RED SSE subscriber]
                          │
                          ▼
                   Parse SSE event
                          │
                          ▼
              highland/state/garage/bay_one/{entity}  ← RETAINED
                          │
                          ▼
              highland/event/garage/bay_one/{event}   (on transitions)
                          │
                          ▼
              MQTT Discovery (on startup)     → HA entity registration
```

```
highland/command/garage/bay_one/{target}
        │
        ▼
Command Handler
        │
        ▼
POST http://{blaq_ip}/...
```

### Flow Groups

**Wall Remote** — Two independent streams for ZEN37 bay one buttons:
- `Bay One Cover`: MQTT in (scene/001) → `Filter Single Tap` → `Smart Reversing` → MQTT out
- `Bay One Light`: MQTT in (scene/003) → `Filter Single Tap` → `Toggle Light` → MQTT out

**Command Handler** — MQTT in (`highland/command/garage/bay_one/#`) → `Route Command` function → HTTP Request → `Log Response` function

**Sinks** — On Startup inject → SSE client (`node-red-contrib-sse-client`) → `Parse Event` function → MQTT out (state topics, retained)

**Home Assistant Discovery** — On Startup inject → `Build Sensors` function (emits all six Discovery configs) → MQTT out (retained)

**Error Handling** — Flow-wide catch → debug

---

## MQTT Topics

### State Topics (All Retained)

| Topic | State values |
|-------|-------------|
| `highland/state/garage/bay_one/door` | `OPEN` \| `CLOSED` + `current_operation`: `IDLE` \| `OPENING` \| `CLOSING` + `position`: 0.0–1.0 (decimal) |
| `highland/state/garage/bay_one/light` | `ON` \| `OFF` + `color_mode`: `onoff` |
| `highland/state/garage/bay_one/remote_lock` | `LOCKED` \| `UNLOCKED` |
| `highland/state/garage/bay_one/obstruction` | `ON` \| `OFF` |
| `highland/state/garage/bay_one/motion` | `ON` \| `OFF` |
| `highland/state/garage/bay_one/motor` | `ON` \| `OFF` |
| `highland/state/garage/bay_one/synced` | `true` \| `false` |
| `highland/state/garage/bay_one/learn` | `ON` \| `OFF` |
| `highland/state/garage/bay_one/openings` | `{ count: integer }` |

All state payloads: `{ "timestamp": "...", "source": "garage_bridge", "state": "..." }`

> **Confirmed from blaQ SSE stream:** `position` is a decimal 0.0–1.0 (not 0–100). `state` remains `OPEN` during both opening and closing — direction is determined by `current_operation`. Mid-travel detection: `position > 0 && position < 1`.

**Confirmed cover entity states:**

| `state` | `current_operation` | `position` | Meaning |
|---------|-------------------|-----------|----------|
| `CLOSED` | `IDLE` | `0` | Fully closed |
| `OPEN` | `IDLE` | `1` | Fully open |
| `OPEN` | `OPENING` | `0–1` | Mid-travel, opening |
| `OPEN` | `CLOSING` | `0–1` | Mid-travel, closing |
| `OPEN` | `IDLE` | `0–1` | Mid-travel, stopped |

**SSE filter key:** `id === 'cover-garage_door'`

### Event Topics (Not Retained)

`highland/event/garage/bay_one/door_opened` | `door_closed` | `obstruction_detected` | `obstruction_cleared` | `motion_detected`

All carry minimal envelope: `{ "timestamp": "...", "source": "garage_bridge" }`

### Command Topics

Door, lock, and remote_lock commands use `{"action": "..."}` payload format. Light commands use HA MQTT JSON schema format `{"state": "ON"}` / `{"state": "OFF"}`.

| Topic | Payload format |
|-------|----------------|
| `highland/command/garage/bay_one/door` | `{"action": "open" \| "close" \| "stop" \| "toggle" \| "set_position", "position": 0.0–1.0}` |
| `highland/command/garage/bay_one/light` | `{"state": "ON" \| "OFF"}` (HA MQTT JSON schema) |
| `highland/command/garage/bay_one/remote_lock` | `{"action": "lock" \| "unlock"}` |

---

## HA Entities via MQTT Discovery

Node-RED publishes MQTT Discovery configs on startup (retained, idempotent). HA auto-creates these entities and they survive HA restarts with no manual config.

| Entity | Type | HA Entity ID | Notes |
|--------|------|-------------|-------|
| Garage Door | `cover` | `cover.garage_bay_one` | `open`/`close`/`stop` commands |
| Garage Light | `light` | `light.garage_bay_one` | JSON schema, on/off + color_mode |
| Remote Lock | `lock` | `lock.garage_bay_one` | locked/unlocked |
| Obstruction | `binary_sensor` | `binary_sensor.garage_bay_one_obstruction` | `device_class: problem` |
| Motor | `binary_sensor` | `binary_sensor.garage_bay_one_motor` | |
| Opening Count | `sensor` | `sensor.garage_bay_one_openings` | Lifetime open count |

> **Note:** Motion and Synced entities were omitted from the Discovery config — Motion requires the optional motion-sensing wall button accessory; Synced is an internal diagnostic state not useful for normal operation.

---

### ZEN37 Wall Remote Topics

| Button | Z-Wave JS UI Topic | Function |
|--------|-------------------|----------|
| Large button 1 | `zwave/garage/garage_wall_remote/central_scene/endpoint_0/scene/001` | Bay one door — Smart Reversing |
| Small button 1 | `zwave/garage/garage_wall_remote/central_scene/endpoint_0/scene/003` | Bay one light — toggle |
| Large button 2 | Reserved | Bay two door (future) |
| Small button 2 | Reserved | Bay two light (future) |

Payload filter: only `value === 0` (single tap) is processed. Double taps, holds, and releases are dropped by `Filter Single Tap`.

```json
{"time": 1234567890, "value": 0, "nodeName": "garage_wall_remote", "nodeLocation": "garage"}
```

The Zooz ZEN37 is a 4-button Z-Wave wall remote (2 large, 2 small buttons) mounted in the garage. It provides a physical toggle for the garage door that requires intelligent direction-awareness — a simple toggle command to the blaQ would be ambiguous mid-travel.

### Button Mapping

| Button | Action | Function |
|--------|--------|----------|
| Large button 1 | Press | Bay 1 door — Smart Reversing toggle |
| Small button 1 | Press | Bay 1 light — toggle |
| Large button 2 | Press | Bay 2 door — reserved (future) |
| Small button 2 | Press | Bay 2 light — reserved (future) |

### State Machine

The bridge tracks `last_direction` in flow context:

| Value | Meaning |
|-------|---------|
| `OPENING` | Door was last observed moving upward |
| `CLOSING` | Door was last observed moving downward |
| `UNKNOWN` | Default on startup before first SSE update |

`last_direction` is updated on every SSE event where `current_operation` transitions to `OPENING` or `CLOSING`. It updates continuously through reversals — no special reset needed.

**Decision logic on ZEN37 bay 1 large button press:**

| Door state | `current_operation` | `last_direction` | Action |
|------------|-------------------|-----------------|--------|
| `CLOSED` | `IDLE` | any | `open` |
| `OPEN` | `IDLE` | any | `close` |
| any | `OPENING` or `CLOSING` | any | `stop` |
| mid-travel | `IDLE` | `OPENING` | `close` |
| mid-travel | `IDLE` | `CLOSING` | `open` |
| mid-travel | `IDLE` | `UNKNOWN` | `close` ← fail-safe |

**Fail-safe rationale:** When `last_direction` is unknown (flow just restarted and door is already mid-travel), the bridge defaults to closing. An open or partially-open door is the higher-risk state; closing is the safer assumption.

**Mid-travel detection:** `position > 0 && position < 1` (position is a decimal 0.0–1.0 confirmed from blaQ SSE stream).

### Command Path Distinction

- **ZEN37** → Smart Reversing logic → blaQ REST API (toggle semantics, direction-aware)
- **HA UI / Node-RED command** → `highland/command/garage/bay_one/door` → blaQ REST API directly (explicit directional commands, no reversing logic needed)

---

## SSE Reconnection

The SSE connection to the blaQ is long-lived. The bridge implements reconnection logic:
- On stream close or error → wait `reconnect_delay_s` (configurable, default 5s) → re-subscribe
- On startup → subscribe immediately, no delay
- Connection state tracked in `global.connections` (see `nodered/HEALTH_MONITORING.md`)
- Log WARN on disconnect; log INFO on reconnect

---

## blaQ Configuration Notes

- blaQ IP is hardcoded in the `Route Command` function node — externalise to config when a suitable home for non-Zigbee/Z-Wave device IPs is established
- Local API must be enabled (default on blaQ)
- The blaQ's native HA ESPHome integration must be **disabled** when the bridge is active — dual control creates state conflicts

---

## Open Questions

- [x] Confirm blaQ firmware version and whether SSE event field names are stable across updates — SSE stream confirmed, field names stable on current firmware
- [x] Determine whether `cover` entity type in MQTT Discovery correctly supports `stop` command — confirmed working
- [x] Confirm Z-Wave JS UI topic format for ZEN37 scene/button events — confirmed: `zwave/garage/garage_wall_remote/central_scene/endpoint_0/scene/00N`, `value === 0` for single tap
- [ ] Externalise blaQ IP address when config home for non-Zigbee/Z-Wave devices is established

---

*Last Updated: 2026-04-16*
