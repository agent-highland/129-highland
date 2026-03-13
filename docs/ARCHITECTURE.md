# Home Assistant Infrastructure Rebuild - Architecture Planning

## Overview

Ground-up rebuild of Home Assistant infrastructure, prioritizing **resiliency**, **separation of concerns**, and **scalability**. Migration from a 6-year veteran's existing HAOS setup to a distributed architecture.

---

## Current State

| Component | Current Implementation |
|-----------|----------------------|
| **HAOS Host** | Dell OptiPlex 7050 MFF |
| **Zigbee Coordinator** | Sonoff Zigbee Dongle Plus (3.0) |
| **Z-Wave Coordinator** | Zooz 800 series USB |
| **Automations** | Node-RED (HA Add-on) |
| **Cameras** | Reolink Wi-Fi/Battery + Home Hub |

### Pain Points
- Hardware limitations preventing advanced automations (image manipulation, AI workflows)
- Node-RED as HA add-on = coupled to HA lifecycle
- HAOS updates occasionally break things; no isolation

---

## Target Architecture

### Design Principles
1. **Resiliency** — Protocol coordinators and automations survive HA restarts/failures
2. **Separation** — Logical AND physical separation where practical
3. **Scalability** — Room to grow without architectural rework
4. **Maintainability** — Updates to one component don't risk others

### Hardware Allocation (Finalized)

| Role | Hardware | CPU | RAM | Storage | Status |
|------|----------|-----|-----|---------|--------|
| **HAOS** | Dell OptiPlex 7050 SFF | i7-7700 4.2GHz | 16GB | 480GB SSD | ✅ Ready |
| **Workflow** | Dell OptiPlex 7050 SFF | i7-7700 4.2GHz | 16GB | 480GB SSD | ✅ Ready |
| **Communication Hub** | MFF (Ryzen 5) | Ryzen 5 3550H | 16GB | 512GB SSD | ✅ Ready |
| **Edge AI Box** | Dell OptiPlex 7050 SFF | i7-7700 4.2GHz | 32GB | 480GB SSD | ✅ Ready |

**Workflow Services:**
- Node-RED (primary automation engine)
- code-server (VS Code web edition) — embedded as HA sidebar panel
- Future utilities as needed

**Storage Notes:**
- 32GB RAM in Edge AI box for LLM headroom alongside Coral TPU vision inference
- Coral TPU (PCIe) installed in Edge AI box
- Reolink NVR ready for camera infrastructure

### New Coordinators
- Sonoff Zigbee USB (latest) — for new Z2M instance
- Sonoff Z-Wave JS UI instance

*Note: New coordinators enable parallel bring-up alongside existing infrastructure for clean migration.*

### Proposed Topology

```
┌─────────────────────────┐
│        HAOS             │
│   Dell OptiPlex SFF     │
│   http(s)://home.local  │
│                         │
│   • Home Assistant      │
│   • Frontend/UI         │
│   • Integrations        │
│   • Supervised updates  │
└───────────┬─────────────┘
            │
            │ MQTT / WebSocket
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│   Communication Hub     │     │       Workflow          │
│   Dell OptiPlex MFF     │     │   Dell OptiPlex SFF     │
│   http(s)://hub.local   │     │ http(s)://workflow.local│
│                         │     │                         │
│   • MQTT Broker         │     │   • Node-RED            │
│   • Zigbee2MQTT         │◄───►│   • (Ubuntu host)       │
│   • Z-Wave JS UI        │     │   • Potential other svcs│
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

## Open Decisions

### Node-RED Hosting
- **Option A:** Dedicated bare-metal Ubuntu box (physical separation) ✓ **SELECTED**
- **Option B:** VM/container on same host as HA (logical separation only)

### Communication Hub Stack
- **Decision:** Docker Compose on minimal Linux
- **OS:** Ubuntu Server 24.04 LTS
- **Services:** Mosquitto, Zigbee2MQTT, Z-Wave JS UI (containerized)
- **Rationale:** Clean, portable, version-controllable, easy backup/rebuild

**Configuration Decisions:**
- **MQTT Auth:** Username/password enabled from the start
- **Static IP:** Yes, DHCP reservation or static assignment
- **USB Passthrough:** By-id symlinks (avoid /dev/ttyUSB roulette)
- **Z2M Frontend:** Enabled, embedded as HA sidebar panel (iframe)
- **Z-Wave JS UI Frontend:** Enabled, embedded as HA sidebar panel (iframe)
- **Z-Wave JS UI MQTT Gateway:** Enabled alongside WebSocket
  - WebSocket → HA native Z-Wave integration (state, dashboards, history)
  - MQTT → Node-RED direct control (resiliency, unified control plane)

*Note: Day-to-day device control via HA integrations; advanced config (OTA, network healing, device interviews) via embedded frontends.*

### Device Communication Model

**Two-Layer Event Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                      MQTT Broker                            │
│                      (Mosquitto)                            │
├─────────────────────────────────────────────────────────────┤
│  Raw Device Topics          │  Semantic Event Topics        │
│  zigbee2mqtt/{device}/...   │  highland/event/{area}/...    │
│  zwave/{node}/...           │  highland/ack/...             │
└─────────────────────────────────────────────────────────────┘
        │                                   ▲
        │ subscribe                         │ publish
        ▼                                   │
┌─────────────────────────────────────────────────────────────┐
│                     Area Flows (Node-RED)                   │
│  • Subscribe to raw device events (zigbee2mqtt/, zwave/)    │
│  • Interpret and publish semantic events (highland/event/)  │
│  • Subscribe to semantic events from other flows            │
│  • Command devices via raw topics                           │
└─────────────────────────────────────────────────────────────┘
```

