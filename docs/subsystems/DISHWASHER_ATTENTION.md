# Dishwasher Attention State Machine

## Purpose & Scope

This document covers the **attention layer** for the dishwasher — two cooperating state machines that run above the power-based cycle detection documented in `subsystems/APPLIANCE_MONITORING.md`. Where cycle detection answers "is the dishwasher running?", the attention layer answers "what state are the dishes in, and does the dishwasher require human action?"

---

## Design Philosophy

### Explicit intent over inference

Human confirmation via button press is the primary signal. Sensors observe behavior but cannot reliably determine intent. The door being fully open is a necessary precondition for unloading — the bottom rack physically requires the door fully flat to extend — but it is not a sufficient indicator. Loading dirty dishes, unloading clean ones, and opening the door out of curiosity are all physically identical from the sensor's perspective. The button eliminates that ambiguity entirely.

Sensor data drives notifications and the guest heuristic fallback. It does not drive the primary state transition.

### Two cooperating state machines

The attention layer is modeled as two independent state machines that share a single handoff signal:

- **Content state** — what is the condition of the items in the dishwasher? Simple two-state machine. `DIRTY` is the default resting state. `CLEAN` persists until unloading is confirmed, at which point content transitions back to `DIRTY`.
- **Attention state** — is human interaction required, and what is the nature of the current interaction? Four states covering the range from "nothing needed" through "someone is actively engaged" to "likely done but unconfirmed."

The handoff: when attention state reaches `NONE`, content state transitions from `CLEAN` to `DIRTY`. The content machine is entirely passive — it just watches for that signal.

### All inputs gated by operational state

The attention state machine only accepts inputs when the operational state (from cycle detection) is `IDLE`. When the dishwasher is `RUNNING`, all inputs — button presses, tilt events, timers — are silently discarded. A button press during a cycle is not an error; it is irrelevant.

---

## Hardware

### ZEN15 Power Switch

**Z2M device name:** `kitchen_dishwasher`

Power monitoring switch, connected inline between the outlet and the dishwasher. Owned entirely by cycle detection — see `subsystems/APPLIANCE_MONITORING.md`. The attention layer consumes its events but does not interact with it directly.

### Tilt Sensor — Aqara DJT11LM (Zigbee)

**Z2M device name:** `kitchen_dishwasher_vibration`

Mounted on the side of the dishwasher door near the top. Reports angle data via Zigbee2MQTT on change — every message is a meaningful signal, silence means the door is stationary.

> **Naming note:** Z2M classifies the DJT11LM as a vibration sensor — its primary marketed capability. The Z2M device name and topic path therefore use `vibration`. We are using it exclusively for tilt/angle measurement.

**Relevant field:** `angle_x` — the axis that tracks door angle given the current mounting orientation.

**Calibrated angle ranges:**

| Range | Zone | Interpretation |
|-------|------|----------------|
| `angle_x` > -5 | Closed / vent | Door is closed (latched ~5, soft-closed ~4) or cracked slightly for venting (~1). No meaningful distinction between latched and unlatched — both are "closed" for our purposes. |
| -5 ≥ `angle_x` > -80 | Middle | Partially open. Ambiguous intent. Silent — no state transitions, no notifications. |
| `angle_x` ≤ -80 | Unload | Door is fully open (~-83 at mechanical stop). The only position from which the bottom rack can be extended. |

**Sensitivity setting:** Medium or low via Z2M. Dishwasher motor vibration and nearby appliances should not generate spurious angle reports.

### Manual Button — SONOFF SNZB-01P (Zigbee)

**Z2M device name:** `kitchen_dishwasher_button`

**Button action mapping (Z2M exposes single, double, long):**

| Action | Meaning |
|--------|---------|
| Single press | "I have unloaded the dishwasher" — primary action |
| Double press | Reserved |
| Long press | Reserved |

Mounting location TBD — counter underside and adjacent cabinetry present physical constraints due to drawer clearance. Evaluate adjacent cabinet side panel or other location that survives daily use without accidental activation.

**Acknowledgment:** Brief HA notification ("Dishwasher marked as empty") confirms the press registered. Voice acknowledgment deferred until voice pipeline is integrated.

**Idempotency:** Button press when attention is already `NONE` (and operational state is `IDLE`) is a safe no-op.

---

## Content State Machine

Simple two-state machine. Passive — only transitions in response to the attention state machine reaching `NONE`.

### States

| State | Meaning |
|-------|---------|
| `DIRTY` | Default resting state. Contains dirty dishes or is empty and ready to load. No attention needed. |
| `CLEAN` | A wash cycle has completed. Dishes are clean. Persists until unloading confirmed. |

### Transitions

```
DIRTY ──(cycle_finished event)──────────────────────────► CLEAN
CLEAN ──(attention state → NONE)────────────────────────► DIRTY
```

`CLEAN` is sticky. It does not change based on door activity, presence, or time elapsed. Only `NONE` in the attention state machine resets it.

---

## Attention State Machine

### States

