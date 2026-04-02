# MQTT Topics — Authoritative Reference

## Purpose & Scope

This document is the authoritative registry of all `highland/` MQTT topics. It defines what exists, who owns it, whether it's retained, and what its payload looks like.

**Relationship to other docs:**
- **`standards/EVENT_ARCHITECTURE.md`** — philosophy, patterns, design rationale. Read that first.
- **`nodered/OVERVIEW.md`** — flow organization conventions, including how flows consume topics.
- **This document** — the reference. What topics actually exist, settled and locked.

---

## Namespace Summary

| Namespace | Purpose | Retained? |
|-----------|---------|----------|
| `highland/event/` | Point-in-time facts. Something happened. | No |
| `highland/state/` | Current operational truth. What is true right now. | **Always** |
| `highland/status/` | Service health and liveness. Infra concerns only. | No (heartbeats); Yes (health snapshots) |
| `highland/command/` | Imperative instructions to a service. | No |
| `highland/ack/` | Acknowledgment infrastructure. | No |

### Key Distinctions

**`highland/event/` vs `highland/state/`**

An event is something that *happened* — it fires and it's gone. Flows that miss it miss it. State is *what is currently true* — retained, always available, immediately meaningful to a restarting flow.

- Precipitation started → `event/` (it happened at a moment)
- Current synthesized weather conditions → `state/` (it's just true right now)
- Scheduler period transitioned to evening → `event/` (the transition happened)
- What period the house is currently in → `state/` (the house is in this period)

**`highland/state/` vs `highland/status/`**

State is operational — automations read it to make decisions. Status is infrastructure health — monitoring flows and dashboards read it. Don't put device sensor data under `status/`. Don't put heartbeats under `state/`.

---

## `highland/state/` Contract

This namespace is the primary integration surface between Node-RED and Home Assistant. Its rules are strict:

1. **Always retained** — that's the point. A restarting consumer gets current truth immediately.
2. **Always JSON** — structured objects, never raw scalars. HA uses `value_template` to extract fields.
3. **Single writer** — one flow owns each topic. No two flows publish to the same state topic.
4. **Source-neutral** — payload never indicates where the data came from. Consumers don't know or care whether a value originated from a physical sensor, a third-party API, or a calculated derivation.
5. **Atomic** — publish the full object on every update. Never partial patches.

### How Home Assistant Consumes `highland/state/`

Node-RED uses **MQTT Discovery** to register sensor entities with HA. HA auto-creates entities and tracks them — zero manual HA config required.

**Discovery topic pattern:**
```
homeassistant/{component}/{node_id}/{object_id}/config
```

- `{component}` — HA component type: `sensor`, `binary_sensor`, `switch`, etc.
- `{node_id}` — logical device grouping (groups related entities into one HA device)
- `{object_id}` — unique entity identifier within that device

**Example — outdoor temperature from weather conditions:**

Discovery config (published by Node-RED on startup, retained):
```
Topic: homeassistant/sensor/highland_weather/outdoor_temperature/config
```
```json
{
  "name": "Outdoor Temperature",
  "unique_id": "highland_weather_outdoor_temperature",
  "state_topic": "highland/state/weather/conditions",
  "value_template": "{{ value_json.temperature }}",
  "device_class": "temperature",
  "unit_of_measurement": "°F",
  "state_class": "measurement",
  "device": {
    "identifiers": ["highland_weather"],
    "name": "Highland Weather",
    "model": "Synthesized",
    "manufacturer": "Highland"
  }
}
```

State topic (the actual value, retained, updated on each synthesis cycle):
```
Topic: highland/state/weather/conditions
```
```json
{
  "temperature": 72.3,
  "humidity": 64,
  "wind_speed": 8.2,
  ...
}
```

HA creates a `sensor.outdoor_temperature` entity, watches `highland/state/weather/conditions`, and applies `value_template` to get `72.3`. When HA restarts, it re-reads the retained discovery config and retained state — it wakes up fully current.

**Key discovery config fields:**

| Field | Purpose |
|-------|---------|
| `device_class` | Semantic type (`temperature`, `humidity`, `pressure`, `wind_speed`, `precipitation`, `motion`, etc.) |
| `unit_of_measurement` | Scale (`°F`, `%`, `inHg`, `mph`, `in`, etc.) |
| `state_class` | History behavior: `measurement` (current value), `total_increasing` (accumulators like daily precip) |
| `value_template` | Jinja2 expression to extract scalar from JSON payload |

**Discovery is idempotent.** Publishing the same config repeatedly (e.g., on every Node-RED startup) is safe — HA ignores duplicate registrations with the same `unique_id`.

**Where discovery configs live:** The Config Loader flow publishes all discovery configs on startup. Sensor definitions live there as data — not scattered across flows.

---

## Standard Payload Envelope

All `highland/event/` and `highland/state/` payloads include:

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "{flow_name}",
  ...domain-specific fields...
}
```

`highland/state/` payloads always include all fields (full object on every publish — no partial updates).

---

## Topic Registry

### Scheduler

**Purpose:** Defines the house's daily rhythm and fires time-based task triggers.

---

#### Period State

**`highland/state/scheduler/period`** ← RETAINED

Current period the house is in. Updated when period transitions.

| | |
|--|--|
| **Publisher** | Scheduler flow |
| **Consumers** | Any flow with time-of-day behavior |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T17:35:00Z",
  "source": "scheduler",
  "period": "evening"
}
```

