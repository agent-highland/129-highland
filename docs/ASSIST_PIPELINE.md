# Assist Pipeline — Planning & Architecture

## Status

Working notes — not yet promoted to project-level documentation. Decisions here are directional; implementation follows baseline infrastructure build.

---

## Overview

HA Assist pipeline provides both chat and voice interfaces for home control and queries. Unlike most of the Highland architecture, Assist lives entirely within Home Assistant — it is explicitly HA-dependent. If HA is down, Assist is down. This is acceptable because:

- Critical automations run via Node-RED/MQTT independent of HA
- Assist is a convenience/accessibility layer on top of a resilient control plane
- Making HA itself stable (dedicated HAOS box) is the correct mitigation, not trying to externalize Assist

---

## Pipeline Components

Assist is modular — four loosely-coupled stages, each independently swappable:

| Stage | Purpose | Selected Option |
|-------|---------|-----------------|
| **Wake Word** | Trigger hands-free listening | openWakeWord (local, HA add-on) |
| **STT** | Audio → text | Whisper (local, HA add-on) |
| **Conversation Agent** | Interpret intent, formulate response | Two-tier (see below) |
| **TTS** | Text → audio | Google Cloud TTS (Neural2 tier) |

---

## Conversation Agent: Two-Tier Strategy

Two agents, routed based on use case. HA supports multiple conversation agents simultaneously with per-pipeline routing.

### Tier 1: Local Ollama (Home-Aware)

**Hosted on:** Edge AI SFF (co-located with Coral TPU vision inference)

**Handles:**
- Device control ("turn off the living room lights")
- Home state queries ("is the back door locked?")
- Video analysis calendar queries ("was my package delivered today?")
- Anything requiring access to home context

**Rationale for Edge AI placement:** HAOS is a locked-down OS — Ollama can only run there via addon, inside a constrained supervisor environment. The Edge AI box runs Ubuntu with full Docker access, making it the correct home for arbitrary inference workloads. Ollama (LLM inference) and Coral TPU (vision inference) are distinct workloads that coexist without meaningful resource contention. Network latency to HAOS over LAN is negligible. 32GB RAM on the Edge AI box provides headroom for capable models (13B-32B range) alongside vision inference overhead.

**Note:** For simple, unambiguous commands, HA's built-in intent engine handles the request before it reaches Ollama (faster, no LLM overhead).

### Tier 2: Cloud LLM (General Purpose)

**Integration:** Extended OpenAI Conversation (HACS) — supports Claude and OpenAI-compatible APIs

**Handles:**
- General knowledge queries
- Tasks that benefit from a capable model but don't need home context

---

## TTS: Google Cloud TTS

**Selected voice tier:** Neural2 (sweet spot of quality vs. cost; Studio tier available if Neural2 proves inadequate)

**Key rationale: Unified voice across the system.** Google Cloud TTS is available both in HA (first-party integration) and to Node-RED via standard HTTP API. A single voice ID and API key means HA Assist responses and Node-RED notification TTS utterances sound identical. Pick the voice once; use it everywhere.

**Credentials:** Store API key in `secrets.json` as `google_tts_api_key` (or consolidate under a `google` block if Calendar API key also present).

---

## Satellite Hardware

Satellites handle mic input and speaker output, offloading STT/TTS/inference to the HAOS server via the Wyoming protocol.

### Available Hardware

| Device | Quantity | Protocol | Notes |
|--------|----------|----------|-------|
| M5Stack ATOM Echo | 2 | Wyoming / ESPHome | Audio-only; good for pipeline validation |
| Echo Show (Gen 1, NOS) | 1 | Android (post-LineageOS) | Primary experiment unit, unopened |
| Echo Show 5 | 1 | Android (post-LineageOS) | Secondary experiment unit ($35) |

### Echo Show Experiment

**Goal:** Flash LineageOS → install ViewAssist → evaluate as wall-mounted voice+visual satellite

**Why this matters:** Touchscreen + voice is a significantly better UX than audio-only, particularly for non-ambulatory users. Visual confirmation of responses, ability to display camera feeds, dashboard cards, etc.

**Timing:** Post-baseline-infrastructure. Hardware on hand, not yet opened.

