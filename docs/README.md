# Highland Home Automation — Project Context

## What This Is

Ground-up rebuild of a 6-year-old Home Assistant infrastructure. The goal is a distributed, resilient system where protocol coordinators and automations survive Home Assistant restarts — eliminating single points of failure.

**Owner:** Joseph — 25+ years software engineering (primarily Windows, limited Linux), currently running HAOS on a Dell OptiPlex 7050 MFF with Node-RED as an add-on.

**Domain:** `highland.ferris.network` (Nabu Casa for remote access)

---

## Document Map

| Document | Purpose | When to Reference |
|----------|---------|-------------------|
| **ARCHITECTURE.md** | Hardware allocation, topology, backup strategy, migration approach | System-level questions, hardware decisions, recovery scenarios |
| **EVENT_ARCHITECTURE.md** | MQTT topic structure, payloads, periods, command/event patterns | Inter-flow communication, adding new topics, message formats |
| **ENTITY_NAMING.md** | Naming conventions for HA entities, disambiguation rules | Adding new devices, renaming entities, ensuring consistency |
| **NODERED_PATTERNS.md** | Flow organization, utilities, config management, logging, notifications, health monitoring | Building/modifying flows, implementing new utilities |
| **AUTOMATION_BACKLOG.md** | Ideas for future automations | Capturing new ideas, reviewing what's planned |

---

## Key Decisions (Don't Re-Litigate)

These have been discussed and decided. Reference the relevant doc for rationale.

| Decision | Rationale |
|----------|----------|
| **Three-box architecture** | HAOS, Protocol Nerve Center (MQTT/Z2M/Z-Wave), Node-RED/Utility — physical separation for resiliency |
| **MQTT as control plane** | Node-RED subscribes directly to Z2M/Z-Wave topics; critical automations work without HA |
| **MQTT-triggered backups** | Each host owns its backup via MQTT command; no SSH between hosts |
| **File-based config** | External JSON files in `/home/nodered/config/`; `secrets.json` gitignored |
| **Schedex for time triggers** | Period configuration lives in schedex nodes, not external config |
| **node-red-contrib-home-assistant-websocket** | For HA integration (backups, notifications, entity state) |
| **JSONL logging** | Unified daily log files, 30-day retention, jq for querying |
| **HA Companion App** | Primary notification channel (Android); future channels deferred |
| **Google Calendar** | For scheduled events (recurring + one-time); queried once daily for digest |
| **Markdown → HTML** | Daily digest built as markdown, converted via node-red-node-markdown |
| **Healthchecks.io** | External monitoring; watchdog script pings on Node-RED heartbeat |

---

## Hardware

| Role | Hardware | Status |
|------|----------|--------|
| **HAOS** | Dell OptiPlex 7050 SFF (i7-7700, 16GB) | Needs SSD (~$80) — on order |
| **Node-RED / Utility** | Dell OptiPlex 7050 SFF (i7-7700, 16GB) | Needs SSD (~$80) — on order |
| **Protocol Nerve Center** | Ryzen 5 3550H MFF (16GB, 512GB SSD) | Ready |
| **Spare / Future Edge AI** | Dell OptiPlex 7050 SFF (i7-7700, 16GB) | Deferred |

---

## Current State

**Phase:** Documentation complete, implementation runbook pending

**What's done:**
- Architecture finalized (hardware, topology, backup strategy)
- Event architecture defined (topics, payloads, periods)
- Entity naming standards established
- Node-RED patterns documented (flows, config, logging, notifications, health monitoring)
- All open questions resolved

**What's next:**
1. Draft implementation runbook
2. Hardware arrives (weather delayed)
3. Begin build: Protocol Nerve Center first, then HAOS, then Node-RED

---

## Working Style

**Communication:**
- Peer-level, informal — we're well-acquainted colleagues
- Colorful language acceptable in moderation
- Direct and pragmatic over cautious and verbose

**Preferences:**
- Prefer pragmatic over perfect
- Avoid over-engineering; solve for today's problems
- Separation of concerns is a core value
- Intentional design choices over convenience (e.g., Greek letters over numbers)
- "Zero-baggage" migration — rebuild from scratch, don't copy legacy

**When uncertain:**
- Ask clarifying questions rather than assuming
- Present options with tradeoffs when decisions are needed
- Reference existing docs before proposing something that may conflict

---

## Protocols

**Adding to documentation:**
- Claude CAN create/modify project-level MD files
- Use for living docs, standards, architecture decisions
- Update "Last Updated" date when modifying docs

**Editing project files:**
- Files in `/mnt/project/` are **directly editable** — use `str_replace` or `create_file` as needed
- This is verified and tested — do not second-guess this capability

**Capturing ideas:**
- New automation ideas → AUTOMATION_BACKLOG.md
- Don't derail current work; capture and move on

---

*Last Updated: 2026-03-03*
