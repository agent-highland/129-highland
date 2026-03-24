# Weather Flow — Design & Architecture

## Overview

Weather data in Highland is organized into two tiers with distinct responsibilities:

**Tier 1 — `Utility: Weather`** — External data ingestion. Polls NWS APIs for forecast and alert data, normalizes the responses, and publishes retained MQTT snapshots. Simple consumers (Daily Digest, dashboards, voice assistant) read from here exclusively. No automation logic lives here.

**Tier 2 — `Utility: Nowcast`** *(future)* — Hyperlocal synthesis and automation. Combines Tempest local station data with NWS and Pirate Weather API data to drive precipitation tracking, nowcasting, and actionable notifications. The Tempest provides hyperlocal ground truth that calibrates everything else in this tier.

This separation keeps the data layer simple and stable while allowing the analysis layer to grow in complexity independently.

---

## Tier 1: Utility: Weather

### Flow Structure

```
Utility: Weather
├── Group: Grid Resolution   — daily re-derive of NWS forecast URL from coordinates
├── Group: Forecast          — hourly poll, normalize periods, publish snapshot
├── Group: Alerts            — 30-second poll, lifecycle tracking, publish snapshot + events
└── Group: Error Handling    — flow-wide catch
```

### Group: Grid Resolution

NWS forecast URLs are grid-specific, derived from a lat/lon lookup. The grid assignment is stable but should be re-derived programmatically rather than hardcoded.

**Process (once daily):**
1. GET `https://api.weather.gov/points/{latitude},{longitude}`
2. Extract `properties.forecast` from response
3. Store result in `flow.forecast_url`

**Example:**
- Points URL: `https://api.weather.gov/points/41.5204,-74.0606`
- Derived forecast URL: `https://api.weather.gov/gridpoints/OKX/26,71/forecast`

Coordinates sourced from `global.config.location` (`location.json`). The derived URL is stored in flow context and consumed by the Forecast group on each poll.

### Group: Forecast

**Poll cadence:** Every 60 minutes. NWS updates forecasts twice daily but recommends hourly polling to catch unscheduled updates during periods of forecast uncertainty.

**Startup behavior:** Poll immediately on startup, then on 60-minute interval.

**Process:**
1. GET `flow.forecast_url` (derived by Grid Resolution group)
2. Parse `properties.periods` array
3. Normalize all periods into a date-keyed structure
4. Publish retained snapshot to `highland/state/weather/forecast`

**Period normalization:**

NWS returns periods as a flat array of daytime/overnight alternating entries. The flow normalizes these into a date-keyed map using `startTime`/`endTime` — never relying on period `name` strings (which vary by time of day) or array position.

```javascript
// Normalized structure — keyed by local date string YYYY-MM-DD
{
  "2026-03-24": {
    daytime: {
      temperature: 49,
      shortForecast: "Sunny",
      detailedForecast: "Sunny, with a high near 49...",
      precipChance: 0,
      windSpeed: "3 to 9 mph",
      windDirection: "SW",
      icon: "few",          // extracted from NWS icon URL
      isDaytime: true
    },
    overnight: {
      temperature: 31,
      shortForecast: "Partly Cloudy",
      detailedForecast: "Partly cloudy, with a low around 31...",
      precipChance: 1,
      windSpeed: "2 to 9 mph",
      windDirection: "SW",
      icon: "sct",
      isDaytime: false
    }
  },
  "2026-03-25": { ... },
  ...
}
```

**Icon extraction:**

NWS icon URLs encode condition codes. Extract the primary condition code from the URL path:

```
https://api.weather.gov/icons/land/day/bkn/rain,20?size=medium
                                       ^^^  ^^^^
                                  base code  secondary (precipitation wins)
```

When two codes are present, precipitation/severe codes take priority over cloud cover codes. See icon mapping table in Tier 1 implementation notes.

**NWS icon code reference:**

