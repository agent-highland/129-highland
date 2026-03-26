# Network Configuration

## Host Inventory

| Role | Hostname | Internal URL | External URL | Notes |
|------|----------|--------------|--------------|-------|
| HAOS | `home` | `http(s)://home.local` | `https://your-domain.example` | Nabu Casa, after decom of old system |
| Communication Hub | `hub` | `http(s)://hub.local` | None | |
| Workflow | `workflow` | `http(s)://workflow.local` | None | |
| Network Video Recorder | `nvr` | `http(s)://nvr.local` | None | |

All hosts: static IP or DHCP reservation. See `ha/SECRETS_TEMPLATE.md` for actual IP assignments.

---

## Port Reference

| Service | Port | Host | Notes |
|---------|------|------|-------|
| MQTT | 1883 | Communication Hub | Primary broker port |
| MQTT WebSocket | 9001 | Communication Hub | Optional, enable if needed |
| Zigbee2MQTT Frontend | 8080 | Communication Hub | Embedded as HA sidebar panel |
| Z-Wave JS UI | 8091 | Communication Hub | Embedded as HA sidebar panel |
| Z-Wave JS WebSocket | 3000 | Communication Hub | HA native Z-Wave integration |
| Home Assistant | 8123 | HAOS | Primary HA port |
| Node-RED | 1880 | Workflow | |
| PostgreSQL | 5432 | Workflow | HA Recorder + video pipeline |
| code-server | 8443 | Workflow | Optional VS Code in browser |

---

## Remote Access

**Method:** Nabu Casa with custom domain (`your-domain.example`)

**Scope:** Nabu Casa tunnel exposes **HA only** (port 8123). Sidebar iframes (Z2M, Z-Wave JS UI, VS Code) are local network only — they are not reachable via Nabu Casa remote.

**Setup timing:** Nabu Casa is configured after the old live system is decommissioned. Enabling it before that point risks dual-access issues with the legacy installation.

**If remote management of hub/workflow is needed:** Tailscale or WireGuard. Not planned for initial deployment.

---

## mDNS / .local Resolution

All Ubuntu hosts run `avahi-daemon` for `.local` hostname resolution on the LAN.

**Docker containers cannot resolve `.local` hostnames via mDNS.** Avahi runs on the host, not inside containers. Workaround: add `extra_hosts` to relevant services in `docker-compose.yml` mapping `.local` names to their actual IP addresses. See `RUNBOOK.md` section 3.9 for the Node-RED implementation.

**HAOS and .local:** HAOS sometimes resolves `.local` hostnames to IPv6 link-local addresses (`fe80::...`), which psycopg2 and other clients cannot use. Workaround: use the IPv4 address directly in connection strings (e.g., PostgreSQL `db_url`, MQTT broker address). See `RUNBOOK.md` section 3.14 for details.

---

## Current Topology

**Flat network — no VLANs.** All hosts and devices on a single LAN segment. This simplifies inter-service communication (MQTT, WebSocket, mDNS) during the initial build.

**Future:** Network segmentation is planned but out of scope for the initial rebuild. When VLANs are eventually implemented, routing and firewall rules will be required for: MQTT (1883/8883), HA (8123), Z-Wave JS WebSocket (3000), Node-RED (1880), PostgreSQL (5432).

---

## USB Serial Devices (Communication Hub)

Serial dongles use stable by-id paths to avoid `/dev/ttyUSB` roulette on reboot.

| Device | By-ID Path |
|--------|------------|
| Zigbee dongle (SONOFF MG24) | `/dev/serial/by-id/FILL_IN` — see `ha/SECRETS_TEMPLATE.md` |
| Z-Wave dongle (SONOFF PZG23) | `/dev/serial/by-id/FILL_IN` — see `ha/SECRETS_TEMPLATE.md` |

Docker Compose maps the host's by-id symlink to a stable container-side device path (e.g., `/dev/ttyUSB0`, `/dev/zwave`). The stability guarantee is on the host side. See `RUNBOOK.md` sections 1.5 and 1.9.

---

*Last Updated: 2026-03-26*