| State | Meaning |
|-------|---------|
| `NONE` | No attention required. Either content is `DIRTY`, machine is `RUNNING`, or unloading has been confirmed. |
| `UNATTENDED` | Content is `CLEAN`, machine is `IDLE`, no human interaction yet confirmed. |
| `VENTING` | Content is `CLEAN`, door is in the vent zone. Human is aware; steam venting in progress. |
| `LIKELY_EMPTY` | Guest heuristic fired — door was at full unload depth for an extended continuous duration. Awaiting confirmation or timeout. |

### Transitions

All inputs silently discarded when operational state is `RUNNING`.

```
NONE
  ──(cycle_finished)────────────────────────────────────► UNATTENDED

UNATTENDED
  ──(button: single press)──────────────────────────────► NONE
  ──(angle_x settles in vent zone, > -5)────────────────► VENTING
  ──(angle_x ≤ -80, sustained ≥ guest_duration_s)───────► LIKELY_EMPTY

VENTING
  ──(door returns to closed zone, angle_x > -5)─────────► UNATTENDED
  ──(venting_timeout_s elapsed, door still in vent zone)► UNATTENDED
  ──(button: single press)──────────────────────────────► NONE

LIKELY_EMPTY
  ──(button: single press)──────────────────────────────► NONE
  ──(likely_empty_timeout_h elapsed)────────────────────► NONE
```

### Transition Logic Detail

**`UNATTENDED` → `VENTING`**

Trigger: `angle_x` settles above -5 (vent zone) after having been lower. The door was opened slightly and stopped. This is the steam venting gesture — intentional, conscious, a human deliberately cracking the door. We enter `VENTING` and start a timeout.

**`VENTING` → `UNATTENDED`**

Two paths:
1. Door returns to closed zone — human closed the door without proceeding to unload. Back to `UNATTENDED`, with contextual notification.
2. `venting_timeout_s` elapses while door is still in vent zone — human opened the door and walked away without closing it. Same destination, same contextual notification.

In both cases the notification acknowledges prior interaction: "You vented the dishwasher — don't forget to unload it." Subsequent nags reference the venting event rather than treating the situation as if no interaction occurred.

**`UNATTENDED` → `LIKELY_EMPTY`**

Trigger: `angle_x` ≤ -80 (unload zone) continuously for ≥ `guest_duration_s`. A genuine unload by someone unfamiliar with the system (guest) keeps the door fully down for several minutes. Grabbing one item does not. Cumulative open time is **not** used — a single continuous open of sufficient duration is required.

The middle zone (-5 to -80) is silent — no transition, no notification, no timer. Partial opens are ignored entirely.

**`LIKELY_EMPTY` → `NONE` (timeout)**

If `LIKELY_EMPTY` receives no button confirmation for `likely_empty_timeout_h`, it transitions to `NONE` automatically. The heuristic fired with reasonable confidence; the timeout prevents the system from waiting indefinitely.

**Button press gating**

Button is valid in `UNATTENDED`, `VENTING`, and `LIKELY_EMPTY` — anywhere a human might be confirming they have unloaded. Button is a no-op in `NONE`. Button is discarded entirely when operational state is `RUNNING`.

---

## Notification Strategy

Notifications are driven by state transitions and elapsed time. The state machine publishes events; the notification flow consumes them and decides whether to fire based on configuration.

| Trigger | Message | DnD |
|---------|---------|-----|
| `NONE` → `UNATTENDED` (cycle finished) | "Dishwasher finished — dishes are clean." | Respect |
| `UNATTENDED` nag, no prior interaction | "Dishwasher has been done for N hours." | Override after `nag_dnd_override_h` |
| `VENTING` → `UNATTENDED` | "You vented the dishwasher — don't forget to unload it." | Respect |
| `UNATTENDED` nag, post-venting | "You vented the dishwasher N minutes ago — dishes are still waiting." | Override after `nag_dnd_override_h` |
| `UNATTENDED` → `LIKELY_EMPTY` | "Looks like the dishwasher was emptied — press the button to confirm." | Respect |
| Button press → `NONE` | "Dishwasher marked as empty." | Respect |

Nag notifications are time-based events, not state transitions — they fire while sitting in `UNATTENDED` at configured intervals. They are canceled by any state transition out of `UNATTENDED`. Post-venting nags are distinct in character from cold nags and reference the prior venting interaction.

The door reaching the unload zone (`angle_x` ≤ -80) does **not** generate a notification. A human standing at a fully-open dishwasher is already aware the dishes are clean.

---

## MQTT Topics

### State Topics (Retained)

**`highland/state/appliance/dishwasher/content`** ← RETAINED

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "dishwasher_attention",
  "state": "CLEAN"
}
```

`state` values: `DIRTY` | `CLEAN`

---

**`highland/state/appliance/dishwasher/attention`** ← RETAINED

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "dishwasher_attention",
  "state": "UNATTENDED",
  "cycle_finished_at": "2026-04-03T09:30:00Z",
  "last_door_event_at": "2026-04-03T09:45:00Z",
  "last_angle_x": -2.1
}
```

`state` values: `NONE` | `UNATTENDED` | `VENTING` | `LIKELY_EMPTY`

---

