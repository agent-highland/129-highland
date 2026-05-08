# AI Platform — Cross-Cutting Decisions

## Status

**Architecture defined; implementation pending.** This is the foundational decision layer for Highland's conversational AI. Voice pipeline (`subsystems/ai/ASSIST_PIPELINE.md`) and memory architecture (`subsystems/ai/PERSISTENT_MEMORY.md`) build on the choices captured here.

---

## Scope

This document covers the **conversational AI platform** — the LLM, hardware target, and cross-cutting decisions that shape both the voice pipeline and the memory architecture.

It does **not** cover:
- Computer vision inference (CodeProject.AI + Coral TPU on the Edge AI box) — see `subsystems/VIDEO_PIPELINE.md`
- Voice pipeline mechanics (STT, TTS, satellites, wake word) — see `subsystems/ai/ASSIST_PIPELINE.md`
- Memory storage, retrieval, and seed knowledge management — see `subsystems/ai/PERSISTENT_MEMORY.md`

The platform decisions here are deliberately separated from those subsystem details so the rationale lives in one place rather than being duplicated or drifting between docs.

---

## Why Local AI

The conversational assistant interacts with privacy-sensitive context: who is home, what they're doing, household routines, personal data. Three forces push toward local inference:

1. **Privacy.** Routine assistant queries about the house should not leave the network. Cloud LLM providers see — and frequently retain — the full prompt, including injected memory context.
2. **Independence.** The assistant should remain functional during cloud outages and rate-limit windows. It's a daily-use surface; reliability matters.
3. **Cost predictability.** Per-token API costs scale with usage. Local inference is fixed-cost (hardware + electricity), which fits a household-scale workload better than metered API access.

The cost argument is real but secondary — it would not justify local-only on its own. Privacy and independence are the load-bearing reasons.

---

## Capability Tier

Local LLM capability is best understood in three tiers, each defined by what the underlying model can reliably do:

| Tier | Model Class | Use Case | VRAM Target |
|------|-------------|----------|-------------|
| **Tier 1** | 3B–8B | Intent parsing only ("turn off the lights") | 4–8GB or CPU |
| **Tier 2** | 14B–32B | Real conversational assistant with tool calling | 16–24GB |
| **Tier 3** | 70B+ | Genuine smart assistant with deep reasoning | 48GB+ or multi-GPU |

### Selected: Tier 2 (32B-class)

The smart-assistant vision — persistent memory, tool calling, deep HA integration, basic world knowledge — is firmly Tier 2 minimum. Tier 1 is sufficient only for utterance-to-service-call mapping and falls apart on real conversation. Tier 3 is noticeably better but with sharply diminishing returns relative to cost (roughly 3× spend for ~30% better capability) and worse latency for voice use.

### What Tier 2 Buys

- **Reliable tool calling** — ~85–90% accuracy on a small toolset; degrades as tool count grows but acceptable for HA's domain
- **Two-step reasoning** — chains like "check sensors → summarize state" work; deeper chains need hand-holding
- **Decent world knowledge** — common topics handled well; niche topics spotty; honest about uncertainty when prompted correctly
- **Acceptable voice latency** — first token ~500ms, completion 1–2s on appropriate hardware

### What Tier 2 Does Not Buy

- Bulletproof tool calling on a large tool surface
- Multi-step (3+) conditional reasoning chains
- Reliable obscure-domain knowledge
- The "feels like talking to a competent assistant" threshold of Tier 3

These are accepted tradeoffs given expected usage patterns. The Google Hub trajectory — heavy novelty use, then settling into a small set of habit-forming patterns — is the realistic reference frame, and Tier 2 is appropriately sized for that reality.

### Upgrade Path

Tier 2 is the starting point, not a permanent ceiling. If usage patterns demonstrate sustained Tier 3 demand, upgrade paths exist:
- Second GPU alongside the first (dual-3090 → 70B Q4)
- Replacement with higher-VRAM card (5090 32GB, future generations)
- Hybrid cloud routing (see below) covering deep-reasoning queries while local handles routine work

The decision to upgrade is reversible and cheap to defer; over-investing before validating usage is the more expensive mistake.

---

## Hardware Target

### Host

The existing ATX mid-tower repurposed for LLM inference:

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 5 3600 |
| RAM | 64GB DDR4 |
| Storage | 1TB SSD + 4TB HDD |
| GPU (current) | RX 580 |
| GPU (target) | **NVIDIA RTX 3090 24GB (used market)** |

### GPU Decision: Used 3090 24GB

**Replaces** the previously planned RTX 3060 12GB upgrade. The capability difference is meaningful:

| Card | VRAM | Tier Reach (Q4) | Used Market |
|------|------|-----------------|-------------|
| RTX 3060 | 12GB | 13B comfortably | ~$200–250 |
| **RTX 3090** | **24GB** | **32B comfortably** | **~$700–800** |

The 3060's 12GB ceiling caps inference at 13B-class models — well below the 32B-class Tier 2 capability target. The 3090's 24GB headroom comfortably hosts 32B Q4, with ~1–2GB to spare for a co-resident embedding model (~500MB) and Whisper STT (~1–2GB) if those services are co-located rather than CPU-hosted.

