# Calendar Integration — Design & Architecture

## Overview

Google Calendar serves as the household's event scheduling layer. A dedicated `Utility: Calendaring` flow owns all Google Calendar API access, polls on a regular cadence, and publishes a rolling 7-day snapshot to MQTT. All other flows consume the snapshot — nothing queries the calendar directly.

---

## Design Goals

- **Single integration point** — `Utility: Calendaring` owns the Google Calendar API relationship; consumers subscribe to MQTT, never query the API directly
- **Zero-friction authoring** — household members use Google Calendar exactly as they do today; no metadata syntax, no new tools
- **Loose coupling** — the flow has no knowledge of consumers; flows subscribe to what they care about
- **Rolling snapshot** — a single retained MQTT topic always contains the next 7 days of events; consumers filter for what they need
- **Stateless re-derivation** — every poll cycle derives complete authoritative state from scratch; no incremental patching
- **Near-real-time** — 15-minute poll cadence ensures same-day additions are picked up quickly; manual reload available for immediate refresh

---

## Calendar Structure

Three dedicated Google sub-calendars, each representing a distinct event category. The default calendar is left empty.

| Calendar | Purpose |
|----------|---------|
| **Appointments** | Timed events — service calls, deliveries, repair visits, anything with a specific start/end time |
| **Reminders** | All-day nudges — HVAC filter changes, maintenance tasks, anything without a specific time |
| **Trash & Recycling** | Recurring pickup schedules only |

Classification is determined purely by which calendar an event belongs to. The bridge has no knowledge of event titles, colors, or descriptions.

Calendar IDs are stored in `secrets.json`:

```json
{
  "google_calendar": {
    "appointments_calendar_id": "...",
    "reminders_calendar_id": "...",
    "trash_calendar_id": "..."
  }
}
```

---

## Core Operating Model

### Rolling 7-Day Snapshot

Every poll cycle queries all three calendars for the next 7 days (168 hours from poll time), classifies all events by source calendar, and overwrites the retained snapshot topic with the complete result. Consumers filter the snapshot for whatever they need — the bridge has no opinion about what's relevant to any given consumer.

**Key properties:**
- **Stateless re-derivation** — every poll overwrites from scratch; no incremental patching
- **Single topic** — one retained `highland/state/calendar/snapshot` contains everything; consumers decide what to extract
- **Rolling window** — the 7-day window advances with each poll; no concept of "today" or "tomorrow" in the bridge itself
- **Multi-day events** — stored once with `start` and `end`; consumers determine if their date of interest falls within the span
- **All-day vs timed events** — normalized at classification time; `all_day: true` flag on each event simplifies consumer logic

### Poll Cadence

| Trigger | Cadence | Purpose |
|---------|---------|---------|
| Periodic interval | Every 15 minutes | Primary cadence; picks up same-day additions within 15 minutes |
| On startup | Immediate | Reconciles state after any outage |
| `highland/command/calendar/reload` | On demand | Immediate refresh — manual or programmatic (e.g. triggered by voice assistant after modifying an event) |

At 3 calendars × 96 polls/day = 288 API calls/day, well within Google's limits (1M calls/day).

### Startup Behavior

Node-RED startup is not a special codepath. On startup the flow immediately runs a full poll cycle — re-derives state, overwrites the retained snapshot, and the periodic interval takes over normally. No special startup logic needed.

### API Error Handling

If the Google Calendar API is unavailable:
- Log ERROR
- Do **not** overwrite the retained snapshot (leave last-known-good in place)
- Retry after 5 minutes
- If still unavailable after retry, notify (medium priority): "Calendar unavailable — snapshot may be stale"

---

## Authentication

OAuth 2.0 via a Google Cloud Console OAuth client. Credentials stored in `secrets.json`. Token refresh is handled by the flow — access tokens expire; the refresh token is used to obtain a new one transparently.

**Setup prerequisite:** Create an OAuth 2.0 client in Google Cloud Console with a redirect URI pointing to Node-RED. This is a one-time setup step performed during flow implementation.

---

## MQTT Topics

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/state/calendar/snapshot` | Yes | Rolling 7-day event snapshot — authoritative current state |
| `highland/command/calendar/reload` | No | Trigger an immediate poll cycle |

---

## Snapshot Payload

```json
{
  "timestamp": "2026-03-24T10:15:00Z",
  "source": "calendaring",
  "window_start": "2026-03-24T10:15:00",
  "window_end": "2026-03-31T10:15:00",
  "events": [
    {
      "id": "google_event_id_1",
      "calendar": "appointments",
      "title": "Plumber",
      "date": "2026-03-24",
      "start": "2026-03-24T14:00:00",
      "end": "2026-03-24T16:00:00",
      "all_day": false
    },
    {
      "id": "google_event_id_2",
      "calendar": "reminders",
      "title": "Replace HVAC filter",
      "date": "2026-03-24",
      "all_day": true
    },
    {
      "id": "google_event_id_3",
      "calendar": "trash",
      "title": "Trash & Recycling",
      "date": "2026-03-27",
      "all_day": true
    },
    {
      "id": "google_event_id_4",
      "calendar": "appointments",
      "title": "Renovation crew",
      "date": "2026-03-25",
      "start": "2026-03-25T08:00:00",
      "end": "2026-03-27T17:00:00",
      "all_day": false
    }
  ]
}
```

**Field notes:**
- `calendar` — classification by source calendar: `appointments`, `reminders`, `trash`
- `date` — the start date in local time (America/New_York); all-day events use this for filtering
- `start` / `end` — local time ISO strings for timed events; absent for all-day events
- `all_day` — flag for consumer logic; avoids parsing `start`/`end` absence
- Multi-day events appear once with their full `start`/`end` span; consumers check if their date of interest falls within

All timestamps are normalized to local time (`America/New_York` from `location.json`) before publishing. Consumers never need to handle timezone conversion.

---

## Consumer Pattern

Consumers subscribe to `highland/state/calendar/snapshot` (retained) and filter for what they need:

```javascript
const snapshot = JSON.parse(msg.payload);
const today = new Date().toISOString().slice(0, 10);
const tomorrow = new Date(Date.now() + 86400000).toISOString().slice(0, 10);

