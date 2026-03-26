# Node-RED — Startup Sequencing

## The Problem

On Node-RED startup or deploy, three things can conflict:

1. MQTT subscriptions are established and the broker **immediately** delivers all retained messages for those topics
2. The flow's own initialization (Config Loader, flow registration, context restoration from disk) is still in progress
3. `Utility: Initializers` may not have finished populating the `initializers` context store yet

A retained message can arrive and trigger handler logic before `global.config` is loaded, before flow context is restored, and before utility functions are available. There is no guaranteed ordering between inject nodes across flows — Node-RED makes no startup ordering guarantees.

---

## The Two-Condition Gate

The correct solution combines the echo probe pattern with an Initializers readiness check. A flow's gate opens only when **both** conditions are true:

1. **Echo probe returned** — guarantees all retained MQTT messages for this session have been processed
2. **Initializers ready** — guarantees utility functions are available in the `initializers` context store

```
On startup:
  1. Set flow.initialized = false  (gate closed)
  2. Subscribe to all topics (including retained state topics)
  3. Subscribe to highland/status/initializers/ready (non-retained)
  4. Publish probe to highland/command/nodered/init_probe/{flow_name} (non-retained)
  5. Retained messages begin arriving → buffer or discard (gate is closed)
  6. Own probe returns → retained messages are done
     → Check global.get('initializers.ready', 'initializers')
     → If true: both conditions met → open gate immediately
     → If undefined: wait for highland/status/initializers/ready message
  7. highland/status/initializers/ready arrives (non-retained)
     → If probe already returned: both conditions met → open gate
  8. Gate opens → process buffered state → normal operation begins
```

### Probe Topic Convention

```
highland/command/nodered/init_probe/{flow_name}
```

Each flow uses its own probe topic to avoid cross-flow interference.

---

## Why Non-Retained for the Ready Signal

Using a retained message for `highland/status/initializers/ready` introduces a stale session problem — a flow restarting at 09:00:02a would see the retained message from the previous session's 08:00:05a initialization and incorrectly open its gate before the current session's utilities are ready.

Using a **non-retained** message combined with a **global flag** in the `initializers` store solves this cleanly:

- If Initializers runs first → sets flag to `true` → dependent flows check it when their probe returns and open immediately
- If a dependent flow starts first → probe returns, flag is `undefined` → flow waits for the non-retained ready message
- On restart → the `initializers` store clears (memory backend resets) → flag is gone → stale ready state is structurally impossible

---

## Initializers Startup Sequence

```javascript
// Step 1: Mark not ready
global.set('initializers.ready', false, 'initializers');

// Step 2: Populate all utility functions
global.set('utils.formatStatus', function(text) { ... }, 'initializers');
// ... other utilities ...

// Step 3: Mark ready and signal dependent flows (non-retained)
global.set('initializers.ready', true, 'initializers');
msg.topic   = 'highland/status/initializers/ready';
msg.payload = { timestamp: new Date().toISOString() };
return msg;
// → publish non-retained to highland/status/initializers/ready
```

---

## Gate Pattern in Sinks Groups

The gate check belongs in the Sinks group — at the point of ingress — before messages reach any processing logic.

Three states must be handled:

```javascript
// Check degraded first — permanent failure, drop message
if (flow.get('degraded')) {
    node.warn('Flow is degraded — dropping message');
    return null;
}

// Check initialized — temporary, waiting
if (!flow.get('initialized')) {
    const buffer = flow.get('state_buffer') || [];
    buffer.push(msg);
    flow.set('state_buffer', buffer);
    return null;
}

// Gate open — proceed
return msg;
```

Degraded is checked before initialized because a degraded flow also has `initialized = false` — without this ordering, messages would buffer forever.

---

## State vs. Event Handling During Init

**Retained state messages** (from `highland/state/#`) — **buffer these**. They represent current truth and will be needed once the gate opens.

**Point-in-time events** (from `highland/event/#`) — **discard these**. If a real-time event fires during the brief init window it is genuinely gone and cannot be recovered. This is acceptable — the window is very short and events are by definition momentary.

---

## Processing the Buffer

Once the gate opens, process buffered state in order:

```javascript
const buffer = flow.get('state_buffer') || [];
flow.set('state_buffer', []);
for (const bufferedMsg of buffer) {
    node.send(bufferedMsg);
}
```

---

## Two Entry Points, One Handler

Flows that respond to period changes (or any retained state) use two entry points feeding a single handler:

```
highland/state/scheduler/period  ──┐  (retained — arrives during init, buffered,
  (startup recovery path)          │   processed after gate opens)
                                   ├──► period logic handler
highland/event/scheduler/evening ──┘  (real-time — arrives during normal operation,
  (real-time transition path)         gate already open)
```

This is a push model, not polling. The retained state delivers once on subscription; events drive everything thereafter.

---

## Degraded State and Recovery

When the Initializer Latch subflow times out, the calling flow sets `flow.degraded = true` and clears the buffer:

```javascript
flow.set('degraded', true);
flow.set('state_buffer', []);  // No point buffering — gate will never open
node.status({ fill: 'red', shape: 'ring', text: 'Degraded: init timeout' });
// Publish CRITICAL log entry to highland/event/log
```

**Recovery procedure:**

The root cause of a degraded flow is always in `Utility: Initializers`. The degraded state in dependent flows is a symptom, not the cause.

1. Identify and fix the issue in `Utility: Initializers`
2. Deploy `Utility: Initializers`
3. Redeploy the affected flow(s)

Step 3 is required because `flow.get('degraded')` persists in context storage across restarts. Redeploying resets flow context and re-runs the startup inject.

---

## Bootstrapping Limitation

There is an inherent bootstrapping limitation in any event-driven system: **you cannot use infrastructure to report infrastructure failures.**

If both Initializers and MQTT are simultaneously unavailable, Node-RED has no self-reporting mechanism. This is not a design flaw — it is a physical reality.

**Accepted fallbacks when MQTT is unavailable:**
- **Node-RED debug sidebar** — node status (red ring, "Degraded") is visible in the editor regardless of MQTT state
- **Node-RED console log** — `node.error()` and `node.warn()` write to Node-RED's own log, visible via `docker compose logs nodered`
- **Healthchecks.io** — the Health Monitor pings Healthchecks.io via direct HTTP, independently of MQTT

The correct mitigation is to **monitor MQTT health** so you know when this condition exists. See `nodered/HEALTH_MONITORING.md`.

---

## Notes

- The init window is typically well under one second. The gate is a safety net, not a performance concern.
- This pattern applies to every flow that subscribes to retained state topics OR uses utilities from the `initializers` store.
- Config Loader and Initializers do not use the two-condition gate — they are the things being waited for, not the things waiting.
- If MQTT is unavailable on startup, the probe never returns. Flows should have a startup timeout after which they log an error and enter degraded state. The Initializer Latch handles this. See `nodered/SUBFLOWS.md`.

---

*Last Updated: 2026-03-26*
