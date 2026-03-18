# Node-RED Patterns & Conventions

## Overview

Design patterns and conventions for Node-RED flows in the Highland home automation system. These patterns prioritize readability, maintainability, and alignment with the event-driven architecture.

---

## Core Principles

1. **Visibility over abstraction** — Keep logic visible in flows; don't hide complexity in subflows unless truly reusable
2. **Horizontal scrolling is the enemy** — Use link nodes and groups to keep flows compact and readable
3. **Pub/sub for inter-flow communication** — Flows talk via MQTT events, not direct dependencies
4. **Centralized error handling** — Flow-wide catch with targeted overrides
5. **Configurable logging** — Per-flow log levels for flexible debugging

---

## Node-RED Environment Configuration

### Using Node.js Modules in Function Nodes

`require()` is not available directly in function nodes in Node-RED 3.x. Modules must be declared in the function node's **Setup tab** and are injected as named variables.

**To use `fs` and `path` (or any other module):**
1. Open the function node
2. Go to the **Setup** tab
3. Add entries under **Modules**:
   - Name: `fs` / Module: `fs`
   - Name: `path` / Module: `path`
4. In the function body, use `fs` and `path` directly — do **not** call `require()`

```javascript
// WRONG — will throw "require is not defined"
const fs = require('fs');

// CORRECT — declared in Setup tab, available as plain variable
const raw = fs.readFileSync(filepath, 'utf8');
```

This applies to any Node.js built-in or npm module used in function nodes.

### Context Storage (settings.js)

Node-RED context must be configured for disk persistence to survive restarts/deploys:

```javascript
// In settings.js
contextStorage: {
    default: {
        module: "localfilesystem"
    }
}
```

This enables:
- Flow context persistence (`flow.set()` / `flow.get()`)
- Global context persistence (`global.set()` / `global.get()`)
- Survives Node-RED restarts and deploys

### Home Assistant Integration

**Primary method:** `node-red-contrib-home-assistant-websocket`

Provides:
- HA entity state access
- Service calls (notifications, backups, etc.)
- Event subscription
- WebSocket connection to HA

**Configuration:**
- Base URL: `http://{ha_ip}:8123`
- Access Token: Long-lived access token from HA (stored in Node-RED credentials, not in config files)

**Use cases:**
| Action | Method |
|--------|--------|
| Trigger HA backup | Service call: `backup.create` |
| Send notification via Companion App | Service call: `notify.mobile_app_*` |
| Check HA entity state | HA API node or WebSocket |
| React to HA events | HA events node |

*Note: Most device control goes through MQTT directly (Z2M, Z-Wave JS). HA integration is primarily for HA-specific features (backups, notifications, entity state that only exists in HA).*

### Watchdog Script

External process that monitors Node-RED and pings Healthchecks.io. Runs on the Node-RED host.

**Location:** `/usr/local/bin/highland-watchdog.sh`

**Implementation:**
```bash
#!/bin/bash
# Highland Watchdog - Monitors Node-RED heartbeat via MQTT
# Pings Healthchecks.io on success, fails silent on miss

MQTT_HOST="hub.local"
MQTT_USER="highland"
MQTT_PASS="from-env-or-file"
HEARTBEAT_TOPIC="highland/status/node_red/heartbeat"
HC_PING_URL="https://hc-ping.com/your-uuid-here"
TIMEOUT=90  # seconds to wait for heartbeat

# Subscribe and wait for one message with timeout
mosquitto_sub -h "$MQTT_HOST" -u "$MQTT_USER" -P "$MQTT_PASS" \
    -t "$HEARTBEAT_TOPIC" -C 1 -W "$TIMEOUT" > /dev/null 2>&1

if [ $? -eq 0 ]; then
    # Heartbeat received, ping Healthchecks.io
    curl -fsS -m 10 --retry 3 "$HC_PING_URL" > /dev/null
else
    # No heartbeat, don't ping (Healthchecks.io will alert after grace period)
    logger "highland-watchdog: No heartbeat from Node-RED"
fi
```

**Cron entry (`/etc/cron.d/highland-watchdog`):**
```
* * * * * root /usr/local/bin/highland-watchdog.sh
```

**Flow side (Health Monitor):**
- Publishes heartbeat to `highland/status/node_red/heartbeat` every 30 seconds
- Watchdog expects at least one heartbeat per 90-second window

**Dependencies:**
- `mosquitto-clients` package (for `mosquitto_sub`)
- `curl` for Healthchecks.io pings

---
