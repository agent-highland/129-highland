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

**Event types published to the stream:**

| Event | Payload field | Values |
|-------|--------------|--------|
| `door` | `state` | `open`, `closed` |
| `door` | `current_operation` | `idle`, `opening`, `closing` |
| `light` | `state` | `on`, `off` |
| `lock` | `state` | `locked`, `unlocked` |
| `obstruction` | `state` | `on`, `off` |
| `motion` | `state` | `on`, `off` |
| `motor` | `state` | `on`, `off` |
| `openings` | `count` | integer |
| `learn` | `state` | `on`, `off` |
| `synced` | `state` | `true`, `false` |

### REST Commands

```
POST http://{blaq_ip}/button/{button_id}/press
POST http://{blaq_ip}/switch/{switch_id}/turn_on
POST http://{blaq_ip}/switch/{switch_id}/turn_off
POST http://{blaq_ip}/switch/{switch_id}/toggle
```

**Button IDs:** `door_open`, `door_close`, `door_stop`, `door_toggle`, `light_toggle`

**Switch IDs:** `remote_lock`, `learn`

---

## Node-RED Bridge Architecture

```
blaQ SSE stream  →  [Node-RED SSE subscriber]
                          │
                          ▼
                   Parse SSE event
                          │
                          ▼
              highland/state/garage/{entity}  ← RETAINED
                          │
                          ▼
              highland/event/garage/{event}   (on transitions)
                          │
                          ▼
              MQTT Discovery (on startup)     → HA entity registration
```

```
highland/command/garage/{target}
        │
        ▼
Command Handler
        │
        ▼
POST http://{blaq_ip}/...
```

### Flow Groups

**Startup** — On startup inject → Publish MQTT Discovery configs → Subscribe to SSE stream

**SSE Sink** — SSE in node → Parse Event → Publish State + Publish Event (on transition)

**Command Handler** — MQTT in (`highland/command/garage/#`) → Route Command → HTTP Request (blaQ REST API)

**Error Handling** — Flow-wide catch + targeted SSE reconnect logic

---

## MQTT Topics

### State Topics (All Retained)

| Topic | State values |
|-------|-------------|
| `highland/state/garage/door` | `OPEN` \| `CLOSED` + `current_operation`: `IDLE` \| `OPENING` \| `CLOSING` |
| `highland/state/garage/light` | `ON` \| `OFF` |
| `highland/state/garage/remote_lock` | `LOCKED` \| `UNLOCKED` |
| `highland/state/garage/obstruction` | `ON` \| `OFF` |
| `highland/state/garage/motion` | `ON` \| `OFF` |
| `highland/state/garage/motor` | `ON` \| `OFF` |
| `highland/state/garage/synced` | `true` \| `false` |
| `highland/state/garage/learn` | `ON` \| `OFF` |
| `highland/state/garage/openings` | `{ count: integer }` |

All state payloads: `{ "timestamp": "...", "source": "garage_bridge", "state": "..." }`

### Event Topics (Not Retained)

`highland/event/garage/door_opened` | `door_closed` | `obstruction_detected` | `obstruction_cleared` | `motion_detected`

All carry minimal envelope: `{ "timestamp": "...", "source": "garage_bridge" }`

### Command Topics

| Topic | `action` values |
|-------|----------------|
| `highland/command/garage/door` | `"open"` \| `"close"` \| `"stop"` \| `"toggle"` |
| `highland/command/garage/light` | `"turn_on"` \| `"turn_off"` \| `"toggle"` |
| `highland/command/garage/remote_lock` | `"lock"` \| `"unlock"` |
| `highland/command/garage/learn` | `"turn_on"` \| `"turn_off"` \| `"toggle"` |

---

## HA Entities via MQTT Discovery

Node-RED publishes MQTT Discovery configs on startup (retained, idempotent). HA auto-creates these entities and they survive HA restarts with no manual config.

| Entity | Type | Notes |
|--------|------|-------|
| Garage Door | `cover` | `open`/`close`/`stop` commands via `highland/command/garage/door` |
| Garage Light | `light` | on/off |
| Remote Lock | `lock` | locked/unlocked |
| Obstruction | `binary_sensor` | `device_class: problem` |
| Motion | `binary_sensor` | `device_class: motion` |
| Motor | `binary_sensor` | |
| Opening Count | `sensor` | Lifetime open count |
| Synced | `binary_sensor` | blaQ/opener sync state |

---

## SSE Reconnection

The SSE connection to the blaQ is long-lived. The bridge implements reconnection logic:
- On stream close or error → wait `reconnect_delay_s` (configurable, default 5s) → re-subscribe
- On startup → subscribe immediately, no delay
- Connection state tracked in `global.connections` (see `nodered/HEALTH_MONITORING.md`)
- Log WARN on disconnect; log INFO on reconnect

---

## blaQ Configuration Notes

- blaQ must be configured with a **static IP** (or DHCP reservation) — the bridge stores the IP in `device_registry.json` or `secrets.json`
- Local API must be enabled (default on blaQ)
- The blaQ's native HA ESPHome integration should be **disabled** if the bridge is active — dual control creates state conflicts

---

## Open Questions

- [ ] Confirm blaQ firmware version and whether SSE event field names are stable across updates
- [ ] Determine whether `cover` entity type in MQTT Discovery correctly supports `stop` command or requires a separate button entity

---

*Last Updated: 2026-03-26*
