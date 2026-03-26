# Event Architecture

## Overview

Event-driven architecture for inter-flow communication using MQTT as the message transport. Publishers emit facts; consumers decide relevance and action.

---

## Core Principles

1. **Publishers emit facts, not commands**
   - ✓ "It's 30 minutes before sunset" (`evening`)
   - ✗ "Turn on exterior lights"

2. **Consumers decide relevance** — each flow subscribes to events it cares about and decides what action to take

3. **Loose coupling** — publishers have zero knowledge of subscribers; flows can be added or removed without changing publishers

4. **Async acknowledgment when needed** — critical flows can request confirmation via ACK events with a `correlation_id`

5. **Publishers don't name events after consumers** — `midnight` not `digest_daily`; the event describes when it fires, not who subscribes

---

## Flow Types

### Area Flows

Named after physical areas: `Garage`, `Living Room`, `Front Yard`. Area flows own their devices, subscribe to events, decide local actions, and publish area-level semantic events (e.g., `presence_detected`).

### Utility Flows

Functional concepts that transcend areas: `Scheduler`, `Security`, `Notifications`, `Logging`. Utility flows orchestrate cross-cutting concerns and publish global events.

---

## MQTT Namespace

| Namespace | Purpose | Retained? |
|-----------|---------|-----------|
| `highland/event/` | Point-in-time facts. Something happened. | **Never** |
| `highland/state/` | Current operational truth. What is true right now. | **Always** |
| `highland/status/` | Service health and liveness. Infrastructure concerns only. | No (heartbeats); Yes (health snapshots) |
| `highland/command/` | Imperative instructions to a service. | No |
| `highland/ack/` | Acknowledgment infrastructure. | No |

### event vs. state

An event is something that *happened* — it fires and it's gone. State is *what is currently true* — retained, always available to a restarting flow.

- Precipitation started → `event/` (it happened at a moment)
- Current synthesized weather conditions → `state/` (it's just true right now)
- Scheduler period transitioned to evening → `event/` (the transition happened)
- What period the house is currently in → `state/` (the house is in this period)

### state vs. status

State is operational — automations read it to make decisions. Status is infrastructure health — monitoring flows and dashboards read it. Don't put device sensor data under `status/`; don't put heartbeats under `state/`.

---

## Naming Convention

- **Style:** lowercase with underscores (`living_room`, `front_porch`)
- **Pattern:** `highland/{namespace}/{source_or_domain}/{event_type}[/{entity}]`

---

## Scheduler Periods

Three periods define the house's daily rhythm. Period **events** fire at the moment of transition (not retained). Current period **state** is always available at `highland/state/scheduler/period` (retained).

| Event | Trigger | Purpose |
|-------|---------|---------|
| `highland/event/scheduler/day` | Sunrise | House wakes up |
| `highland/event/scheduler/evening` | 30 min before sunset | House winds down |
| `highland/event/scheduler/overnight` | 10:00 PM (fixed) | House goes to sleep |

**Period naming rationale:** `overnight` rather than `night` implies continuity through early morning until `day`. Avoids collision with schedex astronomical constants.

**Retention:** Period events are not retained. Current period is always available at the state topic. Flows use both: the state topic on startup for recovery, event topics during normal operation. See `nodered/STARTUP_SEQUENCING.md` for the two-entry-point pattern.

### Task Events

Bespoke point-in-time triggers for specific scheduled jobs.

| Event | Trigger | Subscribers |
|-------|---------|-------------|
| `highland/event/scheduler/midnight` | 00:00:00 | Daily Digest, LoRaWAN mailbox, any flow needing a date-rollover trigger |
| `highland/event/scheduler/backup_daily` | TBD (3:00 AM) | Backup Utility Flow |

Task events answer: *"Is it time to do this specific job?"* They are named for the moment, not the consumer.

---

## When to Use Each Namespace

| Scenario | Approach |
|----------|----------|
| "I need to do something at sunset-ish" | Subscribe to `highland/event/scheduler/evening` |
| "I need to do something at a specific time unrelated to house rhythm" | Create a task event |
| "I need to do something 15 minutes after overnight starts" | Subscribe to `overnight`, add delay in flow |
| "I need current state after restart" | Subscribe to `highland/state/...` (retained) |
| "I need to react at the moment of a transition" | Subscribe to `highland/event/...` |

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

`highland/state/` payloads always include the full object on every publish — no partial updates.

Optional fields for events requiring ACK correlation:
```json
{
  "correlation_id": "abc123",
  "request_ack": true
}
```

---

## Retention Rules

| Topic Pattern | Retained |
|---------------|----------|
| `highland/state/#` | **Always** |
| `highland/status/+/health` | Yes |
| `highland/status/+/heartbeat` | No |
| `highland/event/#` | **Never** |
| `highland/command/#` | No |
| `highland/ack/#` | No |

### The Pattern for Recoverable State

For anything that needs retained representation (like scheduler period), the correct pattern is:

1. Publish the **event** (not retained) — the transition moment
2. Publish **state** (retained) — the new current truth

Flows use both: the state topic on startup for recovery, the event topic during normal operation for real-time reaction.

---

## ACK Pattern for Critical Operations

```
1. Utility publishes event with correlation_id
2. Utility registers expected ACKs with ACK Tracker (highland/ack/register)
3. Area flows perform action
4. Area flows publish ACK to highland/ack
5. ACK Tracker collects ACKs, publishes result after timeout to highland/ack/result
6. Requesting flow handles success/failure
```

See `nodered/NOTIFICATIONS.md` for the ACK Tracker utility flow implementation.

---

## Known Gaps: Device Resiliency

| Device | Status | Notes |
|--------|--------|-------|
| **Eufy Wi-Fi Locks** | ⚠️ Open | Cloud-dependent, no official local API. Lockdown ACK flows may not survive HA or internet outages until resolved. May require hardware replacement. |

---

## Authoritative Reference

`standards/MQTT_TOPICS.md` is the authoritative topic registry. Where this document and MQTT_TOPICS.md conflict, **MQTT_TOPICS.md wins**. This document covers philosophy and patterns; MQTT_TOPICS.md covers what actually exists.

---

*Last Updated: 2026-03-26*