**Risks:**
- LineageOS on Echo Show hardware is community-supported, not mainstream — expect some friction, particularly around mic array drivers
- Proceed with eyes open; not a blocker for anything else

**Fallback:** If Echo Show experiment fails or stalls, Android tablets (e.g., Fire HD with Play Store sideloaded) running ViewAssist provide similar visual+voice UX with less effort. Mic quality will be worse but likely adequate for wall-mounted close-range use. Same software story, less interesting hardware.

### Target Deployment (If Experiment Succeeds)

| Location | Count | Notes |
|----------|-------|-------|
| Living Room | 1 | Open concept — may need 2 depending on coverage |
| Kitchen | 1 | |
| Master Bedroom | 2 | One per nightstand |
| Guest Room One | 1 | |
| Guest Room Two | 1 | |
| Office | 1 | |
| **Total** | **7** | |

Transitional spaces (stairway, hallways) excluded — not worth deploying in spaces where no one stops to have a conversation.

**Sourcing note:** Gen 1 Echo Shows are no longer manufactured. eBay and Facebook Marketplace are primary sourcing channels. When experiment is validated and ready to scale, move quickly rather than leisurely — units in unbootloaderable states or priced by people who know what they have are an increasing share of available inventory.

### ATOM Echoes

Use as audio-only satellites during Echo Show experiment. Gets pipeline end-to-end validated before committing to visual hardware.

---

## Proactive Conversations & Satellite Targeting

### House-Initiated Conversations

Assist supports house-initiated (proactive) conversations — a triggering event causes HA (or NR via a service call) to push a spoken prompt to a satellite, which plays the prompt and then *opens its mic and listens* for a response. The response flows back through the normal STT → conversation agent → action pipeline.

This is distinct from user-initiated flow. The house starts the conversation. For a non-ambulatory user this is significant — the system surfaces things proactively rather than requiring the user to remember to ask.

**Continuing conversations** is a related feature: after an initial wake word trigger and exchange, the satellite remains in listening mode for a follow-up without requiring another wake word. Enables natural multi-turn exchanges ("turn on the living room lights" → "to what brightness?" → "fifty percent") as a single conversation rather than three separate triggers.

*Implementation note: Both features were functional but still maturing as of knowledge cutoff. Check current HA release notes when implementing — this area was moving fast.*

### Satellite Targeting

Proactive prompts and TTS announcements target a **specific satellite entity** (e.g., `assist_satellite.living_room`). You can target one satellite, a group, or all — your choice at the point of the service call. There is no broadcast-by-default behavior.

This enables intelligent targeting by Node-RED: NR knows which room an event occurred in (or, eventually, where a specific person is via presence detection) and can direct a conversation to the appropriate satellite. This is actually a better model than Alexa/Google — the routing logic is yours, running locally, not in someone else's cloud.

**Relevance to accessibility:** For a non-ambulatory user, being able to direct prompts to the satellite in whichever room she's currently in — rather than broadcasting to the whole house or requiring her to be near a specific device — is a meaningful UX improvement worth designing for from the start.

---

## Node-RED Integration Points

NR isn't deeply involved in the Assist pipeline itself, but there are three clean integration points. Understanding the seams matters more than having specific plans for them.

### NR → TTS directly (most common)

When NR has already determined what needs to be said, it calls `tts.speak` targeting a specific satellite/media player. This bypasses the conversation agent entirely — straight to the TTS engine, same Google Cloud voice. This covers the majority of NR notification and announcement use cases.

### NR triggering a proactive Assist conversation

NR detects a triggering event → calls HA service to initiate a proactive conversation on a specific satellite → satellite speaks the prompt and listens → response flows through the full pipeline → NR can optionally subscribe to the outcome event and act on it.

Example: NR detects washing machine cycle complete → initiates conversation on nearest satellite → "The laundry is done. Do you want a reminder in 30 minutes to move it?" → response comes back → NR sets the timer.

### NR → `conversation.process` (less common)

NR can call `conversation.process` to send text directly into the conversation agent and receive a structured response. Effectively uses Assist as an NLU engine. Niche use case — NR is usually the one doing the reasoning — but the capability exists.

### Division of Labor

