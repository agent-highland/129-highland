# Assist Pipeline — Planning & Architecture

## Status

**Working notes — directional, not final.** Implementation follows baseline infrastructure build.

**Dependency:** Baseline infrastructure (Communication Hub → HAOS → Node-RED → Edge AI box) must be stable before beginning.

---

## Overview

HA Assist pipeline provides both chat and voice interfaces for home control and queries. Unlike most of Highland's architecture, Assist lives entirely within Home Assistant — it is explicitly HA-dependent. If HA is down, Assist is down. This is acceptable because:

- Critical automations run via Node-RED/MQTT independent of HA
- Assist is a convenience/accessibility layer on top of a resilient control plane
- Making HA itself stable (dedicated HAOS box) is the correct mitigation

---

## Pipeline Components

Four loosely-coupled stages, each independently swappable:

| Stage | Purpose | Selected Option |
|-------|---------|-----------------|
| **Wake Word** | Trigger hands-free listening | openWakeWord (local, HA add-on) |
| **STT** | Audio → text | Whisper (local, HA add-on) |
| **Conversation Agent** | Interpret intent, formulate response | Two-tier (see below) |
| **TTS** | Text → audio | Google Cloud TTS (Neural2 tier) |

---

## Conversation Agent: Two-Tier Strategy

### Tier 1: Local Ollama (Home-Aware)

**Hosted on:** Edge AI SFF (co-located with Coral TPU vision inference)

**Handles:** Device control, home state queries, video analysis calendar queries, anything requiring access to home context.

**Rationale for Edge AI placement:** HAOS is a locked-down OS — Ollama can only run there via addon in a constrained supervisor environment. The Edge AI box runs Ubuntu with full Docker access. Ollama (LLM inference) and Coral TPU (vision inference) are distinct workloads that coexist without meaningful resource contention. 32GB RAM provides headroom for capable models (13B–32B range).

### Tier 2: Cloud LLM (General Purpose)

**Integration:** Extended OpenAI Conversation (HACS) — supports Claude and OpenAI-compatible APIs

**Handles:** General knowledge queries, tasks that benefit from a capable model but don't need home context.

---

## TTS: Google Cloud TTS

**Selected tier:** Neural2 (sweet spot of quality vs. cost; Studio tier available if Neural2 proves inadequate)

**Key rationale:** Google Cloud TTS is available both in HA (first-party integration) and to Node-RED via standard HTTP API. A single voice ID and API key means HA Assist responses and Node-RED notification TTS utterances sound identical. Pick the voice once; use it everywhere.

**Credentials:** Store API key in `secrets.json` as `google_tts_api_key`.

---

## Satellite Hardware

Satellites handle mic input and speaker output, offloading STT/TTS/inference to the HAOS server via the Wyoming protocol.

| Device | Quantity | Protocol | Notes |
|--------|----------|----------|-------|
| M5Stack ATOM Echo | 2 | Wyoming / ESPHome | Audio-only; good for pipeline validation |
| Echo Show (Gen 1, NOS) | 1 | Android (post-LineageOS) | Primary experiment unit, unopened |
| Echo Show 5 | 1 | Android (post-LineageOS) | Secondary experiment unit |

### Echo Show Experiment

**Goal:** Flash LineageOS → install ViewAssist → evaluate as wall-mounted voice+visual satellite

**Why this matters:** Touchscreen + voice is a significantly better UX than audio-only, particularly for non-ambulatory users. Visual confirmation of responses, ability to display camera feeds, dashboard cards.

**Timing:** Post-baseline-infrastructure. Hardware on hand.

**Fallback:** If Echo Show experiment fails or stalls, Android tablets running ViewAssist provide similar UX with less effort. Mic quality will be worse but likely adequate for wall-mounted close-range use.

### Target Deployment (If Experiment Succeeds)

| Location | Count |
|----------|-------|
| Living Room | 1 |
| Kitchen | 1 |
| Master Bedroom | 2 (one per nightstand) |
| Guest Room One | 1 |
| Guest Room Two | 1 |
| Office | 1 |
| **Total** | **7** |

Transitional spaces (stairway, hallways) excluded — not worth deploying where no one stops to have a conversation.

**Sourcing note:** Gen 1 Echo Shows are no longer manufactured. eBay and Facebook Marketplace are primary sourcing channels. When experiment is validated and ready to scale, move quickly — units in unbootloaderable states or priced by people who know what they have are an increasing share of available inventory.

