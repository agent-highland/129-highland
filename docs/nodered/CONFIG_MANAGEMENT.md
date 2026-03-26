# Node-RED — Configuration Management

## Overview

Centralized configuration using external JSON files. Config is separated from code and from secrets. All files live at `/home/nodered/config/` on the Workflow host, volume-mounted into the Node-RED container at `/config`.

---

## File Structure

```
/home/nodered/config/
├── device_catalog.json         ← git: yes (model battery specs, friendly name overrides)
├── device_registry.json        ← git: yes
├── flow_registry.json          ← git: yes (area→device mappings, if persisted)
├── location.json               ← git: yes (lat/lon, timezone, elevation)
├── notifications.json          ← git: yes (recipient mappings, channels)
├── system.json                 ← git: yes (system identity, HTTP defaults)
├── thresholds.json             ← git: yes (battery, health, etc.)
├── healthchecks.json           ← git: yes (service config)
├── secrets.json                ← git: NO (.gitignore)
└── README.md                   ← git: yes (documents config structure)
```

> **Scheduler configuration** (periods, sunrise/sunset) lives in schedex nodes within the Scheduler flow, not external config. `location.json` is the authoritative source for coordinates — schedex nodes reference it as documentation but require the values to be entered manually in the UI.

---

## Config Categories

| Category | Examples | Version Control |
|----------|----------|-----------------|
| **Structural** | Device registry, flow registry, notification recipients | Yes |
| **Tunable** | Thresholds, scheduler times, timeouts | Yes |
| **Secrets** | API keys, credentials, tokens, passwords | **No** |

---

## Example Config Files

### system.json

```json
{
  "http": {
    "user_agent": "(Highland-SmartHome, highland@your-domain.example)"
  }
}
```

### location.json

```json
{
  "latitude": "REDACTED — see secrets.json",
  "longitude": "REDACTED — see secrets.json",
  "timezone": "America/New_York",
  "elevation_ft": 367
}
```

### thresholds.json

```json
{
  "battery": {
    "warning": 35,
    "critical": 15
  },
  "health": {
    "disk_warning": 70,
    "disk_critical": 90,
    "cpu_warning": 80,
    "cpu_critical": 95,
    "memory_warning": 80,
    "memory_critical": 95,
    "devices_offline_critical_percent": 20
  },
  "ack": {
    "default_timeout_seconds": 30
  },
  "weather": {
    "poll_dormant_to_monitor_probability": 0.40,
    "poll_monitor_to_active_probability": 0.70,
    "poll_monitor_to_active_intensity": 0.01,
    "poll_active_to_monitor_intensity_clear": 0.005,
    "poll_active_to_monitor_sustained_minutes": 30,
    "poll_lightning_cape": 2500,
    "heavy_rain_intensity": 0.30,
    "snow_notification_accumulation_inches": 2.0,
    "ensemble_spread_confidence_ratio": 0.75
  }
}
```

### notifications.json

```json
{
  "people": {
    "joseph": {
      "admin": true,
      "channels": {
        "ha_companion": "notify.mobile_app_joseph_phone"
      }
    },
    "spouse": {
      "admin": false,
      "channels": {
        "ha_companion": "notify.mobile_app_spouse_phone"
      }
    }
  },
  "areas": {
    "living_room": {
      "channels": {
        "tv": {
          "media_player": "media_player.living_room_tv",
          "sources": [
            { "name": "FIOS TV", "type": "android_tv" },
            { "name": "Xbox", "type": "webos" }
          ],
          "endpoints": {
            "android_tv": "notify.living_room_stb",
            "webos": "notify.living_room_lg_tv"
          }
        }
      }
    }
  },
  "daily_digest": {
    "enabled": true,
    "recipients": ["joseph"]
  },
  "defaults": {
    "admin_only": ["joseph"],
    "all": ["joseph", "spouse"]
  }
}
```

**Target addressing:** Notifications use a `targets` array of namespaced strings (`namespace.key.channel`). The `*` wildcard expands all keys in a namespace section.

| Example target | Resolves to |
|----------------|-------------|
| `people.joseph.ha_companion` | Joseph's phone via HA Companion |
| `people.*.ha_companion` | All people's HA Companion |
| `areas.living_room.tv` | Living room TV |

### secrets.json (gitignored)

```json
{
  "mqtt": {
    "username": "svc_nodered",
    "password": "..."
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "user": "...",
    "password": "..."
  },
  "location": {
    "latitude": 00.0000,
    "longitude": -00.0000
  },
  "weather_api_key": "...",
  "google_calendar_api_key": "...",
  "healthchecks_io": {
    "node_red": "https://hc-ping.com/uuid",
    "mqtt": "https://hc-ping.com/uuid",
    "z2m": "https://hc-ping.com/uuid",
    "zwave": "https://hc-ping.com/uuid",
    "ha": "https://hc-ping.com/uuid"
  },
  "email_addresses": {
    "joseph": "Joseph Ferris <joseph@example.com>",
    "home": "Ferris Smart Home <home@example.com>"
  },
  "ai_providers": {
    "openai_api_key": "...",
    "anthropic_api_key": "..."
  }
}
```

See `ha/SECRETS_TEMPLATE.md` for the full credentials reference across all hosts.

---

## Config Loader Utility Flow

Loads all config files into global context on startup, deploy, or MQTT reload command.

**Triggers:**
- Node-RED startup
- Node-RED deploy
- Manual inject
- `highland/command/config/reload`
- `highland/command/config/reload/{config_name}`

**Actions:**
1. Read each JSON file from `/config/`
2. Validate JSON structure
3. Store in global context:
   - `global.config.deviceCatalog`
   - `global.config.deviceRegistry`
   - `global.config.location`
   - `global.config.notifications`
   - `global.config.system`
   - `global.config.thresholds`
   - `global.config.healthchecks`
   - `global.config.secrets`
4. Publish `highland/status/config/loaded` (retained)
5. Log: "Config loaded: {list}"

On partial failure (one file fails to parse), log the error and continue loading remaining files. Don't crash Node-RED over a config file — load what's available and report what failed in the status payload.

---

## Accessing Config in Flows

```javascript
// System identity (User-Agent for external HTTP calls)
const userAgent = global.get('config')?.system?.http?.user_agent;

// Location
const location = global.get('config.location');
const { latitude, longitude, timezone } = location;

// Thresholds
const batteryWarn = global.get('config.thresholds.battery.warning');
const batteryCrit = global.get('config.thresholds.battery.critical');

// Secrets
const apiKey = global.get('config.secrets.weather_api_key');

// Notification recipients
const adminRecipients = global.get('config.notifications.defaults.admin_only');

// Device info
const device = global.get('config.deviceRegistry')?.['foyer_entry_door'];
const friendlyName = device?.friendly_name;
```

---

## Structural Validation

On load, validate each config file:
- JSON parses correctly
- Required fields present for each entry type
- Log errors, don't crash Node-RED

---

*Last Updated: 2026-03-26*
