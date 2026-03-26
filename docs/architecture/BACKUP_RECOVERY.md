# Backup & Recovery

## Philosophy

Each host owns its own backup. No SSH between hosts required. Backups are MQTT-triggered (or cron-fallback) and results are published back to the bus so the Backup Utility Flow can track outcomes and notify on failure.

---

## Per-Host Backup Strategy

| Component | What to Back Up | Method |
|-----------|----------------|--------|
| **HAOS** | Full HA backup (config, database, add-ons) | Native HA backup + Nabu Casa cloud sync |
| **Communication Hub** | Docker Compose file, Mosquitto config, Z2M data, Z-Wave JS data | Local script tars config volumes |
| **Workflow** | Docker Compose file, Node-RED flows (JSON), config files, PostgreSQL | Local script exports flows + tars data |

---

## Retention Policy

| Type | Retention | Notes |
|------|-----------|-------|
| Logs (JSONL) | 30 days | Daily rotation, cron cleanup |
| Local backups | 7 days | Per-host, cron cleanup |
| Nabu Casa (HA) | 3–5 backups | Managed by Nabu Casa |
| NAS (future) | TBD | Define when NAS is available |

---

## MQTT-Triggered Backup Architecture

```
Scheduler (Node-RED)
      │
      │ highland/event/scheduler/backup_daily (3:00 AM)
      ▼
Backup Utility Flow
      │
      ├──► highland/command/backup/trigger/hub
      │         │
      │         ▼
      │    Communication Hub backup script
      │         │
      │         └──► highland/event/backup/completed {host: "hub"}
      │
      ├──► Node-RED backs itself up locally
      │         │
      │         └──► highland/event/backup/completed {host: "workflow"}
      │
      └──► HA REST API: trigger HA backup
                │
                └──► Nabu Casa handles cloud sync

Backup Utility collects results, notifies on failure
```

---

## Communication Hub Backup Script

Location: `/usr/local/bin/highland-backup.sh`

Triggered by MQTT listener (`mosquitto_sub`) or cron fallback. Tars `/opt/highland/mosquitto`, `/opt/highland/zigbee2mqtt`, `/opt/highland/zwavejs`. Destination: `/var/backups/highland/`. Publishes result to MQTT on completion or failure.

See `RUNBOOK.md` — Post-Build section for the full script.

**Cron:** Runs at 3:15 AM daily as cron fallback (MQTT-triggered backup fires at 3:00 AM via Scheduler flow).

---

## Workflow Host Backup

Handled by the Backup Utility Flow in Node-RED rather than a separate script:
- Flow export via Node-RED admin API
- Tar `/home/nodered/config/` and `/home/nodered/data/`
- Tar `/home/nodered/assets/` (weather icons, static assets)

**Workflow host filesystem layout:**

```
/home/nodered/
├── config/         ← git-tracked config JSON files (secrets.json gitignored)
├── data/           ← Node-RED context storage (disk-backed flow/global context)
└── assets/
    └── weather-icons/
        └── 64/     ← Meteocons filled PNGs (236 files)
```

Volume mounts into Node-RED container:
- `/home/nodered/config` → `/config`
- `/home/nodered/assets` → `/assets`

---

## Recovery Scenarios

| Scenario | Recovery Approach |
|----------|-------------------|
| **HAOS failure** | Restore from Nabu Casa cloud or local backup; Communication Hub + Workflow continue running during recovery |
| **Communication Hub failure** | Redeploy Docker Compose, restore config volumes from backup; devices remain paired in coordinator database |
| **Workflow failure** | Redeploy Docker Compose, import flow JSON, restore config files; MQTT events queue until back online |
| **Total loss** | Restore all from backup destination; re-pair devices if coordinator database lost |

---

## MQTT Topics

| Topic | Purpose |
|-------|---------|
| `highland/command/backup/trigger` | Trigger backup on receiving host |
| `highland/command/backup/trigger/{host}` | Trigger backup on a specific host |
| `highland/event/backup/completed` | Backup completed successfully |
| `highland/event/backup/failed` | Backup failed |

See `standards/MQTT_TOPICS.md` for full payload schemas.

---

*Last Updated: 2026-03-26*
