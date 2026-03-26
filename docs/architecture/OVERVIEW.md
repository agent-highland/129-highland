# Architecture Overview

## Design Principles

1. **Resiliency** — Protocol coordinators and automations survive HA restarts and failures
2. **Separation of concerns** — Logical and physical separation where practical
3. **Scalability** — Room to grow without architectural rework
4. **Maintainability** — Updates to one component don't risk others

**Core rule:** Node-RED is the automation engine. Home Assistant is the consumer and dashboard layer. Critical automations must survive HA being offline. HA recovers fully and automatically on restart via MQTT Discovery — no manual intervention.

---

## Hardware Allocation

| Role | Hardware | CPU | RAM | Storage | Status |
|------|----------|-----|-----|---------|--------|
| **HAOS** | Dell OptiPlex 7050 SFF | i7-7700 4.2GHz | 16GB | 480GB SSD | ✅ Ready |
| **Workflow** | Dell OptiPlex 7050 SFF | i7-7700 4.2GHz | 16GB | 480GB SSD | ✅ Ready |
| **Communication Hub** | OptiPlex MFF (Ryzen 5 3550H) | Ryzen 5 3550H | 16GB | 512GB SSD | ✅ Ready |
| **Edge AI Box** | Dell OptiPlex 7050 SFF | i7-7700 4.2GHz | 32GB | 480GB SSD | ✅ Ready |

**Communication Hub services:** Mosquitto MQTT broker, Zigbee2MQTT, Z-Wave JS UI (all containerized via Docker Compose)

**Workflow services:** Node-RED, PostgreSQL (Docker Compose)

**Edge AI Box services:** CodeProject.AI + Coral TPU (vision inference), Ollama (LLM inference)

**Storage notes:** 32GB RAM in Edge AI Box for LLM headroom alongside Coral TPU vision inference. Coral TPU occupies the single PCIe slot.

---

## System Topology

```
┌─────────────────────────┐
│        HAOS             │
│   Dell OptiPlex SFF     │
│   home.local            │
│                         │
│   • Home Assistant      │
│   • Frontend/UI         │
│   • MQTT Discovery      │
│   • Supervised updates  │
└───────────┬─────────────┘
            │
            │ MQTT / WebSocket
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│   Communication Hub     │     │       Workflow          │
│   Dell OptiPlex MFF     │     │   Dell OptiPlex SFF     │
│   hub.local             │     │   workflow.local        │
│                         │     │                         │
│   • MQTT Broker         │     │   • Node-RED            │
│   • Zigbee2MQTT         │◄───►│   • PostgreSQL          │
│   • Z-Wave JS UI        │     │                         │
└─────────────────────────┘     └─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│   Edge AI Box           │
│   (SFF + Coral TPU)     │
│                         │
│   • CodeProject.AI      │
│   • Ollama (LLM)        │
│   • Camera triage       │
└─────────────────────────┘
```

---

## Two-Layer Event Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      MQTT Broker (Mosquitto)                │
├─────────────────────────────────────────────────────────────┤
│  Raw Device Topics              │  Semantic Event Topics    │
│  zigbee2mqtt/{device}/...       │  highland/event/{area}/.. │
│  zwave/{node}/...               │  highland/state/{domain}/ │
└─────────────────────────────────────────────────────────────┘
        │                                   ▲
        │ subscribe                         │ publish
        ▼                                   │
┌─────────────────────────────────────────────────────────────┐
│                     Area Flows (Node-RED)                   │
│  • Subscribe to raw device events                           │
│  • Interpret and publish semantic events                    │
│  • Command devices via raw topics                           │
└─────────────────────────────────────────────────────────────┘
```

Node-RED subscribes directly to Z2M and Z-Wave MQTT topics. Critical automations (security, safety) function without HA. HA also subscribes — parallel, not serial. HA uses MQTT Discovery with `value_template` extraction; no manual HA configuration required for new entities.

---

## Device Communication Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Z2M MQTT Gateway | Enabled | Zigbee devices on MQTT bus, Node-RED can control directly |
| Z-Wave JS UI MQTT Gateway | Enabled alongside WebSocket | WebSocket → HA native integration; MQTT → Node-RED direct control |
| MQTT Auth | Username/password, per-service accounts | Auditability, future ACL flexibility |
| USB passthrough | By-id symlinks | Stability across reboots |

---

## Migration Strategy

### Zero-Baggage Parallel Build

The new infrastructure is built alongside the existing live system. This is not a lift-and-shift — it's a greenfield build with a running reference implementation.

**Principles:**
- New coordinators = new Zigbee/Z-Wave networks (devices migrate individually)
- Flows rebuilt from scratch, not copied (re-evaluated against new conventions)
- Old system remains live as fallback until new system is proven
- Devices and automations migrate one at a time, validated before proceeding
- No legacy naming, no cruft

**Migration sequence:**
1. Baseline hardware — install OS, Docker, base services on all boxes
2. Communication Hub online — Mosquitto, Z2M, Z-Wave JS UI running (empty networks)
3. HAOS online — fresh install, connect to MQTT and Z-Wave JS via WebSocket
4. Workflow online — fresh install, connect to MQTT, establish event architecture
5. Migrate devices — one at a time, pair to new coordinators, verify in HA
6. Rebuild flows — one at a time, reference old flows for logic but implement fresh
7. Validate — run parallel until confidence achieved
8. Decommission old system

---

## Stretch Goal: Dedicated LLM Inference Box

**Status:** Deferred — post-baseline only. Do not begin until HAOS, Node-RED, and Edge AI box are all stable.

The Edge AI box has one PCIe slot occupied by the Coral TPU, leaving no room for a GPU. A separate dedicated Ollama inference box unblocks the persistent memory architecture for Marvin (the AI assistant).

**Candidate hardware:** Existing ATX mid-tower (AMD Ryzen 5 3600, 64GB DDR4, 1TB SSD + 4TB HDD, RX 580 → replace with RTX 3060 12GB)

Required change: swap RX 580 for RTX 3060 12GB (~$200–250 used). CUDA on RTX 3060 is clean and well-supported; ROCm on RX 580 is inconsistent. 12GB VRAM runs 13B models comfortably.

See `subsystems/ai/ASSIST_PIPELINE.md` and `subsystems/ai/PERSISTENT_MEMORY.md` for full context on why this is a blocker.

---

## Related Documents

| Document | Content |
|----------|---------|
| `architecture/NETWORK.md` | Hostnames, ports, remote access, VLAN future |
| `architecture/BACKUP_RECOVERY.md` | Per-host backup strategy, retention, recovery scenarios |
| `standards/EVENT_ARCHITECTURE.md` | MQTT philosophy and patterns |
| `standards/MQTT_TOPICS.md` | Authoritative topic registry |
| `RUNBOOK.md` | Step-by-step build guide |

---

*Last Updated: 2026-03-26*
