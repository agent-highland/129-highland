# Landroid Integration

Integration of Worx Landroid Vision robotic mower into Highland.

**Status:** 📋 Phase 1 in progress — `landroid_cloud` installed, scratchpad flows collecting operational data.

---

## Hardware

**Worx Landroid Vision 1-Acre 4WD (WR344)**

- Wire-free boundary detection via camera/AI (no perimeter wire required)
- Built-in WiFi; communicates exclusively via Worx cloud (AWS IoT Core)
- No local API
- Sited: side yard (late afternoon direct sun only)
- Zones mapped: side yard + rear yard combined as single L-shaped zone (0.12 acres); front yard (est. 0.25–0.30 acres) mapping in progress

**Reolink Argus Eco Ultra + Solar Panel** *(to be purchased)*

- 4K, 125° FOV, color night vision
- Solar/battery powered, WiFi
- Strap mounts included for both camera and solar panel — supports tree or pole mounting where no fixed structure is available
- Connects to the RLN16-410 NVR over WiFi; NVR exposes RTSP stream to the Highland video pipeline identically to PoE cameras

---

## Integration Philosophy

Connectivity is **additive nicety, not a requirement.** The mower functions fully as a standalone appliance without any Highland integration. If the integration breaks temporarily or permanently, the mower continues to operate on its own schedule with its own onboard rain sensor — exactly as most users run their mower. This shapes every integration decision: keep it thin, keep it optional, don't build dependencies on it.

This also drives the phasing decision. The mower is an **informational device** from Highland's perspective, not a critical automation surface. Building a custom MQTT bridge before establishing baseline infrastructure is not a good use of time. Phase 1 uses the existing `landroid_cloud` community integration to get visibility and a control surface quickly. Phase 2, if warranted, migrates to a custom bridge for deeper Node-RED integration.

---

## Mowing Schedule

| Day | Zone |
|-----|------|
| Monday | Front |
| Tuesday | Rear + Side |
| Wednesday | Front |
| Thursday | Rear + Side |
| Friday | Front |
| Saturday | Rear + Side |
| Sunday | Off |

Front yard gets 3 sessions per week given its larger size (est. 0.25–0.30 acres) and visibility. Rear and side are mapped as a single combined L-shaped zone (open passage between them, ~20–25 feet at narrowest). Schedule configured in the Worx app; Highland does not own scheduling in Phase 1.

All sessions start at **9:00 AM** — late enough for dew to burn off (front yard slope receives early morning sun), early enough to complete before evening moisture accumulates.

---

## Observed State Behavior

Captured from live WR344 operation. Use this as the reference for flow design.

### Normal Mow Cycle

```
starting → mowing → returning → docked → charging → mowing (resumes after charge)
```

### Low Battery Mid-Session

```
mowing → [error: battery_low] → returning → docked
```

- `battery_low` error fires at the 10% discharge threshold
- Error clears approximately 2.5 minutes after docking, once battery climbs back above 10%
- Mower resumes mowing autonomously once sufficiently charged
- `battery_low` followed by a rising battery level is normal operational behavior — not a notification-worthy event
- `battery_low` persisting without a rising battery level indicates Mowen did not make it back to dock — escalate to mobile

### Rain Delay

```
mowing → [error: rain_delay] → returning → rain_delayed
```

- `rain_delayed` is a distinct at-base state — the mower does not pass through `docked` on a rain delay return
- Any logic checking "is Mowen at base" must match on both `docked` and `rain_delayed`
- The onboard rain sensor sensitivity gap: light rain does not trigger the onboard sensor; it fires at approximately the Tempest light-to-moderate precipitation transition
- Tempest-based suppression via `lawn_mower.dock` covers the light rain gap the onboard sensor misses
- The onboard rain sensor can take up to 7 hours to report dry after precipitation ends — excessive in practice; rain delay duration reduced to 30 minutes in the Worx app as a mitigation

### Coverage Map Retention and Completion Priority