The cost delta (~$500) is real but measured against expected daily use over multi-year hardware lifetime, the per-day cost is trivial. The capability cliff between 13B and 32B is steep enough that paying for the larger card up front is the right call.

**ROCm vs CUDA:** ROCm support on RX 580 is inconsistent. CUDA on RTX 3090 is mature and well-supported across all relevant tooling (Ollama, faster-whisper, embedding models). Eliminating ROCm fragility is a side benefit of the GPU swap.

### Why VRAM Matters More Than Compute

Local LLM performance is gated almost entirely by VRAM capacity, not raw compute. A model that fits cleanly in VRAM runs at memory bandwidth speed; a model that spills to system RAM or swap runs at glacial speed. The 3090's 24GB is sized to fit the target model class with operational headroom, not to maximize tokens per second.

---

## Hybrid Cloud Routing — Open Question

A hybrid approach routes specific query types to a cloud LLM while keeping home-context queries local:

| Query Type | Route | Privacy Surface |
|------------|-------|-----------------|
| Home control, sensor state, household context | Local | Stays in-network |
| Personal/family memory queries | Local | Stays in-network |
| General world knowledge ("what's the capital of...") | Cloud (optional) | Public information; no household exposure |
| Deep reasoning, niche expertise | Cloud (optional) | No household exposure |

The architecturally clean property: cloud routing handles only queries that don't expose home or personal information. The split is privacy-preserving by construction.

### Status: Deferred Decision

Hybrid is **not committed**. It remains a viable lever post-deployment if Tier 2 limitations bite in real use. Reasons to defer:

- It introduces external dependency where there was none — modest but real
- Tier 2 may prove sufficient; routing complexity is unjustified if so
- Routing logic (when to dispatch to cloud) is non-trivial to get right
- The decision is reversible — adding hybrid later is easier than removing it

### Triggers to Activate Hybrid

- Sustained user friction with Tier 2 world-knowledge gaps
- Specific high-value queries that consistently underperform locally
- Voice latency on local model is unacceptable for query types cloud could handle faster

If activated, the cloud endpoint should support tool calling and have favorable retention/training policies. Specific provider selection deferred until activation is decided.

---

## Voice Latency Budget

Voice interfaces feel responsive below ~2 seconds of perceived latency and broken above ~3 seconds. The platform must support that budget end-to-end:

| Stage | Target | Notes |
|-------|--------|-------|
| Wake word detection | <100ms | On-device on the satellite |
| STT (typical utterance) | 300–500ms | faster-whisper, small or medium model |
| LLM first token | ~500ms | 32B Q4 on RTX 3090 |
| LLM completion (typical) | 1–2s | Streams to TTS as it generates |
| TTS playback start | <200ms after first LLM tokens | Piper streams |
| **Perceived round-trip** | **~2s** | First audio response from end of user speech |

Any stage exceeding its budget is a tuning problem worth investigating individually, not a reason to relax the overall target. Hybrid cloud routing changes this budget — adds network round-trip on top of cloud inference time — and that latency cost is part of the tradeoff in routing decisions.

---

## What Powers What

| Service | Hardware | Notes |
|---------|----------|-------|
| LLM inference (Marvin) | LLM box, GPU | 32B-class via Ollama |
| Embedding model (memory) | LLM box, GPU or CPU | Small (~500MB), can co-reside |
| STT (Whisper) | LLM box, GPU or workflow.local CPU | small/medium model |
| TTS (Piper) | workflow.local, CPU | Lightweight, no GPU needed |
| Wake word | Voice satellite (on-device) | microWakeWord on ESP32-class |
| Vector store | workflow.local, PostgreSQL | pgvector extension on existing Postgres |
| Memory orchestration | workflow.local, Node-RED | Subflows, no separate service |

The split keeps the LLM box GPU dedicated to inference and embedding work; everything else lives where it already runs comfortably. No new hosts introduced.

---

## Open Questions

- [ ] Specific 32B model selection — Qwen 2.5 32B, Llama 3.3 70B-Q4 (might fit at aggressive quant), Hermes 3, or a tool-call-tuned variant
- [ ] Embedding model — `nomic-embed-text` is the default candidate; alternatives if dimensionality or quality prove inadequate
- [ ] Whether STT runs on the LLM GPU (lower latency, shared resource) or CPU on workflow.local (decoupled, slightly slower)
- [ ] Hybrid cloud routing activation — deferred to post-deployment validation
- [ ] Cloud provider selection if hybrid activates — favorable retention policy required

---

## Related Documents

| Document | Content |
|----------|---------|
| `architecture/OVERVIEW.md` | Four-box infrastructure, where the LLM box fits |
| `subsystems/ai/ASSIST_PIPELINE.md` | Voice pipeline, satellites, wake word, conversation flow |
| `subsystems/ai/PERSISTENT_MEMORY.md` | Memory architecture, RAG, knowledge ingestion, safety framework |
| `subsystems/VIDEO_PIPELINE.md` | Computer vision inference (separate concern from this doc) |

---

*Last Updated: 2026-05-08*
