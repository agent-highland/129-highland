# Landroid Integration

Integration of Worx Landroid Vision robotic mower into Highland via MQTT bridge.

**Status:** 📋 Planned — hardware in-hand, base not yet sited. MQTT bridge and camera implementation deferred until first mow season.

---

## Hardware

**Worx Landroid Vision 1-Acre 4WD (WR344)**

- Wire-free boundary detection via camera/AI (no perimeter wire required)
- Built-in WiFi; communicates exclusively via Worx cloud (AWS IoT Core)
- No local API
- Planned location: side yard (late afternoon direct sun only)

**Reolink Argus Eco Ultra + Solar Panel** *(to be purchased)*

- 4K, 125° FOV, color night vision
- Solar/battery powered, WiFi
- Strap mounts included for both camera and solar panel — supports tree or pole mounting where no fixed structure is available
- Connects to the RLN16-410 NVR over WiFi; NVR exposes RTSP stream to the Highland video pipeline identically to PoE cameras

---

## Integration Philosophy

Connectivity is **additive nicety, not a requirement.** The mower functions fully as a standalone appliance without any Highland integration. If the bridge breaks temporarily or permanently, the mower keeps mowing on its own schedule. This shapes every integration decision — keep it thin, keep it optional, don't build dependencies on it.

---

## Architecture

### How Worx MQTT Works

The mower communicates exclusively with AWS IoT Core (MQTT over TLS, port 8883). Worx's REST API (`api.worxlandroid.com`) provides:

- Authentication
- Per-account TLS certificate (valid through 2050, same cert for all clients)
- AWS IoT endpoint hostname
- Device serial number and topic prefix

The mower publishes status payloads to `<prefix>/<serial>/commandOut` and receives command payloads on `<prefix>/<serial>/commandIn`. Status is published on a ~10-minute heartbeat and immediately in response to any inbound command.

### Bridge Approach: Mosquitto Bridge

The preferred integration path is a **Mosquitto bridge** on the Communication Hub — an additional bridge config that connects the existing Mosquitto instance to Worx's AWS IoT endpoint. No new services, no new containers.

```
Mower ──► AWS IoT Core ──► Mosquitto Bridge ──► highland/state/landroid/#
                                             ◄── highland/command/landroid/#
```

The bridge subscribes to `<prefix>/<serial>/commandOut` from AWS and republishes locally. Commands flow in reverse. From Node-RED's perspective, the mower is just another MQTT device.

**Setup requirements (one-time):**
1. Authenticate against the Worx REST API to retrieve the TLS cert and AWS endpoint
2. Write a `landroid-bridge.conf` for Mosquitto (additional bridge, not modifying the main config)
3. Cert files live in `/etc/mosquitto/certs/landroid/` on the Communication Hub

**Known fragility:** Worx has historically rotated certs or changed API responses without notice. If the bridge goes silent, the cert retrieval step may need to be re-run. This is acceptable given the "additive nicety" posture.

### Alternative: `pyworxcloud` Python Bridge

`pyworxcloud` is a reverse-engineered Python library (used by the HACS HA integration) that handles auth and AWS MQTT, republishing to a local broker via a thin script. More portable than the Mosquitto bridge approach but adds a Python runtime dependency. Only pursue this if the Mosquitto bridge proves unworkable.

### Camera Integration Path

```
Argus Eco Ultra (WiFi) ──► RLN16-410 NVR ──► RTSP ──► Highland video pipeline
```

The NVR is the integration point. WiFi cameras connect to the NVR over the network; the NVR exposes RTSP streams identically for both WiFi and PoE cameras. The Argus Eco Ultra is a fully first-class participant in the video pipeline — no Home Hub, no separate handling, no special cases.

---

## MQTT Topics

| Topic | Direction | Retained | Notes |
|-------|-----------|----------|-------|
| `highland/state/landroid/status` | Publish | Yes | Normalized mower state |
| `highland/event/landroid/error` | Publish | No | Error state changes |
| `highland/command/landroid/control` | Subscribe | No | Start / stop / return to base |

