# Highland Secrets & Credentials

Unified reference for all credentials, keys, tokens, and service accounts across the Highland infrastructure.

> **⚠️ IMPORTANT:** This file is a **blank template** — it contains no real values and is safe to commit.
> The populated copy must be stored **outside this repository** in a password manager or encrypted document.
> Never commit a version of this file containing real credentials.

---

## Table of Contents

1. [Network Infrastructure](#1-network-infrastructure)
2. [Hardware Reference](#2-hardware-reference)
3. [System Access (SSH & OS)](#3-system-access-ssh--os)
4. [MQTT Broker](#4-mqtt-broker)
5. [Database (PostgreSQL)](#5-database-postgresql)
6. [Protocol Coordinators](#6-protocol-coordinators)
   - [Zigbee (Z2M)](#61-zigbee-z2m)
   - [Z-Wave (Z-Wave JS UI)](#62-z-wave-z-wave-js-ui)
7. [Home Assistant](#7-home-assistant)
8. [External Monitoring](#8-external-monitoring)
9. [Email (Outbound — SMTP)](#9-email-outbound--smtp)
10. [Email (Inbound — IMAP)](#10-email-inbound--imap)
11. [External APIs](#11-external-apis)
    - [Pirate Weather](#111-pirate-weather)
    - [Google Calendar](#112-google-calendar)

---

## 1. Network Infrastructure

| Host | Hostname | IP Address | Hardware |
|------|----------|------------|----------|
| Communication Hub | `hub.local` | `FILL_IN` | Dell OptiPlex 7050 MFF |
| HAOS | `home.local` | `FILL_IN` | Dell OptiPlex 7050 SFF |
| Workflow | `workflow.local` | `FILL_IN` | Dell OptiPlex 7050 SFF |
| NVR | `nvr.local` | `FILL_IN` | Reolink NVR |
| Router/Gateway | — | `FILL_IN` | — |

---

## 2. Hardware Reference

Serial dongles on the Communication Hub. Stable by-id paths — update only if hardware changes.

| Device | By-ID Path |
|--------|------------|
| Zigbee dongle (SONOFF MG24) | `/dev/serial/by-id/FILL_IN` |
| Z-Wave dongle (SONOFF PZG23) | `/dev/serial/by-id/FILL_IN` |

---

## 3. System Access (SSH & OS)

### SSH Keypair

Used for admin access to Ubuntu hosts (hub, workflow).

| Field | Value |
|-------|-------|
| Private key path | `FILL_IN` (e.g. `~/.ssh/highland`) |
| Public key fingerprint | `FILL_IN` |
| Passphrase | `FILL_IN` |

Authorized on: `hub`, `workflow`

### OS User Accounts

| Host | Username | Password |
|------|----------|----------|
| hub | `highland` | `FILL_IN` |
| workflow | `highland` | `FILL_IN` |

---

## 4. MQTT Broker

Mosquitto broker on the Communication Hub. Per-service accounts — each service authenticates independently.
Stored in `/opt/highland/mosquitto/config/password.txt`.

| Account | Password | Used By |
|---------|----------|---------|
| `svc_zigbee2mqtt` | `FILL_IN` | Zigbee2MQTT container |
| `svc_zwavejs` | `FILL_IN` | Z-Wave JS UI container |
| `svc_nodered` | `FILL_IN` | Node-RED |
| `svc_homeassistant` | `FILL_IN` | Home Assistant MQTT integration |
| `svc_scripts` | `FILL_IN` | Backup scripts, watchdog, CLI tools |

---

## 5. Database (PostgreSQL)

Deployed on the Workflow host. Shared between HA Recorder and the video pipeline.

| Field | Value |
|-------|-------|
| Host | `workflow.local` |
| Port | `5432` |
| Username | `highland` |
| Password | `FILL_IN` |
| Database (HA Recorder) | `homeassistant` |

---

## 6. Protocol Coordinators

### 6.1 Zigbee (Z2M)

Network key encrypts all Zigbee traffic. Losing this key requires re-pairing all Zigbee devices.

```
network_key: FILL_IN
# Format: [161, 178, 195, 212, 229, 246, 161, 178, 195, 212, 229, 246, 161, 178, 195, 212]
```

### 6.2 Z-Wave (Z-Wave JS UI)

S2 keys secure Z-Wave device pairing. Required to restore paired devices if the controller is replaced.
Export from Z-Wave JS UI: **Settings → Backup → Export**

| Key Type | Value |
|----------|-------|
| S2 Unauthenticated | `FILL_IN` |
| S2 Authenticated | `FILL_IN` |
| S2 Access Control | `FILL_IN` |
| S0 (Legacy) | `FILL_IN` |

---

## 7. Home Assistant

### Admin Account

| Field | Value |
|-------|-------|
| Username | `FILL_IN` |
| Password | `FILL_IN` |
| Internal URL | `http://home.local:8123` |
| External URL | `https://your-domain.example` |

### Long-Lived Access Tokens

| Token Name | Value | Used By |
|------------|-------|---------|
| `node-red` | `FILL_IN` | Node-RED HA integration |

### Nabu Casa

| Field | Value |
|-------|-------|
| Account email | `FILL_IN` |
| Account password | `FILL_IN` |
| Remote URL | `FILL_IN` |

---

## 8. External Monitoring

Healthchecks.io ping URLs. Treat like passwords — knowledge of the URL allows resetting the monitor.

| Check | Period | Ping URL |
|-------|--------|----------|
| `highland-node-red` | 1 min | `https://hc-ping.com/FILL_IN` |
| `highland-hub-backup` | 24h | `https://hc-ping.com/FILL_IN` |
| `highland-workflow-backup` | 24h | `https://hc-ping.com/FILL_IN` |

---

## 9. Email (Outbound — SMTP)

Used for Daily Digest notifications.

| Field | Value |
|-------|-------|
| Provider | `FILL_IN` |
| SMTP host | `FILL_IN` |
| SMTP port | `587` |
| Username | `FILL_IN` |
| Password | `FILL_IN` |
| From address | `FILL_IN` |
| To address | `FILL_IN` |

---

## 10. Email (Inbound — IMAP)

Used by the LoRaWAN mailbox flow to parse USPS Informed Delivery emails.
Hosted at `highland@your-domain.example`.

| Field | Value |
|-------|-------|
| IMAP host | `FILL_IN` |
| IMAP port | `993` (SSL) |
| SMTP host | `FILL_IN` |
| SMTP port | `587` |
| Username | `highland@your-domain.example` |
| Password | `FILL_IN` |

---

## 11. External APIs

### 11.1 Pirate Weather

| Field | Value |
|-------|-------|
| API key | `FILL_IN` |
| Plan | `FILL_IN` |
| Endpoint | `https://api.pirateweather.net/forecast` |

### 11.2 Google Calendar

Used for the Daily Digest and camera suppression integration (each camera has a dedicated calendar guest address).

| Field | Value |
|-------|-------|
| API key | `FILL_IN` |
| Service account email | `FILL_IN` |
| Main calendar ID | `FILL_IN` |

**Camera opt-out addresses** — invite these as guests to suppress monitoring for the event duration:

| Camera | Calendar Email |
|--------|----------------|
| `FILL_IN` | `FILL_IN@your-domain.example` |
| `FILL_IN` | `FILL_IN@your-domain.example` |
| `FILL_IN` | `FILL_IN@your-domain.example` |

---

*Template version: 2026-03-26 — Add new sections here as infrastructure grows.*
