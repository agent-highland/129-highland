# Appliance Monitoring — Cycle Detection

## Purpose & Scope

Power-based cycle detection for the three ZEN15-monitored appliances: washing machine, dryer, and dishwasher. Reliable start/end detection that works correctly for each appliance's unique power signature — without false-starts, premature termination, or missing actual completion.

Node-RED owns all cycle state. HA receives entities via MQTT Discovery and can display state, trigger notifications, and expose Assist commands — but it is a consumer, not the source of truth.

**Inspired by:** [ha_washdata](https://github.com/3dg1luk43/ha_washdata) — this design extracts the essential logic (state machine + energy gate) and reimplements it natively in Node-RED.

---

## Why Simple Thresholds Fail

Each appliance has a specific failure mode that ruins naive "power below X for Y seconds" logic:

**Washing machine** — Rinse cycles drop power to near-zero before spinning hard again. A 3-minute off-delay fires right in the middle of a rinse-spin sequence.

**Dryer** — The heating element is thermostat-controlled and cycles on/off throughout the entire run. Near the end, there's a cool-down phase: element off, drum motor still running (~200–300W). You're stuck choosing between "fires too early on every cool-down" or "waits several minutes past actual completion."

**Dishwasher** — The worst case. Has genuine zero-power gaps between wash/rinse/heated-dry phases. Air-dry mode is literally zero watts for 20+ minutes at the end. You cannot distinguish "between phases" from "done" with a power threshold alone.

**The solution:** Instead of asking "is power below threshold?", ask "has effectively zero *energy* flowed in the last N seconds?" Combined with per-appliance off-delays calibrated to each device's behavior, this collapses all three failure modes.

---

## Data Pipeline

```
ZEN15 (Z-Wave) → Z-Wave JS UI → MQTT → Node-RED
                                         ↓
                                  State Machine
                                         ↓
                              MQTT Topics (state + events)
                                         ↓
                              HA via MQTT Discovery
                              PostgreSQL (cycle records)
```

### Z-Wave JS UI MQTT Topics

**Action required at bring-up:** Subscribe to `zwave/{node_name}/#` for each appliance node and observe what topics Z-Wave JS UI publishes. The power value (Watts) is the one we care about. Confirm unit is Watts (ZEN15 reports in W, not kW). Update config with confirmed topic paths.

**Polling vs. push:** Configure ZEN15 in Z-Wave JS UI for report-on-change with a threshold (2–5W recommended) plus a periodic fallback (60s). Prefer report-on-change for responsive detection.

---

## State Machine

```
OFF ──────────────────────► STARTING ──► RUNNING ◄──► PAUSED
 ▲                              │             │
 │                          (abort)           │
 │                              ▼             ▼
 └──── FINISHED ◄───────── ENDING ◄──────────┘
       INTERRUPTED
       FORCE_STOPPED
```

| State | Meaning |
|-------|---------|
| `off` | No cycle in progress |
| `starting` | Power above threshold; accumulating energy to confirm real start |
| `running` | Cycle confirmed and actively running |
| `paused` | Power dropped during cycle; waiting to see if it resumes |
| `ending` | Power low long enough to start considering cycle complete; energy gate applied |
| `finished` | Cycle completed normally; auto-expires to `off` after 30 minutes |
| `interrupted` | Cycle ended too short to be a valid run |
| `force_stopped` | Watchdog forced termination (sensor went silent) |

### Dynamic Pause/End Thresholds

The pause threshold is calibrated to the sensor's actual update cadence:

```
pause_delay_s = max(configured_min, 3 × p95_update_interval)
end_candidate_delay_s = max(pause_delay_s + 15s, configured_min_end)
```

`p95_update_interval` is tracked as a rolling statistic from the last 20 received readings. If the ZEN15 reports every 30s, the pause threshold auto-adjusts to ~90s minimum.

---

## Per-Appliance Configuration

### Washing Machine

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `start_threshold_w` | 2.0 W | Any real motor draw |
| `stop_threshold_w` | 2.0 W | Symmetric hysteresis |
| `start_duration_s` | 5 s | Debounce |
| `start_energy_wh` | 0.2 Wh | |
| `pause_delay_s` | 90 s (min) | Rinse gap tolerance |
| `end_candidate_delay_s` | 180 s (min) | |
| `off_delay_s` | 300 s | 5 min after energy gate passes |
| `end_energy_wh_gate` | 0.05 Wh | Effectively zero |
| `min_off_gap_s` | 480 s | 8 min — soak cycle handling |
| `completion_min_s` | 600 s | 10 min to be a valid cycle |

### Dryer

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `start_threshold_w` | 5.0 W | Element draws hard |
| `stop_threshold_w` | 3.0 W | Cool-down still has motor |
| `start_energy_wh` | 0.5 Wh | Element kicks in immediately |
| `pause_delay_s` | 60 s (min) | Thermostat cycling is fast |
| `off_delay_s` | 300 s | 5 min — cool-down phase |

**Note:** `stop_threshold_w` is set above zero because the drum motor runs throughout cool-down. The energy gate is what actually confirms the cycle is done.

### Dishwasher

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `start_threshold_w` | 2.0 W | Pump/heater |
| `pause_delay_s` | 300 s (min) | Phase gaps can be long |
| `end_candidate_delay_s` | 600 s (min) | Air-dry is 20+ min of zero |
| `off_delay_s` | 1800 s | 30 min — handles air-dry |
| `min_off_gap_s` | 2000 s | ~33 min — drying pauses |
| `completion_min_s` | 900 s | 15 min |

**Note:** The `off_delay_s` of 30 minutes is not a bug — it's the only way to reliably handle air-dry mode without profile matching.

---

## Energy Gate Logic

Trapezoidal integration in a Function node:

```javascript
function integrateWh(samples) {
    if (samples.length < 2) return 0;
    let wh = 0;
    for (let i = 1; i < samples.length; i++) {
        const dt_hours = (samples[i].t - samples[i-1].t) / 3_600_000;
        const avg_p = (samples[i].p + samples[i-1].p) / 2;
        wh += avg_p * dt_hours;
    }
    return wh;
}
```

At the ENDING transition check, compute energy over the window covering the last `off_delay_s` seconds. If ≤ `end_energy_wh_gate`, the cycle is done. If it exceeds the gate (real power drew in that window), hold in ENDING.

---

## MQTT Topics

**State (retained):** `highland/state/appliance/{appliance}/cycle`

`{appliance}` values: `washing_machine` | `dryer` | `dishwasher`

**Events (not retained):** `highland/event/appliance/{appliance}/cycle_started` | `cycle_finished` | `cycle_interrupted`

**Commands:** `highland/command/appliance/{appliance}/force_end` | `reset`

See `standards/MQTT_TOPICS.md` for full payload schemas.

---

## HA Entities via MQTT Discovery

Per appliance — using `washing_machine` as example:

| Entity | Type | Notes |
|--------|------|-------|
| Cycle State | `sensor` | `off`/`starting`/`running`/`paused`/`ending`/`finished`/`interrupted` |
| Cycle Duration | `sensor` | Unit: `s`, device_class: `duration` |
| Cycle Energy | `sensor` | Unit: `Wh`, device_class: `energy` |
| Current Power | `sensor` | Unit: `W`, device_class: `power` |
| Cycle Active | `binary_sensor` | `on` when state not in terminal states |

**Binary sensor value_template:**
```
{{ value_json.state not in ['off', 'finished', 'interrupted', 'force_stopped'] }}
```

---

## PostgreSQL Schema

Cycle records written on cycle completion:

```sql
CREATE TABLE appliance_cycles (
    id              SERIAL PRIMARY KEY,
    cycle_id        TEXT NOT NULL UNIQUE,
    appliance       TEXT NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ NOT NULL,
    duration_s      INTEGER NOT NULL,
    energy_wh       NUMERIC(8,3) NOT NULL,
    max_power_w     NUMERIC(8,2) NOT NULL,
    status          TEXT NOT NULL,
    termination_reason TEXT,
    matched_profile TEXT,
    match_confidence NUMERIC(4,3),
    power_trace     JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

`power_trace` stores raw power samples for Phase 2 profile learning. Initially nullable — enable when Phase 2 is implemented.

---

## Watchdog

If the ZEN15 stops reporting (sensor offline, Z-Wave JS UI restart), a watchdog catches the stuck state:

- If state is `running` or `paused` and no reading has arrived in `no_update_timeout_s` (default: 600s), force-transition to `force_stopped`
- Notification fired; termination reason: `"watchdog"`

Implementation: inject node set to `no_update_timeout_s`, reset on every received reading.

---

## Phase 2 — Profile Matching (Backlog)

Once the core state machine is proven stable across several weeks, Phase 2 adds smart termination by matching power traces against known cycle profiles via DTW (Dynamic Time Warping). Enables estimated time remaining and earlier cycle completion detection without waiting for full off-delay.

See `AUTOMATION_BACKLOG.md` for status.

---

## Open Questions

- [ ] ZEN15 MQTT topic paths — exact paths from Z-Wave JS UI to be confirmed at bring-up. Update `config.zwave_topic` per appliance once verified.
- [ ] ZEN15 report parameters — configure per-node: 5W threshold, 10s min interval, 60s periodic (initial recommendation; tune per appliance).
- [ ] Power trace storage — decide whether to store from day one or enable later. ~2KB per cycle at 30s intervals for 60 minutes. Probably worth enabling from the start.

---

*Last Updated: 2026-03-26*