Raw AWS topics (`<prefix>/<serial>/commandOut`) are consumed by the bridge and never exposed directly on the Highland namespace.

---

## Normalized State

The raw Worx MQTT payload is JSON and contains a large number of fields. A Node-RED normalization node extracts the relevant subset:

| Field | Source | Notes |
|-------|--------|-------|
| `state` | `dat.ls` | See state code map below |
| `error` | `dat.le` | Error code; 0 = no error |
| `locked` | `dat.lk` | 1 = lock enabled |
| `battery_pct` | `dat.bt.p` | 0–100 |
| `battery_charging` | `dat.bt.c` | Boolean |
| `rain_delay_active` | `dat.rain.s` | Boolean |
| `rain_delay_remaining_min` | `dat.rain.cnt` | Minutes |
| `last_seen` | derived | Timestamp of last received payload |

**State codes (`dat.ls`):**

| Code | Meaning |
|------|---------|
| 1 | Home / Idle |
| 2 | Error |
| 3 | Mowing |
| 4 | Leaving base |
| 5 | Going home |
| 7 | Charging |
| 8 | Searching wire |
| 32 | Cutting edge |
| 33 | Searching home |

*These are known codes from community reverse engineering. Validate against observed payloads once hardware is in-hand.*

---

## Security & Anti-Theft

### Threat Model

Two distinct theft vectors:

**Vector 1 — Stolen from dock (mower charging/idle)**
The mower is sitting on the base between mow sessions. Someone picks it up and walks off.

**Vector 2 — Stolen while running (mower actively mowing)**
Someone intercepts the mower mid-cycle in the yard.

### On-Device Defenses

The mower has built-in security independent of any Highland integration:

- **Lift sensor** — physical tilt/lift sensor always active when powered on; stops blades immediately on lift regardless of mowing state
- **Security PIN** — locks the mower; wrong PIN triggers audible alarm
- **Lock function** — when enabled, triggers an audible alarm if the mower is lifted and carried outside the yard perimeter (GPS/geofence-based on Vision models, not wire-based)
- **WiFi kill** — mower stops operating after three consecutive days outside WiFi coverage

The WA0865 alarm module (sold separately) adds a dedicated high-decibel alarm with its own backup battery, ensuring the alarm fires even during charging when the main battery could theoretically be removed.

### MQTT Signal: `dat.le` (Lift Error)

The lift sensor populates `dat.le` (error code field) with a non-zero value when the mower is lifted unexpectedly. This fires in both theft vectors — the sensor is always active when the unit is powered on.

**Critical limitation for the docked-theft vector:** The mower heartbeats to AWS approximately every 10 minutes. While actively mowing it communicates more frequently and an error state likely triggers an immediate publish. While docked and charging it may be in a lower-activity comms state, meaning the MQTT message could lag the physical lift event by up to ~10 minutes. Additionally, if the mower is carried out of WiFi range, it goes silent on MQTT immediately. The MQTT notification is therefore a **corroborating signal**, not a primary real-time alarm.

**`dat.lk` (lock state)** is also worth monitoring — a transition to unlocked without a known authorized action is an additional signal.

*Note: Community error code tables were reverse-engineered from wire-based models. The exact `dat.le` value for the lifted state on Vision-series hardware needs validation against observed WR344 payloads.*

### Dedicated Security Camera

A **Reolink Argus Eco Ultra** pointed at the charging base serves as the primary real-time detection layer, directly addressing the MQTT timing gap for the docked-theft vector.

**Why this works well for a fixed-asset use case:**

The mower base is a fixed, known scene. Unlike general perimeter surveillance where the LLM must interpret an arbitrary scene, the camera here has a single binary question: *is a person interacting with the mower?* This makes analysis cheaper, faster, and higher-confidence.

The prompt pattern:

> *"This is a fixed camera pointed at a robotic lawn mower on its charging base. Does the image show a person approaching, touching, lifting, or otherwise interacting with the mower? The mower should be stationary and unattended. Answer yes/no and briefly describe what you see."*