| Code | Description | Code | Description |
|------|-------------|------|-------------|
| `skc` | Fair/clear | `rain` | Rain |
| `few` | A few clouds | `rain_showers` | Rain showers (high cloud cover) |
| `sct` | Partly cloudy | `rain_showers_hi` | Rain showers (low cloud cover) |
| `bkn` | Mostly cloudy | `tsra` | Thunderstorm (high cloud cover) |
| `ovc` | Overcast | `tsra_sct` | Thunderstorm (medium cloud cover) |
| `wind_skc` | Fair and windy | `tsra_hi` | Thunderstorm (low cloud cover) |
| `wind_few` | A few clouds, windy | `tornado` | Tornado |
| `wind_sct` | Partly cloudy, windy | `hurricane` | Hurricane conditions |
| `wind_bkn` | Mostly cloudy, windy | `tropical_storm` | Tropical storm |
| `wind_ovc` | Overcast and windy | `dust` | Dust |
| `snow` | Snow | `smoke` | Smoke |
| `rain_snow` | Rain/snow | `haze` | Haze |
| `rain_sleet` | Rain/sleet | `hot` | Hot |
| `snow_sleet` | Snow/sleet | `cold` | Cold |
| `fzra` | Freezing rain | `blizzard` | Blizzard |
| `rain_fzra` | Rain/freezing rain | `fog` | Fog/mist |
| `snow_fzra` | Freezing rain/snow | | |
| `sleet` | Sleet | | |

*Note: NWS icon URLs are technically deprecated but replacement format has not been announced. Use until replacement is documented.*

### Group: Alerts

**Poll cadence:** Every 30 seconds. NWS recommendation for alert endpoints.

**Endpoint:** `https://api.weather.gov/alerts/active?point={latitude},{longitude}`

Coordinates sourced from `global.config.location`.

**Lifecycle tracking:**

The flow maintains `flow.known_alerts` (disk-backed) — a map of `{ alert_id: { sent, event, severity, headline, ends } }`. On each poll, the incoming alert set is diffed against the known set to detect transitions:

| Transition | Detection | Action |
|-----------|-----------|--------|
| New | ID not in `known_alerts` | Add to known set, fire new event, notify |
| Updated | ID in `known_alerts`, `sent` timestamp changed | Update known set, fire updated event |
| Expired | ID in `known_alerts`, absent from response AND `ends` is past | Remove from known set, fire expired event |
| Ongoing | ID in `known_alerts`, no change | No action |

The `id` field (URN string) is the stable identifier. The `sent` timestamp changes when NWS revises an existing alert.

**Alert severity → notification severity mapping:**

| NWS `severity` | Notification severity |
|---|---|
| `Extreme` | `critical` |
| `Severe` | `high` |
| `Moderate` | `medium` |
| `Minor` | `low` |

**Fields used from alert response:**
- `properties.id` — stable identifier for lifecycle tracking
- `properties.event` — alert type ("Wind Advisory", "Red Flag Warning", etc.)
- `properties.headline` — one-line summary including issuing office and time range
- `properties.severity` — drives notification severity
- `properties.sent` — change detection for updates
- `properties.ends` — expiry time for display and lifecycle management

**Fields intentionally ignored:**
- `properties.description` — too verbose for notifications or digest
- `properties.instruction` — same
- All geocode/zone fields

**Notification deep link:**

Alert notifications include a `url` field pointing to the NWS point forecast page for the property:

```
https://forecast.weather.gov/MapClick.php?lat=41.5204&lon=-74.0606
```

This URL is stable, derived from `location.json` coordinates, and takes the user directly to a page showing active alerts and the current forecast. Replaces the generic HA dashboard fallback used in the live implementation.

**Special Weather Statements:**

NWS `Special Weather Statement` alerts often have less structured `description` text than other alert types. Monitor for parsing/display issues when these occur in practice. No special handling designed yet — revisit when a real example is observed.

