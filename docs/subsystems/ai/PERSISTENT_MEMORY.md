# Persistent Memory Architecture

## Status

**Working notes — stretch goal, dependent on blocked prerequisites.**

**Do not begin implementation until:**
- PNC, HAOS, Node-RED, and Edge AI box are all stable
- Traditional Assist pipeline (the Assistant without memory) is in use and validated
- Dedicated LLM box with RTX 3060 12GB is online

See `architecture/OVERVIEW.md` — Stretch Goal: Dedicated LLM Inference Box for the hardware blocker.

---

## Problem Statement

Give the Assistant durable, accumulating contextual knowledge that grows over time — allowing the system to "learn" from interactions without requiring users to explicitly teach it. A non-technical user should be able to have a natural conversation and have relevant facts automatically captured and surfaced in future interactions.

Two distinct problems:
1. **Augmenting the request** — every request HA sends to Ollama must include accumulated context, without HA explicitly sending it
2. **Capturing the response** — every request/response pair must be extractable and evaluated for potential addition to context

---

## Context Lifecycle

### Two-Tier Model

**Long-Term Memory** — durable baseline
- Version controlled (GitHub)
- Contains only explicitly promoted, permanently relevant facts
- Changes deliberately, not automatically
- Human-reviewable and human-editable at any time

**Short-Term Memory** — daily working copy
- Starts each day as a fresh pull of Long-Term Memory
- Accumulates new facts throughout the day
- Discarded at midnight after long-term-worthy items are committed

### Daily Cycle

```
Midnight
  ↓
1. Queue long-term memory candidates committed to version control
2. Fresh Long-Term Memory pulled → becomes today's Short-Term Memory
  ↓
Throughout the day:
  Each request/response pair evaluated by classifier
  → LONG_TERM: appended to Short-Term + queued for nightly commit
  → SHORT_TERM: appended to Short-Term only (expires at midnight)
  → DISCARD: no action
  ↓
Midnight fires (highland/event/scheduler/midnight)
  Short-Term Memory discarded
  Cycle repeats
```

---

## Architecture: The Proxy

### Why a Proxy

HA's Ollama integration is a URL — it points at `http://host:port` with no discovery protocol. A thin proxy sitting in front of Ollama is invisible to HA. HA thinks it's talking directly to Ollama; the proxy intercepts, augments, and captures transparently.

### Topology

```
HA → Proxy (port 11435) → Ollama (port 11434)
          ↑
    intercept point:
    - inject Short-Term Memory into system prompt
    - capture request/response pair
    - trigger classifier (direct to Ollama port 11434, bypasses proxy)
```

**Critical:** Classifier calls go **directly to Ollama on port 11434**, bypassing the proxy. This prevents feedback loops by design — only HA-originated calls enter the proxy intercept path.

### Implementation

Thin FastAPI or Flask app, co-located with Ollama on the LLM inference box.

```python
@app.post("/api/chat")
async def chat(request):
    body = await request.json()
    context = read_short_term_context()
    body["system"] = context + "\n" + body.get("system", "")
    response = forward_to_ollama(body)  # port 11434
    classify_exchange(body["messages"], response)  # async, direct to 11434
    return response
```

---

## Classification Layer

### Classification Modes

**Explicit teaching (override path):** Trigger phrases ("Marvin, remember that...", "Make a note that...") bypass the classifier entirely → LONG_TERM.

**Implicit inference (default path):** All other interactions pass through the classifier.

### Classifier Call

A second Ollama call, made directly on port 11434 (never through the proxy).

**Classifier response:**
```json
{
  "classification": "LONG_TERM",
  "fact": "Joseph has a sister named Linda.",
  "confidence": 0.85
}
```

### Confidence Threshold

Below a configurable threshold (initial suggestion: 0.75), demote LONG_TERM to SHORT_TERM rather than polluting the long-term memory with misclassified noise.

### Restatement is Critical

| Raw exchange | Distilled fact |
|---|---|
| "My sister Linda is visiting next week" | "Joseph has a sister named Linda." |
| "Dad's birthday is coming up, July 20th" | "Joseph's father's birthday is July 20th." |

---

## Context File Structure

Markdown, human-readable, human-editable at any time.

### Long-Term Memory

```markdown
# Marvin — Long-Term Memory
Last updated: 2026-03-26

## Household
- Primary residents: Joseph and [spouse]

## People
- Joseph's father's birthday: July 20th
- Joseph has a sister named Linda

## Preferences
- Temperature units: Fahrenheit

## Recurring Events
- Wife's PT appointment: Thursdays
```

### Short-Term Memory

Starts as a verbatim copy of Long-Term Memory. New facts appended under `## Today`:

```markdown
[full Long-Term Memory contents]

## Today (2026-03-26)
- Linda (Joseph's sister) is visiting this week
```