`period` values: `"day"` | `"evening"` | `"overnight"`

---

#### Period Transition Events

Point-in-time triggers — fire at the moment of transition. Not retained (use state topic to query current period).

**`highland/event/scheduler/day`**
**`highland/event/scheduler/evening`**
**`highland/event/scheduler/overnight`**

| | |
|--|--|
| **Publisher** | Scheduler flow (schedex) |
| **Consumers** | Flows that need to react at the moment of transition |
| **Retained** | No |

```json
{
  "timestamp": "2026-03-09T17:35:00Z",
  "source": "scheduler"
}
```

---

#### Task Events

Bespoke point-in-time triggers for specific scheduled jobs.

**`highland/event/scheduler/midnight`**
Daily boundary trigger. Fires at 00:00:00. Both the Daily Digest flow and the LoRaWAN mailbox flow subscribe to this event.

**`highland/event/scheduler/backup_daily`**
Triggers the backup orchestration flow.

| | |
|--|--|
| **Publisher** | Scheduler flow |
| **Retained** | No |

All task events carry the minimal payload:
```json
{
  "timestamp": "2026-03-09T00:00:00Z",
  "source": "scheduler"
}
```

---

### Weather

**Architecture:** Two separate Node-RED flows handle different aspects of weather data. Both are live and publishing.

- **`Utility: Weather Forecasts`** — NWS grid resolution (cronplus daily at 11:55 PM) + hourly forecast fetch from NWS forecast API.
- **`Utility: Weather Alerts`** — NWS active alerts endpoint, 30-second cronplus poll, alert lifecycle tracking.

Future Tier 2 flows (Tempest, Pirate Weather synthesis) will extend this namespace when implemented. The black-box synthesis architecture described in `subsystems/WEATHER_FLOW.md` is the target end state.

**Internal-only (never on the bus):**
- NWS raw API responses
- `flow.forecast_url` — grid-resolved forecast endpoint stored in flow context
- `flow.known_alerts` — alert lifecycle tracking map (disk-backed)

---

#### State Topics (Retained)

**`highland/state/weather/conditions`** ← RETAINED

Synthesized current conditions. Combines Tempest observations, model data, and derived calculations. Updated on each synthesis cycle.

| | |
|--|--|
| **Publisher** | Weather flow |
| **Consumers** | HA (via MQTT Discovery), any flow needing current conditions |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "weather",
  "temperature": 72.3,
  "feels_like": 70.1,
  "humidity": 64,
  "dew_point": 58.2,
  "pressure": 29.92,
  "wind_speed": 8.2,
  "wind_gust": 14.1,
  "wind_bearing": 225,
  "uv_index": 4,
  "solar_radiation": 312,
  "visibility": 10.0,
  "cloud_cover": 0.35,
  "precipitation_active": false,
  "precipitation_type": "none",
  "precipitation_rate": 0.0,
  "lightning_last_strike_distance": null,
  "lightning_last_strike_at": null
}
```

*Field list is illustrative; finalize against Tempest API and Pirate Weather v2 field set during implementation. All `-999` sentinel values from Pirate Weather must be null-coerced before inclusion.*

---

**`highland/state/weather/forecast`** ← RETAINED

NWS forecast normalized to a date-keyed period map. Updated hourly by `Utility: Weather Forecasts`. Periods extend ~7 days.

| | |
|--|--|
| **Publisher** | Utility: Weather Forecasts |
| **Consumers** | Utility: Daily Digest |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-26T14:00:00Z",
  "source": "nws",
  "forecast_url": "https://api.weather.gov/gridpoints/OKX/26,71/forecast",
  "periods": {
    "2026-03-26": {
      "daytime": {
        "temperature": 54,
        "shortForecast": "Partly Sunny",
        "detailedForecast": "Partly sunny, with a high near 54.",
        "precipChance": 4,
        "windSpeed": "2 to 13 mph",
        "windDirection": "SW",
        "icon": "sct",
        "isDaytime": true
      },
      "overnight": {
        "temperature": 40,
        "shortForecast": "Mostly Cloudy",
        "detailedForecast": "Mostly cloudy, with a low around 40.",
        "precipChance": 14,
        "windSpeed": "6 to 10 mph",
        "windDirection": "S",
        "icon": "bkn",
        "isDaytime": false
      }
    }
  }
}
```