The WR344 maintains a persistent coverage map across all mapped zones simultaneously. When a session is interrupted — whether by manual stop, rain delay, or end of schedule window — the firmware queues the incomplete coverage and resumes from the point of interruption on the next session, regardless of which zone the daily schedule calls for.

**Observed behavior:**
- Rear yard session manually stopped mid-coverage, then resumed two days later from the exact point of interruption
- Coverage map retention confirmed across rain delay interruptions
- After being sent home mid rear-yard session and fully recharged, Mowen did not restart autonomously (Option A confirmed)
- When `start_mowing` was sent externally, Mowen headed to the front yard — a different zone — to complete coverage interrupted by the previous day's rain delay, rather than continuing the scheduled rear yard session
- Completion priority overrides the daily zone schedule: the firmware works through all outstanding incomplete coverage before treating the current zone as fresh

**The full completion priority behavior (confirmed):**

Finish front yard incomplete coverage → finish rear yard incomplete coverage → resume normal schedule

Observed end-to-end: front yard interrupted by rain delay and rear yard interrupted by manual dock were both queued simultaneously. External `start_mowing` caused Mowen to complete the front yard first (older interruption), then resume the rear yard from the exact point of interruption. Both zones, both interruption types, exact position resume confirmed across all.

**Implication for Tempest-owned rain suppression:**

Node-RED does not need to track which zone was interrupted or in what order. The coverage prioritization is entirely Mowen's responsibility. Node-RED's role is purely: watch conditions, send `lawn_mower.dock` when conditions are bad, send `lawn_mower.start_mowing` when conditions clear. Mowen handles the rest.

### Schedule Window Behavior After Interruption

**Option A confirmed** — Mowen does not autonomously restart after being sent home mid-session. He docks, charges fully, and waits. No autonomous behavior occurs within the remaining schedule window. A `lawn_mower.start_mowing` command or the next scheduled session trigger is required to resume.

This simplifies the Tempest-owned rain suppression design — there is no risk of Mowen heading back out autonomously after being sent home for weather conditions. Node-RED fully controls the resume decision.

### Battery Voltage Profile

`sensor.vision_cloud_4wd_battery_voltage` exposed by `landroid_cloud`. Precision set to two decimal places for trend detection.

The WR344 does **not** trickle charge. The firmware uses a deliberate charge/discharge cycle to preserve lithium battery longevity — the charger runs to full, switches off, allows the battery to discharge to a lower threshold, then charges again. This is by design.

**Observed voltage boundaries:**

| Condition | Voltage |
|-----------|--------|
| Full charge ceiling | ~20.0–20.5V |
| Docked discharge lower threshold | ~17.0–17.3V |
| Active mow discharge floor (at `battery_low`) | ~17V or below |

**`binary_sensor.vision_cloud_4wd_charging` is non-functional** — it does not reflect actual charging state. Use voltage trend derived from `battery_voltage` instead.

**Inferred charging state heuristic:**

| Condition | Inferred state |
|-----------|---------------|
| Voltage < 20V and rising between polls | Charging |
| Voltage ≥ 20V | Fully charged / at ceiling |
| Voltage < 20V and falling between polls | Discharging |

Implementation: store previous voltage reading in context, compare to current reading on each update. Require two consecutive deltas in the same direction before committing to a state — a single flat or ambiguous reading between polls should not trigger a state change.

Mowing sessions are distinguishable from docked discharge cycles by discharge rate — motor load during mowing draws significantly more current than standby electronics, producing a visibly steeper voltage drop.

### `battery_low` Error Payload Structure

```json
{
  "payload": "battery_low",
  "data": {
    "old_state": { "state": "no_error" },
    "new_state": { "state": "battery_low" }
  },
  "topic": "sensor.vision_cloud_4wd_error"
}
```

- `msg.payload` contains the error string directly — use this as the routing key in switch nodes
- `msg.data.old_state.state` provides the previous state for transition validation
- `msg.data.new_state.last_changed` provides ISO timestamp of the transition

---

## Maintenance

### Blade Runtime Sensors

Two blade runtime sensors are exposed by `landroid_cloud`:

| Entity | Purpose |
|--------|---------|
| `sensor.vision_cloud_4wd_blade_runtime_total` | Odometer — cumulative lifetime hours on the blade disc; never resets; useful for disc replacement decisions and asset history |
| `sensor.vision_cloud_4wd_blade_runtime_since_reset` | Trip meter — hours since last blade replacement; resets when blade replacement is logged in the Worx app; use this as the maintenance trigger |

**Workflow:**
1. Replace blades
2. Log the replacement in the Worx app — this resets `blade_runtime_since_reset` to zero
3. Highland watches `blade_runtime_since_reset` and sends a notification when it crosses the replacement threshold
4. Repeat

**Threshold:** Community consensus for the WR344 specifically is 150–200 hours of blade runtime between replacements — meaningfully higher than Worx's generic 1–3 month guidance, which applies across the broader Landroid lineup. At the current schedule across ~0.4 acres total, this works out to roughly 3–4 months during active mowing season. Set the initial notification threshold at 150 hours and adjust based on observed blade condition at first replacement.

### Regular Maintenance Schedule

**After every few sessions (visual check, ~5 minutes):**
- Clear grass and debris from the cutting disc area — buildup impedes airflow and cutting quality
- Check the Vision AI camera lenses — dirt, spider webs, and water spots degrade obstacle detection
- Inspect wheels for wrapped debris (string, hair, grass around axles)
- Check dock charging contacts on both mower and base — debris or oxidation causes docking errors

**Monthly:**
- Full undercarriage cleaning — compressed air or stiff brush; accumulated soil affects cutting height consistency
- Inspect the bumper sensor — front bumper is a physical contact sensor; must move freely
- Check blade disc bolts are tight
- Inspect the camera housing for cracks or moisture ingress
- Check blade condition visually; replace if visibly worn regardless of runtime hours

**Pre-season:**
- Full inspection and clean
- Fresh blades
- Verify charging base is level — unlevel base is a common cause of `charging_station_docking_error`
- Confirm dock charging contacts are clean

**Post-season / winter storage:**
- Full clean including undercarriage
- Remove battery if storing in a sub-freezing environment — lithium battery degradation accelerates below freezing
- Store Mowen and the base indoors or in a protected location
- The Landroid Vision Garage provides UV and weather protection during the season, reducing seasonal wear

### Blade Replacement Notification

HA automation watches `sensor.vision_cloud_4wd_blade_runtime_since_reset`. When it crosses the configured threshold, send a mobile notification prompting blade inspection and replacement. This is a reminder, not an urgent alert — route through the standard mobile notification channel, not TV.

After replacing blades, log the replacement in the Worx app to reset `blade_runtime_since_reset`. If the reset does not happen within 24 hours of the notification, send a follow-up reminder — the sensor is only useful if the reset discipline is maintained.

---

## Phase 1: `landroid_cloud` HA Integration

### Approach

Install the `landroid_cloud` HACS custom component. It handles all AWS MQTT complexity internally via `pyworxcloud`, creates well-formed HA entities automatically, and requires no custom infrastructure. This is the right starting point for an informational device — low setup cost, immediately useful, and the fallback behavior (mower runs autonomously) is acceptable.

### Installation

1. Install `landroid_cloud` via HACS
2. Restart Home Assistant
3. Add the integration via Settings → Devices & Services using Worx app credentials
4. Install `landroid-card` via HACS for the dashboard

### Entities Created

| Entity | Type | Notes |
|--------|------|-------|
| Lawn mower | `lawn_mower` | Primary control entity |
| GPS tracker | `device_tracker` | State is permanently `away` — lat/lon coordinates are in attributes, not state; poll attributes directly if position data is needed |
| Next schedule | Sensor | Timestamp + schedule details as attributes |
| Battery | Sensor | Charge level |
| Error state | Sensor | Current error code string |
| Rain delay | Sensor | Active/inactive + remaining time |
| Zone | `select` | Current zone — read-only on Vision hardware |
| Blade runtime since reset | Sensor | Hours since last blade replacement logged in Worx app |
| Blade runtime total | Sensor | Cumulative lifetime blade disc hours |