**Resiliency Benefit:**
- Node-RED subscribes directly to Z2M and Z-Wave MQTT topics
- Critical automations (security, safety, core lighting) function without HA
- HA also subscribes for state display, history, dashboards — parallel, not serial

**Example flow:**
```
zigbee2mqtt/garage_motion_sensor → [Garage Flow] → highland/event/garage/motion_detected
                                        │
                                        ├──► (turn on lights locally)
                                        │
                                        └──► (other flows react to semantic event)
```

### Virtualization vs Bare Metal
- User prefers HAOS for supervised updates via UI
- Bare metal for HAOS, bare metal Ubuntu for Node-RED currently favored
- Proxmox could consolidate but adds complexity

---

## Network

| System | Hostname | Internal URL | External URL |
|--------|----------|--------------|--------------|
| HAOS | `home` | `http(s)://home.local` | `https://highland.ferris.network` (after decom) |
| Communication Hub | `hub` | `http(s)://hub.local` | None |
| Workflow | `workflow` | `http(s)://workflow.local` | None |
| Network Video Recorder | `nvr` | `http(s)://nvr.local` | None |

- **Current topology:** Flat network, no VLANs
- **Remote access:** Nabu Casa with custom domain (FQDN) — enabled after live system decommissioned
- **Future:** Network segmentation project planned but out of scope for this rebuild

**Remote Access Scope:**
- Nabu Casa tunnel exposes **HA only** (port 8123)
- Sidebar iframes (Z2M, Z-Wave JS UI, VS Code) are **local network only**
- If remote management access needed later: Tailscale/WireGuard recommended

*Note: Flat network simplifies inter-service communication. When VLANs are eventually implemented, ensure proper routing/firewall rules for MQTT (1883/8883), HA (8123), Z-Wave JS WebSocket, Node-RED (1880), etc.*

---

## Migration Strategy

### Approach: Zero-Baggage Parallel Build

The new infrastructure is built alongside the existing live system. This is not a lift-and-shift — it's a greenfield build with a running reference implementation.

**Principles:**
- New coordinators = new Zigbee/Z-Wave networks (devices migrate individually)
- Flows are rebuilt from scratch, not copied (re-evaluated against new conventions)
- Old system remains live as fallback until new system is proven
- Devices/automations migrate one at a time, validated before proceeding
- No legacy naming, no cruft, no "I'll fix that later"

**Benefits:**
- Clean end state with consistent conventions
- Every flow is re-understood during rebuild (maintainability)
- Can A/B test automations if needed
- No single cutover risk — gradual confidence building

**Tradeoffs:**
- Longer timeline
- Temporary duplication of effort
- Some devices may need to be re-paired (Zigbee/Z-Wave network change)

### Migration Sequence (High-Level)

1. **Baseline hardware** — Install OS, Docker, base services on all boxes
2. **Communication Hub online** — Mosquitto, Z2M, Z-Wave JS UI running (empty networks)
3. **HAOS online** — Fresh install, connect to MQTT, Z-Wave JS via WebSocket
4. **Workflow online** — Fresh install, connect to MQTT, establish event architecture
5. **Migrate devices** — One at a time, pair to new coordinators, verify in HA
6. **Rebuild flows** — One at a time, referencing old flows for logic but implementing fresh
7. **Validate** — Run parallel until confidence achieved
8. **Decommission old system**

---

## Backup & Recovery Strategy

### Current State (HAOS)
- Native HA backup, triggered manually or on schedule
- Nabu Casa cloud backup (last 3-5 backups stored off-site)
- Proven: Successfully used for rollback twice in six months

### Target State (Distributed)

Each host owns its own backup, triggered via MQTT command. No SSH between hosts required.

| Component | What to Backup | Method |
|-----------|----------------|--------|
| **HAOS** | Full HA backup (config, database, add-ons) | Native HA backup + Nabu Casa cloud |
| **Communication Hub** | Docker Compose file, Mosquitto config, Z2M data, Z-Wave JS data | Local script tars config volumes |
| **Workflow Host** | Docker Compose file, flows (JSON), settings, credentials, config files | Local script exports flows + tars data |

### MQTT-Triggered Backup Architecture

Each host has a local backup script that:
1. Subscribes to `highland/command/backup/trigger` (or `highland/command/backup/trigger/{hostname}`)
2. Performs local backup (tar, export, etc.)
3. Stores backup in local staging directory
4. Publishes result to `highland/event/backup/completed` or `highland/event/backup/failed`