`icon` is the NWS condition code extracted from the NWS icon URL path (e.g. `sct`, `rain`, `tsra`, `snow`). Priority hierarchy resolves compound icon URLs — precipitation/severe codes take precedence over cloud cover codes. See `subsystems/WEATHER_FLOW.md` for the full NWS icon code reference and mapping table.

---

**`highland/state/weather/alerts`** ← RETAINED

Active NWS weather alerts for the configured location. Updated every 30 seconds by `Utility: Weather Alerts`. `alerts` is an empty array when no alerts are active.

| | |
|--|--|
| **Publisher** | Utility: Weather Alerts |
| **Consumers** | Utility: Daily Digest, notification flows |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-26T14:00:00Z",
  "source": "weather_alerts",
  "alerts": [
    {
      "id": "urn:oid:2.49.0.1.840.0.abc123",
      "event": "Winter Storm Warning",
      "headline": "Winter Storm Warning issued March 26 at 2:00PM EDT",
      "severity": "Severe",
      "urgency": "Expected",
      "certainty": "Likely",
      "onset": "2026-03-26T18:00:00-04:00",
      "expires": "2026-03-27T12:00:00-04:00"
    }
  ]
}
```

`severity` values (NWS): `Extreme` | `Severe` | `Moderate` | `Minor` | `Unknown`

---

**`highland/state/weather/precipitation`** ← RETAINED

Current precipitation event state. Distinct from conditions — this is the active event lifecycle, not point-in-time intensity readings.

| | |
|--|--|
| **Publisher** | Weather flow |
| **Consumers** | HA (via MQTT Discovery), automation flows |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T14:30:00Z",
  "source": "weather",
  "state": "active",
  "type": "rain",
  "intensity": 0.08,
  "accumulation_today": 0.22,
  "event_started_at": "2026-03-09T13:45:00Z",
  "event_duration_minutes": 45
}
```

`state` values: `"none"` | `"imminent"` | `"active"` | `"tapering"` | `"done"`

---

---

#### Radar Topics

**`highland/status/weather/radar/{product}`** ← RETAINED

Current execution status of a radar product. Updated by the hub daemon at the start and end of each product run.

| | |
|--|--|
| **Publisher** | Hub weather daemon (`highland-weather-daemon`) |
| **Consumers** | `Utility: Weather Radar` (Event Listeners group) |
| **Retained** | Yes |

`{product}` values: `reflectivity` (additional products as added)

`payload` values: `"idle"` | `"running"` | `"error"`

---

**`highland/state/weather/radar/{product}/last_updated`** ← RETAINED

Unix timestamp of the most recent successful GIF render for a product.

| | |
|--|--|
| **Publisher** | Hub weather daemon |
| **Consumers** | `Utility: Weather Radar`, dashboards |
| **Retained** | Yes |

`payload`: Unix timestamp as string (e.g. `"1775040000"`)

---

**`highland/event/weather/radar/{product}/rendered`**

Fires on successful GIF assembly and HAOS delivery.

| | |
|--|--|
| **Publisher** | Hub weather daemon |
| **Consumers** | `Utility: Weather Radar`, logging pipeline |
| **Retained** | No |

```json
{
  "product": "reflectivity",
  "output_path": "/var/lib/highland/weather/assets/loops/reflectivity.gif",
  "timestamp": 1775040000
}
```

---

**`highland/event/weather/radar/{product}/error`**

Fires when a product run fails.

| | |
|--|--|
| **Publisher** | Hub weather daemon |
| **Consumers** | `Utility: Weather Radar`, logging pipeline |
| **Retained** | No |

```json
{
  "product": "reflectivity",
  "error": "RainViewer API timeout",
  "timestamp": 1775040000
}
```

---

**`highland/event/weather/radar/base_map/rendered`**

Fires when the Stadia Maps base map is rebuilt.

| | |
|--|--|
| **Publisher** | Hub weather daemon |
| **Consumers** | Logging pipeline |
| **Retained** | No |

```json
{ "timestamp": 1775040000 }
```

---

**`highland/command/weather/config`** ← RETAINED

Full weather daemon config payload. Published by `Utility: Weather Radar` on startup and on demand. The config listener on the hub receives this and writes it to disk.

| | |
|--|--|
| **Publisher** | `Utility: Weather Radar` (Node-RED) |
| **Consumer** | Hub `highland-weather-config-listener` |
| **Retained** | Yes |

