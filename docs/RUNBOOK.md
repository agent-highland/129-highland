# Implementation Runbook

Step-by-step guide for building the Highland home automation infrastructure.

---

## Pre-Flight Checklist

### Hardware Inventory

| System | Hardware | Storage | Status |
|--------|----------|---------|--------|
| Communication Hub | Ryzen 5 3550H MFF | 512GB SSD | ✅ Ready |
| HAOS | Dell OptiPlex 7050 SFF | 480GB SSD | ✅ Ready |
| Workflow | Dell OptiPlex 7050 SFF | 480GB SSD | ✅ Ready |
| Network Video Recorder | Reolink NVR | Internal | ✅ Ready (boxed) |

**USB Devices:**
- [x] SONOFF Zigbee 3.0 Dongle Plus (MG24) — for Communication Hub
- [x] SONOFF Z-Wave 800 Dongle Plus (PZG23) — for Communication Hub

### Network Planning

| System | Hostname | Internal URL | External URL | Notes |
|--------|----------|--------------|--------------|-------|
| Communication Hub | `hub` | `http(s)://hub.local` | None | DHCP reservation or static |
| HAOS | `home` | `http(s)://home.local` | `https://highland.ferris.network` (after decom) | DHCP reservation or static |
| Workflow | `workflow` | `http(s)://workflow.local` | None | DHCP reservation or static |
| Network Video Recorder | `nvr` | `http(s)://nvr.local` | None | DHCP reservation or static |

**Ports to note:**
| Service | Port | Host |
|---------|------|------|
| MQTT | 1883 | Communication Hub |
| MQTT (WebSocket) | 9001 | Communication Hub (if needed) |
| Zigbee2MQTT Frontend | 8080 | Communication Hub |
| Z-Wave JS UI | 8091 | Communication Hub |
| Home Assistant | 8123 | HAOS |
| Node-RED | 1880 | Workflow |
| Node-RED Admin API | 1880 | Workflow |
| code-server | 8443 | Workflow (if installed) |

### Credentials to Prepare

Generate/document these before starting:

| Credential | Purpose | Store In |
|------------|---------|----------|
| MQTT username/password | Broker auth | secrets.json, Mosquitto config |
| HA long-lived access token | Node-RED → HA integration | Node-RED credentials |
| Healthchecks.io ping URLs | External monitoring | secrets.json |
| SMTP credentials | Daily digest email | secrets.json |
| Google Calendar API key | Daily digest | secrets.json |

**SSH keys:**
- [ ] Generate SSH keypair for admin access to Ubuntu hosts

### Downloads to Stage

Download these in advance to save time:

| Item | URL |
|------|-----|
| Ubuntu Server 24.04 LTS ISO | https://ubuntu.com/download/server |
| HAOS image (Generic x86-64) | https://www.home-assistant.io/installation/generic-x86-64 |
| Balena Etcher (or Rufus) | https://www.balena.io/etcher |

**Docker images (will pull during setup, but good to note):**
- `eclipse-mosquitto:latest`
- `koenkk/zigbee2mqtt:latest`
- `zwavejs/zwave-js-ui:latest`
- `nodered/node-red:latest`
- `codercom/code-server:latest` (optional)

---

## Phase 1: Communication Hub

The backbone. MQTT broker + protocol coordinators.

### 1.1 Ubuntu Server Installation

1. Flash Ubuntu Server 24.04 to USB
2. Boot Ryzen MFF from USB
3. Install with options:
   - Hostname: `hub`
   - Username: `highland` (or your preference)
   - Enable OpenSSH server
   - No additional snaps
4. Complete installation, reboot, remove USB

### 1.2 Base Configuration

```bash
# Login and update
sudo apt update && sudo apt upgrade -y

# Set static IP (if not using DHCP reservation)
sudo nano /etc/netplan/00-installer-config.yaml
```

**Example netplan (adjust for your network):**
```yaml
network:
  version: 2
  ethernets:
    enp0s31f6:  # Your interface name
      dhcp4: no
      addresses:
        - 192.168.1.10/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
```