### Event Topics (Not Retained)

**`highland/event/appliance/dishwasher/attention_changed`**

Published on every attention state transition.

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "dishwasher_attention",
  "previous_state": "UNATTENDED",
  "new_state": "NONE",
  "trigger": "button"
}
```

`trigger` values: `cycle_finished` | `button` | `voice` | `vent_closed` | `vent_timeout` | `guest_heuristic` | `likely_empty_timeout`

---

**`highland/event/appliance/dishwasher/content_changed`**

Published when content state transitions.

```json
{
  "timestamp": "2026-04-03T10:00:00Z",
  "source": "dishwasher_attention",
  "previous_state": "CLEAN",
  "new_state": "DIRTY",
  "trigger": "attention_none"
}
```

---

## Home Assistant Discovery

Published by `Area: Kitchen` flow on startup (retained, idempotent). Content state exposed as a plain sensor under a new logical device separate from the Z2M-managed `kitchen_dishwasher` device.

**Device:** `highland_dishwasher_attention`

| Entity | Type | State Topic | Value Template | Notes |
|--------|------|------------|----------------|-------|
| Dishwasher Content | `sensor` | `highland/state/appliance/dishwasher/content` | `{{ value_json.state }}` | `DIRTY` or `CLEAN` |
| Dishwasher Attention | `sensor` | `highland/state/appliance/dishwasher/attention` | `{{ value_json.state }}` | Optional — richer dashboard detail |

The content sensor is the primary dashboard entity. The attention sensor is optional but useful for showing the nuanced interaction state.

---

## Node-RED Flow Architecture

Tab: `Area: Kitchen`

```
[MQTT In: cycle events]
        │
        ▼
[Function: Attention State Machine]  ← single Function node, all state in flow context
        │
        ├──► [MQTT Out: highland/state/appliance/dishwasher/content]     retained
        ├──► [MQTT Out: highland/state/appliance/dishwasher/attention]   retained
        ├──► [MQTT Out: highland/event/appliance/dishwasher/attention_changed]
        ├──► [MQTT Out: highland/event/appliance/dishwasher/content_changed]
        └──► [Function: Notification Builder]
                    │
                    ▼
             [MQTT Out: highland/event/notify]

[MQTT In: zigbee2mqtt/kitchen_dishwasher_vibration]
        │
        ▼
[Function: Tilt Handler]   ← extracts angle_x, gates on operational state
        │
        ▼
[Link Out → Attention State Machine]

[MQTT In: zigbee2mqtt/kitchen_dishwasher_button]
        │
        ▼
[Function: Button Handler]  ← filters single press, gates on operational state
        │
        ▼
[Link Out → Attention State Machine]

[Inject: nag timer]
[Inject: venting timeout]
        │
        ▼
[Link Out → Attention State Machine]
```

---

## Configuration Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `vent_zone_threshold` | -5 | `angle_x` above this = vent/closed zone |
| `unload_zone_threshold` | -80 | `angle_x` at or below this = unload zone |
| `venting_timeout_s` | 900 | 15 min in vent zone before auto-return to `UNATTENDED` |
| `guest_duration_s` | 360 | Continuous time in unload zone to trigger `LIKELY_EMPTY` (6 min) |
| `likely_empty_timeout_h` | 2 | Hours before `LIKELY_EMPTY` self-resolves to `NONE` |
| `nag_first_h` | 4 | Hours in `UNATTENDED` before first nag |
| `nag_repeat_h` | 2 | Hours between subsequent nags |
| `nag_dnd_override_h` | 6 | Hours before nag overrides DnD |
| `notify_on_likely_empty_timeout` | false | Notification on auto-timeout resolution |

---

## Open Questions / Pending Actions

- **Button mounting location** — TBD. Counter underside ruled out (drawer clearance). Adjacent cabinet side panel or similar to be evaluated.
- **Venting timeout tuning** — 15 minutes is a starting estimate. Adjust after observing real venting behavior.
- **Guest duration tuning** — 6 minutes is a reasoned estimate. Review guest interactions in logs and adjust.
- **Voice integration** — "Marvin, dishwasher is empty" as additional path to `NONE`. Deferred until voice pipeline built. No state machine changes required — add a new input link to the existing Function node.
- **MQTT_TOPICS.md** — update to reflect the two-topic model (`content` + `attention`) replacing the single `attention` topic.

---

## Changelog

| Date | Change |
|------|--------|
| 2026-04-03 | Full redesign. Replaced single-state-machine approach with two cooperating state machines: content (`DIRTY`/`CLEAN`) and attention (`NONE`/`UNATTENDED`/`VENTING`/`LIKELY_EMPTY`). Introduced `VENTING` state to model steam venting gesture explicitly. Content state is sticky — only transitions on attention reaching `NONE`. All inputs gated by operational state. Tilt thresholds calibrated from live sensor: vent zone > -5, unload zone ≤ -80, silent middle zone. Hardware confirmed: ZEN15 (`kitchen_dishwasher`), SNZB-01P (`kitchen_dishwasher_button`), DJT11LM (`kitchen_dishwasher_vibration`). |