Payload is the full `weather.json` structure — see `hub/weather/weather.json.template` for schema.

---

**`highland/command/weather/radar/{product}/enable`**

Runtime enable/disable of a specific radar product. The config listener patches the on-disk config; the daemon picks up the change on its next tick.

| | |
|--|--|
| **Publisher** | `Utility: Weather Radar` (Product Control group) |
| **Consumer** | Hub `highland-weather-config-listener` |
| **Retained** | No |

`payload` values: `"true"` | `"false"`

---

---

### Weather Station

**Architecture:** WeatherFlow Tempest station broadcasting UDP on port 50222 (LAN). A `socat` relay on the Workflow host (systemd: `highland-tempest-relay`) forwards to port 50223 where Node-RED's Docker container can receive it. `Utility: Weather Station` decodes and normalizes all Tempest message types.

**Message types handled:**
- `obs_st` — full observation every 1 minute → state + observation event
- `evt_precip` — optical rain sensor trigger → precipitation start event
- `evt_strike` — lightning detection → lightning event
- `rapid_wind` — wind every 3 seconds → plumbing in place, not yet processed

---

#### State Topics (Retained)

**`highland/state/weather/station`** ← RETAINED

Normalized Tempest observation. Published once per minute on every `obs_st` message. All values converted to US customary units.

