# Email Ingress — Design & Architecture

## Scope

The `Utility: Email Ingress` flow owns the **email intake surface** for the Highland system. It connects to the household Gmail account, watches for new messages via IMAP IDLE, normalizes them into a common schema, and publishes them to MQTT for consumption by downstream flows.

It is deliberately content-agnostic. It knows nothing about USPS, calendar invites, warranty notifications, or any other email source's meaning. Its only concerns are: hold a reliable IMAP connection, detect new mail, identify which consumer (if any) should receive it, publish in a consistent shape, and manage message lifecycle.

This is the email equivalent of `Utility: Device Registry` — a single owner for a cross-cutting concern so consuming flows don't each reimplement the same plumbing.

---

## Why a Dedicated Flow

Every email-driven subsystem would otherwise duplicate:

- IMAP connection management (credentials, TLS, reconnect handling, IDLE re-arming)
- Gmail rate-limit awareness (15 simultaneous IMAP connections per account)
- `Message-Id` deduplication across restarts
- Gmail label convention enforcement (`Highland/*` namespace)
- IMAP health monitoring and heartbeat
- App password revocation detection
- Processed-email retention and purge

Owning all of that in one place makes each consumer trivially simple — a consumer's entire job is "subscribe to my label's topic, parse content, publish semantic state." No IMAP code anywhere else in the system.

---

## Gmail Account

The flow connects to a single dedicated household Gmail account (referenced in `secrets.json` as `gmail.account`). The same account is used for Google Calendar and is the registered mailbox for USPS Informed Delivery. See `subsystems/DELIVERIES.md § Gmail Account` for rationale on the dedicated-account choice.

### Authentication

IMAP access requires 2FA on the account (Google deprecated "less secure app access" in 2022):

1. 2FA is enabled on the account.
2. A single app password is generated, scoped to "Mail," and named **"Node-RED Highland IMAP"** — an obvious name so a human reviewing the account's security page doesn't mistake it for suspicious activity.
3. The app password is stored in `secrets.json` under `gmail.app_password`.
4. The `Utility: Config Loader` flow populates `global.config.secrets.gmail` at startup, per the established project pattern.

App passwords are revocable independently of the account's primary credential.

---

## Architecture — Single Connection, All Mail Watch, Label Dispatch

### Why not one watcher per Highland folder?

Two options were considered and rejected:

