# Node-RED — Utility: Daily Digest

## Purpose

Nightly HTML email summarizing the state of the home. Delivered every morning so you don't need to go look. Built and tested — confirmed delivering via midnight scheduler event.

---

## Timing

**Trigger:** `highland/event/scheduler/midnight`

**Delay:** 5 seconds after midnight — gives the Calendaring flow time to complete its midnight poll before the Digest assembles, and ensures the date has rolled over correctly.

---

## Data Sources

The Digest is a pure reader. All data comes from retained MQTT topics stored to flow context continuously by the Sinks group. No on-demand API calls at send time.

| Section | Source Topic | Notes |
|---------|-------------|-------|
| Appointments | `highland/state/calendar/snapshot` | Filtered to today, sorted by start time |
| Reminders | `highland/state/calendar/snapshot` | Filtered to today |
| Trash & Recycling | `highland/state/calendar/snapshot` | Today or tomorrow look-ahead |
| Weather | `highland/state/weather/forecast` + `highland/state/weather/alerts` | Day/night panels + 5-day + alert banners |
| Battery Status | `highland/state/battery/states` | Devices in `low` or `critical` only |
| System Health | `global.connections` | MQTT and HA connection state |

---

## Flow Groups

**Sinks** — Four MQTT in nodes (`calendar/snapshot`, `weather/forecast`, `weather/alerts`, `battery/states`), each wiring to a Store function that parses and saves to flow context. Datatype set to `auto-detect` — Store functions use `typeof msg.payload === 'string' ? JSON.parse(msg.payload) : msg.payload` to handle both pre-parsed objects and raw strings.

**Digest Pipeline** — Single linear group:
- `Midnight` MQTT in → `Delay 5s` → `Assemble Data` → `Build Email` → `Build SMTP Message` → `Send` (email out node)
- `Manual Send` link in bypasses the midnight trigger and delay — wires directly to `Assemble Data` for on-demand testing

---

## Email Design

HTML generated via template literal in `Build Email` function node. Inline styles only — email client compatibility. Sections render conditionally: sections with no data are omitted entirely.

**Sections:**
1. Appointments — timed events with hour badge and title
2. Reminders — all-day nudges as a simple list
3. Trash & Recycling — today/tomorrow pickup, two-column layout
4. Weather — optional alert banners (severity-coded), day/night panels with icons + precip/wind, 5-day forecast
5. Battery Status — devices in `low` or `critical` state, color-coded by severity
6. System Health — green grid when all operational; status badges per service

---

## Weather Icons

**Icon set:** Meteocons by Bas Milius (`basmilius/weather-icons`, dev branch) — MIT license, filled style.

**Storage:** 236 PNGs at `/home/nodered/assets/weather-icons/64/` on the Workflow host, volume-mounted into the Node-RED container at `/assets/weather-icons/64/`.

**Delivery:** CID inline attachments. `Build Email` tracks which icons are referenced in a `usedIcons` Set and passes `msg.usedIcons` downstream. `Build SMTP Message` reads each file via `fs.readFileSync()` and builds `msg.attachments` with `cid`, `filename`, `content`, and `contentType`. HTML references icons as `<img src="cid:name.png">`.

> **Why CID and not `path` property:** nodemailer's `path` property was found to produce empty base64 content in this environment. `fs.readFileSync()` in `Build SMTP Message` is the reliable approach.

**NWS → Meteocons mapping** lives in `getMeteoconName(code, isDaytime)` inside `Build Email`. NWS condition codes (e.g. `sct`, `rain`, `tsra`) map to Meteocons filenames (e.g. `partly-cloudy-day`, `rain`, `thunderstorms-day`). Day/night variants selected from `isDaytime` field on each forecast period.

**Icon display note:** Meteocons 64px PNGs have significant transparent padding; icons display correctly at stated dimensions despite appearing smaller than the raw pixel count suggests.

---

## Recipients & Sender

Resolved at send time from config:

```javascript
// notifications.json
{
  "daily_digest": {
    "enabled": true,
    "recipients": ["joseph"]
  }
}

// secrets.json
{
  "email_addresses": {
    "joseph": "Joseph Ferris <joseph@example.com>",
    "home": "Ferris Smart Home <home@example.com>"
  }
}
```

`Build SMTP Message` maps named recipients to addresses, sets `msg.from = emailAddresses['home']`, and attaches icons. `node-red-node-email` handles SMTP delivery (port 465, `secure: true`).

---

## Gmail Compatibility Note

Gmail blocks `data:` URI images and inline SVG. CID attachments are the correct approach for embedded images. Do not attempt to embed images via data URIs.

---

## Backlog

Template externalization is captured in `AUTOMATION_BACKLOG.md` — `Build Email` is a large inline function node. Moving the HTML template to a file in `/home/nodered/config/templates/daily-digest.html` and loading via Config Loader at startup into `global.config.digestTemplate` is the target state but is deferred.

---

*Last Updated: 2026-03-26*