| | |
|--|--|
| **Publisher** | Utility: Weather Station |
| **Consumers** | HA (via MQTT Discovery), any flow needing current conditions |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-04-02T14:22:00Z",
  "wind_lull": 0.0,
  "wind_avg": 2.9,
  "wind_gust": 4.1,
  "wind_direction": 245,
  "pressure": 29.97,
  "temperature": 41.3,
  "humidity": 98.48,
  "uv_index": 0.08,
  "solar_radiation": 27,
  "rain_accumulated": 0.012,
  "precipitation_type": "None",
  "lightning_distance": null,
  "lightning_count": 0,
  "battery": 2.668,
  "report_interval": 1,
  "source": "tempest",
  "serial_number": "ST-00022280"
}
```

`precipitation_type` values: `"None"` | `"Rain"` | `"Hail"` | `"Rain+Hail"`

`lightning_distance` is `null` when no lightning detected within the observation window, otherwise distance in miles.

All units: temperature °F, wind mph, pressure inHg, rain inches, distance miles.

---

#### Event Topics (Not Retained)

**`highland/event/weather/station/observation`**

Fires on every successfully processed `obs_st` message. Consumers subscribe to this event and read `highland/state/weather/station` on receipt rather than polling the state topic.

| | |
|--|--|
| **Publisher** | Utility: Weather Station |
| **Consumers** | Any flow reacting to weather changes |
| **Retained** | No |

```json
{
  "timestamp": "2026-04-02T14:22:00Z",
  "serial_number": "ST-00022280",
  "source": "tempest"
}
```

---

**`highland/event/weather/station/precipitation_start`**

Fires immediately when the Tempest optical sensor detects rain onset (`evt_precip`). More responsive than waiting for the next `obs_st`.

| | |
|--|--|
| **Publisher** | Utility: Weather Station |
| **Consumers** | Notification flows, automation flows |
| **Retained** | No |

```json
{
  "timestamp": "2026-04-02T14:22:00Z",
  "serial_number": "ST-00022280",
  "source": "tempest"
}
```

---

**`highland/event/weather/station/lightning`**

Fires on every lightning strike detected by the Tempest hardware (`evt_strike`).

| | |
|--|--|
| **Publisher** | Utility: Weather Station |
| **Consumers** | Notification flows, automation flows |
| **Retained** | No |

```json
{
  "timestamp": "2026-04-02T14:22:00Z",
  "distance_miles": 4.2,
  "energy": 12500,
  "serial_number": "ST-00022280",
  "source": "tempest"
}
```

---

#### Home Assistant Discovery

Published by `Utility: Weather Station` on startup (retained). All entities group under the `Highland Weather Station` device (`highland_weather_station`). State topic for all sensors: `highland/state/weather/station`.

| Entity | `unique_id` | `device_class` | Unit | `state_class` |
|--------|------------|---------------|------|---------------|
| Temperature | `highland_station_temperature` | `temperature` | °F | `measurement` |
| Humidity | `highland_station_humidity` | `humidity` | % | `measurement` |
| Pressure | `highland_station_pressure` | `atmospheric_pressure` | inHg | `measurement` |
| Wind Speed | `highland_station_wind_avg` | — | mph | `measurement` |
| Wind Gust | `highland_station_wind_gust` | — | mph | `measurement` |
| Wind Lull | `highland_station_wind_lull` | — | mph | `measurement` |
| Wind Direction | `highland_station_wind_direction` | — | ° | `measurement` |
| UV Index | `highland_station_uv_index` | — | — | `measurement` |
| Solar Radiation | `highland_station_solar_radiation` | `irradiance` | W/m² | `measurement` |
| Rain Accumulated | `highland_station_rain_accumulated` | `precipitation` | in | `total_increasing` |
| Precipitation Type | `highland_station_precipitation_type` | — | — | — |
| Lightning Distance | `highland_station_lightning_distance` | `distance` | mi | — |
| Lightning Count | `highland_station_lightning_count` | — | — | `total_increasing` |
| Battery | `highland_station_battery` | `voltage` | V | `measurement` (diagnostic) |

---

#### Infrastructure

**socat relay** (`highland-tempest-relay.service` on Workflow host)

Receives Tempest UDP broadcast on port 50222 and unicasts to `127.0.0.1:50223`. Required because Docker bridge networking does not forward broadcast traffic to containers.

```
socat UDP-RECV:50222,reuseaddr UDP-SENDTO:127.0.0.1:50223
```

Node-RED docker-compose maps `50223/udp` into the container. The `Utility: Weather Station` UDP In node listens on port 50223.

---

#### Event Topics (Not Retained)

**`highland/event/weather/precipitation_start`**

```json
{
  "timestamp": "2026-03-09T13:45:00Z",
  "source": "weather",
  "type": "rain",
  "intensity": 0.04
}
```

**`highland/event/weather/precipitation_end`**

```json
{
  "timestamp": "2026-03-09T15:30:00Z",
  "source": "weather",
  "type": "rain",
  "duration_minutes": 105,
  "accumulation": 0.31
}
```

**`highland/event/weather/precipitation_type_change`**

Fires when precipitation transitions between types during an active event (e.g., rain → freezing rain).

```json
{
  "timestamp": "2026-03-09T14:00:00Z",
  "source": "weather",
  "previous_type": "rain",
  "new_type": "ice"
}
```

**`highland/event/weather/lightning_detected`**

```json
{
  "timestamp": "2026-03-09T16:22:00Z",
  "source": "weather",
  "distance_miles": 4.2,
  "energy": 12500
}
```

**`highland/event/weather/wind_gust`**

Fires when a wind gust crosses a configured threshold. Threshold defined in `thresholds.json`.

```json
{
  "timestamp": "2026-03-09T14:15:00Z",
  "source": "weather",
  "gust_speed": 38.2,
  "threshold_crossed": 35.0
}
```

**`highland/event/weather/alert/new`** — Previously unseen alert ID appears in NWS response.

**`highland/event/weather/alert/updated`** — Known alert ID reappears with changed `updated` timestamp.

**`highland/event/weather/alert/expired`** — Known alert ID no longer present in NWS response.

All alert events carry:
```json
{
  "timestamp": "...",
  "source": "weather_alerts",
  "alert": {
    "id": "urn:oid:2.49.0.1.840.0.abc123",
    "event": "Winter Storm Warning",
    "headline": "...",
    "severity": "Severe"
  },
  "link": "https://forecast.weather.gov/..."
}
```

---

### Calendar

**`highland/state/calendar/snapshot`** ← RETAINED

Rolling 7-day window of events from all three Google sub-calendars (Appointments, Reminders, Trash & Recycling). Rebuilt from scratch on every 15-minute poll cycle.

| | |
|--|--|
| **Publisher** | Utility: Calendaring |
| **Consumers** | Utility: Daily Digest |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-26T14:32:08Z",
  "source": "calendaring",
  "window_start": "2026-03-26T14:32:08Z",
  "window_end": "2026-04-02T14:32:08Z",
  "events": [
    {
      "calendar": "appointments",
      "date": "2026-03-26",
      "title": "HVAC Service",
      "start": "2026-03-26T10:00:00-04:00",
      "end": "2026-03-26T12:00:00-04:00"
    },
    {
      "calendar": "trash",
      "date": "2026-03-27",
      "title": "Trash & Recycling Pickup"
    }
  ]
}
```

`calendar` values: `appointments` | `reminders` | `trash`. All-day events have `date` only; timed events have `start` and `end` in ISO 8601 with timezone offset.

---

**`highland/state/calendar/camera_suppression`** ← RETAINED

Authoritative snapshot of currently active camera suppression events. Updated on every Calendar Bridge poll cycle. `active` is an empty array when no events are active.

| | |
|--|--|
| **Publisher** | Calendar Bridge flow |
| **Consumers** | Camera pipeline flow |
| **Retained** | Yes |

```json
{
  "timestamp": "2026-03-09T16:05:00Z",
  "source": "calendar_bridge",
  "active": [
    {
      "event_id": "google_event_id",
      "title": "Backyard BBQ",
      "start": "2026-03-09T16:00:00Z",
      "end": "2026-03-09T22:00:00Z",
      "cameras": ["rear_yard", "rear_patio"]
    }
  ]
}
```

