# LoRa Integration — Design & Architecture

## Overview

LoRaWAN-based sensor network extending smart home coverage beyond the practical range of Zigbee and Z-Wave. Designed for low-power, outdoor, long-range use cases where the property's ~275ft driveway makes other protocols impractical.

---

## Infrastructure

### Gateway

**Device:** DFRobot DFR1093-915 Private LoRaWAN Gateway (~$99)
**Frequency:** US915 (902–928 MHz)
**Radio:** Semtech SX1302 8-channel chip

**Architecture note:** The gateway runs an internal SIoT MQTT broker (not configurable to an external broker natively). Data path uses a lightweight relay flow on the gateway's built-in Node-RED instance:

```
LoRa Node → Gateway radio → SIoT broker (localhost:1883)
  → Gateway Node-RED relay flow
  → Mosquitto on Communication Hub (hub.local:1883)
  → Node-RED on Workflow host
```

**Relay flow pattern (on gateway):**
- MQTT In — subscribes to SIoT broker at `localhost:1883`
- Function node — reshapes payload into `highland/` namespace, adds timestamp and source
- MQTT Out — publishes to `hub.local:1883` with highland MQTT credentials

> **Security note:** Change SIoT default credentials (`siot`/`dfrobot`) before the gateway touches the LAN.

---

## Use Case 1: Trash & Recycling Bin Monitoring

### Problem

Property is ~275ft from the road via a shared driveway with no line of sight to the street. Two bins (trash, recycling) need monitoring for: "Did I put them out?" and "Were they picked up?"

### Sensor Hardware

**Device:** Milesight EM320-TILT (US915), ~$84 each × 2
- 3-axis MEMS accelerometer, ±1° accuracy
- IP67 weatherproof
- 2× ER14505 Li-SOCl2 batteries, ≥3 year life at 10-min reporting
- NFC configuration via Milesight ToolBox app
- Immediate uplink on threshold crossing

**Mounting:** Screw-mount to bin side panel (not lid) with consistent orientation. Verify axis mapping via NFC app before deployment.

### Zone Detection (Home vs. Away)

Uses **RSSI** (and SNR) from LoRaWAN uplink packets — reported by the gateway on every received transmission, no additional hardware required.

- Calibrate once: note RSSI at street (~275ft) vs. home position
- Rolling average of last N readings (3–5) to reduce noise before zone state change
- Threshold sits in the ~15–20dB delta between home and street positions

**Limitation:** RSSI is sufficient for "probably on property" vs. "probably at the street" — which is all that's needed.

### Pickup Detection

**Trigger:** X-axis rotation ≥ 85° while bin is in `AWAY` state and past debounce window.

**Rationale:** Automated truck arm rotates bin 90–135° during dump cycle. No normal bin activity (rolling, wind, bumps) produces rotation anywhere near that magnitude.

**Configuration:** Threshold set on EM320-TILT via NFC app. Device fires immediate uplink on threshold crossing — zero detection latency.

### Bin State Machine

```
HOME ──────────────────────────────────────────────────────┐
  │                                                         │
  │ RSSI crosses outbound threshold                         │
  ▼                                                         │
AWAY_SETTLING (15-min debounce)                             │
  │                                 │                       │
  │ 15 min elapsed                  │ RSSI crosses back     │
  ▼                                 ▼                       │
AWAY ──────────────────────────► HOME                       │
  │                                                         │
  │ X-axis ≥ 85° threshold                                  │
  ▼                                                         │
PICKED_UP                                                   │
  │                                                         │
  │ RSSI crosses inbound threshold                          │
  ▼                                                         │
RETURNED ────────────────────────────────────────────────► HOME
```

**Debounce rationale:** 15-minute AWAY_SETTLING window prevents rolling/jostling during bin placement from triggering false pickup events.

### Calendar Integration

Pickup schedule stored in household Google Calendar. `trash_day` boolean unlocks context-aware behavior:

| Condition | Action |
|-----------|--------|
| Trash day, bins still `HOME` at configurable morning time | Reminder notification |
| Trash day, bin transitions to `PICKED_UP` | "Trash picked up, bins at street" notification |
| Bin in `PICKED_UP` state past configurable evening time | "Don't forget to bring the bins in" reminder |
| Not trash day, bin transitions to `AWAY` | No action |

**Independent tracking:** Trash and recycling run separate state machines. It is normal for trash to be `AWAY` while recycling is `HOME`.

### MQTT Topics

**State (retained):**

| Topic | Purpose |
|-------|---------|
| `highland/state/driveway/trash_bin` | Current trash bin state |
| `highland/state/driveway/recycling_bin` | Current recycling bin state |

```json
{
  "timestamp": "...",
  "source": "lora_bin_monitor",
  "state": "AWAY",
  "rssi": -108,
  "snr": -4.2,
  "battery_pct": 94
}
```

`state` values: `"HOME"` | `"AWAY_SETTLING"` | `"AWAY"` | `"PICKED_UP"` | `"RETURNED"`

**Events (not retained):**

`highland/event/driveway/{bin}/zone_changed` | `picked_up` | `returned`

`{bin}` values: `trash_bin` | `recycling_bin`

---

## Use Case 2: Mailbox Delivery Detection

### Problem

Mailbox is ~275ft from the house at the end of the driveway with no line of sight. Three mailboxes share one post. Goals: "Has mail been delivered?" and "Have I retrieved it?"

