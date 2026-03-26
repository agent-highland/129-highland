# Calendar Integration — Design & Architecture

## Overview

Google Calendar serves as the household's event scheduling layer. A dedicated Node-RED Calendar Bridge flow polls the household calendar and materializes relevant events as MQTT triggers.

The primary initial consumer is the video pipeline's camera kill switch — certain calendar events should automatically suppress camera monitoring for their duration.

---

## Design Goals

- **Single integration point** — Calendar Bridge flow owns the calendar API relationship; consumers subscribe to MQTT events, never query the calendar directly
- **Zero-friction authoring** — household members use Google Calendar exactly as they do today
- **Loose coupling** — the bridge has no knowledge of consumers
- **Active event state** — retained MQTT topics allow restarting flows to immediately know what is currently active
- **Stateless re-derivation** — every poll cycle derives complete authoritative state from scratch; no incremental patching

---

## Camera Suppression: Attendee-Based Approach

### Core Concept

Each camera has a dedicated email address on the `your-domain.example` domain. To suppress a camera during a calendar event, **invite it as a guest**. That's the entire authoring workflow.

```
camera.driveway@your-domain.example
camera.rear_yard@your-domain.example
camera.rear_patio@your-domain.example
camera.front_porch@your-domain.example
```

The Calendar Bridge reads the attendee list of each event. Any event containing one or more camera addresses triggers suppression for those cameras from event start to event end.

### Why This Works

- **Self-documenting** — the guest list tells you exactly which cameras are affected
- **Native Google Calendar UX** — inviting a guest is a standard action on any platform
- **Granular by default** — invite only the specific cameras you want suppressed
- **One-time explanation** — "invite the camera to opt it out" is counterintuitive once, obvious forever
- **Event type is irrelevant** — the bridge doesn't care if it's a BBQ, repair visit, or anything else

The bridge does NOT care about event title, color, description, or type. If no camera addresses are in the guest list, the bridge ignores the event for camera purposes entirely.

### Email Notifications

Google Calendar will attempt to send invite emails to camera addresses. Create a catch-all alias on `your-domain.example` that discards silently. The bridge operates via Calendar API polling, not email.

---

## Core Operating Model

### Stateless Re-Derivation

Every poll cycle derives the complete authoritative picture from scratch. The bridge does not accumulate or patch state incrementally — it re-reads calendar reality and overwrites retained state with whatever is true right now.

- **Event ended while down** — next poll excludes it; retained state corrected
- **Event started while down** — next poll includes it; retained state reflects it
- **Overlapping events on the same camera** — bridge computes the union; no consumer-side reference counting needed
- **Deleted or modified events** — next poll corrects automatically

### Startup = Immediate Poll

On startup, the bridge:
1. Cancels all existing calendar timers (stale from before restart)
2. Immediately runs the full poll cycle
3. Re-derives state, overwrites retained topics, reschedules timers fresh

**Startup must cancel existing timers before polling** to prevent a stale pre-restart timer from firing alongside the fresh reconciliation.

### Poll Cadence

| Trigger | Cadence | Purpose |
|---------|---------|---------|
| Periodic | Every 30 minutes | Picks up events added same-day |
| On startup | Immediate | Reconciles state after any outage |
| Manual | `highland/command/calendar/reload` | On-demand for just-added events |

### Lookahead Window

**72 hours.** Weekend planning frequently happens by Thursday; a Friday addition for a Saturday event should land in the first poll after it's created. Multi-day events are handled by daily re-evaluation.

---

## Two Classes of Consumers

### Stateful Consumers

These just need to know "what is true right now." Subscribe to the retained `highland/state/calendar/camera_suppression`. On receipt, apply current state directly.

### Ceremony Consumers

These have side effects that should fire exactly once per event, not on every reconciliation. The bridge tracks whether ceremony has already fired for a given event ID (keyed in disk-backed flow context). On poll, if a start event is detected and ceremony has not been marked for that event ID, fire the start event and mark it.

**Persistent MQTT sessions required:** Ceremony consumers must use persistent MQTT sessions (`clean_session: false` with a stable client ID). This closes the race window where the bridge fires a ceremony event but the consumer is momentarily offline.

### `already_active` Flag

Start event payloads include `already_active: true` when the event was already in progress at poll time. Gives ceremony consumers a clean signal to skip ceremony and only reconcile state.

---

## Calendar Bridge Flow

### Poll Cycle

```
Trigger (startup | 30-min timer | manual reload)
        │
        ▼
Cancel all existing calendar timers
        │
        ▼
Query Google Calendar API (next 72h)
        │
        ├── For each event with camera attendees:
        │     ├── Compute currently active? (start <= now <= end)
        │     ├── If active and ceremony not fired → fire start event, mark ceremony
        │     ├── If active → include in retained state union
        │     └── If upcoming → schedule start/end timers
        │
        ├── Overwrite retained state: highland/state/calendar/camera_suppression
        │
        └── Log: "Calendar bridge: N active, M upcoming camera suppression events"
```

### Camera Address Registry

Maintained in config (alongside or separate from `device_registry.json`). Maps email address → camera entity ID:

```json
{
  "camera_guests": {
    "camera.driveway@your-domain.example": "driveway",
    "camera.rear_yard@your-domain.example": "rear_yard",
    "camera.rear_patio@your-domain.example": "rear_patio",
    "camera.front_porch@your-domain.example": "front_porch"
  }
}
```

### API Error Handling

- Log error, do **not** overwrite retained state (leave last-known-good in place)
- Retry once after 5 minutes
- If still unavailable, notify (normal priority): "Calendar bridge unavailable — retained state may be stale"

---

## MQTT Topics

| Topic | Type | Notes |
|-------|------|-------|
| `highland/state/calendar/camera_suppression` | RETAINED | Authoritative current suppression snapshot |
| `highland/event/calendar/camera_suppression/start` | NOT RETAINED | Ceremony event — fires once per calendar event |
| `highland/event/calendar/camera_suppression/end` | NOT RETAINED | Ceremony event |
| `highland/command/calendar/reload` | NOT RETAINED | Force immediate re-poll |

See `standards/MQTT_TOPICS.md` for full payload schemas.

---

## Relationship to Video Pipeline

Calendar-driven suppression complements rather than replaces the manual kill switch.

| Scenario | Mechanism |
|----------|-----------|
| Planned event (cameras invited) | Calendar bridge fires start → kill switch auto-enabled |
| Spontaneous event (not on calendar) | Manual kill switch from HA dashboard |
| Event ends | Calendar bridge fires end → kill switch auto-disabled |
| Override during calendar-driven suppression | Manual kill switch toggle always available |

---

## Future: One-Shot via AI Assistant

Once Marvin has calendar write capability, the authoring workflow collapses further:

*"Hey Marvin, add a BBQ Saturday 3 to 10pm, suppress the rear yard and driveway cameras."*

Marvin creates the event and adds the camera addresses to the guest list in one step. The Calendar Bridge Flow is unchanged — it still just reads attendee lists.

---

## Open Questions

- [ ] Camera address format confirmed as convention; verify no conflicts with other `your-domain.example` mail routing
- [ ] Ceremony state persistence eviction strategy: remove entries for events whose end time is in the past to prevent unbounded growth

---

*Last Updated: 2026-03-26*