**Supported mower states:** `mowing`, `docked`, `returning`, `rain_delayed`, `error`, `edge_cut`, `starting`, `escaped_digital_fence`

**Vision-series limitation:** Zones and schedules are read-only via this integration — they can be read from the mower but not modified from HA. This is acceptable; schedule management stays in the Worx app.

### Available HA Actions

- `lawn_mower.start_mowing` — send mower out; behavior when called after an interrupted session (resume vs. fresh start) is unconfirmed on Vision hardware — validate empirically
- `lawn_mower.dock` — stop and return to base
- `lawn_mower.pause` — stop in place
- `landroid_cloud.ots` — one-time schedule (start with runtime parameter)

### Dashboard

Install `landroid-card` (by Barma-lej, available via HACS) for a purpose-built mower dashboard card. Displays mower state, battery, error status, and control buttons in a single card.

### Known Fragility

- Depends on Worx cloud API remaining stable and accessible
- A March 2025 release caused excessive API retries that locked up HA — resolved since, but worth noting as a precedent for fragility
- `pyworxcloud` is reverse-engineered; Worx API changes can break it without notice

These risks are acceptable given the "additive nicety" posture. If the integration breaks, the mower continues to operate autonomously.

---

## Phase 1 Automations

All automations in Phase 1 live in HA (YAML automations), not Node-RED. This is consistent with the principle of not building custom Node-RED subsystems for informational devices before baseline infrastructure is complete.

### Rain Suppression

The rain suppression logic runs 24/7 as a continuous Tempest monitor, but handles rain events differently depending on whether they are schedule-impacting or not. The moment Node-RED sends a `lawn_mower.dock` command for rain reasons, it assumes manual control of that day's session — the schedule is suspended until the next scheduled window.

**Non-schedule-impacting rain (no action required):**

Rain that ends far enough before the schedule window that the cooldown fully elapses before Mowen would start. If rain ended at 2am with a 30-minute cooldown, by 9am conditions have been clear for nearly 7 hours — no reason to intervene. Log the event for Daily Digest, do not set the `controlled_day` flag, let the schedule run normally.

Condition: `rain_end_time + cooldown_minutes <= schedule_start` → non-impacting, no action

**Schedule-impacting rain — two trigger scenarios:**

**Scenario A — Rain starts mid-session:**
- Mower state is `mowing`
- Tempest detects active precipitation
- Send `lawn_mower.dock` → set `controlled_day` flag

**Scenario B — Rain present at or near session start:**
- Schedule fires → mower transitions to `starting`
- Evaluate: is active precipitation present OR has cooldown not yet elapsed since last rain end?
  - `now < rain_end_time + cooldown_minutes` → conditions not yet clear → send `lawn_mower.dock` immediately, set `controlled_day` flag
  - Conditions already clear → do not intervene, schedule runs normally

Scenario B requires catching the `starting` state transition. The dock command must be sent immediately — `starting` is a brief transient state before `mowing` begins.

**Controlled day — resume logic:**

Once `controlled_day` is set, Node-RED evaluates resume conditions continuously:

1. No active precipitation (Tempest precipitation rate = 0)
2. Cooldown elapsed since last rain end
3. NWS minutely forecast clear for at least `forecast_clear_minutes` ahead
4. Current time is at least `resume_cutoff_minutes` before `schedule_end`

All conditions met → send `lawn_mower.start_mowing`. Remain in controlled mode — rain may return.

If rain returns while Mowen is out → Scenario A fires again, send `dock`, restart cooldown.

**End-of-schedule gate:**

At `schedule_end` time, if `controlled_day` is set AND mower state is not `docked` → send `lawn_mower.dock`.

Clear `controlled_day` flag when: schedule end time has passed AND mower is docked AND no active precipitation. Safety clear at midnight regardless of state.

**Flag persistence:**