```bash
sudo netplan apply

# Set timezone
sudo timedatectl set-timezone America/New_York

# Install essentials
sudo apt install -y curl git htop mosquitto-clients jq

# Reboot to confirm static IP persists
sudo reboot
```

### 1.3 Docker Installation

```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Add user to docker group
sudo usermod -aG docker $USER

# Logout and back in for group to take effect
exit
# SSH back in

# Verify
docker --version
docker compose version
```

### 1.4 Directory Structure

```bash
# Create highland services directory
sudo mkdir -p /opt/highland/{mosquitto,zigbee2mqtt,zwavejs}
sudo mkdir -p /opt/highland/mosquitto/{config,data,log}
sudo mkdir -p /opt/highland/zigbee2mqtt/data
sudo mkdir -p /opt/highland/zwavejs/data

# Set ownership
sudo chown -R $USER:$USER /opt/highland

# Create backup directory
sudo mkdir -p /var/backups/highland
sudo chown $USER:$USER /var/backups/highland
```

### 1.5 USB Device Setup

```bash
# Plug in Zigbee and Z-Wave USB devices

# Find device paths (by-id for stability)
ls -la /dev/serial/by-id/

# Note the paths, e.g.:
# /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_xxx-if00-port0
# /dev/serial/by-id/usb-Zooz_800_xxx-if00-port0
```

### 1.6 Mosquitto Configuration

**Create `/opt/highland/mosquitto/config/mosquitto.conf`:**
```
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout

listener 1883
allow_anonymous false
password_file /mosquitto/config/password.txt

# WebSocket listener (optional)
# listener 9001
# protocol websockets
```

**Create password file:**
```bash
# Install mosquitto package for password utility (or use Docker)
docker run --rm -v /opt/highland/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto mosquitto_passwd -c /mosquitto/config/password.txt highland

# Enter password when prompted
```

### 1.7 Zigbee2MQTT Configuration

**Create `/opt/highland/zigbee2mqtt/data/configuration.yaml`:**
```yaml
homeassistant: true
permit_join: false
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883
  user: highland
  password: YOUR_MQTT_PASSWORD
serial:
  port: /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_xxx-if00-port0
frontend:
  port: 8080
advanced:
  network_key: GENERATE_NEW_KEY
  log_level: info
```

**Generate network key:**
```bash
# Generate a random network key
openssl rand -hex 16
# Format as array: [0xAA, 0xBB, 0xCC, ...]
```

### 1.8 Z-Wave JS UI Configuration

Z-Wave JS UI is configured via its web interface after first launch. Key settings to configure:

- **Serial port:** `/dev/serial/by-id/usb-Zooz_800_xxx-if00-port0`
- **MQTT Gateway:** Enabled
- **MQTT Host:** `mosquitto` (Docker network)
- **MQTT Port:** 1883
- **MQTT Auth:** highland / password
- **MQTT Prefix:** `zwave`

### 1.9 Docker Compose

**Create `/opt/highland/docker-compose.yml`:**
```yaml
version: '3.8'

services:
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
      # - "9001:9001"  # WebSocket, if needed
    volumes:
      - /opt/highland/mosquitto/config:/mosquitto/config
      - /opt/highland/mosquitto/data:/mosquitto/data
      - /opt/highland/mosquitto/log:/mosquitto/log

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    depends_on:
      - mosquitto
    ports:
      - "8080:8080"
    volumes:
      - /opt/highland/zigbee2mqtt/data:/app/data
    devices:
      - /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_xxx-if00-port0:/dev/ttyUSB0
    environment:
      - TZ=America/New_York

  zwavejs:
    image: zwavejs/zwave-js-ui:latest
    container_name: zwavejs
    restart: unless-stopped
    depends_on:
      - mosquitto
    ports:
      - "8091:8091"
      - "3000:3000"  # WebSocket for HA
    volumes:
      - /opt/highland/zwavejs/data:/usr/src/app/store
    devices:
      - /dev/serial/by-id/usb-Zooz_800_xxx-if00-port0:/dev/zwave
    environment:
      - TZ=America/New_York
```