**`highland/event/calendar/camera_suppression/start`** — Ceremony event, fires once per calendar event when suppression begins. Requires persistent MQTT sessions on consuming flows. Includes `already_active: bool` field.

**`highland/event/calendar/camera_suppression/end`** — Fires when suppression ends.

See `subsystems/CALENDAR_INTEGRATION.md` for full payload schemas and consumer patterns.

---

### Driveway Bins

**Architecture:** Two independent LoRaWAN EM320-TILT tilt sensors. Zone detection uses RSSI/SNR from gateway uplink metadata — no additional hardware. State machines are independent. See `subsystems/LORA.md`.

**`highland/state/driveway/trash_bin`** ← RETAINED
**`highland/state/driveway/recycling_bin`** ← RETAINED

```json
{
  "timestamp": "2026-03-09T07:45:00Z",
  "source": "lora_bin_monitor",
  "state": "AWAY",
  "rssi": -108,
  "snr": -4.2,
  "battery_pct": 94
}
```

`state` values: `"HOME"` | `"AWAY_SETTLING"` | `"AWAY"` | `"PICKED_UP"` | `"RETURNED"`

**`highland/event/driveway/{bin}/zone_changed`** — Bin transitions between HOME and AWAY zones (after debounce).

**`highland/event/driveway/{bin}/picked_up`** — X-axis rotation ≥ 85° while bin is `AWAY`. Indicates truck serviced the bin.

**`highland/event/driveway/{bin}/returned`** — Bin transitions from `PICKED_UP` or `AWAY` back to `HOME`.

`{bin}` values: `trash_bin` | `recycling_bin`

---

### Mailbox

**Architecture:** LoRaWAN door/contact sensor (Milesight EM300-MCS) combined with USPS Informed Delivery email parsing. Neither signal alone is sufficient. See `subsystems/LORA.md`.

**`highland/state/mailbox/delivery`** ← RETAINED

```json
{
  "timestamp": "2026-03-09T14:22:00Z",
  "source": "lora_mailbox",
  "state": "MAIL_WAITING",
  "last_door_event": "2026-03-09T14:20:00Z",
  "advisory_received_at": "2026-03-09T06:03:00Z",
  "confirmation_received_at": "2026-03-09T14:21:00Z"
}
```

`state` values: `"IDLE"` | `"UNCLASSIFIED"` | `"ADVISORY_RECEIVED"` | `"MAIL_WAITING"` | `"DELIVERY_EXCEPTION"` | `"RETRIEVED"`

**`highland/event/mailbox/door_activity`** — Any open/close event (unclassified at publication time).

**`highland/event/mailbox/mail_expected`** | **`mail_delivered`** | **`mail_retrieved`** | **`delivery_exception`** — State transition events. All carry `{ previous_state, new_state }` payload.

---

### Garage Door

**Architecture:** Node-RED bridge to Konnected GDO blaQ via SSE stream + REST API. HA never speaks to the device directly. See `subsystems/GARAGE_DOOR.md`.

**`highland/state/garage/door`** ← RETAINED

```json
{
  "timestamp": "...",
  "source": "garage_bridge",
  "state": "CLOSED",
  "current_operation": "IDLE"
}
```

`state` values: `"OPEN"` | `"CLOSED"` — `current_operation` values: `"IDLE"` | `"OPENING"` | `"CLOSING"`

**Also retained:** `highland/state/garage/light` | `remote_lock` | `obstruction` | `motion` | `motor` | `synced` | `learn` | `openings`

**Events (not retained):** `highland/event/garage/door_opened` | `door_closed` | `obstruction_detected` | `obstruction_cleared` | `motion_detected`

**Commands:**

| Topic | `action` values |
|-------|----------------|
| `highland/command/garage/door` | `"open"` \| `"close"` \| `"stop"` \| `"toggle"` |
| `highland/command/garage/light` | `"turn_on"` \| `"turn_off"` \| `"toggle"` |
| `highland/command/garage/remote_lock` | `"lock"` \| `"unlock"` |
| `highland/command/garage/learn` | `"turn_on"` \| `"turn_off"` \| `"toggle"` |

---

### Appliance Monitoring

**Architecture:** Power-based cycle detection via ZEN15 smart plugs. See `subsystems/APPLIANCE_MONITORING.md` for full state machine and payload schemas.

**`highland/state/appliance/{appliance}/cycle`** ← RETAINED

`{appliance}` values: `washing_machine` | `dryer` | `dishwasher`