The `controlled_day` flag persists across the schedule window. A rain event at 8pm that sets it will still be active at 9am the next morning if conditions haven't fully cleared. However, if 9am arrives with no active precipitation and cooldown elapsed (Scenario B gate evaluates false), Mowen heads out normally and the flag is cleared — the schedule owns the day.

**`rain_delayed` state note:**

Once "Mow When It Is Raining" is disabled in the Worx app, `rain_delayed` will no longer appear as a mower state. The "is Mowen at base" check simplifies to `docked` only. Node-RED owns all rain delay state from that point.

**Config:**

```json
"rain_suppression": {
    "schedule_start": "09:00",
    "schedule_end": "18:00",
    "resume_cutoff_minutes": 15,
    "cooldown_minutes": 30,
    "forecast_clear_minutes": 30
}
```

All values are tunable. Thresholds cannot be finalized before a full mow season of observed Tempest data — start conservative and adjust based on actual ground conditions.

**This logic belongs in Node-RED (`Utility: Landroid`)**, not HA YAML automations. The Tempest data, NWS minutely forecast, and stateful flag management are already in Node-RED. This is the use case that justifies promoting Landroid automation to Node-RED ahead of a full Phase 2 migration.

### Error Notification

When `sensor.vision_cloud_4wd_error` transitions away from `no_error`, route the notification based on error string value. `msg.payload` from the state-changed node contains the error string directly and is used as the routing key.

**Maintenance mode:** An `input_boolean.landroid_maintenance_mode` helper suppresses the urgent notification tier when active. Toggle on before picking up Mowen for any intentional interaction; toggle off when done. Auto-timeout after 60 minutes to prevent accidentally leaving suppression active. Dashboard visibility for maintenance mode state is required — must be obvious when it is active.

**Error state map and notification tiering:**

