# Highland Home Automation — Documentation Index

## What This Is

Ground-up rebuild of an existing Home Assistant infrastructure. The goal is a distributed, resilient system where protocol coordinators and automations survive Home Assistant restarts — eliminating single points of failure.

**Core philosophy:** Node-RED is the automation engine. Home Assistant is the consumer and dashboard layer. Critical automations survive HA outages. HA recovers fully on restart via MQTT Discovery — no manual intervention required.

**Current phase:** Documentation complete, hardware in-hand — ready to build.

---

## Quick Reference: Key Decisions

These have been discussed and decided. Reference the relevant doc for rationale.

| Decision | Where to find it |
|----------|-----------------|
| Four-box architecture (HAOS, Hub, Workflow, Edge AI) | `architecture/OVERVIEW.md` |
| MQTT as control plane | `standards/EVENT_ARCHITECTURE.md` |
| Node-RED owns automations; HA is consumer only | `architecture/OVERVIEW.md` |
| MQTT-triggered backups, each host owns its backup | `architecture/BACKUP_RECOVERY.md` |
| File-based config, `secrets.json` gitignored | `nodered/CONFIG_MANAGEMENT.md` |
| Schedex for time triggers | `nodered/SCHEDULING.md` |
| JSONL logging, 30-day retention | `nodered/LOGGING.md` |
| UTC for storage/transport, local for scheduling/presentation | `standards/TIME_HANDLING.md` |
| HA Companion App as primary notification channel | `nodered/NOTIFICATIONS.md` |
| Google Calendar for scheduled events | `subsystems/CALENDAR_INTEGRATION.md` |
| Attendee-based camera suppression | `subsystems/CALENDAR_INTEGRATION.md` |
| Healthchecks.io for external monitoring | `nodered/HEALTH_MONITORING.md` |
| PostgreSQL on Workflow host (shared: HA Recorder + video pipeline) | `RUNBOOK.md` §3.13 |
| OWM + Open-Meteo multi-source weather analysis | `subsystems/WEATHER_FLOW.md` |

---

## Documentation Map

### Architecture

System-level decisions, hardware, topology, network, and backup strategy.

| Document | When to Reference |
|----------|------------------|
| [`architecture/OVERVIEW.md`](architecture/OVERVIEW.md) | Hardware allocation, system topology, two-layer event architecture, migration strategy, LLM inference box stretch goal |
| [`architecture/NETWORK.md`](architecture/NETWORK.md) | Host inventory, port reference, remote access, mDNS gotchas, USB serial devices |
| [`architecture/BACKUP_RECOVERY.md`](architecture/BACKUP_RECOVERY.md) | Per-host backup strategy, MQTT-triggered backup architecture, retention policy, recovery scenarios |

---

### Standards

Conventions and contracts that everything else builds on. Read these before building flows or adding devices.

| Document | When to Reference |
|----------|------------------|
| [`standards/EVENT_ARCHITECTURE.md`](standards/EVENT_ARCHITECTURE.md) | MQTT namespace philosophy, event vs. state, scheduler periods, ACK pattern, payload conventions |
| [`standards/MQTT_TOPICS.md`](standards/MQTT_TOPICS.md) | **Authoritative topic registry.** Adding/modifying topics, payload schemas, publisher/consumer mapping. When in doubt, this wins. |
| [`standards/ENTITY_NAMING.md`](standards/ENTITY_NAMING.md) | HA entity naming conventions, disambiguation hierarchy, areas/floors, MQTT topic alignment |
| [`standards/TIME_HANDLING.md`](standards/TIME_HANDLING.md) | Timestamp conventions — UTC for storage and transport, local time for scheduling and presentation, ISO 8601 format, DST handling |

---

### Node-RED

Everything needed to build and maintain Node-RED flows in Highland.