```json
{
  "timestamp": "...",
  "source": "appliance_monitor",
  "appliance": "washing_machine",
  "state": "running",
  "cycle_start": "...",
  "duration_s": 900,
  "energy_wh": 42.3,
  "power_w": 387.2,
  "max_power_w": 1820.0,
  "matched_profile": null,
  "estimated_remaining_s": null
}
```

`state` values: `off` | `starting` | `running` | `paused` | `ending` | `finished` | `interrupted` | `force_stopped`

**Events:** `highland/event/appliance/{appliance}/cycle_started` | `cycle_finished` | `cycle_interrupted`

**Commands:** `highland/command/appliance/{appliance}/force_end` | `reset`

---

### Security

**`highland/state/security/mode`** ← RETAINED

```json
{ "timestamp": "...", "source": "security", "mode": "home" }
```

`mode` values: `"home"` | `"away"` | `"lockdown"`

**Events:** `highland/event/security/lockdown` | `highland/event/security/away`

---

### Logging

**`highland/event/log`** — All log entries from all systems. QoS 2.

| | |
|--|--|
| **Publisher** | Any flow or system |
| **Consumers** | Utility: Logging (exclusive writer to disk) |
| **Retained** | No |
| **QoS** | 2 |

```json
{
  "timestamp": "...",
  "system": "node_red",
  "source": "weather",
  "level": "ERROR",
  "message": "Pirate Weather API timeout",
  "context": { "attempt": 3, "error": "ETIMEDOUT" }
}
```

`level` values: `"VERBOSE"` | `"DEBUG"` | `"INFO"` | `"WARN"` | `"ERROR"` | `"CRITICAL"`

> **Essential service:** `Utility: Logging` does not use the Initializer Latch — it inlines all required helpers directly so it remains functional even if Initializers fails.

---

### Notifications

**`highland/event/notify`**

```json
{
  "timestamp": "...",
  "source": "security",
  "severity": "high",
  "title": "Lock Failed to Engage",
  "message": "Front Door Lock did not respond within 30 seconds",
  "targets": ["people.joseph.ha_companion"],
  "icon": "mdi:lock-alert",
  "dnd_override": true,
  "actionable": true,
  "actions": [
    { "id": "retry", "label": "Retry Lock" },
    { "id": "dismiss", "label": "Dismiss" }
  ],
  "sticky": true,
  "group": "security_alerts",
  "correlation_id": "lockdown_20260309_2200"
}
```

**`highland/event/notify/action_response`**

```json
{
  "timestamp": "...",
  "source": "notification",
  "action": "retry",
  "correlation_id": "lockdown_20260309_2200",
  "device": "mobile_joseph"
}
```

**`highland/command/notify/clear`** — Dismiss a previously delivered notification by `correlation_id`.

See `nodered/NOTIFICATIONS.md` for the full Utility: Notifications implementation.

---

### Battery

**`highland/state/battery/states`** ← RETAINED

Full battery state map for all tracked devices.

| | |
|--|--|
| **Publisher** | Utility: Battery Monitor |
| **Consumers** | Utility: Daily Digest |
| **Retained** | Yes |

```json
{
  "timestamp": "...",
  "source": "battery_monitor",
  "states": {
    "office_desk_presence": { "state": "normal", "level": 100, "last_notified_critical": null },
    "front_door_lock": { "state": "low", "level": 28, "last_notified_critical": null }
  }
}
```

**`highland/event/battery/low`** | **`critical`** | **`recovered`**

```json
{
  "timestamp": "...",
  "source": "battery_monitor",
  "entity": "garage_motion_sensor",
  "level": 32,
  "previous_state": "normal",
  "new_state": "low",
  "battery": { "type": "CR2032", "quantity": 1 }
}
```

---

### Backup

**`highland/event/backup/completed`**

Published by the Hub backup script and the Backup Utility Flow on successful completion.

Hub payload:
```json
{
  "host": "hub",
  "file": "hub_backup_20260327_031500.tar.gz",
  "timestamp": "2026-03-27T03:15:00-04:00"
}
```

Workflow payload:
```json
{
  "host": "workflow",
  "status": "completed",
  "timestamp": "2026-03-27T03:15:02.000Z"
}
```

HA audit payload (last backup within 26-hour window):
```json
{
  "host": "ha",
  "status": "completed",
  "elapsed_hours": 14.2,
  "timestamp": "2026-03-27T03:15:01.000Z"
}
```

---

**`highland/event/backup/failed`**

Published on backup failure or stale HA backup detection. Triggers failure notification via Result Collection group.

Hub payload:
```json
{
  "host": "hub",
  "error": "tar failed",
  "timestamp": "2026-03-27T03:15:00-04:00"
}
```

