# Highland Home Automation

Ground-up rebuild of a Home Assistant infrastructure. The goal is a distributed, resilient system where protocol coordinators and automations survive Home Assistant restarts — eliminating the single point of failure inherent in a monolithic HAOS setup.

---

## Architecture at a Glance

**Four-box physical separation:**

| Role | Hardware | Services |
|------|----------|----------|
| **HAOS** | Dell OptiPlex 7050 SFF | Home Assistant OS — UI, dashboards, voice |
| **Communication Hub** | Dell OptiPlex MFF | Mosquitto MQTT, Zigbee2MQTT, Z-Wave JS UI |
| **Workflow** | Dell OptiPlex 7050 SFF | Node-RED (automation engine), PostgreSQL |
| **Edge AI** | Dell OptiPlex 7050 SFF | CodeProject.AI + Coral TPU, Ollama |

**Core principle:** Node-RED is the automation engine. Home Assistant is the consumer and dashboard layer. Critical automations survive HA restarts via MQTT. HA self-configures on restart via MQTT Discovery — no manual intervention required.

---

## Documentation

All documentation lives in this `docs/` folder. Start with the index:

**[`docs/INDEX.md`](INDEX.md)** — complete documentation map with descriptions and status for every document

**Quick links:**

| Need | Document |
|------|----------|
| System overview and topology | [`architecture/OVERVIEW.md`](architecture/OVERVIEW.md) |
| Step-by-step build guide | [`RUNBOOK.md`](RUNBOOK.md) |
| MQTT topic reference | [`standards/MQTT_TOPICS.md`](standards/MQTT_TOPICS.md) |
| Node-RED conventions | [`nodered/OVERVIEW.md`](nodered/OVERVIEW.md) |
| HA entity naming | [`standards/ENTITY_NAMING.md`](standards/ENTITY_NAMING.md) |
| Credentials template | [`ha/SECRETS_TEMPLATE.md`](ha/SECRETS_TEMPLATE.md) |

---

## Repository Structure

```
docs/
├── INDEX.md                          ← Start here — full documentation map
├── RUNBOOK.md                        ← Step-by-step build guide
├── AUTOMATION_BACKLOG.md             ← Ideas for future automations
│
├── architecture/
│   ├── OVERVIEW.md                   ← Hardware, topology, migration strategy
│   ├── NETWORK.md                    ← Hostnames, ports, remote access
│   └── BACKUP_RECOVERY.md            ← Backup strategy and recovery scenarios
│
├── standards/
│   ├── EVENT_ARCHITECTURE.md         ← MQTT philosophy and patterns
│   ├── MQTT_TOPICS.md                ← Authoritative topic registry
│   └── ENTITY_NAMING.md              ← HA entity naming conventions
│
├── nodered/
│   ├── OVERVIEW.md                   ← Flow types, conventions, flow registration
│   ├── ENVIRONMENT.md                ← Node.js modules, context stores, settings.js
│   ├── STARTUP_SEQUENCING.md         ← Two-condition gate, degraded state
│   ├── CONFIG_MANAGEMENT.md          ← Config files, Config Loader, secrets
│   ├── DEVICE_REGISTRY.md            ← Device Registry, Command Dispatcher, ACK Tracker
│   ├── SUBFLOWS.md                   ← Initializer Latch, Connection Gate
│   ├── LOGGING.md                    ← JSONL framework, Utility: Logging
│   ├── NOTIFICATIONS.md              ← Framework, Utility: Notifications
│   ├── SCHEDULING.md                 ← Utility: Scheduling
│   ├── HEALTH_MONITORING.md          ← Health Monitor, Utility: Connections
│   ├── BATTERY_MONITOR.md            ← Utility: Battery Monitor
│   └── DAILY_DIGEST.md               ← Utility: Daily Digest
│
├── ha/
│   ├── HA_CONFIG.md                  ← YAML config structure and automations
│   └── SECRETS_TEMPLATE.md           ← Credentials template (no real values)
│
└── subsystems/
    ├── APPLIANCE_MONITORING.md       ← Washer/dryer/dishwasher cycle detection
    ├── CALENDAR_INTEGRATION.md       ← Google Calendar bridge, camera suppression
    ├── GARAGE_DOOR.md                ← Konnected GDO blaQ integration
    ├── LORA.md                       ← LoRaWAN: bin monitoring + mailbox detection
    ├── VIDEO_PIPELINE.md             ← Video analysis pipeline
    ├── WEATHER_FLOW.md               ← Weather data synthesis
    └── ai/
        ├── ASSIST_PIPELINE.md        ← HA Assist voice pipeline (planned)
        └── PERSISTENT_MEMORY.md      ← AI memory architecture (blocked)
```

---

## Privacy

This repository is public. All identifying information has been redacted:
- GPS coordinates reference `secrets.json` (gitignored)
- Domain names use `your-domain.example`
- Email addresses use generic descriptions
- API keys and passwords use placeholder values

The populated `secrets.json` and HA `secrets.yaml` are stored outside this repository.

---

*Last Updated: 2026-03-26*
