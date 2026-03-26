# Video Analysis Pipeline — Design & Architecture

## Overview

Node-RED-owned video analysis pipeline providing motion-triggered capture and a three-stage analysis ladder: local triage (mechanical gate), remote triage (intelligent scene assessment), and remote deep analysis (full clip analysis).

**Target state:** Operates without Home Assistant as a dependency in the motion detection and capture path (direct `reolink_aio` integration).

**Initial implementation:** Uses HA's Reolink integration as a known-working scaffold while NVR API capabilities are validated. See [Phased Implementation Approach](#phased-implementation-approach) below.

---

## Design Goals

- **HA independence (target state)** — Motion detection and capture must not require HA to be running
- **Three-stage escalation ladder** — Local triage (free) gates remote triage (cheap); remote triage gates deep analysis (expensive)
- **Local triage is mechanical, not intelligent** — Edge AI eliminates obvious non-events only; threat assessment enters at remote triage
- **Progressive notification** — Alert on motion, update as analysis completes, dismiss if nothing of interest
- **Property-aware filtering** — Zone-based detection; cameras observing adjacent properties require spatial filtering
- **Cost-controlled** — Remote triage operates on a single annotated still; deep analysis only reached after escalation

---

## Analysis Stages

| Stage | Technology | Input | Cost | Job |
|-------|-----------|-------|------|-----|
| **Local Triage** | CPAI + Coral TPU | Still keyframe | Free | Eliminate non-events. Is there anything worth looking at? |
| **Remote Triage** | Gemini (still) | Annotated still | Cheap | Intelligent assessment. Is this worth full analysis? |
| **Remote Deep Analysis** | Gemini (clip) | Video clip | Expensive | Full scene description, narrative, threat assessment |

---

## Event Flow (Target Architecture — NVR)

```
NVR detects motion → starts recording (NVR-managed)
        │
        ▼
reolink_aio sidecar receives push event → MQTT
        │
        ▼
Pull still from active NVR stream
        │
        ▼
─────── STAGE 1: LOCAL TRIAGE ──────────────────────────────
CPAI object detection → bounding boxes
Zone filtering (bottom-center anchor)
        │
        ├── No in-scope detections → done (notification dismissed)
        │
        └── In-scope detections found
                │
                ▼
        Build annotated still (in-scope bounding boxes only)
                │
                ▼
─────── STAGE 2: REMOTE TRIAGE ─────────────────────────────
Gemini receives: annotated still + triage context
        │
        ├── Nothing of interest → done (notification dismissed)
        │
        └── Worth escalating
                │
                ▼
        Pull clip from NVR-managed recording
        Early notification sent (category, triage summary)
                │
                ▼
─────── STAGE 3: REMOTE DEEP ANALYSIS ──────────────────────
Gemini receives: video clip (+ annotated still for reference)
Full scene description, behavior, threat assessment, narrative
        │
        ▼
Notification updated with full description
```

---

## Phased Implementation Approach

### Phase 1: HA-Dependent Scaffold (Initial Build)

The pipeline's motion trigger and still/clip capture path will initially use HA's Reolink integration. This is a deliberate pragmatic choice:

- **Known working** — HA's Reolink integration is validated in the live system today
- **Unblocks implementation** — the full CPAI triage → Gemini analysis chain can be built and tuned without waiting on NVR API validation
- **Pipeline logic is unaffected** — Node-RED still owns the state machine, cooldown, kill switch, zone filtering, and all analysis stages

In Phase 1, Node-RED receives motion events via HA's event bus and uses HA service calls for still/clip capture where needed.

Any flow sections that use HA as the capture path should be clearly marked:

```javascript
// TODO: Replace with direct reolink_aio path once NVR API is validated
// See subsystems/VIDEO_PIPELINE.md — Phased Implementation Approach
```

### Phase 2: Direct reolink_aio Integration (Target State)

Once the NVR is physically available and the following are validated:
- Push event mechanism (TCP Baichuan vs. ONVIF SWN)
- Per-channel still capture via API
- Stream accessibility during active NVR recording
- Clip extraction capabilities

...the HA-dependent input path is replaced with the `reolink_aio` sidecar. The rest of the pipeline is unchanged — contained swap at the input end only.

---

## Components

### Reolink Sidecar Service

Thin Python service using `reolink_aio` (starkillerOG). Deployed as a Docker container on Workflow alongside Node-RED.

**MQTT interface:**

| Direction | Topic | Purpose |
|-----------|-------|---------|
| Publish | `highland/event/camera/{camera_id}/motion` | Motion detected |
| Publish | `highland/status/camera_sidecar/health` | Sidecar health |
| Subscribe | `highland/command/camera/{camera_id}/capture_still` | Trigger still capture |
| Subscribe | `highland/command/camera/{camera_id}/capture_clip` | Trigger clip capture |
| Publish | `highland/event/camera/{camera_id}/still_ready` | Still captured, path in payload |
| Publish | `highland/event/camera/{camera_id}/clip_ready` | Clip captured, path in payload |

---

### Stage 1: Local Triage (CodeProject.AI)

Local inference on Edge AI box (SFF + Coral TPU). Purely mechanical — classifies objects and returns bounding boxes. No threat assessment.

**CPAI modules:**
- `ObjectDetectionCoral` — Coral TPU object detection
- `ipcam-general` (MikeLud) — person + vehicle only; highest accuracy for those targets
- `ipcam-animal` (MikeLud) — species-level: bird, cat, dog, horse, sheep, cow, bear, deer, rabbit, raccoon, fox, skunk, squirrel, pig
- `ipcam-delivery` (MikeLud) — carrier identification: Amazon, DHL, FedEx, UPS, USPS, etc.

**CPAI as a multi-call pipeline:**

