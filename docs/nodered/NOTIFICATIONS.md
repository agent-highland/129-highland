# Node-RED — Notification Framework

## Concept

Notifications answer: *"How urgently does a human need to know about this?"*

Notifications are separate from logging. A CRITICAL log may auto-generate a notification, but many notifications have nothing to do with errors — weather alerts, reminders, security events.

---

## Notification Topic

**Single topic — all details in payload:**

```
highland/event/notify
```

See `standards/MQTT_TOPICS.md` for full payload schema.

---

## Severity Levels

| Severity | DND Override | Use Case |
|----------|--------------|----------|
| `low` | No | Informational; can wait (fog advisory, routine status) |
| `medium` | No | Worth knowing soon, but not urgent |
| `high` | Yes | Needs attention now (lock failure, unexpected motion) |
| `critical` | Yes | Emergency (fire, flood, intrusion) |

---

## Notification Payload Fields

| Field | Required | Description |
|-------|----------|-------------|
| `targets` | **Yes** | Namespaced target array: `["people.joseph.ha_companion"]` |
| `severity` | Yes | `low`, `medium`, `high`, `critical` |
| `title` | Yes | Short summary |
| `message` | Yes | Full detail |
| `dnd_override` | No | Derived from severity if not specified |
| `media` | No | Image and/or video URLs |
| `actionable` | No | Can recipient respond? Default = false |
| `actions` | No | Available response actions |
| `sticky` | No | Notification persists until dismissed; default = false |
| `group` | No | Group related notifications together |
| `correlation_id` | No | For linking response back to originating event; also used as notification tag |

**Both `targets` and `severity` are required** — every notification represents a deliberate design-time decision about who receives it and how. No implicit defaulting.

---

## Target Addressing

Targets use a `namespace.key.channel` format:

| Example target | Resolves to |
|----------------|-------------|
| `people.joseph.ha_companion` | Joseph's phone via HA Companion |
| `people.*.ha_companion` | All people's HA Companion |
| `areas.living_room.tv` | Living room TV |
| `areas.*.tv` | All area TVs |

The `*` wildcard expands all keys in a namespace section.

---

## HA Companion App (Android)

Initial notification delivery channel. Configured via `notifications.json`.

### Android Notification Channels

Pre-configured in HA Companion App for user control over sound/vibration/DND:

| Channel ID | Purpose | DND Override |
|------------|---------|--------------|
| `highland_low` | Informational | No |
| `highland_default` | Standard alerts | No |
| `highland_high` | Urgent alerts | Yes |
| `highland_critical` | Emergency | Yes |

### Severity → HA Companion Mapping

| Severity | HA Priority | Channel | Persistent |
|--------------|-------------|---------|------------|
| `low` | `low` | `highland_low` | No |
| `medium` | `default` | `highland_default` | No |
| `high` | `high` | `highland_high` | No (unless `sticky: true`) |
| `critical` | `high` | `highland_critical` | Yes |

### Clearing Notifications

To programmatically dismiss a notification (e.g., lock succeeded on retry):

```yaml
service: notify.mobile_app_joseph_phone
data:
  message: "clear_notification"
  data:
    tag: "lockdown_20260224_2200"
```

Same `tag` as the original notification, message `"clear_notification"`. Use `highland/command/notify/clear` on the bus — the Notification Utility handles the HA service call.

---

## Action Responses

When a user taps a notification action, HA fires an event. The Notification Utility captures this and publishes to MQTT:

**Published to MQTT:**
```json
{
  "timestamp": "...",
  "source": "notification",
  "action": "retry",
  "correlation_id": "lockdown_20260224_2200",
  "device": "mobile_joseph"
}
```

Topic: `highland/event/notify/action_response`

Originating flows subscribe to `highland/event/notify/action_response`, filter by `correlation_id`, and handle accordingly.

---

## Utility: Notifications Flow

Centralized delivery flow. All notification traffic enters via MQTT and is dispatched here — no other flow calls HA notification services directly.

