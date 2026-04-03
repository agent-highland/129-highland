# Node-RED — Utility: Scheduling

## Purpose

Publishes period transitions and fixed task events to the MQTT bus. All time-based triggers in Highland originate here — no other flow contains scheduling logic.

---

## Topics

| Topic | Retained | Purpose |
|-------|----------|---------|
| `highland/state/scheduler/period` | Yes | Current period — ground truth for all period-aware flows |
| `highland/event/scheduler/day` | No | Fired on transition to daytime |
| `highland/event/scheduler/evening` | No | Fired on transition to evening |
| `highland/event/scheduler/overnight` | No | Fired on transition to overnight |
| `highland/event/scheduler/midnight` | No | Fired daily at 00:00:00 |
| `highland/event/scheduler/backup_daily` | No | Triggers backup orchestration (System Event — limited consumers) |

---

## Periods

Three periods driven by `node-red-contrib-schedex` using solar events and fixed times:

| Period | On time | On offset | Off time | Off offset |
|--------|---------|-----------|----------|------------|
| `day` | `sunrise` | 0 | `sunset` | -30 min |
| `evening` | `sunset` | -30 min | `22:00` | 0 |
| `overnight` | `22:00` | 0 | `sunrise` | 0 |

Schedex coordinates pulled from `config.secrets.location` (lat/lon). All 7 days enabled.

---

## Flow Groups

**Dynamic Periods** — Three schedex nodes (Day, Evening, Overnight), each wiring through a `link call` return pattern into a shared `Publish Dynamic Period` group: `Is Active?` switch → `Prepare Dynamic` function → two MQTT out nodes (event + state)

**Fixed Events** — Midnight inject (cron `00 00 * * *`) → `Prepare Fixed` function → MQTT out. General-purpose events intended for broad consumption by any interested flow.

**System Events** — CronPlus nodes for deterministic task triggers with limited consumers. Uses `node-red-contrib-cron-plus` (6-field cron: second minute hour day month weekday). Each node wires to a `Prepare Event` function → MQTT out. Sets `node.status()` on each fire for editor visibility.

| Task | Expression | Topic | QoS |
|------|-----------|-------|-----|
| `Backup Daily` | `0 15 3 * * *` (3:15 AM daily) | `highland/event/scheduler/backup_daily` | 2 |
| `Registry Refresh` | `0 0,15,30,45 * * * *` (every 15 min at :00/:15/:30/:45) | `highland/command/config/reload/device_registry` | 1 |

**Sinks** — On Startup inject → `Recover Last State` function → Dynamic Period `link call`

**Test Cases** — Manual injects for each period and midnight; wired directly to the respective publish entry points for on-demand testing

---

## Startup Recovery

On startup, `Recover Last State` sends `send_state` to all three schedex nodes via dynamic `link call`. Each schedex node emits its current state if it is within its active window — exactly one of the three responds with a non-empty payload. The `Is Active?` switch drops empty off-window responses.

**`startup_recovering` flag:** Set to `true` for 2 seconds in the `volatile` store on startup. During this window, `Prepare Dynamic` suppresses event publication and publishes state only. After the window, all transitions publish both event and state.

This prevents spurious period events from firing at every Node-RED restart.

---

## Period-Aware Flow Pattern

Flows that respond to period changes use **two entry points, one handler:**

```
highland/state/scheduler/period  ──┐  (retained — delivered on subscription,
  (startup recovery path)          │   covers restart/init)
                                   ├──► period logic handler
highland/event/scheduler/evening ──┘  (non-retained — real-time transition)
```

This is a push model, not polling. The retained state delivers once on subscription; events drive everything thereafter.

**State-following flows** (lights, ambiance): read retained period on startup, act immediately. No reconciliation needed.

**Safety-critical flows** (locks, security): read retained period on startup, query actual device state, reconcile if misaligned.

---

## Payloads

**Period event / state:**
```json
{
  "period": "evening",
  "timestamp": "2026-03-26T19:47:12.000Z",
  "source": "scheduler"
}
```

**Midnight / task events:**
```json
{
  "timestamp": "2026-03-26T00:00:00.000Z",
  "source": "scheduler",
  "task": "midnight"
}
```

---

## Implementation Notes

- `send_state` dispatched to schedex nodes via dynamic `link call` — each schedex node is a named `Link In` target (`Day`, `Evening`, `Overnight`)
- Spreading a string payload with `{...msg.payload}` produces a character-indexed object — always pass string payloads directly as `msg.payload`
- `Prepare Fixed` sets `node.status()` on every midnight fire for "last fired" visibility in the editor
- Midnight cron uses Node-RED's 5-field format: `"00 00 * * *"` (minute hour day month weekday)
- System Events use `node-red-contrib-cron-plus` 6-field format: `"s m h d M wd"` — e.g. `"0 15 3 * * *"` for 3:15:00 AM daily
- **Fixed vs System Events distinction:** Fixed Events (`midnight`) are general-purpose signals for any interested flow. System Events (`backup_daily`) are task triggers with a specific, limited set of consumers; named after the task rather than the time.

---

*Last Updated: 2026-04-03*
