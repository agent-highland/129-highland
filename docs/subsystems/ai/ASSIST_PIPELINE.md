# Assist Pipeline — Planning & Architecture

## Status

**Architecture defined; implementation pending.** Cross-cutting AI platform decisions (model tier, hardware sizing, hybrid cloud routing) live in `architecture/AI_PLATFORM.md`. Memory architecture lives in `subsystems/ai/PERSISTENT_MEMORY.md`. This doc owns the voice pipeline, satellite strategy, and persona.

**Dependency:** Baseline infrastructure (Communication Hub → HAOS → Node-RED → LLM box) must be stable before beginning.

---

## Overview

HA Assist provides chat and voice interfaces for home control and queries. The pipeline is built on the **Wyoming protocol**, which decouples each stage into independently swappable services that talk over the network. Each stage can run on a different host without architectural impact.

The pipeline is a convenience and accessibility surface on top of Highland's resilient control plane. Critical automations run via Node-RED/MQTT independent of HA, so an Assist outage is non-load-bearing for the house's actual operation.

---

## Pipeline Components

Four loosely-coupled stages, each independently swappable via Wyoming:

| Stage | Purpose | Selected Option |
|-------|---------|-----------------|
| **Wake Word** | Trigger hands-free listening | microWakeWord (on-device, custom "Marvin" model) |
| **STT** | Audio → text | faster-whisper (local) |
| **Conversation Agent** | Interpret intent, formulate response | Local 32B-class via Ollama with NR orchestration |
| **TTS** | Text → audio | Piper local, `en_GB-alan-medium` voice |

Model and hardware rationale is in `architecture/AI_PLATFORM.md`. This doc describes the pipeline structure, not the model choice.

---

## Persona: Marvin

The assistant's personality is **Marvin from The Hitchhiker's Guide to the Galaxy** — the depressive paranoid android. Dry wit, mild existential despair, brain-the-size-of-a-planet condescension delivered with weary politeness. The persona is a deliberate choice for tone enjoyment, not a functional requirement.

### Persona Lives in the System Prompt

The persona is implemented entirely in the LLM's system prompt. The voice itself (`en_GB-alan-medium`) is just a neutral British male — clear, slightly formal. The character comes from how the model writes its responses. This separation matters:

- Voice and persona can be tuned independently
- The persona can be adjusted (more or less dour) without changing voice models
- Other voice models can substitute later without losing the character

### Wake Word: "Marvin"

A custom wake word matching the persona. Implementation considerations:

- **microWakeWord** on the voice satellite is the target — runs on ESP32-S3 class hardware, on-device detection, only streams audio after activation
- A community-trained "Marvin" model exists in the HA wakeword collection but has had training quality issues (early versions trained on misspelled phonetic input). A fresh custom-trained model via HA's Colab training notebook may produce better results
- **openWakeWord** is a fallback path — runs centrally, more model flexibility, but requires continuous audio stream from satellite to server (worse for privacy and bandwidth)

Initial deployment plan: train a custom microWakeWord "Marvin" model, validate detection rate and false-trigger rate over a few days of real use, fall back to openWakeWord if microWakeWord proves insufficient.

---

## Conversation Agent

### Local LLM via Ollama

The conversation agent is a 32B-class model hosted on the dedicated LLM box. Tier rationale, hardware target, and voice latency budget are in `architecture/AI_PLATFORM.md`.

### Orchestration Layer

The LLM does not run in isolation. Memory retrieval (from the conversational memory store), seed knowledge retrieval (from the document store), and tool call dispatch are all handled by an orchestration layer between HA and Ollama.

The orchestrator pattern: HA Assist points at the orchestrator's endpoint instead of Ollama directly. The orchestrator:

1. Receives the prompt from HA Assist
2. Embeds the query, retrieves relevant memory and seed knowledge from `pgvector`
3. Assembles the augmented prompt (system prompt + memory block + seed knowledge block + conversation)
4. Forwards to Ollama
5. Captures the response and any tool calls
6. Dispatches tool calls (most via Node-RED → MQTT for HA control; some via direct service calls)
7. Returns the final response to HA Assist for TTS

This is the same pattern as the previous architecture's "proxy" concept, but the orchestrator can be implemented as a Node-RED HTTP endpoint rather than a separate Python service. See `subsystems/ai/PERSISTENT_MEMORY.md` for full detail on the memory side.

### Hybrid Cloud Routing — Open Question

Routing some queries to a cloud LLM (deep reasoning, niche world knowledge) is a deferred architectural decision. Full discussion in `architecture/AI_PLATFORM.md`. Initial deployment is local-only; hybrid is a lever to pull post-deployment if Tier 2 limitations bite in real use.

