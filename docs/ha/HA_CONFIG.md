# Home Assistant Configuration

## Philosophy

HA is a **consumer and dashboard layer** in Highland — not the automation engine. Most devices are exposed through Zigbee2MQTT and Z-Wave JS UI via MQTT Discovery. Direct HA configuration changes are expected to be infrequent.

Configuration is kept lean: only what HA cannot do via the UI or MQTT Discovery goes into YAML.

---

## Directory Structure

```
/config/
  configuration.yaml          ← orchestrator only, all !includes
  secrets.yaml                ← gitignored, actual values
  secrets_template.yaml       ← committed, placeholder values
  recorder.yaml               ← PostgreSQL recorder config
  rest_commands.yaml          ← outbound HTTP calls
  scenes.yaml                 ← scenes (if needed)
  automations/                ← !include_dir_merge_list
    health_monitoring.yaml    ← Healthchecks.io ping automations
  scripts/                    ← !include_dir_merge_named (placeholder)
```

---

## configuration.yaml

Thin orchestrator — includes only, no inline config:

```yaml
default_config:

frontend:
  themes: !include_dir_merge_named themes

automation: !include_dir_merge_list automations/
script: !include_dir_merge_named scripts/
scene: !include scenes.yaml

recorder: !include recorder.yaml
rest_command: !include rest_commands.yaml
```

**Include type rules:**
- `!include_dir_merge_list` — for automations and other list-based config
- `!include_dir_merge_named` — for scripts, scenes, and other dictionary-based config
- `!include` — for single files

---

## Secrets

HA uses a `!secret` directive to reference values from `secrets.yaml`. This keeps credentials out of version-controlled files.

**secrets.yaml** (gitignored — never commit):
```yaml
recorder_db_url: "postgresql://highland:PASSWORD@IP/homeassistant"
healthchecks_home_assistant_url: "https://hc-ping.com/uuid"
healthchecks_ha_z2m_edge_url: "https://hc-ping.com/uuid"
healthchecks_ha_zwave_edge_url: "https://hc-ping.com/uuid"
```

**secrets_template.yaml** (committed — documents required keys):
```yaml
recorder_db_url: "postgresql://highland:YOUR_PASSWORD@YOUR_WORKFLOW_IP/homeassistant"
healthchecks_home_assistant_url: "https://hc-ping.com/YOUR-UUID"
healthchecks_ha_z2m_edge_url: "https://hc-ping.com/YOUR-UUID"
healthchecks_ha_zwave_edge_url: "https://hc-ping.com/YOUR-UUID"
```

---

## recorder.yaml

Points HA's recorder at the PostgreSQL instance on the Workflow host:

```yaml
db_url: !secret recorder_db_url
purge_keep_days: 30
```

> **Note:** File contains only the values — no `recorder:` top-level key. That key lives in `configuration.yaml` via `!include`.

> **HAOS and .local hostnames:** HAOS sometimes resolves `.local` hostnames to IPv6 link-local addresses (`fe80::...`), which psycopg2 cannot use. Use the IPv4 address directly in `recorder_db_url`. See `RUNBOOK.md` section 3.14.

---

## rest_commands.yaml

Outbound HTTP calls. Currently used for Healthchecks.io pings from HA automations:

```yaml
healthchecks_io_home_assistant:
  url: !secret healthchecks_home_assistant_url
  method: GET

healthchecks_io_home_assistant_zigbee_edge:
  url: !secret healthchecks_ha_z2m_edge_url
  method: GET

healthchecks_io_home_assistant_zwave_edge:
  url: !secret healthchecks_ha_zwave_edge_url
  method: GET
```

> **Note:** No `rest_command:` top-level key — that lives in `configuration.yaml`.

---

## automations/health_monitoring.yaml

Healthchecks.io ping automations. HA self-reports and edge checks run on a 1-minute time pattern:

```yaml
- alias: Home Assistant Heartbeat Monitor
  triggers:
    - trigger: time_pattern
      minutes: /01
  conditions: []
  actions:
    - action: rest_command.healthchecks_io_home_assistant
      data: {}
  mode: single

- alias: HA / Zigbee Edge Monitor
  triggers:
    - trigger: time_pattern
      minutes: /01
  condition:
    - condition: state
      entity_id: binary_sensor.zigbee2mqtt_bridge_connection_state
      state: "on"
      for:
        seconds: 10
  actions:
    - action: rest_command.healthchecks_io_home_assistant_zigbee_edge
      data: {}
  mode: single

- alias: HA / Z-Wave Edge Monitor
  triggers:
    - trigger: time_pattern
      minutes: /01
  condition:
    - condition: state
      entity_id: sensor.usb_controller_status
      state: "ready"
      for:
        seconds: 10
  actions:
    - action: rest_command.healthchecks_io_home_assistant_zwave_edge
      data: {}
  mode: single
```

**Notes:**
- `binary_sensor.zigbee2mqtt_bridge_connection_state` uses `"on"` not `"connected"` — the UI label is friendly but the underlying state is a standard binary sensor value
- The `for: seconds: 10` debounce prevents momentary flapping from suppressing the ping
- Z-Wave JS uses `sensor.usb_controller_status` with state `"ready"`

---

## Healthchecks.io Integration

HA pings Healthchecks.io directly via `rest_command`, independent of Node-RED. This provides two-signal coverage:

| Check | What it proves |
|-------|----------------|
| `Home Assistant` | HA is alive and running |
| `Home Assistant / Zigbee Edge` | HA can see Zigbee devices via Z2M/MQTT |
| `Home Assistant / Z-Wave Edge` | HA can see Z-Wave devices via Z-Wave JS UI |

See `nodered/HEALTH_MONITORING.md` for the full failure signature matrix.

---

## Adding New Configuration

| Type | Where it goes |
|------|---------------|
| New automation | `automations/{domain}.yaml` |
| New script | `scripts/` directory |
| New rest_command | `rest_commands.yaml` |
| New secret reference | `secrets.yaml` + `secrets_template.yaml` |
| New integration | HA UI (Settings → Devices & Services) — not YAML |
| New device | MQTT Discovery via Z2M or Z-Wave JS — not YAML |

---

*Last Updated: 2026-03-26*
