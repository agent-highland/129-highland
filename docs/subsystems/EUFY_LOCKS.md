# Eufy Locks Integration

## Scope

Highland has two Eufy Wi-Fi smart locks:

- **Front door** — Eufy deadbolt
- **Garage egress** — Eufy C33 lever lock

The Eufy doorbell is on the same account but is **explicitly out of scope** for this integration. The Reolink doorbell is the preferred doorbell going forward. The eufy-bridge must filter to lock device types only — the doorbell must not be wired into the Highland MQTT bus.

---

## Approach

A custom TypeScript microservice (`eufy-bridge`) built directly on the [`eufy-security-client`](https://github.com/bropat/eufy-security-client) npm library. This is intentionally **not** built on `eufy-security-ws` — that project lags behind the client library and adds an unnecessary WebSocket translation layer.

The bridge has one job: translate between eufy-security-client events/commands and the Highland MQTT bus. Node-RED never knows it's Eufy — it sees MQTT topics like everything else.

---

## Architecture

```
eufy-security-client (npm dependency)
    ↕ events / method calls
eufy-bridge (TypeScript service, Docker on Comm Hub)
    ↕ MQTT
highland/ bus → consumed by Node-RED, HA
```

### Docker

- Runs on the **Comm Hub** alongside Mosquitto, Zigbee2MQTT, Z-Wave JS UI
- `--network host` required for local P2P UDP broadcast discovery of the HomeBase/locks
- `/data` volume for persistent auth token and session data

---

## MQTT Topics

| Topic | Direction | Retained | Purpose |
|-------|-----------|----------|---------|
| `highland/state/lock/<serial>/locked` | Publish | Yes | Current locked/unlocked state |
| `highland/state/lock/<serial>/battery` | Publish | Yes | Battery level percent |
| `highland/event/lock/<serial>/changed` | Publish | No | Lock state change event |
| `highland/command/lock/<serial>/set` | Subscribe | No | Lock/unlock commands from Node-RED |

`<serial>` is the Eufy device serial number, which is stable and unique per device.

---

## Configuration

Key `EufySecurityConfig` options:

```typescript
{
    username: string;           // Eufy account email (see Credentials section)
    password: string;           // Eufy account password
    country: 'US';
    persistentDir: '/data';     // Docker volume — persists auth tokens across restarts
    p2pConnectionSetup: 0;      // Local preferred over cloud
    pollingIntervalMinutes: 10; // Background cloud refresh; consider disabling (set 0) for lock-only use
    eventDurationSeconds: 10;
}
```

### Polling

Lock/unlock events arrive in real-time via Eufy's internal MQTT path — cloud polling is not required for event delivery. For a lock-only bridge, `pollingIntervalMinutes` can potentially be set to `0` to disable background polling entirely. Test with polling enabled first to establish a baseline, then evaluate disabling it.

---

## Credentials

### Primary vs. Secondary Account

The eufy-security-client imitates a mobile app session. Every time the service starts, it forces all other sessions on that account to log off. The standard mitigation is to create a dedicated secondary Eufy account, share the home and devices to it with admin rights, and use those credentials in the bridge.

**Decision:** Attempt deployment with the primary account first. Given low Eufy app usage, session conflicts are unlikely to be a practical problem. Create a secondary account only if conflicts become a real-world issue.

---

## Device Filtering

At startup, the bridge must enumerate all devices on the account and filter to lock device types only. The Eufy doorbell shares the same account and must not produce MQTT events or respond to commands via this bridge.

Filter using the device type check provided by the library (e.g. `Device.isLock(device)`).

---

## Secondary Account Note

If a secondary account is ever needed: create a second Eufy account with a dedicated email, share the home from the primary account with admin rights, log into the Eufy app once on a phone with the secondary account to verify device visibility, then use those credentials in `secrets.json`.

---

## Implementation Notes

- Node.js >= 20.0.0 required (breaking change in recent eufy-security-client versions)
- The Docker image intentionally reverts CVE-2023-46809 (RSA PKCS#1 padding) because eufy-security-client requires it for cloud decryption — this is a known and documented tradeoff
- `trustedDeviceName` in config should be set to something identifiable (e.g. `highland-eufy-bridge`) — visible in the Eufy app's trusted devices list when 2FA is active

---

## Fallback Path

The Eufy integration is the primary plan. The existing hardware will be exhausted before considering a pivot. However, if the eufy-security-client approach proves unworkable — for example, due to Eufy API changes that break the library and are not promptly fixed, or fundamental reliability issues with the C33 over the bridge — the fallback is:

**Two Yale Assure Lock 2 Touch with Z-Wave (ZW3), keyless:**
- Front door: deadbolt, direct replacement
- Garage egress: requires adding a deadbolt to the door (currently lever-only) before installation

These locks integrate directly into Z-Wave JS UI with no cloud dependency. Fingerprints are enrolled once via the Yale Access app over Bluetooth and stored locally on the lock. All subsequent integration is via Z-Wave → ZWJSUI → MQTT, identical to other Z-Wave devices in Highland.

If this path is ever activated, reopen GitHub issues #8 and #9 for tracking.

---

*Last Updated: 2026-04-07*