// Today's appointments
const todayAppointments = snapshot.events.filter(e =>
    e.calendar === 'appointments' && e.date === today
);

// Trash pickup today or tomorrow (for look-ahead)
const trashRelevant = snapshot.events.filter(e =>
    e.calendar === 'trash' && (e.date === today || e.date === tomorrow)
);

// Today's reminders
const todayReminders = snapshot.events.filter(e =>
    e.calendar === 'reminders' && e.date === today
);
```

Because the snapshot is retained, a consumer receives it immediately on subscription — startup recovery is automatic with no special codepath.

---

## Flow Structure: Utility: Calendaring

### Groups

**Sinks** — Periodic interval inject (15 min) + On Startup inject + MQTT in (`highland/command/calendar/reload`) → `Trigger Poll` link out

**Poll** — Link in → `Fetch Calendars` (HTTP calls to Google Calendar API for all three calendars) → `Classify Events` → `Build Snapshot` → MQTT out (`highland/state/calendar/snapshot`, retained)

**Token Management** — Handles OAuth token refresh; exposes valid access token to Poll group via flow context

**Error Handling** — flow-wide catch → log + conditional notify

### Key Function Nodes

**`Fetch Calendars`** — Makes three parallel HTTP GET requests to the Google Calendar API events endpoint, one per calendar, using the stored OAuth access token. Window: now to now + 7 days.

**`Classify Events`** — Normalizes each event from the Google API response format:
- Maps `calendarId` → `calendar` category (`appointments`, `reminders`, `trash`)
- Extracts `start.dateTime` (timed) or `start.date` (all-day) and normalizes to local time
- Sets `all_day` flag
- Produces a flat array of normalized events across all three calendars

**`Build Snapshot`** — Assembles the final payload, sorts events chronologically by date/start, publishes to retained topic.

---

## Trash & Recycling — Look-Ahead Pattern

Trash and recycling events appear in the snapshot tagged `calendar: "trash"`. The Daily Digest (and any future trash-aware flow) reads the snapshot and filters for trash events within its look-ahead window:

- **Thursday's digest:** finds a trash event dated Friday → surfaces as "tomorrow"
- **Friday's digest:** finds the same event dated today → surfaces as "today"

Holiday schedule changes and weather-related delays are handled by modifying the individual recurring event instance in Google Calendar. The next poll picks up the change automatically.

---

## Future: Camera Suppression

*This section documents a planned capability, not the current implementation. Camera suppression requires the NVR and video pipeline infrastructure to be in place first.*

### Attendee-Based Approach

Each camera has a dedicated email address on the household domain. To suppress a camera during a calendar event, invite it as a guest. The calendaring flow reads attendee lists and fires suppression events for any event containing camera guest addresses.

```
camera.driveway@highland.ferris.network
camera.rear_yard@highland.ferris.network
camera.rear_patio@highland.ferris.network
camera.front_porch@highland.ferris.network
```

### Additional Topics (future)

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/state/calendar/camera_suppression` | Yes | Active suppression state — union of all currently active suppression events |
| `highland/event/calendar/camera_suppression/start` | No | Ceremony event — fires once per event when suppression begins |
| `highland/event/calendar/camera_suppression/end` | No | Ceremony event — fires once per event when suppression ends |

### Ceremony Tracking (future)

Start/end events should fire exactly once per event, not on every reconciliation. The flow tracks ceremony state by event ID in disk-backed flow context. Eviction strategy: remove entries for events whose end time is in the past to prevent unbounded growth.

### Camera Address Registry (future)

```json
{
  "camera_guests": {
    "camera.driveway@highland.ferris.network": "driveway",
    "camera.rear_yard@highland.ferris.network": "rear_yard",
    "camera.rear_patio@highland.ferris.network": "rear_patio",
    "camera.front_porch@highland.ferris.network": "front_porch"
  }
}
```

See VIDEO_PIPELINE.md for kill switch design and state management.

---

## Future: AI Assistant Integration

Once the AI assistant has calendar write capability:

*"Hey, our 4pm plumber appointment has been moved to 4:30pm."*

The assistant modifies the event in Google Calendar and publishes to `highland/command/calendar/reload`. The flow polls immediately and downstream flows react within seconds. No changes to the calendaring flow itself — the MQTT hook makes this work without any additional plumbing.

---

## Open Questions

- [ ] OAuth redirect URI — confirm Node-RED endpoint during implementation
- [ ] Token refresh implementation — determine whether to use a Node-RED OAuth library or implement refresh token exchange manually in a function node
- [ ] Ceremony state eviction strategy (future, camera suppression) — remove entries for events past their end time; determine eviction trigger (on poll, on startup, or periodic)

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| **EVENT_ARCHITECTURE.md** | Calendar events follow standard MQTT payload conventions |
| **NODERED_PATTERNS.md** | `Utility: Calendaring` is a utility flow; Config Loader handles API credentials |
| **VIDEO_PIPELINE.md** | Future consumer — calendar events drive camera kill switch |
| **AUTOMATION_BACKLOG.md** | AI calendar assistant (natural language event creation) is a related backlog item |

---

*Last Updated: 2026-03-24*