```
Backup Orchestration

Scheduler (Node-RED)
      |
      | highland/event/scheduler/backup_daily (3:00 AM)
      v
Backup Utility Flow
      |
      +---> highland/command/backup/trigger/hub
      |           |
      |           v
      |     Communication Hub backup script
      |           |
      |           +---> highland/event/backup/completed {host: "hub"}
      |
      +---> (Node-RED backs itself up locally)
      |           |
      |           +---> highland/event/backup/completed {host: "workflow"}
      |
      +---> HA REST API: trigger backup
                  |
                  +---> (Nabu Casa handles cloud sync)

Backup Utility collects results, notifies on failure
```

### Backup Scripts (Per Host)

**Communication Hub (`/usr/local/bin/highland-backup.sh`):**
```bash
#!/bin/bash
# Triggered by: MQTT listener (mosquitto_sub) or cron fallback
# Tars: /opt/highland/mosquitto, /opt/highland/zigbee2mqtt, /opt/highland/zwavejs
# Destination: /var/backups/highland/
# Publishes result to MQTT
```

**Workflow Host:**
- Flow export via Node-RED admin API
- Tar `/home/nodered/config/` and `/home/nodered/data/`
- Handled by Backup Utility Flow (backs itself up)

### Retention Policy

| Type | Retention | Notes |
|------|-----------|-------|
| **Logs (JSONL)** | 30 days | Daily rotation, cron cleanup |
| **Local backups** | 7 days | Per-host, cron cleanup |
| **Nabu Casa (HA)** | 3-5 backups | Managed by Nabu Casa |
| **NAS (future)** | TBD | Define when NAS is available |

### Recovery Scenarios

| Scenario | Recovery Approach |
|----------|-------------------|
| **HAOS failure** | Restore from Nabu Casa cloud or local backup; Communication Hub + Workflow continue running |
| **Communication Hub failure** | Redeploy Docker Compose, restore config volumes from backup; devices remain paired in coordinator database |
| **Workflow failure** | Redeploy Docker Compose, import flow JSON, restore config files; MQTT events queue until back online |
| **Total loss** | Restore all from backup destination; re-pair devices if coordinator database lost |

---

## Future Considerations

- **Edge AI integration** — Coral TPU for camera feed triage before cloud analysis (Gemini)
- **Network Video Recorder deployment** — Replace/supplement Home Hub
- **Weather automation expansion** — Map compositing, image manipulation (needs horsepower)
- **Room-based automations** — Light switches, presence detection, etc.

### Stretch Goal: Dedicated LLM Inference Box

**Status:** Deferred — post-baseline infrastructure only. Do not begin until PNC, HAOS, NR, and Edge AI box are all stable.

**Candidate hardware:** Existing ATX mid-tower (previously development desktop / Plex server, currently idle)

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 5 3600 6-Core |
| RAM | 64GB DDR4-3600 |
| Storage | 1TB Crucial MX500 SSD + 4TB WD Black HDD |
| GPU | Sapphire RX 580 8GB → **replace with RTX 3060 12GB** |
| PSU | EVGA 600W 80+ Bronze |
| Case | ATX Mid-Tower |

**Required change:** Swap RX 580 for RTX 3060 12GB (~$200-250 used). RX 580 ROCm support on Ollama is inconsistent; CUDA on RTX 3060 is clean and well-supported. 12GB VRAM runs 13B models comfortably with headroom.

**Role:** Dedicated Ollama LLM inference, decoupled from Edge AI box (which retains Coral TPU for vision inference in its single PCIe slot). With this box online, Node-RED can own conversation orchestration directly — talking to Ollama outside of HA's Assist pipeline — which unblocks the persistent memory architecture.

**Why this unblocks memory:** HA's conversation agent pipeline is fully sandboxed — pipeline events are internal-only, no writable surface is reachable from within the conversation agent, and external context injection is equally constrained (verified via ~2 hours live experimentation on HA 2026.3.1). The only clean solution is NR owning orchestration directly, which requires a capable Ollama instance. See ASSIST_PIPELINE.md for full details.

**Services on this box:**
- Ollama (LLM inference — primary conversation agent for Marvin)
- Conversation history / memory store (markdown files, human-readable)
- Future LLM workloads as needed

---

## Related Documents

| Document | Status | Content |
|----------|--------|---------|
| **ENTITY_NAMING.md** | ✅ Complete | Naming conventions, disambiguation, device patterns |
| **EVENT_ARCHITECTURE.md** | ✅ Complete | MQTT topics, payloads, periods, two-layer model |
| **NODERED_PATTERNS.md** | ✅ Complete | Flow organization, logging, notifications, config management, health monitoring |
| **GARAGE_DOOR.md** | ✅ Complete | Konnected GDO blaQ — integration design, API surface, Node-RED bridge architecture |

## Deferred Items

| Item | Reason |
|------|--------|
| Version control strategy (GitHub) | Mechanics depend on actual config structure during implementation |
| Docker Compose files | Draft when ready to build |
| Retention policy (backups, logs) | Define when NAS is available |
| HA Dashboard design | Revisit when automation baseline is stable |

---

*Last Updated: 2026-03-12*