Multiple CPAI modules may run sequentially against the same event. Node-RED owns the accumulated detection object and makes each call independently. For enrichment calls (face recognition, LPR), Node-RED crops the image to the relevant bounding box before submitting — a face recognition model operating on a face-sized crop is materially more accurate than one operating on a full scene image.

**Closed-set classifier behavior:**

CPAI's animal models are closed-set classifiers — an animal outside the training set is forced to the closest match. Practical consequences:

| Animal | Likely CPAI output |
|--------|-------------------|
| Coyote | `dog` |
| Fisher cat | `cat` or `dog` |
| Groundhog | `cat` or no detection |
| Large snake | No detection |

A coyote classified as `dog` still passes the animal gate and escalates to Gemini, which will identify it correctly. The genuine blind spot is an animal that produces no detections at all — those events are silently dropped at local triage.

**NVR history as calibration mechanism:** The NVR maintains a ground truth record of all motion events. Comparing NVR event history against pipeline escalations will surface blind spots empirically over time.

**CPAI status note:** Original maintainers departed in late 2024; active development has slowed. For a local inference service running in Docker with a stable, bounded use case, an abandoned-but-working tool is acceptable — pin a known-good version, validate Coral TPU support before committing. If CPAI becomes untenable, the REST call pattern from Node-RED makes swapping the inference backend relatively low-friction.

---

### Stage 2: Remote Triage (Gemini — still)

Intelligent scene assessment on the annotated still. First point where threat assessment, behavioral inference, and contextual reasoning occur.

**Input:** Annotated still (in-scope bounding boxes drawn) + prompt including what local triage detected, count/category of in-scope subjects, camera position/geometry context, property boundary description.

**Gate decision:** Remote triage result determines whether the pipeline escalates to deep analysis or terminates.

---

### Stage 3: Remote Deep Analysis (Gemini — clip)

Full scene analysis on the video clip. Only reached after remote triage escalates.

**Input:** Video clip + annotated still for reference + prompt with full scene context.

**Output:** Full scene description, behavior assessment, threat level, classification for event storage, confidence scoring, final notification content.

---

## Zone Filtering

Per-camera zone configuration:

```json
{
  "camera_driveway": {
    "zones": {
      "our_property": {
        "polygon": [[x1,y1],[x2,y2],[x3,y3],[x4,y4]],
        "default": "analyze"
      },
      "neighbor_north": {
        "polygon": [[x1,y1],[x2,y2],[x3,y3],[x4,y4]],
        "default": "suppress",
        "exceptions": ["emergency_vehicle"]
      }
    }
  }
}
```

**Logic:** Zone membership tested using the **bottom-center point** of each bounding box (ground contact point, not centroid). This correctly handles perspective distortion from elevated/angled camera mounts — a person at the rear of the driveway has feet on-property even if their head appears to float over the neighbor's yard in the 2D frame.

---

## Classification Targets

| Target | Examples | Notes |
|--------|----------|-------|
| `animal` | deer, bear, raccoon, coyote | Species matters for threat escalation |
| `vehicle` | car, truck, emergency vehicle | Type matters for zone exception logic |
| `person` | any human | Never auto-cooldown |

---

## Cooldown / Governor

| Target | Auto-Cooldown | Rationale |
|--------|---------------|-----------|
| `animal` | Yes | Lingering/repeated animal presence is low-stakes |
| `vehicle` | Yes | Parked vehicle doesn't need repeated analysis |
| `person` | **No** | Too high-stakes to suppress automatically; use kill switch for expected presence |

**Cooldown override conditions** — any of these break cooldown and force analysis: new classification target detected, species/type escalation, composition change, count change (threshold TBD), zone change.

---

## Kill Switch (Manual Override)

**Kill switch = hard off.** When active: no triage, no analysis, no notifications. Everything suppressed regardless of detection.

**Cooldown = soft gate.** Algorithm-driven, mid-pipeline, temporary.

| Scenario | Mechanism |
|----------|-----------|
| Deer lingering in yard for an hour | Cooldown (automatic) |
| Backyard gathering | Kill switch (manual) |
| Expected delivery window | Kill switch (manual) |
| Bear detected during deer cooldown | Cooldown override (automatic) |

---

## Progressive Notification Pattern

| Stage | Action |
|-------|--------|
| Motion detected | Send: "Motion detected — [camera friendly name]" |
| Local triage negative | Clear notification |
| Local triage positive | Update: category + count of in-scope detections |
| Remote triage negative | Clear notification |
| Remote triage positive | Update: remote triage summary |
| Deep analysis complete | Update with full description and narrative |
| Nothing of interest after deep analysis | Clear notification |

---

## Open Questions

- [ ] Video pipeline MQTT topics need formal registration in `standards/MQTT_TOPICS.md`
- [ ] Does NVR API support on-demand still capture per channel?
- [ ] Is clip capture via `reolink_aio` or direct RTSP pull (ffmpeg)?
- [ ] Can RTSP stream be read concurrently while NVR is actively recording?
- [ ] PostgreSQL schema for video events
- [ ] Retention policy — events vs. media (keyframes, clips) likely different
- [ ] Dual-lens 180° cameras present as two independent channels on the NVR — deduplication strategy TBD
- [ ] Street-facing camera feasibility pending late spring foliage assessment
- [ ] Validate Coral TPU compatibility with pinned CPAI version before committing

---

## Deferred

| Item | Reason |
|------|--------|
| Contextual exceptions on adjacent property (smoke, fire, altercation) | Requires Gemini in triage path; add after baseline pipeline is proven |
| Facial recognition / LPR as local triage enrichment | Architecture defined; deferred until baseline pipeline is stable |
| Custom model training for out-of-distribution animals | Deferred until blind spot profile is known from real-world calibration |

---

*Last Updated: 2026-03-26*
