# Utility: Internet Connectivity

Monitors internet connectivity and throughput from the Workflow host. Two independent pipelines publish state to MQTT for consumption by Home Assistant, alerting, and the Healthchecks.io dead man's switch.

---

## Architecture

```
CronPlus (1 min)  →  Build Requests  →  Ping (×3 parallel)  →  Coalesce  →  MQTT out
CronPlus (1 hr)   →  Speed Test      →  Parse               →  MQTT out
On Startup        →  Build Sensors   →  MQTT out (Discovery)
```

Both pipelines are fully independent. The connectivity pipeline runs every minute; the throughput pipeline runs every hour.

---

## Connectivity Pipeline

### Detection

Three DNS resolvers pinged in parallel via `node-red-configurable-ping`:

| Host | Provider |
|------|----------|
| `8.8.8.8` | Google DNS |
| `1.1.1.1` | Cloudflare DNS |
| `208.67.222.222` | OpenDNS |

**Why three hosts:** Any single host could be down or unreachable for reasons unrelated to local connectivity. Requiring all three to fail before declaring an outage eliminates false positives from individual endpoint issues.

### Coalesce Logic

`Build Requests` fires all three pings in parallel, tracking expected hosts in `flow.all_hosts`. Each ping result (RTT in ms, or `false` on failure) is stored in `flow.host_*` context. `Coalesce` waits for all three results before evaluating.

**Aggregate state derivation:**

| Condition | State |
|-----------|-------|
| All three up | `up` |
| All three down | `down` |
| Mixed | `degraded` |

`flow.connectivity_state` tracks the previous aggregate state. `connectivity_changed` event fires only on transitions.

### MQTT Topics

**State (retained):**

| Topic | Payload |
|-------|---------|
| `highland/state/network/connectivity` | `{ state, timestamp, source }` |
| `highland/state/network/ping/8_8_8_8` | `{ state, rtt, timestamp, source }` |
| `highland/state/network/ping/1_1_1_1` | `{ state, rtt, timestamp, source }` |
| `highland/state/network/ping/208_67_222_222` | `{ state, rtt, timestamp, source }` |

`state` is `up` or `down` per host; `rtt` is milliseconds or `null` on failure.

**Events (non-retained):**

| Topic | Payload |
|-------|---------|
| `highland/event/network/connectivity_changed` | `{ previous, current, timestamp, source }` |

### Flow Context

| Key | Value | Set by |
|-----|-------|--------|
| `flow.all_hosts` | Array of `host_*` keys for current cycle | `Build Requests` |
| `flow.host_8_8_8_8` | RTT ms or `false` | `Coalesce` |
| `flow.host_1_1_1_1` | RTT ms or `false` | `Coalesce` |
| `flow.host_208_67_222_222` | RTT ms or `false` | `Coalesce` |
| `flow.connectivity_state` | `up` \| `degraded` \| `down` | `Coalesce` |

---

## Throughput Pipeline

### Speed Test

`node-red-contrib-speedtest` wraps the Speedtest CLI. Pinned to server ID `73509` (Contabo, Carlstadt NJ) for consistent comparable results — server variance was observed to cause significant upstream measurement discrepancy across servers.

**Cadence:** Hourly via CronPlus.

### Parse

Raw bandwidth values from Speedtest are in bytes/sec. Conversion: `Math.round(bandwidth * 8 / 1000000)` → Mbit/s.

`msg.results` carries the structured derived values:

```json
{
  "throughput": {
    "downstream": 941,
    "upstream": 830
  },
  "ping": {
    "jitter": 0.115,
    "latency": 10.929,
    "packet_loss": 0
  }
}
```

### MQTT Topics

**State (retained):**

| Topic | Payload |
|-------|---------|
| `highland/state/network/speedtest` | `{ throughput, ping, server, result_url, timestamp, source }` |

```json
{
  "timestamp": "2026-04-28T...",
  "source": "speedtest_monitor",
  "throughput": { "downstream": 941, "upstream": 830 },
  "ping": { "jitter": 0.115, "latency": 10.929, "packet_loss": 0 },
  "server": { "name": "Contabo", "location": "Carlstadt, NJ" },
  "result_url": "https://www.speedtest.net/result/c/..."
}
```

---

## Home Assistant Discovery

All entities published under a single device `Highland Internet Connectivity` (identifiers: `highland_network`).

| Entity | Type | HA Entity ID | Source |
|--------|------|-------------|--------|
| Internet Connectivity | `binary_sensor` | `binary_sensor.internet_connectivity` | `highland/state/network/connectivity` |
| Google DNS | `binary_sensor` | `binary_sensor.internet_google_dns` | `highland/state/network/ping/8_8_8_8` |
| Cloudflare DNS | `binary_sensor` | `binary_sensor.internet_cloudflare_dns` | `highland/state/network/ping/1_1_1_1` |
| OpenDNS | `binary_sensor` | `binary_sensor.internet_open_dns` | `highland/state/network/ping/208_67_222_222` |
| Google DNS RTT | `sensor` | `sensor.internet_google_dns_rtt` | `highland/state/network/ping/8_8_8_8` |
| Cloudflare DNS RTT | `sensor` | `sensor.internet_cloudflare_dns_rtt` | `highland/state/network/ping/1_1_1_1` |
| OpenDNS RTT | `sensor` | `sensor.internet_open_dns_rtt` | `highland/state/network/ping/208_67_222_222` |
| Download Speed | `sensor` | `sensor.internet_download_speed` | `highland/state/network/speedtest` |
| Upload Speed | `sensor` | `sensor.internet_upload_speed` | `highland/state/network/speedtest` |
| Speedtest Ping Latency | `sensor` | `sensor.internet_speedtest_ping_latency` | `highland/state/network/speedtest` |
| Speedtest Ping Jitter | `sensor` | `sensor.internet_speedtest_ping_jitter` | `highland/state/network/speedtest` |
| Speedtest Packet Loss | `sensor` | `sensor.internet_speedtest_packet_loss` | `highland/state/network/speedtest` |

**Notes:**
- `binary_sensor` connectivity entities use `payload_on: 'up'`, `payload_off: 'down'` — `degraded` aggregate state shows as `off`
- Download/Upload Speed use `device_class: 'data_rate'` with `unit_of_measurement: 'Mbit/s'` (not `Mbps` — HA requires the SI form)
- RTT and ping sensors use `device_class: 'duration'`, `unit_of_measurement: 'ms'`
- Throughput and ping sensors include `state_class: 'measurement'` for long-term HA statistics

---

## Pending

- **#55** — Outage detection and notification (consecutive failure tracking)
- **#56** — Slow speed detection and notification (thresholds TBD after baseline established)
- **#57** — Healthchecks.io dead man's switch gating on connectivity state
- **#58** — State change and result logging

---

*Last Updated: 2026-04-28*