**Pipeline:**

Fits directly into the existing video pipeline's three-stage ladder (see `subsystems/VIDEO_PIPELINE.md`):

1. **CPAI triage** — person detection gate; no person in frame → discard immediately
2. **Gemini snapshot analysis** — focused prompt above; yes → escalate
3. **Notification** — immediate push via Utility: Notifications with keyframe attached

**Two-signal confirmation:**

| Signal | Covers Vector 1 (docked) | Covers Vector 2 (running) | Latency |
|--------|--------------------------|---------------------------|---------|
| Camera (person detection) | ✅ | ✅ (if in frame) | Seconds |
| `dat.le` MQTT lift error | ✅ (with delay caveat) | ✅ (likely immediate) | 0–10 min |

Camera fires first; `dat.le` confirms the lift happened. Either signal alone warrants a notification; both together warrant an urgent alert.

**Camera siting — to resolve once base is placed:**

- Mount at roughly 6–8 feet; higher than that and the camera is staring at the tops of heads rather than capturing the human-object interaction
- Position at ~45° to the side of the base rather than directly in front or behind — gives visibility of both approach and the lift action as a coherent sequence
- FOV should capture a 10–15 foot approach radius, not just the base itself — the goal is to detect someone walking toward the mower before they reach it
- Avoid pointing into late afternoon sun (side yard context) — if solar panel orientation and lens direction conflict, favor the lens angle and accept slightly lower solar yield; afternoon sun in this location is strong enough that an oblique panel angle will still charge adequately
- Strap mounts support tree or pole mounting where no fixed structure is available

**Authorized interaction handling:**

Accept false positives during legitimate maintenance rather than building suppression logic. Time-of-day context handles most of the ambiguity: a 3am alert is categorically different from a 2pm one.

---

## Planned Automations

### Rain Delay Awareness

Node-RED monitors `rain_delay_active`. If a mow session was expected but rain delay is active, include in the Daily Digest or send a low-priority notification. No action required — informational only.

### Error Notification

When `error` transitions from 0 to non-zero, send a notification via Utility: Notifications. Include the error code and a human-readable description. Map known error codes to friendly strings; unknown codes fall through with the raw value.

### Anti-Theft Alert

When the camera pipeline detects a person interacting with the mower base, send an urgent push notification with the keyframe. If `dat.le` also transitions to a lift error code within a configurable window (suggest 15 minutes), escalate to a second notification confirming the lift occurred. Single-signal (camera only) is informational; dual-signal is urgent.

### Calendar Suppression

If an active calendar suppression is in effect (guests, parties, etc.), send a pause command via `highland/command/landroid/control`. Resume when suppression clears. Low priority — the mower's built-in schedule handles most cases.

### Daily Digest Integration

Include mower status summary in the Daily Digest: last mow time, current state, battery level, any active errors or rain delay.

---

## Open Questions

- Exact topic prefix for WR344 — confirm from actual API response (community reports `PRM100` but this may be model-dependent)
- Whether Vision-series models use the same MQTT protocol as older Landroid models — needs validation once hardware is in-hand
- Exact `dat.le` error code for the lifted state on WR344 — validate against observed payloads
- Whether `dat.le` transitions immediately while docked, or only during active mow — validate once hardware is in-hand
- Rain delay threshold strategy: defer to mower's own logic or supplement with NWS/Tempest forecast data?
- Camera mount point — TBD once base is sited; tree/pole strap mount likely given side yard location

---

## Implementation Notes

- Do not implement the MQTT bridge until base is sited and at least one mow cycle has been observed
- Run cert extraction manually first to validate before writing the Mosquitto bridge config
- Camera siting should be decided concurrently with base placement — treat them as a paired decision
- Treat the MQTT bridge as optional infrastructure — a failure here should never produce noise unless it's been working and then stops
- The camera integrates via RLN16-410 NVR over WiFi; no special handling required beyond normal video pipeline onboarding

---

*Last Updated: 2026-04-13*