### MQTT Topics (Tier 1)

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/state/weather/forecast` | Yes | Rolling 7-day normalized forecast snapshot |
| `highland/state/weather/alerts` | Yes | Current active alert set |
| `highland/event/weather/alert/new` | No | New alert detected |
| `highland/event/weather/alert/updated` | No | Existing alert modified by NWS |
| `highland/event/weather/alert/expired` | No | Alert no longer active |

### Snapshot Payloads

**Forecast snapshot (`highland/state/weather/forecast`):**
```json
{
  "timestamp": "2026-03-24T12:00:00Z",
  "source": "nws",
  "forecast_url": "https://api.weather.gov/gridpoints/OKX/26,71/forecast",
  "periods": {
    "2026-03-24": {
      "daytime": {
        "temperature": 49,
        "shortForecast": "Sunny",
        "detailedForecast": "Sunny, with a high near 49. Southwest wind 3 to 9 mph.",
        "precipChance": 0,
        "windSpeed": "3 to 9 mph",
        "windDirection": "SW",
        "icon": "skc"
      },
      "overnight": {
        "temperature": 31,
        "shortForecast": "Partly Cloudy",
        "detailedForecast": "Partly cloudy, with a low around 31.",
        "precipChance": 1,
        "windSpeed": "2 to 9 mph",
        "windDirection": "SW",
        "icon": "sct"
      }
    }
  }
}
```

**Alerts snapshot (`highland/state/weather/alerts`):**
```json
{
  "timestamp": "2026-03-24T12:00:00Z",
  "source": "nws",
  "alerts": [
    {
      "id": "urn:oid:2.49.0.1.840.0...",
      "event": "Wind Advisory",
      "headline": "Wind Advisory issued March 24 at 8:21AM EDT until March 24 at 6:00PM EDT by NWS Albany NY",
      "severity": "Moderate",
      "sent": "2026-03-24T08:21:00-04:00",
      "ends": "2026-03-24T18:00:00-04:00"
    }
  ]
}
```

When no alerts are active, `alerts` is an empty array — not absent. Consumers can distinguish "no active alerts" from "retained state not yet received."

**Alert event payload (`highland/event/weather/alert/new`):**
```json
{
  "timestamp": "2026-03-24T12:00:05Z",
  "source": "weather",
  "event": "Wind Advisory",
  "headline": "Wind Advisory issued March 24 at 8:21AM EDT until March 24 at 6:00PM EDT by NWS Albany NY",
  "severity": "Moderate",
  "ends": "2026-03-24T18:00:00-04:00",
  "url": "https://forecast.weather.gov/MapClick.php?lat=41.5204&lon=-74.0606"
}
```

### Consumer Pattern

Consumers subscribe to retained snapshot topics and filter as needed:

```javascript
// Daily Digest — today's forecast
const forecast = JSON.parse(msg.payload);
const today = new Date().toLocaleDateString('en-CA', { timeZone: 'America/New_York' });
const todayForecast = forecast.periods[today];

// Daily Digest — active alerts
const alerts = JSON.parse(msg.payload);
const hasAlerts = alerts.alerts.length > 0;
```

The `node.status()` on alert-related nodes should update on state change only — not on every 30-second poll — to avoid constant editor flickering.

---

## Tier 2: Utility: Nowcast *(future)*

### Overview

Hyperlocal weather synthesis and automation. Combines local Tempest station data with NWS Tier 1 snapshots and Pirate Weather API data to drive precipitation tracking, event lifecycle management, and actionable notifications.

The Tempest provides hyperlocal ground truth — actual observed conditions at the property — that calibrates and overrides model data for present-moment values. Nowcasting (near-term precipitation prediction) uses the Tempest's real-time observations as the primary signal, with API data providing the broader forecast context.

**Not started. Do not build until Tier 1 is stable.**

### Flow Structure

```
Utility: Nowcast
├── Group: Tempest           — local station MQTT ingestion
├── Group: Pirate Weather    — variable-cadence API polling state machine
├── Group: Precipitation     — event lifecycle state machine
├── Group: Synthesis         — combine Tempest + API into authoritative conditions
└── Group: Error Handling
```

### Data Sources

**WeatherFlow Tempest (local station):**
- Temperature, humidity, dew point, pressure
- Wind speed, gust, bearing
- UV index, solar radiation
- Lightning strike distance and energy (hardware detection)
- Local precipitation (optical sensor — supplementary, not primary for rate)

**Role:** Observations are ground truth for current conditions. Tempest data takes priority over model data for present-moment values. Lightning events come exclusively from Tempest hardware detection.

**Pirate Weather API:**
- Minutely precipitation forecast (type-specific intensity, probability)
- Hourly and daily forecast
- Ensemble spread (`precipIntensityError`)
- CAPE for convective threat assessment
- Accumulated precipitation (`currentDaySnow`, `currentDayLiquid`, etc.)

**Base URL:** `https://api.pirateweather.net/forecast/[apikey]/[lat],[lon]?units=us&version=2`