---

## Proactive Conversations & Satellite Targeting

### House-Initiated Conversations

Assist supports house-initiated (proactive) conversations — a triggering event causes NR (via HA service call) to push a spoken prompt to a satellite, which plays the prompt and then opens its mic and listens for a response.

**Continuing conversations** is a related feature: after an initial wake word trigger and exchange, the satellite remains in listening mode for a follow-up without requiring another wake word.

### Satellite Targeting

Proactive prompts and TTS announcements target a **specific satellite entity** (e.g., `assist_satellite.living_room`). No broadcast-by-default. NR knows which room an event occurred in and can direct a conversation to the appropriate satellite. This is actually a better model than Alexa/Google — the routing logic is yours, running locally.

**Accessibility relevance:** For a non-ambulatory user, directing prompts to whichever room she's currently in — rather than broadcasting to the whole house — is a meaningful UX improvement worth designing for from the start.

---

## Node-RED Integration Points

### NR → TTS directly (most common)

When NR has already determined what needs to be said, it calls `tts.speak` targeting a specific satellite/media player. This bypasses the conversation agent entirely — straight to the TTS engine, same Google Cloud voice.

### NR triggering a proactive Assist conversation

NR detects a triggering event → calls HA service to initiate a proactive conversation on a specific satellite → satellite speaks the prompt and listens → response flows through the full pipeline.

Example: NR detects washing machine cycle complete → initiates conversation on nearest satellite → "The laundry is done. Do you want a reminder in 30 minutes to move it?" → response comes back → NR sets the timer.

### NR → `conversation.process` (less common)

NR can call `conversation.process` to send text directly into the conversation agent and receive a structured response — effectively using Assist as an NLU engine.

### Division of Labor

- **NR owns** event detection, triggering logic, satellite targeting
- **Assist owns** the voice conversation (STT, conversation, TTS)
- **NR optionally owns** follow-through when the conversation outcome requires MQTT device control or complex automation

---

## Deferred: Persistent Memory

**Status: Blocked — see `subsystems/ai/PERSISTENT_MEMORY.md` for full details.**

**Blockers:**
1. HA pipeline events (`assist_pipeline_event`) do not propagate to the external WebSocket API or HA event bus. Verified on HA 2026.3.1. Clean per-interaction capture requires NR to own conversation orchestration entirely, talking to Ollama directly.
2. No viable local LLM inference hardware with GPU acceleration. The Edge AI box PCIe slot is occupied by Coral TPU. CPU-only inference on i7-7700 produces marginal latency for voice use.

*Revisit triggers: HA exposes pipeline events externally, or dedicated LLM inference hardware becomes viable.*

---

## Deferred: Wall-Mounted Dashboard / Room Displays

Significant topic, not yet planned. Intersection of satellite hardware selection, per-room dashboard design, accessibility requirements, and ViewAssist integration. Deserves a dedicated planning session.

---

## Implementation Sequence

All of this follows baseline infrastructure being stable.

1. **Google Cloud TTS** — Configure integration, select voice, validate output quality
2. **Whisper STT** — Install HA add-on. Enables text or voice input via Companion app; no satellite hardware required yet
3. **Ollama + Conversation Agent** — Stand up on Edge AI box, wire to Assist pipeline, validate home control and calendar queries via dashboard chat
4. **ATOM Echo satellites** — Get wake word + full audio pipeline end-to-end
5. **Echo Show experiment** — LineageOS + ViewAssist; parallel workstream once baseline infra is stable

---

## Open Questions

- [ ] Ollama model selection — which model(s) for home control vs. general use?
- [ ] Google TTS voice selection — Neural2 vs Studio tier; specific voice ID
- [ ] Wake word selection (openWakeWord library vs. custom trained)
- [ ] ATOM Echo placement for initial testing
- [ ] Echo Show LineageOS build process and any known blockers for Gen 1
- [ ] Speaker recognition — monitoring [EuleMitKeule/speaker-recognition](https://github.com/EuleMitKeule/speaker-recognition): HA addon using Resemblyzer neural voice embeddings. Trains on audio samples per user, returns speaker name + confidence score. Early project but implementation looks legitimate. If this matures, closes the "who asked?" gap for shared room satellites.

---

*Last Updated: 2026-03-26*
