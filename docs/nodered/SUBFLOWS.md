# Node-RED — Subflows

## Overview

Subflows in Highland are used sparingly — only for truly reusable components that are used identically across many flows. Two subflows are currently defined:

- **Initializer Latch** — gates flow execution until `Utility: Initializers` is ready
- **Connection Gate** — guards message flow based on live connection state

Both follow the same interface convention: descriptive environment variables (labelled in the Node-RED UI), consistent status indicators, and output routing semantics you can reason about at a glance.

---

## Initializer Latch

### Purpose

Gates flow execution until `Utility: Initializers` has populated the `initializers` context store. Drop this into any flow's startup sequencing group.

### Interface

- **1 input** — any message from any source (inject, MQTT, HTTP, etc.)
- **Output 1 (OK)** — messages pass through once condition is met; buffered messages drain in order
- **Output 2 (TIMEOUT)** — single signal message when condition is never met within retry window

### Environment Variables

| Variable | UI Label | Default | Purpose |
|----------|----------|---------|---------|
| `RETRY_INTERVAL_MS` | Retry Interval (ms) | 250 | Delay between retries |
| `MAX_RETRIES` | Max Retries | 20 | Maximum retry attempts before timeout |
| `CONTEXT_PREFIX` | Scope | `''` | Prefix for flow context keys — set when multiple instances exist on the same flow tab |

Total timeout at defaults: 250ms × 20 = 5 seconds.

### Internal Behavior

1. Every incoming message is buffered immediately
2. On first message, starts polling `global.get('initializers.ready', 'initializers')`
3. If flag is `true` → sets `flow.initialized = true`, clears `flow.degraded`, drains buffer via Output 1
4. If max retries exceeded → sets `flow.degraded = true`, discards buffer, emits signal via Output 2
5. If already initialized → passes message through Output 1 directly (no buffering)
6. If already degraded → drops message silently

### Calling Flow Responsibilities

- Wire Output 1 to normal processing logic — messages arrive as if nothing happened
- Wire Output 2 to error handler — log CRITICAL, set node status, clear buffer

### Degraded Recovery

`flow.degraded` persists in context storage across restarts. Recovery requires:

1. Fix the issue in `Utility: Initializers`
2. Deploy `Utility: Initializers`
3. Redeploy the affected flow(s) — this resets flow context and gives the latch a fresh start

See `nodered/STARTUP_SEQUENCING.md` for full degraded state and recovery details.

---

## Connection Gate

### Purpose

Guards message flow based on the live state of a connection. Used wherever a flow needs to deliver a message via a connection-dependent path — routing to a fallback or holding briefly for recovery rather than blindly attempting delivery when a connection is down.

Distinct from the Initializer Latch — the latch is a one-time startup concern; the gate handles repeated up/down transitions during normal operation.

### Interface

- **1 input** — any message
- **Output 1 (Pass)** — connection is up; message delivered here, either immediately or after recovery
- **Output 2 (Fallback)** — connection is down and not recovered within retention window; caller handles alternative delivery or discard

### Environment Variables

| Variable | UI Label | Purpose | Default |
|----------|----------|---------|---------|
| `CONNECTION_TYPE` | Connection | Which connection to check: `home_assistant`, `mqtt` | — |
| `RETENTION_MS` | Retention (ms) | How long to poll for recovery before routing to Output 2. 0 = no retention. | `0` |
| `CONTEXT_PREFIX` | Scope | Prefix for flow context keys — required when multiple instances on the same flow tab | `''` |

### Behavior

| Scenario | Result |
|----------|--------|
| Connection up | Output 1 immediately |
| Connection down, `RETENTION_MS` = 0 | Output 2 immediately |
| Connection down, `RETENTION_MS` > 0, recovers within window | Output 1 when recovery detected |
| Connection down, `RETENTION_MS` > 0, window expires | Output 2 after window |

**Output 1 always means connected state** — either the connection was up at arrival, or it recovered within the retention window.

**Output 2 always means unrecovered down state** — guaranteed to exit one output or the other; no silent drops.

**Latest-only retention** — if a new message arrives while a retention poll is in progress, the existing poll is cancelled and a new one starts for the latest message. Earlier messages are discarded.

**Poll interval** is internalized at 500ms — not configurable per instance. Callers tune behavior via `RETENTION_MS` only.

### Node Status Values

| Status | Meaning |
|--------|---------|
| Green dot — Passed | Gate open, Output 1 immediately |
| Red dot — Fallback | Gate closed, no retention, Output 2 immediately |
| Yellow ring — Waiting... | Gate closed, polling for recovery |
| Green ring — Recovered | Recovered within window, Output 1 |
| Red ring — Expired | Window elapsed without recovery, Output 2 |

Dot = immediate decision; ring = delayed decision.

### Usage Example

```
Notification msg → Connection Gate (CONNECTION_TYPE=home_assistant, RETENTION_MS=120000)
                        ↓ Output 1                    ↓ Output 2
                   HA Companion delivery          Pushover delivery
```

### Internal Structure

**Evaluate Gate** — single function node with two outputs. Reads `global.connections.{CONNECTION_TYPE}`, applies retention policy, sets `node.status()`.

**Status Monitor** — `status` node scoped to `Evaluate Gate`, wired to `Set Status` function node, surfaces status onto the subflow instance in the parent flow without requiring a third output.

### Flow Context Keys

| Key | Store | Value |
|-----|-------|-------|
| `{PREFIX}retention_poll` | volatile | interval handle or null |

### Notes

- Output 2 being unwired is valid — messages that would route to Output 2 are silently discarded (caller has made a deliberate choice that delivery is best-effort)
- `CONTEXT_PREFIX` env var is labelled **Scope** in the Node-RED UI — consistent with Initializer Latch convention
- Timer handles stored in `volatile` context store — non-serializable, intentionally lost on restart

---

*Last Updated: 2026-03-26*