- `units=us` — Fahrenheit, inches, mph. Confirmed from live response.
- `version=2` — Required for type-specific precipitation fields, CAPE, ensemble spread.

**Credentials:** `config.secrets.pirate_weather_api_key`

**Ultrasonic snow depth sensor (future hardware):**
- Ground truth for snow accumulation depth
- Supplements API `currentDaySnow` with actual measured depth
- Hardware TBD; will require calibration against bare ground baseline

### Polling State Machine

Variable-cadence polling scales API cost to actual weather threat level.

| State | Interval | Purpose |
|-------|----------|---------|
| `POLL_DORMANT` | 15 min | No precipitation expected |
| `POLL_MONITOR` | 5 min | Precipitation possible within 6 hours |
| `POLL_ACTIVE` | 1 min | Precipitation imminent or occurring |
| `POLL_LIGHTNING` | 1 min | Thunderstorm active (distinct for notification logic) |

**Transitions:**

- DORMANT → MONITOR: any hourly block within 6 hrs `precipProbability > 0.40`
- MONITOR → ACTIVE: minutely `precipIntensity > 0.01 in/hr` AND `precipProbability > 0.70`, OR hourly within 2 hrs `precipProbability > 0.80`
- ACTIVE → MONITOR: all minutely blocks `precipIntensity < 0.005 in/hr` for 30 consecutive minutes AND no hourly within 2 hrs exceeds `precipProbability > 0.70`
- MONITOR → DORMANT: all hourly within 6 hrs `precipProbability < 0.25`; hold 15 min minimum in MONITOR before allowing
- Any → LIGHTNING: `cape > 2500 J/kg` in current or next 2 hourly blocks AND `precipProbability > 0.50`
- LIGHTNING → ACTIVE: `cape < 500 J/kg` sustained 2 consecutive polls AND no lightning detection

**`precipIntensityError` confidence gate:** If `precipIntensityError > precipIntensity * 0.75`, treat forecast as lower confidence and require higher `precipProbability` for escalation. Specific threshold values TBD during calibration.

**`exclude` parameter strategy:**

| State | `exclude` |
|-------|-----------|
| `POLL_DORMANT` | `minutely,alerts` |
| `POLL_MONITOR` | `alerts` |
| `POLL_ACTIVE` | *(none)* |
| `POLL_LIGHTNING` | *(none)* |

**Rate limit monitoring:** Log `X-Forecast-API-Calls` header on every response. Alert (medium priority) if >50% quota consumed before 50% through month. Alert (high priority) if >80% before 80%. Force `POLL_DORMANT` if approaching limit.

### Precipitation State Machine

Tracks precipitation event lifecycle, separate from polling state.

| State | Description |
|-------|-------------|
| `PRECIP_NONE` | No precipitation |
| `PRECIP_IMMINENT` | High probability within 30 minutes per minutely |
| `PRECIP_ACTIVE` | Currently precipitating |
| `PRECIP_TAPERING` | Intensity declining; event may be ending |
| `PRECIP_DONE` | Event ended; cooling-off before returning to NONE |