| Document | When to Reference |
|----------|------------------|
| [`nodered/OVERVIEW.md`](nodered/OVERVIEW.md) | Flow types, tab naming, groups, link nodes, node naming, flow registration pattern |
| [`nodered/ENVIRONMENT.md`](nodered/ENVIRONMENT.md) | Node.js modules in function nodes, context stores, settings.js setup, HA integration node, User-Agent for external HTTP |
| [`nodered/STARTUP_SEQUENCING.md`](nodered/STARTUP_SEQUENCING.md) | Two-condition gate, echo probe, Initializers ready signal, state vs. event handling during init, degraded state and recovery |
| [`nodered/CONFIG_MANAGEMENT.md`](nodered/CONFIG_MANAGEMENT.md) | Config file structure, Config Loader flow, all example config file schemas, secrets.json |
| [`nodered/DEVICE_REGISTRY.md`](nodered/DEVICE_REGISTRY.md) | Utility: Device Registry — HA-sourced registry, models block, Command Dispatcher, ACK Tracker |
| [`nodered/INITIALIZERS.md`](nodered/INITIALIZERS.md) | Utility: Initializers — registered helpers, `utils.formatStatus`, adding new helpers |
| [`nodered/SUBFLOWS.md`](nodered/SUBFLOWS.md) | Initializer Latch, Connection Gate — interfaces, environment variables, behavior |
| [`nodered/LOGGING.md`](nodered/LOGGING.md) | JSONL framework, log levels, per-flow threshold, Utility: Logging, auto-notify behavior, jq queries |
| [`nodered/NOTIFICATIONS.md`](nodered/NOTIFICATIONS.md) | Notification payload, severity levels, target addressing, Utility: Notifications, HA Companion delivery, action responses |
| [`nodered/SCHEDULING.md`](nodered/SCHEDULING.md) | Utility: Scheduling, periods and triggers, startup recovery, period-aware flow pattern |
| [`nodered/HEALTH_MONITORING.md`](nodered/HEALTH_MONITORING.md) | Health Monitor flow, Healthchecks.io architecture, failure signature matrix, Utility: Connections |
| [`nodered/BATTERY_MONITOR.md`](nodered/BATTERY_MONITOR.md) | Utility: Battery Monitor — states, notifications, device catalog, startup recovery |
| [`nodered/DAILY_DIGEST.md`](nodered/DAILY_DIGEST.md) | Utility: Daily Digest — data sources, email design, Meteocons CID icons, recipients |
| [`nodered/INTERNET_CONNECTIVITY.md`](nodered/INTERNET_CONNECTIVITY.md) | Utility: Internet Connectivity — connectivity pipeline (multi-host ping), throughput pipeline (Speedtest), HA Discovery, pending alerting and logging |

---

### Home Assistant

HA configuration and credentials reference.

| Document | When to Reference |
|----------|------------------|
| [`ha/HA_CONFIG.md`](ha/HA_CONFIG.md) | YAML directory structure, configuration.yaml, recorder.yaml, rest_commands.yaml, health monitoring automations |
| [`ha/SECRETS_TEMPLATE.md`](ha/SECRETS_TEMPLATE.md) | **Credentials template.** All credentials across all hosts — populate offline, never commit real values |

---

### Subsystems

Domain-specific designs. Each subsystem is fully designed and ready for implementation.

| Document | Status | When to Reference |
|----------|--------|------------------|
| [`subsystems/APPLIANCE_MONITORING.md`](subsystems/APPLIANCE_MONITORING.md) | ✅ Designed | ZEN15 cycle detection, energy gate, per-appliance config, PostgreSQL schema |
| [`subsystems/DISHWASHER_ATTENTION.md`](subsystems/DISHWASHER_ATTENTION.md) | ✅ Designed | Dishwasher attention state machine — tilt sensor, button, guest heuristic, CLEAN_UNATTENDED/LIKELY_EMPTY states |
| [`subsystems/WASHER_DRYER_ATTENTION.md`](subsystems/WASHER_DRYER_ATTENTION.md) | ✅ Designed | Washer & dryer attention state machines — PIR presence, button, UNATTENDED/LIKELY_ATTENDED states |
| [`subsystems/CALENDAR_INTEGRATION.md`](subsystems/CALENDAR_INTEGRATION.md) | ✅ Designed | Google Calendar bridge, attendee-based camera suppression, stateless re-derivation |
| [`subsystems/DELIVERIES.md`](subsystems/DELIVERIES.md) | 📋 Planned | `Utility: Deliveries` flow — delivery informational layer; USPS Informed Delivery letter-mail baseline (consumes `Utility: Email Ingress`); future phases add multi-carrier packages and LoRa mailbox sensor fusion; owns `highland/state/deliveries/*` contract |
| [`subsystems/EMAIL_INGRESS.md`](subsystems/EMAIL_INGRESS.md) | 📋 Planned | `Utility: Email Ingress` flow — single owner for IMAP polling of household Gmail account; normalizes messages and publishes to `highland/event/email/<label>/received` with ACK-gated lifecycle |
| [`subsystems/GARAGE_DOOR.md`](subsystems/GARAGE_DOOR.md) | ✅ Designed | Konnected GDO blaQ, SSE stream integration, REST commands, MQTT Discovery |
| [`subsystems/LORA.md`](subsystems/LORA.md) | ✅ Designed | LoRaWAN gateway relay, bin monitoring state machine, mailbox activity sensor (delivery logic owned by `subsystems/DELIVERIES.md`) |
| [`subsystems/PARKING_ASSIST.md`](subsystems/PARKING_ASSIST.md) | ✅ Designed | Per-bay ultrasonic vehicle position indicator, AtomS3 Lite + Atomic RS485 Base + Ultrasonic Unit U098-B1 sensor nodes, six-band distance FSM, button-based calibration; indicator hardware pending (WS2812B strip vs. industrial tower light with buzzer) |
| [`subsystems/STAIR_LIGHTING.md`](subsystems/STAIR_LIGHTING.md) | ✅ Designed | Continuous wall-side RGB IC FCOB accent lighting in single-chamber U-channel, M5Stack Atom + Atomic RS485 Base + ToF Unit sensor nodes integrated at channel endpoints, GLEDOPTO GL-C-015WL-D WLED controller in adjacent bedroom, direction-inference FSM, schedule + outdoor lux active window gating |
| [`subsystems/VIDEO_PIPELINE.md`](subsystems/VIDEO_PIPELINE.md) | ✅ Designed | Three-stage analysis ladder, CPAI triage, Gemini analysis, zone filtering, cooldown/kill switch |
| [`subsystems/WEATHER_FLOW.md`](subsystems/WEATHER_FLOW.md) | 🔄 Tier 1 + Radar + Station + Analysis Live | NWS forecast + alerts live; radar pipeline live; Tempest station live; Weather Analysis live (OWM minutely + Open-Meteo convective, Tempest ground truth, MinuteCast notifications); Weather Lightning live |
| [`subsystems/EUFY_LOCKS.md`](subsystems/EUFY_LOCKS.md) | 📋 Planned | eufy-bridge TypeScript service, MQTT integration for Eufy Wi-Fi locks, lock-only scope, secondary account strategy |
| [`subsystems/LANDROID.md`](subsystems/LANDROID.md) | 📋 Planned | Worx Landroid Vision WR344 — Mosquitto bridge to AWS IoT Core, normalized state, error/rain delay notifications |
| [`subsystems/MATTER.md`](subsystems/MATTER.md) | 📋 Planned | `matter.js`-based bridge service, dedicated PoE Thread Border Router, `matter/#` private topic surface translated to `highland/#`, triggered by specific device need |
| [`subsystems/ai/ASSIST_PIPELINE.md`](subsystems/ai/ASSIST_PIPELINE.md) | 📋 Planned | HA Assist voice pipeline, two-tier conversation agent, Echo Show experiment, satellite targeting |
| [`subsystems/ai/PERSISTENT_MEMORY.md`](subsystems/ai/PERSISTENT_MEMORY.md) | ⏸ Blocked | AI memory architecture — blocked on hardware and HA pipeline event access |

