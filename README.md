# 129 Highland — Smart Home Infrastructure

Ground-up rebuild of a pre-existing Home Assistant infrastructure. The goal is a distributed, resilient system where protocol coordinators and automations survive Home Assistant restarts — eliminating single points of failure.

**Core philosophy:** Node-RED is the automation engine. Home Assistant is the consumer and dashboard layer. Critical automations survive HA outages. HA self-configures on restart via MQTT Discovery — no manual intervention required.

---

## Architecture at a Glance

```
┌─────────────────────────┐
│        HAOS             │
│   Dell OptiPlex SFF     │
│   home.local            │
│   • Home Assistant      │
│   • Frontend/UI         │
│   • MQTT Discovery      │
└───────────┬─────────────┘
            │ MQTT / WebSocket
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│   Communication Hub     │     │       Workflow          │
│   Dell OptiPlex MFF     │     │   Dell OptiPlex SFF     │
│   hub.local             │     │   workflow.local        │
│   • MQTT Broker         │◄───►│   • Node-RED            │
│   • Zigbee2MQTT         │     │   • PostgreSQL          │
│   • Z-Wave JS UI        │     │                         │
└─────────────────────────┘     └─────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│   Edge AI Box           │
│   (SFF + Coral TPU)     │
│   • CodeProject.AI      │
│   • Ollama (LLM)        │
│   • Local inference     │
└─────────────────────────┘
```

---

## Documentation

All documentation lives in [`docs/`](./docs/). Start here:

**[`docs/INDEX.md`](docs/INDEX.md)** — complete documentation map with descriptions and status for every document

**[`docs/README.md`](docs/README.md)** — repository structure, quick links, and privacy notes

---

## Repository Structure

```
129-highland/
├── docs/           # All documentation (start with docs/INDEX.md)
├── config/         # Node-RED config files (device registry, thresholds, etc.)
├── ha/             # Home Assistant configuration
├── hub/            # Communication Hub Docker Compose and configs
├── node-red/       # Flows, package.json, utilities
└── scripts/        # Backup scripts, utilities
```

---

*Project epoch: 2026-03-03*