Workflow payload:
```json
{
  "host": "workflow",
  "status": "failed",
  "error": "tar exited with rc 1",
  "timestamp": "2026-03-27T03:15:02.000Z"
}
```

HA audit payload (last backup older than 26 hours):
```json
{
  "host": "ha",
  "status": "failed",
  "elapsed_hours": 29.4,
  "timestamp": "2026-03-27T03:15:01.000Z"
}
```

---

### Status / Health

**`highland/status/initializers/ready`** — Non-retained. Session-scoped signal.

**`highland/status/config/loaded`** ← RETAINED

```json
{
  "loaded": ["device_registry", "notifications", "thresholds", "secrets"],
  "errors": [],
  "scope": "all",
  "timestamp": "..."
}
```

**`highland/status/mqtt/probe`** — Non-retained round-trip probe for MQTT edge health check. Empty payload.

**`highland/status/{service}/heartbeat`** — Simple liveness ping, not retained.

**`highland/status/{service}/health`** ← RETAINED — Detailed health snapshot. `status` values: `"healthy"` | `"degraded"` | `"unhealthy"`

**Monitored services:** `mqtt` | `z2m` | `zwave` | `ha` | `node_red`

---

### Commands

| Topic | Purpose |
|-------|---------|
| `highland/command/backup/trigger` | Trigger backup on receiving host |
| `highland/command/backup/trigger/{host}` | Trigger backup on specific host |
| `highland/command/config/reload` | Reload all config files |
| `highland/command/config/reload/{config_name}` | Reload specific config file |
| `highland/command/calendar/reload` | Force calendar bridge re-poll |

---

### ACK Infrastructure

**`highland/ack/register`** — Register ACK expectation

```json
{ "correlation_id": "lockdown_20260309_2200", "expected_sources": ["foyer_entry_door", "garage_entry_door"], "timeout_seconds": 30, "source": "security" }
```

**`highland/ack`** — ACK response

```json
{ "ack_correlation_id": "lockdown_20260309_2200", "source": "foyer_entry_door", "timestamp": "..." }
```

**`highland/ack/result`** — Outcome after timeout

```json
{ "correlation_id": "lockdown_20260309_2200", "expected": 2, "received": 1, "sources": ["foyer_entry_door"], "missing": ["garage_entry_door"], "success": false }
```

---

## Retention Summary

| Topic Pattern | Retained |
|---------------|----------|
| `highland/state/#` | **Always** |
| `highland/status/+/health` | Yes |
| `highland/status/+/heartbeat` | No |
| `highland/event/#` | **No** |
| `highland/command/#` | No |
| `highland/ack/#` | No |

---

## Wildcard Subscription Patterns

| Pattern | Use Case |
|---------|----------|
| `highland/state/#` | All retained operational state |
| `highland/state/weather/#` | All weather state |
| `highland/state/driveway/#` | Both bin states |
| `highland/state/garage/#` | All garage state |
| `highland/state/appliance/#` | All appliance cycle state |
| `highland/event/weather/station/#` | All weather station events |
| `highland/event/weather/radar/#` | All radar events |
| `highland/event/weather/radar/+/rendered` | Any product rendered |
| `highland/event/weather/radar/+/error` | Any product error |
| `highland/status/weather/radar/+` | All radar product status |
| `highland/state/weather/radar/+/last_updated` | All radar last-updated state |
| `highland/command/weather/radar/+/enable` | All radar enable/disable commands |
| `highland/event/scheduler/#` | All scheduler events |
| `highland/event/driveway/#` | All bin events |
| `highland/event/mailbox/#` | All mailbox events |
| `highland/event/garage/#` | All garage events |
| `highland/event/appliance/#` | All appliance cycle events |
| `highland/event/appliance/+/cycle_finished` | Any machine finishing |
| `highland/event/+/leak/#` | Any leak in any area |
| `highland/event/+/motion_detected` | Any motion in any area |
| `highland/status/#` | All health and heartbeat |
| `highland/status/+/health` | Health snapshots only |
| `highland/command/backup/#` | All backup commands |
| `highland/command/garage/#` | All garage commands |
| `highland/command/appliance/#` | All appliance commands |

---

## Domains Pending Definition

| Domain | Notes |
|--------|-------|
| **Presence / Occupancy** | FP300 sensors; `highland/state/{area}/occupancy` likely pattern |
| **Video Pipeline** | Kill switch, detection events, triage results |
| **HA Assist / Voice** | Marvin persona events; may not need bus presence |
| **Area sensors** | Zigbee environmental, motion, contact |
| **Lighting state** | TBD if lighting state needs bus representation beyond Z2M raw topics |
| **Entry Door Locks** | Z-Wave/Zigbee door locks; state beyond what Z2M/ZWaveJS already exposes |

---

*Last Updated: 2026-04-02*