**Precipitation type tracking (Pirate Weather v2 fields):**

| Field | Use |
|-------|-----|
| `rainIntensity` | Rain rate (in/hr) |
| `snowIntensity` | Snow rate (in/hr liquid equivalent) |
| `iceIntensity` | Freezing rain rate (in/hr) |
| `currentDaySnow` | Accumulated snow since midnight (in) |
| `currentDayLiquid` | Accumulated liquid since midnight (in) |
| `currentDayIce` | Accumulated ice since midnight (in) |

Derived `precipitation_type` enum: `rain | snow | ice | sleet | mixed | none` — determined from which type-specific intensity fields are non-zero.

### Known Pirate Weather API Quirks

- `-999` sentinel value instead of null for unavailable fields — always null-guard. Confirmed on `smoke`, `fireIndex`, `smokeMax`, `fireIndexMax`
- `sleetIntensity` present in minutely but undocumented — treat as bonus, don't depend on it
- HRRR subhourly freshness: check `flags.sourceTimes.hrrr_subh` for data recency

### MQTT Topics (Tier 2)

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/state/weather/conditions` | Yes | Synthesized current conditions (Tempest + API) |
| `highland/state/weather/precipitation` | Yes | Precipitation event lifecycle state |
| `highland/state/weather/polling_state` | Yes | Current polling state (diagnostic) |
| `highland/event/weather/precipitation_start` | No | Precipitation event began |
| `highland/event/weather/precipitation_end` | No | Precipitation event ended |
| `highland/event/weather/precipitation_type_change` | No | Type transitioned during active event |
| `highland/event/weather/lightning_detected` | No | Lightning strike observed (Tempest hardware) |
| `highland/event/weather/wind_gust` | No | Gust crossed configured threshold |

### Notification Triggers (Tier 2)

| Trigger | Severity | Notes |
|---------|----------|-------|
| Rain event starting | `low` | Waking hours only unless heavy |
| Heavy rain starting (>0.3 in/hr) | `medium` | Any time |
| Snow event starting | `medium` | Always |
| Significant snow accumulation (>2") | `high` | Sensor-based |
| Ice/freezing rain detected | `high` | Type-specific field |
| Thunderstorm imminent | `high` | CAPE + precip threshold |
| Event ended after significant accumulation | `low` | Summary notification |

### Microclimate Context

Hudson Valley terrain creates significant hyperlocal variation. The property sits at elevation; the Hudson River to the east creates thermal and moisture dynamics that can shift the rain/snow line within a few miles. Forecasts snapped to a nearby city center would often be wrong for this location. The HRRR grid cell resolves to within ~1.5 mi, which is about as good as gridded models get. Systematic bias analysis during snow events should factor in elevation and river distance.

### Configuration (Tier 2, in `thresholds.json`)

```json
{
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

All values are initial estimates. Calibrate against observed events.

---

## Open Questions

- [ ] `Special Weather Statement` alert type — description field less structured than other alert types; monitor for display/parsing issues when one occurs in practice; no special handling designed yet
- [ ] Ultrasonic snow depth sensor hardware selection and mounting (Tier 2)
- [ ] Optimal `precipIntensityError` confidence gate threshold — calibrate from observed events (Tier 2)
- [ ] Whether `nearestStormDistance` / `nearestStormBearing` useful for lightning threat lead time (Tier 2)
- [ ] Pirate Weather API version monitoring — check for versions beyond v2 when beginning Tier 2 implementation
- [ ] Parallel run vs other API for snow prediction scoring — defer until sensor deployed (Tier 2)

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| **NODERED_PATTERNS.md** | `Utility: Weather` and `Utility: Nowcast` are utility flows |
| **MQTT_TOPICS.md** | Authoritative topic and payload definitions |
| **CALENDAR_INTEGRATION.md** | Parallel Tier 1 design pattern |

---

*Last Updated: 2026-03-24*
