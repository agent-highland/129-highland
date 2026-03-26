# Node-RED — Environment & Configuration

## Using Node.js Modules in Function Nodes

`require()` is not available directly in function nodes in Node-RED 3.x. Modules must be declared in the function node's **Setup tab** and are injected as named variables.

**To use `fs`, `path`, or any other module:**
1. Open the function node
2. Go to the **Setup** tab
3. Add entries under **Modules**: Name: `fs` / Module: `fs`, Name: `path` / Module: `path`
4. In the function body, use the variable directly — do **not** call `require()`

```javascript
// WRONG — will throw "require is not defined"
const fs = require('fs');

// CORRECT — declared in Setup tab, available as plain variable
const raw = fs.readFileSync(filepath, 'utf8');
```

This applies to any Node.js built-in or npm module used in function nodes.

---

## Context Storage (settings.js)

Node-RED context is configured with three named stores:

```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    },
    initializers: {
        module: "memory"
    },
    volatile: {
        module: "memory"
    }
}
```

### `default` (localfilesystem)

Persists to disk. Used for flow state, config cache, and any value that must survive a Node-RED restart. This is the store used when no store name is specified.

### `initializers` (memory)

In-memory only. Used exclusively for runtime utilities populated by `Utility: Initializers` at startup — functions, helpers, and other values that cannot be JSON-serialized and therefore cannot use `localfilesystem`. These are re-populated on every restart.

### `volatile` (memory)

In-memory only. Used for transient, non-serializable runtime values that must not be persisted — timer handles, open connection references, or anything that would cause a circular reference error if Node-RED attempted to serialize it. Values here are intentionally lost on restart. Seeing `'volatile'` as the third argument signals that the value is transient by design.

### Usage Convention

```javascript
// Utility: Initializers — storing a helper function
global.set('utils.formatStatus', function(text) { ... }, 'initializers');

// Any function node — retrieving it
const formatStatus = global.get('utils.formatStatus', 'initializers');

// Default store — no store name needed
global.set('config', configObject);
const config = global.get('config');
```

The store name in `global.get` / `global.set` is self-documenting — seeing `'initializers'` as the third argument tells you exactly where the value was defined.

---

## settings.js Changes Required

Two changes are required after Node-RED is deployed. Edit `/opt/highland/nodered/data/settings.js`:

**1. Credential secret (read from environment variable — never hardcode):**
```javascript
credentialSecret: process.env.NODE_RED_CREDENTIAL_SECRET,
```

**2. Context storage (replace default section with three-store config):**
```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    },
    initializers: {
        module: "memory"
    },
    volatile: {
        module: "memory"
    }
},
```

After editing, apply with `--force-recreate` — a plain restart does not pick up settings.js changes:

```bash
cd /opt/highland
docker compose up -d --force-recreate nodered
```

The credential secret must be added to the `nodered` service environment in `docker-compose.yml`:

```yaml
environment:
  - TZ=America/New_York
  - NODE_RED_CREDENTIAL_SECRET=your-generated-secret-here
```

Generate the secret with: `openssl rand -hex 32`

> **Docker environment variable gotcha:** `docker compose restart` does NOT pick up environment variable changes in docker-compose.yml. Always use `docker compose up -d --force-recreate {service}` when adding or changing environment variables.

---

## Home Assistant Integration

**Package:** `node-red-contrib-home-assistant-websocket`

**Configuration:**
- Base URL: `http://home.local:8123` (or IP address if mDNS fails — see RUNBOOK.md 3.9)
- Access Token: Long-lived access token from HA profile page (stored in Node-RED credentials, not config files)

**Use cases:**

| Action | Method |
|--------|--------|
| Trigger HA backup | Service call: `backup.create` |
| Send notification via Companion App | Service call: `notify.mobile_app_*` |
| Check HA entity state | HA API node or WebSocket |
| React to HA events | HA events node |

Most device control goes through MQTT directly (Z2M, Z-Wave JS). HA integration is primarily for HA-specific features (backups, notifications, entity state that only exists in HA).

> **mDNS inside Docker:** Docker containers cannot resolve `.local` hostnames via mDNS. If the HA server node stays stuck on "connecting", add `extra_hosts` to the Node-RED service in `docker-compose.yml` mapping `home.local` to its actual IP address, then `docker compose up -d --force-recreate nodered`. See `RUNBOOK.md` section 3.9.

---

## External HTTP Calls

Any flow making outbound HTTP requests to external APIs must set a `User-Agent` header. This is standard practice and explicitly required by some APIs (notably NWS, which uses it to contact misbehaving clients).

### Standard User-Agent

Stored in `system.json`, accessed via `global.config.system`:

```javascript
const userAgent = global.get('config')?.system?.http?.user_agent;
msg.headers = {
    'Accept': 'application/json',
    'User-Agent': userAgent
};
```

Do not hardcode the User-Agent string in individual flows. All external HTTP calls read it from config.

### system.json Value

```json
{
  "http": {
    "user_agent": "(Highland-SmartHome, highland@your-domain.example)"
  }
}
```

The NWS API specifically requests User-Agent in `(app_name, contact_email)` format. This value is suitable for all APIs.

**Applies to:** NWS forecast, NWS alerts, Google Calendar, Pirate Weather, Noonlight, and any future external API. No exceptions.

---

## Installed Palette Packages

| Package | Purpose |
|---------|---------|
| `node-red-contrib-home-assistant-websocket` | HA integration |
| `node-red-node-markdown` | Markdown → HTML for Daily Digest |
| `node-red-contrib-schedex` | Sunrise/sunset scheduling |
| `node-red-node-email` | SMTP/IMAP for Daily Digest and mailbox polling |

---

*Last Updated: 2026-03-26*