| State value | Meaning | Routing |
|-------------|---------|---------|
| `lifted` | Mower lifted unexpectedly | 📺 TV + mobile — suppressed if maintenance mode active |
| `trapped` | Mower stuck | 📺 TV + mobile (urgent — needs intervention) |
| `trapped_timeout` | Stuck for extended period | 📺 TV + mobile (urgent) |
| `upside_down` | Mower fell over | 📺 TV + mobile (urgent — blades may have been running) |
| `outside_wire` | Escaped the mowing zone | 📺 TV + mobile (urgent) |
| `excessive_slope` | Reached terrain it can't handle | 📺 TV + mobile (urgent) |
| `unreachable_charging_station` | Cannot return to base | 📺 TV + mobile (urgent — won't recover on its own) |
| `blade_motor_blocked` | Blade obstruction | 📱 Mobile only |
| `wheel_motor_blocked` | Wheel obstruction | 📱 Mobile only |
| `charge_error` | Charging fault | 📱 Mobile only |
| `battery_temperature_error` | Battery thermal fault | 📱 Mobile only |
| `map_error` | Navigation map failure | 📱 Mobile only |
| `mapping_exploration_failed` | Could not complete mapping | 📱 Mobile only |
| `camera_error` | Vision AI camera fault | 📱 Mobile only |
| `missing_charging_station` | Cannot locate base | 📱 Mobile only |
| `timeout_finding_home` | Timed out returning to base | 📱 Mobile only |
| `close_door_to_mow` | User action required to start | 📱 Mobile only |
| `close_door_to_go_home` | User action required to return | 📱 Mobile only |
| `charging_station_docking_error` | Docking fault | 📱 Mobile only |
| `insufficient_sensor_data` | Sensor fusion failure | 📱 Mobile only |
| `mapping_exploration_required` | Mapping required before mowing | 📱 Mobile only |
| `blade_height_adjustment_blocked` | Height adjustment fault | 📱 Mobile only |
| `unknown` | Unrecognized error code | 📱 Mobile only |
| `rain_delay` | Rain delay active | 📋 Daily Digest only |
| `battery_low` | Low battery | 📋 Daily Digest only — fires at 10% discharge threshold, self-clears ~2.5 minutes after docking once battery rises above 10%; escalate to mobile only if active AND battery level is not rising |
| `locked` | Mower is locked | 📋 Daily Digest only (deliberate state) |
| `battery_trunk_open_timeout` | Battery compartment issue | 📋 Daily Digest only |
| `training_start_disallowed` | Training blocked | 🔇 Log only (user-initiated; fires when cancelling a mapping operation) |
| `wire_missing` | Wire not detected | 🔇 Log only (wire-based error; should not fire on WR344) |
| `reverse_wire` | Wire polarity error | 🔇 Log only (wire-based error; should not fire on WR344) |
| `wire_sync` | Wire synchronization error | 🔇 Log only (wire-based error; should not fire on WR344) |
| `ota_error` | Firmware update failed | 🔇 Log only (not user-actionable) |
| `hbi_error` | Hardware bus error | 🔇 Log only (contact support) |
| `rfid_reader_error` | RFID reader fault | 🔇 Log only (contact support) |
| `headlight_error` | FiatLux headlight fault | 🔇 Log only (contact support) |

**Wire-based error codes** (`wire_missing`, `reverse_wire`, `wire_sync`) should not fire on the WR344 which has no perimeter wire. If they do appear, log them and investigate — they may indicate something unexpected in Vision firmware behavior.

**TV notification** uses the existing Android TV notification infrastructure. Include a concise title and the friendly error description.

### Blade Replacement Reminder

Watch `sensor.vision_cloud_4wd_blade_runtime_since_reset`. When it crosses the configured threshold (initial value: 150 hours — based on WR344 community consensus; adjust based on observed blade condition at first replacement), send a mobile notification prompting blade inspection and replacement.

After replacing blades, log the replacement in the Worx app to reset the sensor. If the reset does not occur within 24 hours of the notification firing, send a follow-up reminder — the sensor is only useful if reset discipline is maintained.

### Daily Digest Contribution

Include in the morning digest: mower state, last mow timestamp, battery level, blade runtime since last reset, any active errors or rain delay. Node-RED Daily Digest flow consumes these via HA state.

---

## Security & Anti-Theft

### Threat Model

Two distinct theft vectors:

**Vector 1 — Stolen from dock (mower charging/idle)**
The mower is sitting on the base between mow sessions. Someone picks it up and walks off.

**Vector 2 — Stolen while running (mower actively mowing)**
Someone intercepts the mower mid-cycle in the yard.

### On-Device Defenses

The mower has built-in security independent of any Highland integration:

- **Lift sensor** — physical tilt/lift sensor always active when powered on; stops blades immediately on lift regardless of mowing state
- **Security PIN** — locks the mower; wrong PIN triggers audible alarm
- **Lock function** — when enabled, triggers an audible alarm if the mower is lifted and carried outside the yard perimeter (GPS/geofence-based on Vision models)
- **WiFi kill** — mower stops operating after three consecutive days outside WiFi coverage

### Dedicated Security Camera

A **Reolink Argus Eco Ultra** pointed at the charging base serves as the primary real-time detection layer. Camera is tree-mounted pointing directly at the dock; exact tree TBD once a suitable candidate is confirmed, but mount type and orientation are settled.

The mower base is a fixed, known scene. The camera has a single binary question to answer: *is a person interacting with the mower?* This makes analysis cheaper, faster, and higher-confidence than general perimeter surveillance.

The prompt pattern:

> *"This is a fixed camera pointed at a robotic lawn mower on its charging base. Does the image show a person approaching, touching, lifting, or otherwise interacting with the mower? The mower should be stationary and unattended. Answer yes/no and briefly describe what you see."*

**Pipeline** (see `subsystems/VIDEO_PIPELINE.md`):
1. **CPAI triage** — person detection gate; no person in frame → discard
2. **Gemini snapshot analysis** — focused prompt above; yes → escalate
3. **Notification** — immediate push via Utility: Notifications with keyframe attached

**Two-signal confirmation:**

| Signal | Vector 1 (docked) | Vector 2 (running) | Latency |
|--------|-------------------|--------------------|---------|
| Camera (person detection) | ✅ | ✅ (if in frame) | Seconds |
| HA error state (lift error) | ✅ (with delay caveat) | ✅ | Seconds–minutes |

**Camera siting:**

- Tree-mounted, pointing directly at the dock
- Mount at roughly 6–8 feet; higher loses the human-object interaction geometry
- FOV should cover a 10–15 foot approach radius, not just the base itself
- Avoid pointing into late afternoon sun — if solar panel orientation and lens direction conflict, favor the lens angle; afternoon sun in this location is strong enough that an oblique panel angle still charges adequately

**Authorized interaction handling:** Accept false positives during legitimate maintenance. Maintenance mode suppresses the lifted notification tier. Time-of-day context handles residual ambiguity.

---

## Phase 2: Custom MQTT Bridge (If Warranted)

Phase 2 is only worth pursuing if one or more of the following is true:

- The `landroid_cloud` integration proves consistently unreliable
- Automation needs grow beyond what HA can deliver (e.g., Node-RED-owned scheduling, zone targeting, deeper Tempest/NWS synthesis)
- The "HA is consumer only" architectural principle becomes a friction point for the mower specifically

### Approach

A **Mosquitto bridge** on the Communication Hub connects directly to Worx's AWS IoT Core endpoint, bypassing the `landroid_cloud` integration entirely. Node-RED becomes the normalization and automation engine; HA receives entities via MQTT Discovery.

```
Mower ──► AWS IoT Core ──► Mosquitto Bridge ──► highland/state/landroid/#
                                             ◄── highland/command/landroid/#
```

### Setup Requirements (One-Time)

1. Authenticate against the Worx REST API to retrieve TLS cert and AWS endpoint
2. Write `landroid-bridge.conf` for Mosquitto (drop-in, does not modify main config)
3. Cert files live in `/etc/mosquitto/certs/landroid/` on the Communication Hub

**Known fragility:** Worx has historically rotated certs without notice. Re-running cert extraction resolves it. Acceptable given the additive nicety posture.

### Reference Material

Community reverse engineering sources, in order of usefulness:

- **`virtualzone/landroid-bridge`** — original cert extraction and bridge documentation; read the source
- **`pyworxcloud` (PyPI/GitHub)** — most up-to-date payload field mappings and command structures; `dump_mapping.py` decodes raw MQTT payloads into human-readable output
- **`iobroker.worx`** — well-maintained ioBroker adapter; good secondary reference for payload structures
- **HA Community forums** — search "Landroid MQTT bridge"; practical notes from people who've done it
- **`roboter-forum.com`** — German robotic mower forum; raw payload samples posted; Google Translate sufficient

**Critical caveat:** Virtually all community documentation is from wire-based models. Vision-series payload behavior needs validation against observed WR344 payloads. The first MQTT Explorer session watching a live mow cycle is primary research — WR344-specific payload documentation is sparse.

### Phase 2 Validation Items

Before building the Node-RED normalization flow, validate against observed WR344 payloads:

- Exact topic prefix (community reports `PRM100` but may be model-dependent)
- Exact `dat.le` error code for lifted state on Vision hardware
- Whether `dat.le` fires immediately while docked or only during active mow
- Whether position coordinates appear in `commandOut` payload (would enable live position on HA map card)
- Whether zone targeting via `commandIn` works on Vision firmware (would enable Node-RED-owned scheduling)

---

## Open Questions

- Whether `lawn_mower.start_mowing` called from Node-RED behaves identically to the same call made from HA — expected yes given it is the same service call, but confirm empirically when Tempest-based suppression is implemented
- Whether `sensor.vision_cloud_4wd_error` rain_delay state persists for the full countdown duration or clears when the countdown begins — observe during next natural rain delay
- Rain suppression and resume thresholds — requires observed Tempest precipitation data and ground condition observations over a full mow season
- Camera mount point — exact tree TBD once base installation is finalized (leveling and brick edging pending)

---

*Last Updated: 2026-05-06*