A single reed switch cannot distinguish delivery from retrieval from an ad stuffer — all interactions are brief. Detection is delegated to USPS Informed Delivery, with the door sensor serving purely as an activity signal.

### Sensor Hardware

**Device:** Milesight EM300-MCS (US915), ~$55–65
- Magnetic reed switch on 1.5m lead cable — sensor body and radio mount separately from the switch
- IP67 weatherproof
- 4000mAh battery (5-year life)
- Includes temperature and humidity sensors (bonus)

**Mounting:** Sensor body mounts on the back/side of the mailbox post — outside the metal enclosure, radio fully unobstructed. 1.5m cable runs to the door mechanism. Small magnet epoxied to inside of mailbox door; reed switch element fixed to door frame.

**Rationale for split mount:** Metal mailbox enclosures attenuate RF and experience extreme temperature swings. Keeping the radio outside avoids both problems.

### Delivery Detection — USPS Informed Delivery

USPS Informed Delivery sends **two emails** each mail day:

| Email | Timing | Content |
|-------|--------|---------|
| Morning advisory | ~6am | Preview of mail expected that day |
| Delivery confirmation | Shortly after physical delivery | Confirms mail placed in box |

The delivery confirmation email is the load-bearing signal. Real-world lag for this address is TBD — must be calibrated from observed data.

### Mailbox State Machine

```
IDLE
  │
  ├─── Morning advisory email arrives ──────────────────────────────────┐
  │                                                                     │
  │ Door event fires                                                    │
  ▼                                                                     ▼
UNCLASSIFIED                                               ADVISORY_RECEIVED
  │                                                                     │
  │ Delivery confirmation email arrives                                 │ Delivery confirmation email arrives
  ▼                                                                     │
MAIL_WAITING ◄───────────────────────────────────────────────────────── ┘
  │
  │ Door event
  ▼
RETRIEVED → IDLE

(No confirmation by midnight → DELIVERY_EXCEPTION)
(Confirmation eventually arrives → MAIL_WAITING)
```

**Key design points:**

- The delivery confirmation email is the load-bearing signal — when it arrives, it resolves the current state to `MAIL_WAITING` regardless of what preceded it
- **Lag window** (`delivery_email_lag_window_minutes`): if a door event occurred within this window before the confirmation email arrived, it's treated as the delivery event itself
- **Midnight rollover** triggers the `DELIVERY_EXCEPTION` check: if state is still `ADVISORY_RECEIVED` at midnight, no confirmation came that calendar day
- `DELIVERY_EXCEPTION` is not terminal — a delayed confirmation resolves it normally
- The morning advisory is explicitly **not** load-bearing for delivery detection

### Configuration (thresholds.json)

```json
"mailbox": {
  "delivery_email_lag_window_minutes": 30,
  "mail_waiting_reminder_time": "18:00",
  "processed_email_retention_days": 14
}
```

`delivery_email_lag_window_minutes` must be calibrated from real-world data.

### Email Management

IMAP poller in Node-RED (`node-red-node-email`) polls the household IMAP server. After processing, emails are **moved to a folder** (`Highland/Mailbox` or similar) rather than deleted — provides an audit trail for debugging edge cases. Processed emails older than 14 days are purged.

**Why move, not delete:** The 36-hour delivery gap scenario demonstrates that an email parsed one day may be relevant the next. Deleting on parse would have removed the advisory before the confirmation arrived.

### MQTT Topics

**State (retained):** `highland/state/mailbox/delivery`

```json
{
  "timestamp": "...",
  "source": "lora_mailbox",
  "state": "MAIL_WAITING",
  "last_door_event": "...",
  "advisory_received_at": "...",
  "confirmation_received_at": "..."
}
```

`state` values: `"IDLE"` | `"UNCLASSIFIED"` | `"ADVISORY_RECEIVED"` | `"MAIL_WAITING"` | `"DELIVERY_EXCEPTION"` | `"RETRIEVED"`

**Events (not retained):** `highland/event/mailbox/door_activity` | `mail_expected` | `mail_delivered` | `mail_retrieved` | `delivery_exception`

---

## Hardware Summary

| Item | Device | Qty | Unit Cost | Total |
|------|--------|-----|-----------|-------|
| Gateway | DFRobot DFR1093-915 | 1 | $99 | $99 |
| Bin sensors | Milesight EM320-TILT (US915) | 2 | $84 | $168 |
| Mailbox sensor | Milesight EM300-MCS (US915) | 1 | ~$60 | ~$60 |
| **Total** | | | | **~$327** |

---

## Open Questions

**Infrastructure**
- [ ] SIoT topic structure — confirm `ProjectID/DeviceName` format during gateway setup

**Trash & Recycling**
- [ ] RSSI calibration procedure — document threshold values once sensors are deployed
- [ ] Optimal RSSI smoothing window (N readings)
- [ ] Morning reminder time threshold
- [ ] Evening retrieval reminder time threshold
- [ ] EM320-TILT mount orientation per bin — verify axis mapping before finalizing pickup threshold config

**Mailbox**
- [ ] Physical installation — sensor body mount point, cable routing, magnet placement
- [ ] IMAP polling interval and email parsing approach
- [ ] Real-world delivery email lag — calibrate from observed data
- [ ] IMAP folder name for processed emails

---

*Last Updated: 2026-03-26*