### Topics

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `highland/event/notify` | Inbound | Deliver a notification |
| `highland/command/notify/clear` | Inbound | Dismiss a previously delivered notification |
| `highland/event/log` | Outbound | Log delivery outcomes |

### Groups

**Receive Notification** — MQTT in → Initializer Latch → Validate Payload → Build Targets → `link call` (Deliver, dynamic) → Log Event link out

**HA Companion Delivery** — Link In → Connection Gate → Build Service Call → HA service call node → `link out` (return mode)

**Clear Notification** — MQTT in (`highland/command/notify/clear`) → Initializer Latch → Build Clear Call → `link call` (Deliver, dynamic) → Log Event link out

**State Change Logging** — Log Event link in → MQTT Available? switch → Format Log Message → MQTT out / Log to Console

**Test Cases** — Persistent sanity tests; intentionally preserved.

### Build Targets (Fan Out)

Resolves a `targets` array into individual delivery messages. Resolution logic:

1. Split target into `[namespace, key, channel]`
2. Look up `notifications[namespace]` — WARN and skip if unknown
3. Expand `*` to all keys in the namespace section
4. Look up `entry.channels[channel]` — WARN and skip if missing
5. Emit one message per resolved address

`resolveLinkTarget()` maps channel names to their `Link In` node names:

```javascript
function resolveLinkTarget(channel) {
    switch (channel) {
        case 'ha_companion': return 'Home Assistant Companion';
        case 'tv':           return 'Television Delivery';
        default: throw new Error(`Unable to resolve channel: ${channel}`);
    }
}
```

Adding a new channel: add a case here and a new delivery group with a matching `Link In` name.

### `link call` Node (Deliver)

Reads `msg.target` dynamically and routes to the matching `Link In` node name. Set to **dynamic** link type, 30-second timeout. Output wires to Log Event link out — logging happens once on the return path after delivery completes. Timeouts handled by a catch node scoped to the `link call`.

### Connection Gate (HA Companion Delivery)

`CONNECTION_TYPE = home_assistant`, `RETENTION_MS = 0`. Output 2 unwired — if HA is down the message drops. Resiliency is the caller's responsibility via channel selection. See `nodered/SUBFLOWS.md`.

### Return Path

The last node in each delivery group is a `link out` set to **return** mode. This returns the message to the `link call` that dispatched it, completing the call/return cycle and triggering downstream logging.

### MQTT Availability Fallback

When MQTT goes down, the normal log path (`highland/event/log` via MQTT out) is unavailable. The State Change Logging group handles this:

```
Log Event link in → MQTT Available? switch (global.connections.mqtt == 'up')
    ↓ up                                   ↓ else
Format Log Message → MQTT out         Log to Console (node.error/warn)
```

`Log to Console` uses `node.error()` / `node.warn()` which write to Node-RED's internal log regardless of MQTT state — visible via `docker compose logs nodered`.

---

## Channel Selection Philosophy

**Multi-channel is intent, not failover.** Specifying `["ha_companion", "pushover"]` means deliver via both regardless of availability. Specifying `["ha_companion"]` means HA Companion only — the caller has made a conscious decision that delivery is best-effort if HA is down.

**Resiliency is the caller's responsibility.** If a notification requires guaranteed delivery, the caller specifies multiple channels. The Notification Utility delivers what it can via the channels specified.

**Missing channel address → log WARN, skip, continue.** Deliver as much as possible.

---

## Future Channels (Deferred)

| Channel | Notes |
|---------|-------|
| `telegram` | Two-way interaction possible |
| `signal` | Privacy-focused |
| `tv` | HA notification to TV entity |
| `tts` | Text-to-speech on smart speakers — see `highland/event/speak` |
| `email` | SMTP integration |

Adding a new channel: add a case to `resolveLinkTarget()`, build a new delivery group with a matching `Link In` name, wire the return `link out`.

---

*Last Updated: 2026-03-26*