---

### Implementation

| Document | Purpose |
|----------|---------|
| [`RUNBOOK.md`](RUNBOOK.md) | **Step-by-step build guide.** Phase-by-phase infrastructure setup. Work through this sequentially. |
| [`AUTOMATION_BACKLOG.md`](AUTOMATION_BACKLOG.md) | Captured ideas for future automations. Not requirements. |

---

## Current State

**What's done:**
- Architecture finalized (hardware, topology, backup strategy)
- Event architecture and MQTT topic registry established
- Entity naming standards established
- Node-RED patterns fully documented (flows, config, logging, notifications, health monitoring, all utility flows)
- Domain-specific designs complete (video pipeline, weather, calendar, LoRaWAN, garage door, appliances)
- Voice/AI pipeline designed (blocked on hardware prerequisites)
- Implementation runbook complete
- Communication Hub live: Mosquitto, Zigbee2MQTT, Z-Wave JS UI
- HAOS live: Node-RED, PostgreSQL, all utility flows operational
- Radar pipeline live: Python daemon on hub, SFTP delivery to HAOS, Node-RED MQTT integration
- Weather station live: WeatherFlow Tempest via UDP relay, 14 HA entities via MQTT Discovery
- Weather Analysis live: OWM minutely forecast (90s cadence) + Open-Meteo convective instability, Tempest ground truth `minutely[0]` replacement, MinuteCast notifications, `threat_type` entity state, HA Discovery sensor
- Weather Lightning live: hyperlocal Tempest strike detection, proximity/distance notification tiers, state aging to Clear
- Weather Alerts enhanced: three-tier deduplication, expiry pre-filter, per-alert notifications, consolidated state topic, Weather Alerts Card on dashboard, HA Discovery sensor

**What's next:**
- Security system: Node-RED FSM, Noonlight integration, Frient keypads
- Hygrostat: RoC-based bathroom fan, Inovelli Blue LED bar
- LoRaWAN: bin monitoring, mailbox delivery detection
- Garage door: Konnected GDO blaQ bridge
- MinuteCast dashboard visualization: apexcharts-card bar chart (designed — issue #30)
- Remaining subsystems per `AUTOMATION_BACKLOG.md`

---

## Protocols

### Privacy & Security

This repository is public. When creating or updating documents, always redact:
- GPS coordinates — use `secrets.json` references
- Domain names — use `your-domain.example` as placeholder
- Email addresses — use generic descriptions or `secrets.json` references
- Internal hostnames/IPs — use `.local` mDNS names or `192.168.x.x`
- API keys, tokens, passwords — always use placeholders

When in doubt: *"Could someone use this to locate or access the system?"* If yes, redact it.

### Capturing Ideas

New automation ideas → `AUTOMATION_BACKLOG.md`. Don't derail current work; capture and move on.

### Working Style

- Peer-level, informal — well-acquainted colleagues
- Pragmatic over perfect
- Separation of concerns is a core value
- Intentional design choices over convenience
- "Zero-baggage" migration — rebuild from scratch, don't copy legacy

---

*Last Updated: 2026-04-28*

