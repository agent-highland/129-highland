# Video Analysis Pipeline — Design &amp; Architecture

## Overview

Node-RED-owned video analysis pipeline providing motion-triggered capture and a three-stage analysis ladder: local triage (mechanical gate), remote triage (intelligent scene assessment), and remote deep analysis (full clip analysis).

**Target state:** Operates without Home Assistant as a dependency in the motion detection and capture path (direct reolink_aio integration). **Initial implementation** uses HA's Reolink integration as a known-working scaffold while NVR API capabilities are validated. See [Phased Implementation Approach](#phased-implementation-approach) below.

---

## Design Goals

- **HA independence (target state)** — Motion detection and capture must not require HA to be running. Initial implementation uses HA as a scaffold; migrated once NVR API is validated.
- **Three-stage escalation ladder** — Local triage (free) gates remote triage (cheap); remote triage gates deep analysis (expensive). Each stage only fires if the previous passes.
- **Local triage is mechanical, not intelligent** — Edge AI eliminates obvious non-events (noise reduction only). Threat assessment enters at remote triage, not before.
- **Progressive notification** — Alert on motion, update as analysis completes, dismiss if nothing of interest
- **Property-aware filtering** — Zone-based detection; cameras observing adjacent properties require spatial filtering, not just object detection
- **Cost-controlled** — Remote triage operates on a single annotated still; deep analysis (clip) only reached after remote triage escalates

---

## Analysis Stages

Three distinct stages with increasing capability and cost. Each stage gates the next.

| Stage | Technology | Input | Cost | Job |
|-------|-----------|-------|------|-----|
| **Local Triage** | CPAI + Coral TPU | Still keyframe | Free | Eliminate non-events. Is there anything worth looking at? |
| **Remote Triage** | Gemini (still) | Annotated still | Cheap | Intelligent assessment. Is this worth full analysis? |
| **Remote Deep Analysis** | Gemini (clip) | Video clip | Expensive | Full scene description, narrative, threat assessment |

**Key principle:** Local triage is purely mechanical — it classifies objects and provides bounding boxes. It has no concept of threat, intent, or context. Threat assessment enters the pipeline at remote triage, not before.

---

## Event Flow

### Target Architecture (NVR)

```
NVR detects motion → starts recording (NVR-managed)
        │
        ▼
Sidecar receives push event
highland/event/camera/{camera_id}/motion published to MQTT
        │
        ▼
Pull still from active NVR stream
        │
        ▼
─────────────────────────────────────
STAGE 1: LOCAL TRIAGE
─────────────────────────────────────
CPAI object detection → bounding boxes
Zone filtering (Node-RED, bottom-center anchor)
        │
        ├── No in-scope detections → done
        │   NVR recording continues/expires naturally
        │   Early notification dismissed
        │
        └── In-scope detections found
                │
                ▼
        Build annotated still:
        • Draw bounding boxes for in-scope detections only
        • Suppressed detections (adjacent property) not marked
                │
                ▼
─────────────────────────────────────
STAGE 2: REMOTE TRIAGE
─────────────────────────────────────
Gemini receives: annotated still + triage context
Prompt: what was detected, which boxes are in scope,
        property boundary context, camera geometry
        │
        ├── Nothing of interest → done
        │   Notification dismissed
        │
        └── Worth escalating
                │
                ▼
        Pull clip from NVR-managed recording
        Early notification sent (category, triage summary)
                │
                ▼
─────────────────────────────────────
STAGE 3: REMOTE DEEP ANALYSIS
─────────────────────────────────────
Gemini receives: video clip (+ annotated still for reference)
Full scene description, behavior, threat assessment, narrative
        │
        ▼
Notification updated with full description
(or dismissed if nothing of interest after full analysis)
```

**Key dependency:** This architecture requires that the NVR stream is accessible for still pull while recording is actively in progress. To be validated when NVR is available.

---

## Classification Targets

Triage and analysis operate against three classification targets. Everything else is ignored.

| Target | Examples | Notes |
|--------|----------|-------|
| `animal` | deer, bear, raccoon, coyote | Species matters for threat escalation |
| `vehicle` | car, truck, emergency vehicle | Type matters for zone exception logic |
| `person` | any human | Never auto-cooldown; see below |

---

## Cooldown / Governor

Prevents notification fatigue and unnecessary Gemini API spend during sustained activity.

### Cooldown Eligibility

| Target | Auto-Cooldown | Rationale |
|--------|---------------|-----------|
| `animal` | Yes | Lingering/repeated animal presence is low-stakes and high-frequency |
| `vehicle` | Yes | Parked vehicle in driveway doesn't need repeated analysis |
| `person` | **No** | Too high-stakes to suppress automatically; use kill switch for expected presence |

---

## Kill Switch (Manual Override)

For scenarios where prolonged expected activity would generate unavoidable noise — gatherings, yard work, expected deliveries — a per-camera kill switch provides a hard circuit-breaker that sits upstream of all processing.

**Kill switch = hard off.** When active: no triage runs, no analysis runs, no notifications sent.

**Topic structure:**
```
homeassistant/switch/camera_{id}_kill/config    ← discovery (retained)
highland/state/camera/{id}/kill                 ← Node-RED → HA (retained, ON/OFF)
highland/command/camera/{id}/kill               ← HA → Node-RED
```

---

## Progressive Notification Pattern

Leverages HA Companion App notification update/dismiss via `tag` field.

| Stage | Action |
|-------|--------|
| Motion detected | Send notification: "Motion detected — [camera friendly name]" |
| Local triage negative | Clear notification via tag |
| Local triage positive | Update notification: category + count of in-scope detections |
| Remote triage negative | Clear notification |
| Remote triage positive | Update notification: remote triage summary |
| Deep analysis complete | Update notification with full description and narrative |

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| **NODERED_PATTERNS.md** | MQTT discovery pattern (cooldown/kill switch entities), utility flow conventions |
| **EVENT_ARCHITECTURE.md** | MQTT topic conventions, payload standards |
| **CALENDAR_INTEGRATION.md** | Calendar-driven kill switch automation; gathering events → camera suppression |

---

*Last Updated: 2026-03-10*