---

## TTS: Piper Local

### Why Piper

Piper is the canonical local TTS option in the HA voice ecosystem. CPU-only, fast, lightweight (~30–100MB voice models), good quality at the medium and high voice pack tiers. Runs on `workflow.local` alongside other lightweight services.

Higher-quality options exist (Kokoro, XTTS-v2 with voice cloning) but introduce more compute cost and operational complexity. Piper hits the "good enough you stop noticing it" bar with effectively zero resource cost.

### Voice Selection: `en_GB-alan-medium`

Neutral British male, fits the Marvin persona. **Medium voice pack minimum** — low-quality voice packs stumble on punctuation and emphasis in ways that break immersion. High-quality voice packs are preferred where available for the chosen voice.

### Voice Unification

A single Piper voice serves all TTS sources — HA Assist responses, Node-RED-triggered TTS notifications, proactive conversations. There is no per-caller voice configuration to keep in sync. Whoever calls Piper gets the same Marvin voice.

This was a key rationale for the previous Google Cloud TTS choice; Piper preserves that property while removing the cloud dependency.

---

## Satellite Hardware

Satellites handle mic input and speaker output, offloading STT/TTS/inference to the LLM box and HA via the Wyoming protocol.

### Selected Path: HA Voice Preview Edition (Primary)

**HA Voice PE** is the path of least resistance: official HA hardware, ESP32-S3 with mic array and speaker, on-device wake word, Wyoming-native, plug-and-play. Roughly $60/unit. No DIY assembly, no firmware tinkering required for baseline functionality.

### Alternative: M5Stack ATOM Echo (DIY / Test)

ATOM Echo is the cheap-and-cheerful DIY satellite — ESP32-based, ESPHome integration, more setup work but more customizable. Useful for initial pipeline validation before committing to the Voice PE rollout, and for spaces where the form factor matters.

### Parallel Project: Echo Show via LineageOS

Two pairs of Echo Show units (1st gen 2017 original, and 1st gen 2019 "checkers/crown" generation) on hand. The LineageOS port for the 2019 generation is active but has known limitations: HA dashboards run well; **microphone support is partial** — gain is low and recordings can use the wrong sample rate. As voice satellites *today*, they're a science project.

The path forward: HA Companion App on Android already supports microWakeWord on-device. Once the LineageOS mic issues get sorted upstream, Companion App + microWakeWord turns these devices into voice satellites with display surfaces. Until then, they earn their keep as dashboard surfaces (View Assist for visual feedback during voice interactions is worth evaluating).

**Approach:** Run as a parallel workstream. Don't block the main voice deployment on Echo Show readiness. Watch the LineageOS thread for mic fixes; revisit when ready.

### Target Deployment

**v1: four rooms covering daily-use surfaces.**

| Location | Count | Rationale |
|----------|-------|-----------|
| Living Room | 1 | Primary social/leisure space |
| Kitchen | 1 | Hands-busy queries (timers, recipes, "what's the temp outside") |
| Master Bedroom | 1 | Lights/alarm/bedtime routines |
| Office | 1 | Daytime use during work |
| **Total** | **4** | |

**v1 cost target:** ~$240 in Voice PE units.

**Future expansion** — driven by observed need:
- Guest rooms (when frequent guests need voice access)
- Bathrooms, basement, garage (low priority; surface only if usage justifies)

The kitchen is the room where a display surface earns its keep most clearly — timers, recipe scrollback, weather glance while hands are full. If the Echo Show / LineageOS path matures, kitchen is the natural pilot location for a display+voice satellite.

---

## Proactive Conversations & Satellite Targeting

### House-Initiated Conversations

The pipeline supports house-initiated (proactive) conversations: a triggering event causes Node-RED to push a spoken prompt to a specific satellite, which plays the prompt and opens its mic to listen for a response.

**Continuing conversations** — after the initial wake-word trigger, the satellite remains in listening mode for follow-up turns without requiring another wake word.

### Satellite Targeting

Proactive prompts and TTS announcements target a **specific satellite entity** (e.g., `assist_satellite.living_room`). No broadcast-by-default. NR knows which room an event occurred in (via existing area-aware flows) and directs conversations to the appropriate satellite.

**Accessibility consideration:** for users with limited mobility, directing prompts to the room they're currently in — rather than broadcasting to the whole house — is a meaningful UX improvement. Worth designing for from the start, even if v1 deployment doesn't fully exercise it.

### Push vs Pull Patterns

Voice interactions split into two architectural patterns with different needs:

| Pattern | Trigger | Targeting | Example |
|---------|---------|-----------|---------|
| **Pull** | User wakes assistant | Wherever the user is | "Marvin, turn off the kitchen lights" |
| **Push** | NR event | Where the user is, or where the event happened | "The laundry is done. Want a reminder in 30 minutes?" |

Push patterns require **presence-aware targeting** — announcing "the laundry is done" to an empty kitchen is useless. The targeting logic sits in Node-RED, on top of the voice pipeline rather than inside it.

---

## Node-RED Integration Points

### NR → TTS directly (most common)

When NR has already determined what needs to be said, it calls `tts.speak` targeting a specific satellite/media player. This bypasses the conversation agent entirely — straight to Piper, same Marvin voice.

### NR triggering a proactive Assist conversation

NR detects a triggering event → calls HA service to initiate a proactive conversation on a specific satellite → satellite speaks the prompt and listens → response flows through the full pipeline → NR follows through on the response if needed.

Example: NR detects washing machine cycle complete → initiates conversation on nearest satellite where someone is present → "The laundry is done. Do you want a reminder in 30 minutes to move it?" → response routes back through STT and the conversation agent → if "yes", NR sets the reminder.

### NR → `conversation.process` (less common)

NR can call `conversation.process` to send text directly into the conversation agent and receive a structured response — using Assist as an NLU engine for non-voice flows.

### Division of Labor

- **NR owns** event detection, triggering logic, satellite targeting, presence awareness
- **Assist owns** the voice conversation flow (STT → orchestrator → TTS)
- **Orchestrator owns** memory retrieval, prompt assembly, tool call dispatch
- **NR owns** follow-through when a conversation outcome requires MQTT device control or scheduled actions

---

## Persistent Memory Integration

The orchestrator layer is where persistent memory plugs in. Every prompt that flows through the orchestrator gets memory and seed-knowledge augmentation before reaching the LLM. Every response gets evaluated for fact extraction.

Full memory architecture, ingestion mechanics, retrieval flow, and safety framework live in `subsystems/ai/PERSISTENT_MEMORY.md`. The pipeline doc references but does not duplicate that content.

---

## Implementation Sequence

All of this follows baseline infrastructure being stable.

1. **Piper TTS** — install on `workflow.local`, select voice pack quality, validate output through HA `tts.speak` and direct API calls from NR
2. **Whisper STT** — install (HA add-on or standalone Wyoming service), validate via HA Companion App text-to-speech round-trip
3. **Voice PE satellite (single unit)** — get one room end-to-end with built-in HA conversation agent (no orchestrator yet) to validate the audio pipeline
4. **Custom Marvin wake word** — train via HA's Colab notebook, deploy to satellite, validate detection rate
5. **LLM box online** — see `architecture/AI_PLATFORM.md` for hardware path; install Ollama with target 32B model
6. **Orchestrator layer (v0)** — minimal HTTP endpoint forwarding to Ollama, no memory yet; verify HA Assist reaches it correctly
7. **Memory layer integration** — see `subsystems/ai/PERSISTENT_MEMORY.md` for full sequence
8. **Voice PE rollout to remaining v1 rooms** — kitchen, master, office
9. **Echo Show experiment** — parallel workstream, not blocking, surface as ready

---

## Wall-Mounted Dashboard / Room Displays

Significant topic, not yet planned. Intersection of satellite hardware selection, per-room dashboard design, accessibility requirements, and ViewAssist integration. Deserves a dedicated planning session post-baseline.

---

## Open Questions

- [ ] Specific Piper voice pack — confirm `en_GB-alan-medium` is the best fit, evaluate high-quality alternatives
- [ ] Custom Marvin wake word — train fresh vs adapt community model
- [ ] Orchestrator implementation — Node-RED HTTP endpoint vs separate Python service vs HA custom integration
- [ ] STT model size — `small` vs `medium` faster-whisper, run on LLM GPU vs `workflow.local` CPU
- [ ] Voice PE placement specifics within each room
- [ ] Echo Show LineageOS mic fix monitoring
- [ ] Speaker recognition production readiness — monitoring `EuleMitKeule/speaker-recognition` (Resemblyzer-based HA addon)

---

## Related Documents

| Document | Content |
|----------|---------|
| `architecture/AI_PLATFORM.md` | LLM tier, hardware, hybrid cloud question, voice latency budget |
| `subsystems/ai/PERSISTENT_MEMORY.md` | Memory architecture, RAG, knowledge ingestion, safety framework |
| `architecture/OVERVIEW.md` | Four-box infrastructure, where the LLM box fits |
| `nodered/NOTIFICATIONS.md` | Notification framework (relevant for Push patterns) |

---

*Last Updated: 2026-05-08*