### 1.10 Launch Services

```bash
cd /opt/highland
docker compose up -d

# Check status
docker compose ps
docker compose logs -f  # Watch logs, Ctrl+C to exit
```

### 1.11 Verification

| Check | Command / Action |
|-------|------------------|
| MQTT broker responding | `mosquitto_sub -h localhost -u highland -P password -t '#' -v` |
| Z2M frontend accessible | Browse to `http://hub.local:8080` |
| Z-Wave JS UI accessible | Browse to `http://hub.local:8091` |
| Z-Wave JS WebSocket | Configure in UI, check logs |

**Test MQTT pub/sub:**
```bash
# Terminal 1: Subscribe
mosquitto_sub -h localhost -u highland -P YOUR_PASSWORD -t 'test/#' -v

# Terminal 2: Publish
mosquitto_pub -h localhost -u highland -P YOUR_PASSWORD -t 'test/hello' -m 'world'
```

---

## Phase 2: HAOS

Home Assistant OS on dedicated hardware.

### 2.1 HAOS Installation

1. Download HAOS image for Generic x86-64
2. Flash to SSD using Balena Etcher
3. Install SSD in Dell OptiPlex 7050 SFF
4. Boot from SSD
5. Wait for initial setup (can take several minutes)
6. Access at `http://home.local:8123`

### 2.2 Initial Configuration