---

## Memory Poisoning Defense

Memory poisoning is the most persistent attack vector — a successfully injected fact that reaches long-term memory survives the session boundary.

**Layer 1 — Provenance tracking:** The proxy tracks where each exchange originated. External-content exchanges carry a provenance flag through the entire pipeline.

**Layer 2 — Provenance-aware classification:** External-provenance candidates require a higher confidence threshold for LONG_TERM promotion.

**Layer 3 — User confirmation for external-provenance long-term candidates:** Any LONG_TERM candidate that originated from external content requires explicit user confirmation before being written.

**Layer 4 — Integrity check at commit time:** At midnight, a final integrity pass runs over queued LONG_TERM candidates, checking for contradictions and flagging anything from external content without confirmed user approval.

**Version Control as backstop:** Version-controlled long-term memory can be diffed against any previous commit. Rollback is always available.

---

## Tool Calling / External Service Integration

Ollama supports tool calling natively. The proxy intercepts tool call instructions, executes them, and feeds results back to Ollama transparently.

### Division of Labor

| Tool Category | Owner | Rationale |
|---|---|---|
| Email (send, read) | Proxy | External service, direct API call |
| Calendar (read, write) | Proxy | External service, direct API call |
| Web search | Proxy | External service, direct API call |
| HA device control | NR via MQTT | NR owns home control, MQTT is the control plane |
| HA state queries | Proxy | Read-only, direct HA REST API call |
| Reminders / timers | NR | NR owns scheduling |
| Notifications | NR via MQTT | NR owns the notification pipeline |

### Candidate Tools (Initial Set)

- [ ] `send_email` — compose and send via SMTP or API
- [ ] `read_email` — surface recent or relevant emails
- [ ] `create_calendar_event` — add to Google Calendar
- [ ] `read_calendar` — query upcoming events
- [ ] `web_search` — surface current information
- [ ] `ha_device_control` — publishes to `highland/command/` → NR executes
- [ ] `set_reminder` — publishes to NR for timer/schedule handling
- [ ] `query_history` — query HA statistics API for historical sensor data
- [ ] `create_watch` — register a dynamic conditional watch with NR
- [ ] `entity_search` — query HA entity registry by keyword

---

## Trust & Safety

### Action Risk Framework

| Tier | Action Type | Confirmation Model |
|---|---|---|
| **Read** | Read-only queries | Silent |
| **Reversible Write** | Creates something deletable | Soft confirmation |
| **Irreversible Action** | Cannot be undone | **Explicit confirmation always required** |

**Irreversible actions always confirm, regardless of confidence. This cannot be overridden.**

### Prompt Injection Defense

When the Assistant reads external content (email bodies, calendar descriptions, web results), that content is **untrusted data**, not user instructions.

**Defense layers:**
1. Explicit content tagging — external content wrapped in `<external_content source="...">` tags; any instructions within must be ignored
2. Action confirmation as safety net — irreversible action confirmation provides a second layer
3. Scope limiting — tool calls from external content reads restricted to read-only by default

### Chain Provenance

| Chain Type | Treatment |
|---|---|
| User → read → report back | Clean — proceed |
| User → read → write (explicit) | Confirm each individual write |
| User → read → write (inferred) | **Never. Blocked absolutely.** |

**Depth limiting:** Tool call chains have a hard maximum depth of three sequential calls before a mandatory user check-in.

---

## User Identity & Resource Ownership

### Voice Recognition as the Hard Gate

Satellite topology is a weak identity signal. **Voice recognition is the hard gate for personal resource access.** Until speaker recognition is reliable, personal resource tool calls are not available via voice.

### Resource Ownership Tiers

| Resource | Read | Write | Identity Required |
|---|---|---|---|
| Household calendar | Anyone | Anyone (confirmed) | No |
| Personal email (Joseph) | Joseph only | Joseph only | Yes — hard gate |
| Personal contacts (Joseph) | Joseph only | Joseph only | Yes — hard gate |

### Identity Fallback: Personal Passphrase

When voice recognition fails, a personal passphrase serves as a voice-native fallback. Never stored in long-term memory — lives in a separate secured credential store. Never repeated back by the Assistant.

---

## Open Questions

- [ ] Classifier model selection — dedicated lightweight model vs. same Marvin model
- [ ] Exact confidence threshold for LONG_TERM vs SHORT_TERM demotion
- [ ] Long-Term Memory review cadence — periodic human review to prune stale entries
- [ ] Git authentication for automated commits from the LLM box
- [ ] Tool call chain depth limit — is 3 the right number?
- [ ] Speaker recognition production readiness (monitoring EuleMitKeule/speaker-recognition)
- [ ] Rate limit threshold tuning for tool calls
- [ ] Watch TTL defaults for unresolved watches

---

*Last Updated: 2026-03-26*