- **NR owns event detection and triggering logic** — knows what happened, decides whether it warrants a voice interaction, decides which satellite to target
- **Assist owns the voice conversation** — STT, conversation, TTS
- **NR optionally owns follow-through** — if the conversation outcome requires MQTT device control or complex automation, NR may be better placed to execute it

Handoff points: HA service calls (NR → Assist) and HA events (Assist outcome → NR subscribes and reacts).

---

## MQTT Topology Note

The NR → Assist integration points described above (TTS service calls, proactive conversation triggers) are HA service calls, not MQTT events. They do not currently have a `highland/` bus presence.

MQTT_TOPICS.md designates **HA Assist / Voice** as a domain pending definition. If future designs require MQTT surface for Assist interactions — e.g., NR publishing a trigger that initiates a proactive conversation, or Assist outcomes being published for downstream NR flows — those topics must follow the established `highland/` namespace conventions from EVENT_ARCHITECTURE.md and MQTT_TOPICS.md.

---

## Implementation Sequence

All of this follows baseline infrastructure (Communication Hub → HAOS → Workflow) being stable.

1. **Google Cloud TTS** — Configure integration, select voice, validate output quality. Foundational dependency; low risk, do first.
2. **Whisper STT** — Install HA add-on. Enables text *or* voice input via Companion app; no satellite hardware required yet.
3. **Ollama + Conversation Agent** — Stand up on Edge AI box, wire to Assist pipeline via HA's Ollama integration, validate home control and calendar queries via dashboard chat. *Note: Edge AI box must be built and online before this step.*
4. **ATOM Echo satellites** — Get wake word + full audio pipeline end-to-end. Validates hardware path before Echo Show effort.
5. **Echo Show experiment** — LineageOS + ViewAssist. Parallel workstream once baseline infra is stable.
6. **Wall-mounted dashboards** — Separate planning session required (see below).

---

## Deferred: Wall-Mounted Dashboard / Room Displays

Significant topic, not yet planned. Intersection of:
- Satellite hardware selection (Echo Show, tablet, dedicated display)
- Per-room dashboard design (what cards, what context per room)
- Accessibility requirements (primary user is non-ambulatory)
- ViewAssist integration

Deserves a dedicated planning session. Capture ideas here as they emerge; full design deferred.

---

## Companion App as Assist Interface

The HA Companion app is not a satellite in the Wyoming protocol sense, but it is a fully capable Assist interface — both text and voice — connecting directly to HA via WebSocket with Assist built in natively.

**Key distinction from room satellites:** The Companion app carries native user identity. Requests made through it are authenticated as a specific HA user account. This solves the "who asked?" problem that room satellites can only approximate via topology.

**Relevance to advanced interactions:** For use cases like "notify me when a specific person arrives," the Companion app is the cleanest path — NR can store the requesting user alongside the expectation and target the response notification back to that user's mobile device with no ambiguity.

**Additional notes:**
- Works over Nabu Casa for remote access (local satellites do not)
- A tablet running the Companion app is a legitimate Assist interface independent of satellite hardware — relevant for accessibility use cases

*To investigate during implementation: exact behavior of user identity in conversation context, how it surfaces to scripts/NR, and whether it can inform satellite targeting for response delivery.*

---

## Open Questions

- [ ] Ollama model selection — which model(s) for home control vs. general use? Depends on HAOS SFF memory headroom under real load.
- [ ] Google TTS voice selection — Neural2 vs Studio tier; specific voice ID
- [ ] Wake word selection (openWakeWord library vs. custom trained)
- [ ] Per-room pipeline routing — e.g., bedroom satellite uses lower-volume TTS response
- [ ] ATOM Echo placement for initial testing
- [ ] Echo Show LineageOS build process and any known blockers for Gen 1
- [ ] Speaker recognition — monitor [EuleMitKeule/speaker-recognition](https://github.com/EuleMitKeule/speaker-recognition): HA addon + custom integration using Resemblyzer neural voice embeddings. Trains on audio samples per user, returns speaker name + confidence score. Plugs into STT and conversation agent pipeline natively. Early project (low star count) but implementation looks legitimate. Python <3.10 requirement for server capabilities worth noting. If this matures, closes the "who asked?" gap for shared room satellites and enables per-user response targeting without relying on Companion app identity or satellite topology heuristics.

---

*Last Updated: 2026-03-11 — Session working notes, not yet promoted to project docs*