1. Create admin account
2. Set home location, timezone, units
3. Name the installation: `Highland` (or preference)
4. Skip integrations for now (we'll add manually)

### 2.3 Network Configuration

If not using DHCP reservation, set static IP:
1. Settings → System → Network
2. Configure IPv4 as static
3. Save and reboot if needed

### 2.4 Nabu Casa Setup

1. Settings → Home Assistant Cloud
2. Sign in to Nabu Casa account
3. Configure remote access
4. **Do NOT expose external URL yet** — wait until live instance is decommissioned
5. Link custom domain (`highland.ferris.network`) when ready

### 2.5 MQTT Integration

1. Settings → Devices & Services → Add Integration
2. Search for "MQTT"
3. Configure:
   - Broker: `hub.local` (or IP address)
   - Port: `1883`
   - Username: `highland`
   - Password: (your MQTT password)
4. Submit and verify connection

### 2.6 Z-Wave JS Integration

1. Settings → Devices & Services → Add Integration
2. Search for "Z-Wave"
3. Select "Z-Wave JS"
4. Choose "Use Z-Wave JS Supervisor add-on" → **No**
5. Enter WebSocket URL: `ws://hub.local:3000`
6. Submit

### 2.7 Zigbee2MQTT Integration

Option A: Use MQTT discovery (automatic if Z2M has `homeassistant: true`)

Option B: Add MQTT integration and devices appear automatically

### 2.8 Sidebar Panels (Iframe)

**For Z2M Frontend:**
1. Edit `configuration.yaml` (via File Editor or SSH)
2. Add:
```yaml
panel_iframe:
  zigbee2mqtt:
    title: "Zigbee2MQTT"
    url: "http://hub.local:8080"
    icon: mdi:zigbee
    require_admin: true
```

**For Z-Wave JS UI:**
```yaml
panel_iframe:
  zwavejs:
    title: "Z-Wave JS"
    url: "http://hub.local:8091"
    icon: mdi:z-wave
    require_admin: true
```

3. Restart Home Assistant
4. Verify panels appear in sidebar

*Note: These panels are local network only. Not accessible via Nabu Casa remote.*

### 2.9 Long-Lived Access Token

Generate for Node-RED integration:
1. Click profile (bottom left)
2. Scroll to "Long-Lived Access Tokens"
3. Create Token → Name: `node-red`
4. **Copy and save securely** — shown only once

### 2.10 Verification

| Check | Action |
|-------|--------|
| MQTT connected | Settings → Devices & Services → MQTT → shows connected |
| Z-Wave JS connected | Settings → Devices & Services → Z-Wave JS → shows connected |
| Z2M devices visible | Check MQTT integration for discovered devices |
| Sidebar panels work | Click Z2M and Z-Wave panels, verify UI loads |
| Local access works | Test via `http://home.local:8123` |

---

## Phase 3: Workflow

Automation engine and utility services (Node-RED).

### 3.1 Ubuntu Server Installation

Same process as Communication Hub:
1. Flash Ubuntu Server 24.04 to USB
2. Boot Dell OptiPlex 7050 SFF from USB
3. Install with options:
   - Hostname: `workflow`
   - Username: `highland`
   - Enable OpenSSH server
4. Complete installation, reboot

### 3.2 Base Configuration

```bash
# Update
sudo apt update && sudo apt upgrade -y

# Set static IP if needed (see Phase 1)

# Set timezone
sudo timedatectl set-timezone America/New_York

# Install essentials
sudo apt install -y curl git htop jq mosquitto-clients

# Reboot
sudo reboot
```

### 3.3 Docker Installation

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
exit
# SSH back in
docker --version
```

### 3.4 Directory Structure

```bash
# Node-RED directories
sudo mkdir -p /opt/highland/nodered/data
sudo mkdir -p /home/nodered/config

# Backup directory
sudo mkdir -p /var/backups/highland

# Log directory
sudo mkdir -p /var/log/highland

# Set ownership
sudo chown -R $USER:$USER /opt/highland
sudo chown -R $USER:$USER /home/nodered
sudo chown -R $USER:$USER /var/backups/highland
sudo chown -R $USER:$USER /var/log/highland
```

### 3.5 Docker Compose

**Create `/opt/highland/docker-compose.yml`:**
```yaml
version: '3.8'

services:
  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - /opt/highland/nodered/data:/data
      - /home/nodered/config:/config:ro
      - /var/log/highland:/var/log/highland
    environment:
      - TZ=America/New_York
    # user: "1000:1000"  # Match host user if needed

  # Optional: code-server for VS Code in browser
  # code-server:
  #   image: codercom/code-server:latest
  #   container_name: code-server
  #   restart: unless-stopped
  #   ports:
  #     - "8443:8080"
  #   volumes:
  #     - /opt/highland:/home/coder/highland
  #     - /home/nodered/config:/home/coder/config
  #   environment:
  #     - TZ=America/New_York
  #     - PASSWORD=your-code-server-password
```

### 3.6 Launch Node-RED

```bash
cd /opt/highland
docker compose up -d

# Verify
docker compose ps
docker compose logs nodered
```

Access Node-RED at `http://workflow.local:1880`

### 3.7 Node-RED Palette Installation

Install via Node-RED UI (Menu → Manage Palette → Install):

| Package | Purpose |
|---------|---------|
| `node-red-contrib-home-assistant-websocket` | HA integration |
| `node-red-node-markdown` | Markdown → HTML for digest |
| `schedex` | Sunrise/sunset scheduling |
| `node-red-contrib-moment` | Date/time handling (optional) |

### 3.8 Settings.js Configuration

Edit `/opt/highland/nodered/data/settings.js`:

```javascript
// Find contextStorage section and update:
contextStorage: {
    default: {
        module: "localfilesystem"
    }
},
```

Restart Node-RED after changes:
```bash
docker compose restart nodered
```

### 3.9 Home Assistant Configuration Node

In Node-RED:
1. Add any HA node to a flow
2. Double-click to configure
3. Add new server:
   - Name: `Highland HA`
   - Base URL: `http://home.local:8123`
   - Access Token: (paste long-lived token from Phase 2)
4. Deploy and verify connection

### 3.10 Config Directory Setup

**Create initial config files in `/home/nodered/config/`:**

```bash
cd /home/nodered/config

# Create empty config files
touch device_registry.json
touch flow_registry.json
touch notifications.json
touch thresholds.json
touch healthchecks.json
touch secrets.json
touch README.md
```

**Initialize with empty/default JSON:**

```bash
echo '{}' > device_registry.json
echo '{}' > flow_registry.json
echo '{}' > notifications.json
echo '{}' > thresholds.json
echo '{}' > healthchecks.json
echo '{}' > secrets.json
```

*Populate these as you build flows. See NODERED_PATTERNS.md for structure.*

### 3.11 HA Sidebar Panel (Optional)

Add Node-RED to HA sidebar:

```yaml
# In HA configuration.yaml
panel_iframe:
  nodered:
    title: "Node-RED"
    url: "http://workflow.local:1880"
    icon: mdi:sitemap
    require_admin: true
```

### 3.12 Verification

| Check | Action |
|-------|--------|
| Node-RED accessible | Browse to `http://workflow.local:1880` |
| MQTT connection works | Add MQTT-in node, subscribe to `#`, see traffic |
| HA connection works | Add HA node, verify green "connected" status |
| Context persistence | Set a global variable, restart Node-RED, verify it persists |

**Test context persistence:**
```javascript
// In a function node
global.set('test_persist', { timestamp: Date.now() });
return msg;

// After restart, in another function node
const test = global.get('test_persist');
node.warn(test);  // Should show the saved value
```

---

## Phase 4: Network Video Recorder

Camera infrastructure (Reolink NVR).

### 4.1 Physical Setup

1. Unbox NVR
2. Connect to network (hardwired)
3. Connect to monitor for initial setup (or use Reolink app)
4. Power on

### 4.2 Network Configuration

1. Access NVR web interface or use Reolink app
2. Set static IP (or DHCP reservation)
3. Set hostname to `nvr` for `nvr.local` resolution
4. Configure NTP for correct time
5. Set timezone

### 4.3 Camera Pairing

1. Add cameras via NVR interface
2. For Wi-Fi cameras: Connect to same network, NVR should discover
3. Configure recording settings (continuous, motion, schedule)
4. Set up storage/retention

### 4.4 Home Assistant Integration

1. Settings → Devices & Services → Add Integration
2. Search for "Reolink"
3. Enter NVR IP address and credentials
4. Cameras should appear as entities

**Entity naming (per ENTITY_NAMING.md):**
- May need to rename entities to match conventions
- e.g., `camera.driveway_feed_fluent`, `camera.driveway_feed_clear`

### 4.5 Verification

| Check | Action |
|-------|--------|
| NVR accessible | Browse to `http://nvr.local` |
| Cameras recording | Check NVR playback |
| HA integration | Verify camera entities in HA |
| Live view works | Test camera streams in HA dashboard |

---

## Post-Build

### Backup Scripts

**Communication Hub (`/usr/local/bin/highland-backup.sh`):**
```bash
#!/bin/bash
# Highland Backup Script - Communication Hub

BACKUP_DIR="/var/backups/highland"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="hub_backup_${TIMESTAMP}.tar.gz"

# Create backup
tar -czf "${BACKUP_DIR}/${BACKUP_FILE}" \
    /opt/highland/mosquitto/config \
    /opt/highland/zigbee2mqtt/data \
    /opt/highland/zwavejs/data \
    /opt/highland/docker-compose.yml

# Cleanup old backups (keep 7 days)
find "${BACKUP_DIR}" -name "hub_backup_*.tar.gz" -mtime +7 -delete

# Publish result to MQTT
if [ $? -eq 0 ]; then
    mosquitto_pub -h localhost -u highland -P "PASSWORD" \
        -t "highland/event/backup/completed" \
        -m "{\"host\":\"hub\",\"file\":\"${BACKUP_FILE}\",\"timestamp\":\"$(date -Iseconds)\"}"
else
    mosquitto_pub -h localhost -u highland -P "PASSWORD" \
        -t "highland/event/backup/failed" \
        -m "{\"host\":\"hub\",\"error\":\"tar failed\",\"timestamp\":\"$(date -Iseconds)\"}"
fi
```

```bash
# Make executable
sudo chmod +x /usr/local/bin/highland-backup.sh

# Add to cron (runs at 3:15 AM)
echo "15 3 * * * root /usr/local/bin/highland-backup.sh" | sudo tee /etc/cron.d/highland-backup
```

**Workflow Host:** Backup handled by Backup Utility Flow (see NODERED_PATTERNS.md).

### Watchdog Script

**Create `/usr/local/bin/highland-watchdog.sh` on Workflow host:**
```bash
#!/bin/bash
# Highland Watchdog - Monitors Node-RED heartbeat

MQTT_HOST="hub.local"
MQTT_USER="highland"
MQTT_PASS="PASSWORD"
HEARTBEAT_TOPIC="highland/status/node_red/heartbeat"
HC_PING_URL="https://hc-ping.com/YOUR-UUID-HERE"
TIMEOUT=90

mosquitto_sub -h "$MQTT_HOST" -u "$MQTT_USER" -P "$MQTT_PASS" \
    -t "$HEARTBEAT_TOPIC" -C 1 -W "$TIMEOUT" > /dev/null 2>&1

if [ $? -eq 0 ]; then
    curl -fsS -m 10 --retry 3 "$HC_PING_URL" > /dev/null
else
    logger "highland-watchdog: No heartbeat from Node-RED"
fi
```

```bash
sudo chmod +x /usr/local/bin/highland-watchdog.sh
echo "* * * * * root /usr/local/bin/highland-watchdog.sh" | sudo tee /etc/cron.d/highland-watchdog
```

### Healthchecks.io Setup

1. Create account at healthchecks.io
2. Create checks for:
   - `highland-node-red` (1 min period, 2 min grace)
   - `highland-hub-backup` (24h period, 1h grace)
   - `highland-workflow-backup` (24h period, 1h grace)
3. Copy ping URLs to `secrets.json`

### Log Rotation

**Create `/etc/logrotate.d/highland` on Workflow host:**
```
/var/log/highland/*.jsonl {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 644 highland highland
}
```

### Initial Utility Flows

Create these flows in Node-RED to establish baseline functionality:

1. **Config Loader** — Load config files on startup
2. **Health Monitor** — Publish heartbeat every 30 seconds
3. **Logging Utility** — Subscribe to `highland/event/log`, write to JSONL

### First Device Migration Test

1. Pick one non-critical Zigbee device
2. Remove from old Z2M instance
3. Pair to new Z2M on Communication Hub
4. Verify appears in HA
5. Create simple test flow in Node-RED
6. Verify end-to-end control

---

## Verification Checklist

### Infrastructure Health

| Check | Status |
|-------|--------|
| Hub: All containers running | ☐ |
| Hub: MQTT accepting connections | ☐ |
| Hub: Z2M frontend accessible | ☐ |
| Hub: Z-Wave JS UI accessible | ☐ |
| HAOS: Running and accessible | ☐ |
| HAOS: MQTT integration connected | ☐ |
| HAOS: Z-Wave JS integration connected | ☐ |
| Workflow: Node-RED accessible | ☐ |
| Workflow: HA integration connected | ☐ |
| Workflow: Context persistence working | ☐ |
| NVR: Cameras recording | ☐ |
| NVR: HA integration working | ☐ |

### Backup & Monitoring

| Check | Status |
|-------|--------|
| Hub backup script installed | ☐ |
| Hub backup cron configured | ☐ |
| Watchdog script installed | ☐ |
| Watchdog cron configured | ☐ |
| Healthchecks.io receiving pings | ☐ |
| Log rotation configured | ☐ |

### First Automation

| Check | Status |
|-------|--------|
| Test device paired to new Z2M | ☐ |
| Device visible in HA | ☐ |
| Node-RED can control device via MQTT | ☐ |
| End-to-end automation working | ☐ |

---

*Last Updated: 2026-03-11*
