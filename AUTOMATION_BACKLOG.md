# Automation Backlog

Captured ideas for future automations. Not requirements — just things worth exploring when time permits.

---

## How to Use This Document

**Adding ideas:**
- Capture the concept, not the implementation
- Note what triggered the idea (pain point, inspiration, etc.)
- Don't over-think it — a sentence or two is fine

**Promoting to implementation:**
- Move to "In Progress" when actively working on it
- Remove from backlog when complete (or abandoned)

---

## Backlog

### Security & Safety

| Idea | Notes | Added |
|------|-------|-------|
| *Example: Lockdown confirmation via TTS* | Announce "House secured" when all locks confirmed | — |

### Lighting & Ambiance

| Idea | Notes | Added |
|------|-------|-------|

### Climate & Comfort

| Idea | Notes | Added |
|------|-------|-------|

### Notifications & Awareness

| Idea | Notes | Added |
|------|-------|-------|
| Externalize Daily Digest HTML template | `Build Email` function node contains the full HTML template inline, making it painful to maintain. Move template to a file in `/home/nodered/config/templates/daily-digest.html`, load via Config Loader at startup into `global.config.digestTemplate`. `Build Email` then performs string replacement against the loaded template rather than generating HTML inline. SVG weather icons could move to a separate lookup file too. | 2026-03-25 |

### Presence & Occupancy

| Idea | Notes | Added |
|------|-------|-------|

### Maintenance & System Health

| Idea | Notes | Added |
|------|-------|-------|
| Host metrics monitoring | CPU load, memory usage, disk usage, and operating temperatures for Hub and Workflow hosts. Lightweight agent script per host publishes to `highland/status/{host}/metrics` (retained); Health Monitor evaluates against thresholds from `thresholds.json`. Hub will also need per-container metrics via `docker stats`. Revisit when baseline is stable and there is a concrete reason to care. | 2026-03-23 |
| Device auto-discovery | When a payload arrives from an unregistered device, create a provisional device_registry.json entry (key, derived name, protocol, topic, partial capabilities inferred from payload fields) and flag for human review via notification or daily digest. Reduces manual registry maintenance from "author from scratch" to "review and enrich". Z2M bridge/devices topic provides protocol, topic, and modelID to bootstrap the entry. | 2026-03-23 |

### AI & Advanced

| Idea | Notes | Added |
|------|-------|-------|
| Calendar management via AI assistant | "Add HVAC service Thursday at 2pm" → writes to Google Calendar | 2025-02-24 |

### Uncategorized

| Idea | Notes | Added |
|------|-------|-------|

---

## Infrastructure Projects

Major efforts that enable new capabilities across the system.

### Video Analysis Pipeline & Database Infrastructure

**Priority:** Future (Post-Phase 1)  
**Complexity:** High  
**Added:** 2026-03-02

#### Overview

Replace LLM Vision integration with a fully Node-RED-owned video analysis pipeline. Eliminates third-party integration dependency while maintaining Assist queryability via calendar-based event storage. Requires PostgreSQL for concurrent write support.

#### Components

**PostgreSQL Infrastructure**
- Deploy PostgreSQL container on Node-RED/Utility box
- Configure HA Recorder to use external PostgreSQL
- Validate concurrent write capability (HA + Node-RED)
- Establish backup/retention strategy

**Video Analysis Flow (Node-RED)**
- Camera motion event trigger
- Keyframe/clip capture
- Gemini triage (worth deep analysis?)
- Gemini deep analysis (full scene description)
- Media storage to HA `/media/analysis/.../`
- Calendar event creation via `calendar.create_event`

**Calendar Event Schema**
```
Title: "{category} - {camera_friendly_name}"
Description:
  camera: {entity_id}
  category: {delivery|person|vehicle|animal|unknown}
  confidence: {0.0-1.0}
  objects: {comma-separated list}
  summary: {narrative description}
  keyframe: {/media/analysis/.../file.jpg}
  clip: {/media/analysis/.../file.mp4}
```

**Assist Integration**
- Create Local Calendar entity: `calendar.video_analysis_timeline`
- Create query script with descriptive `description` field for LLM tool discovery
- Script pulls events via `calendar.get_events`, returns formatted data
- Test with Assist: "Was my package delivered?"

**Retention Management**
- Define retention periods (events: permanent?, media: 30/90 days?)
- Node-RED flow for media cleanup (cron-style)
- Handle orphaned references gracefully

#### Open Questions
- Optimal calendar event structure for LLM parsing (plain text vs structured)
- Multi-camera correlation (same event seen by multiple cameras)
- Event deduplication strategy

---

## In Progress

| Idea | Status | Started |
|------|--------|---------|

---

## Completed / Abandoned

| Idea | Outcome | Date |
|------|---------|------|

---

*Last Updated: 2026-03-25*