**Per-folder IMAP connections** (one connection IDLE'd on each `Highland/*` folder) is simpler to reason about but burns connections against Gmail's 15-per-account ceiling. With multiple personal devices (phone, tablet, watch, computers) already consuming connections, leaving headroom matters.

**IMAP NOTIFY (RFC 5465) on multiple folders over a single connection** is the elegant IMAP-native answer. It was tested against Gmail on 2026-04-22 and found to be **not supported** — Gmail's IMAP server does not advertise `NOTIFY` in its post-auth capability list. Dead end.

### Why All Mail + X-GM-LABELS?

Gmail's `[Gmail]/All Mail` is a virtual folder containing every message in the account regardless of label. Combined with Gmail's `X-GM-EXT-1` IMAP extension, we can fetch `X-GM-LABELS` on every message — telling us, for each new mail, which user labels it has.

This gives us:
- **One IMAP connection** — low connection-budget impact
- **No dependency on a specific feature folder** — Informed Delivery ever changing or going away doesn't break the architecture
- **Auto-discovery of new consumers** — any mail labeled `Highland/<new_thing>` is automatically seen
- **Clean dispatch** — label hierarchy maps directly to MQTT topic hierarchy

Cost: the flow sees every email that hits the account, not just Highland-labeled ones. For each, it fetches envelope + labels, inspects the labels, and silently drops anything without a `Highland/*` label. Fetches are cheap; the "firehose" is trivial at personal-account volumes.

### Validated on Gmail (2026-04-22)

Confirmed working against the household Gmail account:
- IDLE on `[Gmail]/All Mail` fires `exists` events on new mail arrivals
- Gmail-side latency is typically 4–12 seconds between mail landing and event firing (Gmail's internal message-store propagation, not under our control)
- `X-GM-LABELS` fetch returns a mixed array of Gmail system labels (prefixed `\`, e.g. `\Inbox`, `\Important`, `\Sent`) and user labels (plain strings, e.g. `Highland/Deliveries/USPS`)
- Label-prefix filtering cleanly separates Highland-routable mail from everything else

---

## Label Namespace Convention

All automation-relevant email lives under a `Highland/` label prefix in Gmail, organized hierarchically by consuming flow. This establishes a clear boundary between "robot-managed mail" and "human-managed mail," and makes the flow that owns each email visible in the label itself.

```
Highland/
├── Deliveries/
│   ├── USPS                 (USPS Informed Delivery + any future USPS emails)
│   ├── FedEx                (future)
│   ├── UPS                  (future)
│   └── Amazon               (future)
└── Processed                (archival destination after ACK)
```

**Conventions:**

- Each consuming flow gets a namespace under `Highland/` (e.g., `Highland/Deliveries/`).
- Within each namespace, labels identify the **source** of the email (sender/carrier/service), not a product or feature name. `USPS`, not `Informed Delivery` — this survives vendors renaming or replacing product lines.
- A single flat `Highland/Processed` label serves as the archival destination for all consumed mail, regardless of origin namespace.
- Gmail filters (configured manually, once per source) apply the appropriate label to incoming mail and can optionally skip the inbox.
- Humans using the account for unrelated mail interact with regular Gmail; Highland labels are automation-facing.

**Adding a new source to an existing consumer** (e.g., adding FedEx to Deliveries):
1. Create a Gmail filter that applies `Highland/Deliveries/FedEx` to matching incoming mail
2. Add the new topic to the consumer flow's subscription (or rely on its existing wildcard subscription)

**Adding a new consumer entirely** (e.g., Warranties):
1. Decide the namespace name (e.g., `Highland/Warranties/`)
2. Create Gmail filters for each source under that namespace
3. Build the consumer flow, subscribed to `highland/event/email/<namespace>/+/received`

No ingress code or config changes required — the flow auto-discovers new labels as mail arrives carrying them.

### Label-to-topic derivation

The Gmail label is transformed into a consumer-facing MQTT topic suffix: strip the `Highland/` prefix, lowercase, replace whitespace with underscores. **Slashes in the label are preserved as MQTT topic separators.** This aligns Gmail's hierarchical label structure with MQTT's hierarchical topic structure, making wildcard subscriptions natural.

| Gmail Label | Derived Topic Suffix | Full MQTT Topic |
|-------------|---------------------|-----------------|
| `Highland/Deliveries/USPS` | `deliveries/usps` | `highland/event/email/deliveries/usps/received` |
| `Highland/Deliveries/FedEx` | `deliveries/fedex` | `highland/event/email/deliveries/fedex/received` |
| `Highland/Warranties/HVAC` | `warranties/hvac` | `highland/event/email/warranties/hvac/received` |

**Consumer subscription patterns:**

- Specific source: `highland/event/email/deliveries/usps/received`
- All sources for a flow: `highland/event/email/deliveries/+/received`
- Every Highland email event: `highland/event/email/#` (diagnostic only; not a typical production subscription)

---

## Message Lifecycle

```
[new message arrives in Gmail, picks up one or more labels]
         │
         ▼
[IDLE on All Mail fires 'exists' event]
         │
         ▼
[Ingress fetches new UID(s) with X-GM-LABELS]
         │
         ├── No Highland/* label? → silently drop
         │
         ▼
[Dedup check — Message-Id seen before?]
         │
         ├── Yes → silently drop
         │
         ▼
[Normalize → publish to MQTT]
         │
         ▼
[Consumer processes, publishes ACK]
         │
         ▼
[Ingress archives via Gmail label operations]
```

### Archival via Gmail label operations

In Gmail's IMAP model, there are no "folders" in the POP3 sense — everything is a label. Moving a message to a folder is equivalent to adding the destination label and removing the source label. Our archival operation after ACK:

1. Apply `Highland/Processed` label to the message
2. Remove the source `Highland/<namespace>/<source>` label from the message

After archival, the message still exists in All Mail (and in any Gmail system locations like `[Gmail]/All Mail` or `[Gmail]/Sent Mail`), but no longer carries a consumer-facing Highland label. It's out of scope for future dispatch but available for audit.

Implementation uses `X-GM-EXT-1`'s IMAP extensions for label modification. imapflow does not expose dedicated label methods; instead, labels are managed through the standard flag methods with `useLabels: true` passed in the options:

```javascript
// Add a Gmail label
await client.messageFlagsAdd(uid, ['Highland/Processed'], { uid: true, useLabels: true });

// Remove a Gmail label
await client.messageFlagsRemove(uid, ['Highland/Deliveries/USPS'], { uid: true, useLabels: true });
```

This is a Gmail-specific operation — it requires `X-GM-EXT-1` capability, which Gmail's IMAP server advertises.

### Key behaviors

- **State lives in Gmail labels, not in local context.** Flow-side tracking is just for dedup and in-flight ACK management. Gmail's label state is the authoritative record of what has been processed.
- **Idempotent.** A rolling record of recently-published `Message-Id` values in flow context (default store, disk-backed) prevents duplicate publishes if the same mail is seen twice. TTL: 24 hours, enforced by scheduled sweep.
- **ACK-gated archival.** After publishing, ingress waits for `highland/ack/email` with the matching `message_id` before archiving. This prevents loss if a consumer is down or restarting at publish time.
- **Fallback archival TTL.** If no ACK arrives within a configured window (default 24 hours), the message is archived anyway with a warning log. Prevents indefinite accumulation of unacked state when a consumer is genuinely broken or unconfigured.
- **Retention purge.** Messages carrying `Highland/Processed` older than a configurable retention (default 14 days) are moved to `[Gmail]/Trash` by a scheduled sweep. Gmail's built-in 30-day Trash auto-purge finalizes permanent deletion. This Trash-based indirection provides a grace window for recovery if the sweeper misbehaves. This is the only destructive operation the flow performs.

---

## MQTT Topics

### Events (not retained)

One topic per derived label, with MQTT topic hierarchy mirroring Gmail label hierarchy. Topics exist only as consumers arrive — there is no static registry.

| Topic | Fires When |
|-------|-----------|
| `highland/event/email/<namespace>/<source>/received` | A new email with the corresponding Highland label is parsed and ready for consumption |

**Currently in use:**

| Gmail Label | Topic | Consumer |
|-------------|-------|----------|
| `Highland/Deliveries/USPS` | `highland/event/email/deliveries/usps/received` | `Utility: Deliveries` |

**Planned labels (as consumers come online):** captured in `AUTOMATION_BACKLOG.md`.

### Payload Schema

```json
{
  "message_id": "<CAxxxx@mail.gmail.com>",
  "label": "deliveries/usps",
  "gmail_label": "Highland/Deliveries/USPS",
  "from": "USPSInformeddelivery@email.informeddelivery.usps.com",
  "from_name": "USPS Informed Delivery",
  "to": "<household gmail>",
  "subject": "Your Daily Digest",
  "received_at": "2026-04-22T07:15:23-04:00",
  "body_text": "...",
  "body_html": "...",
  "attachment_count": 3
}
```

**Design notes:**

- `message_id` is the canonical identifier. Consumers use it for their own deduplication and must echo it in the ACK.
- `label` is the derived topic-suffix form (including slashes for hierarchy). `gmail_label` is the original Gmail label for reference/debugging.
- `body_text` and `body_html` are both included when available. Consumers pick whichever they prefer.
- **Attachments are not inlined.** `attachment_count` is metadata only. Binary attachments are deliberately excluded from the payload to keep MQTT traffic reasonable. No current consumer needs attachment bytes; if one ever does, a separate fetch-by-message-id mechanism will be added.
- HTML bodies for some senders (Informed Delivery digests with embedded scan references) can be large — several hundred KB. Mosquitto handles this fine at our volume (single-digit emails per day per label).

### ACK

Consumers publish an ACK after successful processing:

**`highland/ack/email`** — not retained

```json
{
  "message_id": "<CAxxxx@mail.gmail.com>",
  "consumer": "utility_deliveries",
  "processed_at": "2026-04-22T07:15:45-04:00",
  "status": "ok"
}
```

`status` values: `"ok"` | `"rejected"` | `"parse_error"`.

**On `status: "ok"`:** ingress archives the message (apply `Highland/Processed`, remove the source Highland label). Normal path.

**On `status: "rejected"`:** consumer saw the message but decided it wasn't theirs (e.g., sender mismatch for a shared label, or subject pattern the consumer doesn't handle). Ingress still archives — it has been "seen" — but tags the log entry. Current design assumes one consumer per label.

**On `status: "parse_error"`:** consumer tried to process but failed. Ingress leaves the message in its source label for retry, logs a warning, and publishes a notification on repeated failures (threshold configurable, default 3 attempts). This is the "something is broken" path — a parser bug or an unexpected email variant shouldn't silently vanish into `Highland/Processed`.

### Health

**Deferred — no current consumer.** A retained `highland/status/email_ingress/health` topic was originally planned for external uptime monitoring and dashboard surfacing. On 2026-04-22, this was deferred:

- External uptime monitoring in Highland goes through Healthchecks.io via HTTP pings, not MQTT subscription — a health topic doesn't feed that pipeline.
- No dashboard consumer identified. Email Ingress is infrastructure; human-facing state belongs to consuming flows (`sensor.mail_status`, etc.).
- No cross-flow coordinator needs ingress health as an input — consumers already depend on ingress implicitly via MQTT event subscriptions.

The operationally-meaningful failure mode (app password revocation, persistent auth failure) is better served by direct notification to the operator than by a retained status topic. That path is tracked under the Open Question about ingress failure notifications.

If a real consumer for this topic emerges later (HA sensor, external monitoring integration), the health publisher can be built at that point, sized to the actual consumer's needs.

---

## IMAP Connection Pattern

The flow holds a single long-lived IMAP connection via the `imapflow` npm package. This is the first flow in the Highland project to manage a persistent third-party library connection from a function node; the pattern is worth documenting explicitly.

### Module import

Per `nodered/ENVIRONMENT.md § Named Exports and the Destructure Pattern`, every function node that uses `imapflow` or `mailparser` declares them in its Setup tab with `Pkg` suffixes:

| Module name | Import as |
|---|---|
| `imapflow` | `imapflowPkg` |
| `mailparser` | `mailparserPkg` |

And destructures at the top of the function body:

```javascript
const { ImapFlow } = imapflowPkg;
const { simpleParser } = mailparserPkg;
```

### Context store usage

All internal state lives in the flow's own context — `flow.*`, not `global.*`. The flow's internal state is nobody else's business; consumers interact exclusively via MQTT. Only `global.config.secrets.gmail` (credentials from the Config Loader) and `global.utils.formatStatus` (shared status helper) are read from the global scope. Everything else is `flow.*`.

The IMAP client handle and the mailbox lock are stored in the `volatile` context store — in-memory, non-serializable, cleared on restart. Per `nodered/ENVIRONMENT.md § Context Storage`, `volatile` is the correct store for open connection handles. Note that `volatile` is a *store*, orthogonal to the `flow` vs `global` *scope* — `flow.get('key', 'volatile')` is valid and correct.

**Keys in use:**

| Key | Scope | Store | Purpose |
|-----|-------|-------|---------|
| `imap_client` | flow | volatile | The `ImapFlow` instance itself |
| `imap_connected_at` | flow | default | Timestamp of last successful connect (disk-persisted for health reporting across restarts) |
| `imap_last_error` | flow | default | Most recent connection error message, for degraded-status reporting |
| `imap_idle_lock` | flow | volatile | The mailbox lock held while IDLE is active |
| `imap_idle_running` | flow | volatile | Boolean guard preventing double-start of IDLE |
| `imap_last_uid_all_mail` | flow | default | High-water UID for All Mail; survives restarts to avoid backlog reprocessing |
| `imap_pending_ack` | flow | default | Map of `message_id` → `{ uid, gmail_label, published_at }` for in-flight ACK tracking |
| `imap_seen_<message_id>` | flow | default | Dedup entries with `{ seen_at }` timestamp for 24h TTL enforcement |
| `imap_consecutive_failures` | flow | default | Counter of consecutive connection failures since last success. Reset on successful connect. Drives notification threshold. |
| `imap_notification_sent` | flow | default | `correlation_id` of currently-active failure notification, or `null`. Used to avoid re-firing and to clear on recovery. |
| `imap_last_failure_at` | flow | default | Timestamp of most recent connection failure (ms since epoch) |
| `imap_last_failure_tier` | flow | default | Tier of most recent notification (`"auth"` or `"connection"`), or `null` |
| `retention_sweep_running` | flow | volatile | Re-entrancy guard for the Retention Sweeper. Prevents stacked async sweeps when the CronPlus is manually triggered rapidly. Cleared on Node-RED restart (as appropriate for a mid-operation guard). |

### Connection lifecycle events

`imapflow` emits `error` and `close` events when the connection drops. Handlers guard against races where a stale handler clobbers a newer connection reference:

```javascript
client.on('close', () => {
    if (flow.get('imap_client', 'volatile') === client) {
        flow.set('imap_client', null, 'volatile');
    }
});
```

"Only clear the handle if it's still pointing at me."

### Deploy cleanup

The IMAP-connection-managing function node uses the `On Stop` tab to gracefully log out on Node-RED deploy/restart, preventing connection leaks:

```javascript
// On Stop
const client = flow.get('imap_client', 'volatile');
if (client) {
    client.logout().catch(() => {});
    flow.set('imap_client', null, 'volatile');
}
```

### Async Function Node Hygiene

Several function nodes in this flow launch async work (`(async () => {...})()`) to invoke imapflow operations. Three related hazards apply:

1. **Connection contention.** imapflow's mailbox lock serializes operations on the underlying connection — not per-mailbox. Only one mailbox can be "selected" at a time on an IMAP connection (a fundamental protocol constraint). If one function holds the connection's mailbox lock indefinitely (such as the Watcher during IDLE), any other function waiting for a lock on **any** mailbox on the same connection will queue forever, never erroring, never resolving.
2. **Stacking** — if the function fires rapidly (manual CronPlus triggers, debouncing bugs, etc.), multiple pending async IIFEs queue up. Each one races to acquire resources (mailbox locks, connection calls).
3. **Shutdown noise** — on Node-RED redeploy/restart, the connection is torn down while pending async work is still in flight. Those operations fail with `"Connection not available"` or similar, which the catch block would normally log as errors. These aren't real errors — they're the expected consequence of shutting down.

**Connection contention resolution.** The main flow's client is permanently busy IDLE-locked on `[Gmail]/All Mail`. Any sweeper or background operation that needs to interact with a *different* mailbox on the main client will block forever (discovered on 2026-04-23 — the Retention Sweeper initially tried to acquire a lock on `Highland/Processed` on the main client and silently enqueued one orphan lock request per firing, accumulating until restart when they all rejected simultaneously with `"Connection not available"`).

The resolution is to **open a dedicated short-lived connection** for such operations. The connection lives only for the duration of the work (open → act → logout) and does not interfere with the main flow's client. Gmail's 15-connection-per-account budget accommodates this fine; our use of one sweep connection for a few seconds per day is negligible.

Currently applied in: Retention Sweeper (connects, searches, moves to Trash, logs out per run).

**Re-entrancy guards.** For functions where stacking is possible (currently: the Retention Sweeper via manual CronPlus triggering), set a `<name>_running` flag in `volatile` context at entry and clear it in the `finally` block:

```javascript
if (flow.get('retention_sweep_running', 'volatile')) {
    node.warn('Already running; skipping this trigger');
    return null;
}
flow.set('retention_sweep_running', true, 'volatile');

(async () => {
    try {
        // ... work ...
    } finally {
        flow.set('retention_sweep_running', false, 'volatile');
    }
})();
```

**Shutdown-error silencing.** Catch blocks in async IMAP operations should distinguish shutdown-triggered errors from real failures, logging only the latter:

```javascript
} catch (err) {
    const isShutdownError = err.message && (
        err.message.includes('Connection not available') ||
        err.message.includes('Connection closed')
    );
    if (isShutdownError) {
        node.status({ fill: 'grey', shape: 'ring', text: formatStatus('Aborted (connection closed)') });
    } else {
        node.warn(`Error: ${err.message}`);
        // ... real error handling ...
    }
}
```

Currently applied in: Retention Sweeper. Consider extending to `Publish New Message` and `Archive Message` if shutdown noise from those becomes operationally bothersome.

---

## Operator Notifications

When the flow detects a failure condition that requires human action, it publishes a notification via the standard Highland notification contract (`highland/event/notify`). When the condition clears, it publishes a corresponding clear command (`highland/command/notify/clear`). Consumers receive notifications via whatever channels `Utility: Notifications` is configured to route through — typically HA Companion App.

This replaces the originally-planned retained health topic (see § Health). Direct notification is the right signal for the operationally-meaningful failure modes; a retained status topic with no consumer would have been infrastructure-in-search-of-a-consumer.

### Failure tiers

Four tiers of failure were considered during design. Only Tiers 1 and 2 are currently implemented; Tiers 3 and 4 are deferred pending need.

| Tier | Implemented | Condition | Rationale |
|------|:-----------:|-----------|-----------|
| 1 | ✅ | Persistent IMAP auth failure | App password revoked/invalidated. Mail stops flowing until operator regenerates and updates `secrets.json`. Blocking failure. |
| 2 | ✅ | Persistent non-auth connection failure | Network, DNS, Gmail outage, TLS issues. Same end state as Tier 1 (no mail flows) but different operator action. |
| 3 | ❌ deferred | Repeated `parse_error` ACKs from a consumer | Consumer parser bug or email format change. System is silently failing to do its job, but ingress itself is healthy. Requires per-consumer counters with windowing — not yet worth the complexity. |
| 4 | ❌ deferred | Stale pending-ACK accumulation | Indicates a consumer is down or no consumer exists for a label. Sweepers already force-archive with warning log; additional notification would be double-reporting. |

### Trigger threshold

Connection failures are counted consecutively in `imap_consecutive_failures`. Notification fires after **3 consecutive failures**, which filters out transient blips while still surfacing real failures within minutes (connection retries are frequent; 3 failures typically means the problem is not self-correcting).

The counter resets to 0 on any successful connection (reused or freshly opened). A notification is only fired once per failure episode — if the flow is already in the "failing and notified" state, subsequent failures increment the counter but do not re-fire.

### Tier discrimination

Auth failures vs. other connection failures are discriminated by string-matching the imapflow error message:

```javascript
function isAuthError(errMessage) {
    if (!errMessage) return false;
    const msg = errMessage.toLowerCase();
    return msg.includes('authentication') ||
           msg.includes('invalid credentials') ||
           msg.includes('auth failed') ||
           msg.includes('login failed');
}
```

Imperfect — a novel error message could be miscategorized — but good enough to differentiate "regenerate your app password" from "check your network" in the notification text. Both paths share the same `notification_id`, so the underlying subscription configuration (targets, severity) is identical; only the title and message differ.

### Notification payloads

**Auth failure notification:**

```json
{
  "notification_id": "email_ingress.imap_failure",
  "timestamp": "2026-04-23T10:15:00Z",
  "source": "email_ingress",
  "title": "Email Ingress: IMAP authentication failed",
  "message": "Gmail app password appears to be invalid. Regenerate in Google Account security settings and update secrets.json. Error: <err message>",
  "correlation_id": "email_ingress_imap_failure_<timestamp>",
  "sticky": true
}
```

**Connection failure notification:**

```json
{
  "notification_id": "email_ingress.imap_failure",
  "timestamp": "2026-04-23T10:15:00Z",
  "source": "email_ingress",
  "title": "Email Ingress: IMAP connection failing",
  "message": "Cannot establish connection to Gmail IMAP after N attempts. Mail is not flowing. Error: <err message>",
  "correlation_id": "email_ingress_imap_failure_<timestamp>",
  "sticky": true
}
```

**Clear command** (published on first successful connection after a notification was active):

```json
{
  "notification_id": "email_ingress.imap_failure",
  "correlation_id": "<same correlation_id as the fired notification>"
}
```

### Severity rationale

`high`, not `critical`. Email Ingress is infrastructure for ancillary subsystems (Deliveries today, more in the future). None of its current or near-term consumers are life-safety, so waking the operator at 3am isn't warranted. `high` still prompts for attention on normal devices but respects an explicit Do Not Disturb window — the mail can wait until morning.

Re-evaluate if email ingress ever becomes a dependency of a life-safety system.

### Recovery semantics

On successful connection after a notification was active:
1. Publish `highland/command/notify/clear` with the stored `correlation_id`
2. Reset `imap_notification_sent`, `imap_consecutive_failures`, `imap_last_failure_tier` to their default states
3. No "recovery" notification is sent — silence is the recovery signal

Rationale: if the operator just fixed a problem, they don't need to be told it's fixed. If the flow self-recovered (transient issue), notifying would be unnecessary interruption.

### Subscription config

Added to `notifications.json`:

```json
"email_ingress.imap_failure": {
    "targets": ["people.joseph.ha_companion"],
    "severity": "high"
}
```

Change target routing by editing this subscription; no flow code changes required.

---

## Flow Outline — `Utility: Email Ingress`

Per `nodered/OVERVIEW.md` conventions: groups are the primary organizing unit; link nodes between groups; no node has more than two outputs.

**Group 1 — Sinks**
- `inject` — Startup (once, ~3s delay after deploy)
- CronPlus — ACK TTL sweeper (every 10min)
- CronPlus — Dedup TTL sweeper (every 1h)
- CronPlus — Retention sweeper (daily)
- `mqtt in` — `highland/ack/email`
- `mqtt in` — `highland/command/config/reload/secrets` (triggers disconnect + reconnect)
- Each sink link-outs to its respective downstream group

**Group 2 — Connection**
- `Ensure Connection` function node: returns existing client if healthy, else opens new one. Stores handle in `volatile`. Wires error/close handlers with identity guards. Tracks consecutive connection failures and emits operator notifications on threshold crossings (see § Operator Notifications).
- Two outputs: output 1 → successful-connection path (to Watcher). Output 2 → notification events / clear commands (to MQTT out, topic set by message).
- `On Stop` cleanup: graceful logout

**Group 3 — Watcher**
- `Start All Mail Watcher` function node: takes mailbox lock on `[Gmail]/All Mail`, records baseline UID, subscribes to `exists` events, fetches new UIDs with `X-GM-LABELS` on each event
- Emits one message per new mail (including those without Highland labels; filtering happens downstream)
- `On Stop` cleanup: release lock

**Group 4 — Label Filter**
- Function node: inspects `X-GM-LABELS` array, finds first `Highland/*` label (if any)
- Drops messages without a Highland label
- Derives the topic-suffix form (e.g., `deliveries/usps`) preserving slashes as MQTT hierarchy
- Attaches derived metadata to msg

**Group 5 — Dedup**
- Function node: checks `imap_seen_<message_id>` context key
- Drops duplicates, logs at debug level
- On new: sets the dedup key with current timestamp

**Group 6 — Normalize**
- Function node: fetches full message source with `mailparser`'s `simpleParser`
- Builds the standard payload schema
- Records in `imap_pending_ack` context

**Group 7 — Publish**
- `mqtt out` to `highland/event/email/<derived_label>/received`
- Non-retained, QoS 1

**Group 8 — ACK Handler**
- Function node: looks up `message_id` in `imap_pending_ack`
- On `ok`/`rejected`: dispatch to Label Archival
- On `parse_error`: increment retry counter, log, notify on threshold
- Removes the pending-ack entry

**Group 9 — Label Archival**
- Function node: applies `Highland/Processed`, removes the source `Highland/*` label on the UID
- Uses `client.messageFlagsAdd()` and `client.messageFlagsRemove()` with `useLabels: true` (X-GM-EXT-1 extension)

**Group 10 — Lifecycle Sweepers**
- ACK TTL sweeper (every 10min): scans `imap_pending_ack` for stale entries, force-archives via Archive Message, logs warning
- Dedup TTL sweeper (every 1h): enumerates `imap_seen_*` keys, removes entries older than dedup TTL
- Retention sweeper (daily, 3am): searches `Highland/Processed` for mail older than retention, moves to `[Gmail]/Trash` for Gmail's auto-purge to finalize

**Group 11 — Error Handling**
- Standard project pattern: `catch` node + log/notify function node

---

## Configuration

Captured in `config/email_ingress.json` (pending final decision on file structure per `DELIVERIES.md` open question).

```json
{
  "imap": {
    "server": "imap.gmail.com",
    "port": 993,
    "tls": true
  },
  "watch_folder": "[Gmail]/All Mail",
  "label_prefix": "Highland/",
  "processed_label": "Highland/Processed",
  "ack_timeout_hours": 24,
  "processed_retention_days": 14,
  "parse_error_notification_threshold": 3,
  "dedup_ttl_hours": 24
}
```

Notably absent compared to earlier drafts: no `folders` array. The architecture is single-folder (All Mail) with dispatch-by-label.

---

## Startup Behavior

Per `nodered/STARTUP_SEQUENCING.md` patterns:

- The flow does **not** immediately process all mail in All Mail at startup. That would republish potentially years of backlog.
- The first thing the watcher does is read `imap_last_uid_all_mail` from persistent context. If present, it fetches any UIDs strictly greater than that value (catching up any mail that arrived while the flow was down). If absent, it records `uidNext - 1` as the baseline and begins watching fresh from that point.
- Existing mail with `Highland/*` labels that predates the flow's first run stays in place, untouched. Manual reprocessing, if ever needed, is a separate utility — not automatic startup behavior. (Likely implemented later as a `highland/command/email_ingress/reprocess` topic accepting a UID range.)

---

## Open Questions

- [ ] **Config file location.** `email_ingress.json` standalone vs folded into a broader file. Same question surfaced in `DELIVERIES.md`; settle both together.
- [ ] **Reprocessing mechanism.** A `highland/command/email_ingress/reprocess` topic for manual reprocessing of historical mail. Primary use case: filter criteria drift — if a Gmail filter stops matching (e.g., USPS changes sender domain), messages land unlabeled in the Inbox. After updating the filter and manually labeling the backlog, a reprocess command would flow those messages through the normal pipeline. Secondary use cases: operator-initiated reprocessing after parser fixes, or deliberate reclassification. This was explicitly considered vs. ambient label-change detection (via IMAP `flags` events) on 2026-04-22 — explicit reprocess was chosen for lower complexity, better signal-to-noise, and cleaner semantics. Payload shape TBD (candidates: `{uid_range}`, `{message_ids}`, `{label, since}`). Priority elevated from "future" to near-term.
- [ ] **Notification routing for ingress failures.** ~~App-password revocation or IMAP auth failure needs to reach a human.~~ **Resolved 2026-04-23** — see § Operator Notifications. Persistent IMAP failures (auth or connection) fire a `high`-severity notification via `notification_id: email_ingress.imap_failure` after 3 consecutive failures. Auth and non-auth failures share a notification subscription but use different title/message strings based on error heuristics. Notification clears automatically on successful reconnect. Tier 3 (per-consumer parse_error tracking) and Tier 4 (stale pending-ACK accumulation) deferred — see § Operator Notifications.
- [ ] **Attachment fetch mechanism.** If a future consumer ever needs attachment bytes, a fetch-by-message-id pattern is the intended approach — but exact topic shape is deferred until a concrete consumer drives it.
- [ ] **Rejection semantics if multiple consumers share a label.** Current design assumes one consumer per label. If that assumption ever breaks, rework needed.
- [ ] **Gmail filter configuration tracking.** The manually-configured filters aren't version-controlled. Consider capturing them in a reference doc as they proliferate, so a rebuild isn't an archaeological dig through Gmail settings.
- [ ] **Health topic — deferred.** The retained `highland/status/email_ingress/health` topic described in earlier drafts was deferred on 2026-04-22 in favor of direct notifications for the failure modes that actually matter (see above). Reconsider if a specific consumer emerges (HA sensor, external dashboard, cross-flow coordination).

---

*Last Updated: 2026-04-23*
