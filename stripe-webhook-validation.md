# Stripe Webhook Validation Guide

A line-by-line manual validation of the durable webhook architecture against your local development
environment, using the Stripe CLI.

This guide assumes you have **never used the Stripe CLI before**. Every command is written out.
Where a command needs a value only you have (an event ID, an order ID), the guide says exactly where
to find it.

**Time required:** about 60–90 minutes for the full pass. Sections 1–6 are setup; 7–14 are the
actual validation and can be done in any order once setup is complete.

---

## What you are validating

Your webhook pipeline now has four layers. This guide exercises each one:

| Layer                  | What it does                                                     |
| ---------------------- | ---------------------------------------------------------------- |
| Signature verification | Rejects anything not signed by Stripe                            |
| Durable claim          | Persists the event and claims it atomically on a unique event ID |
| Handler dispatch       | Runs the business logic and chooses an HTTP status               |
| Outcome recording      | Writes `processed` / `failed` / `permanent_failure` back         |

A `WebhookEvent` document moves through these states:

```
received ──claim──► processing ──success──► processed          (terminal)
                         ├──5xx──► failed ──claim──► processing …
                         └──4xx──► permanent_failure            (terminal, replayable)
```

---

## 1. Install the Stripe CLI

Pick the line for your platform.

**Linux (including WSL) — recommended, no package manager needed:**

```bash
cd ~
curl -fsSL https://github.com/stripe/stripe-cli/releases/latest/download/stripe_linux_x86_64.tar.gz -o stripe.tar.gz
tar -xzf stripe.tar.gz
sudo mv stripe /usr/local/bin/
rm stripe.tar.gz
```

**macOS:**

```bash
brew install stripe/stripe-cli/stripe
```

**Windows (PowerShell, outside WSL):**

```powershell
scoop bucket add stripe https://github.com/stripe/scoop-stripe-cli.git
scoop install stripe
```

Verify it worked:

```bash
stripe --version
```

You should see something like `stripe version 1.21.x`. If you get `command not found`, the binary is
not on your `PATH` — on Linux confirm `/usr/local/bin` is in `echo $PATH`.

---

## 2. Authenticate

```bash
stripe login
```

This prints a pairing code and opens your browser. Confirm the code matches, then approve.

If the browser does not open (common in WSL), copy the URL it prints and paste it into your browser
manually.

Confirm you are connected **to your test account**:

```bash
stripe config --list
```

Look for `test_mode_api_key` in the output. If you see a live key, stop — do not run this guide
against live data.

---

## 3. Configure the webhook secret

The CLI generates its own signing secret when it starts forwarding. You need that secret in your
backend `.env`.

Start the listener first so it prints the secret. **Note the `--skip-verify` flag** — your local
backend serves HTTPS with a self-signed certificate, and without this the CLI refuses to connect:

```bash
stripe listen \
  --forward-to https://localhost:5001/api/payments/webhook \
  --skip-verify
```

It prints something like:

```
> Ready! You are using Stripe API Version [2024-06-20].
  Your webhook signing secret is whsec_a1b2c3d4e5f6... (^C to quit)
```

Copy that `whsec_...` value.

Open `backend/.env` and set:

```
STRIPE_WEBHOOK_SECRET=whsec_a1b2c3d4e5f6...
```

> **The secret changes every time you restart `stripe listen`.** If webhooks suddenly return 400
> after a restart, this is almost always why. Section 14 covers the symptom.

Leave this terminal running. It is **Terminal A** for the rest of the guide.

---

## 4. Start the backend

In a **second terminal (Terminal B)**:

```bash
cd backend
npm run dev
```

Wait for:

```
Server running in development mode on port 5001
🔐 Stripe webhook endpoint: /api/payments/webhook
✅ Redis ping: PONG
```

If Redis is not running locally, start it:

```bash
redis-server --daemonize yes
redis-cli ping     # expect: PONG
```

The webhook path itself does not need Redis any more — the durable store is MongoDB — but stock
reservation and locks still use it, so several handlers will error without it.

---

## 5. Start the frontend (only for Section 12)

You need this **only** to validate the admin UI. Skip it for now if you are working through the
backend sections first.

In a **third terminal (Terminal C)**:

```bash
cd frontend
npm run dev
```

Then open the printed URL (usually `https://localhost:5173`), log in as an **admin** user, and go to
**Admin → System → Webhook Events**.

---

## 6. Open a MongoDB shell

You will inspect documents repeatedly. Keep this open as **Terminal D**:

```bash
mongosh "mongodb://127.0.0.1:27017/<your-dev-db-name>"
```

If you do not know your database name, find it in `backend/.env` under `MONGO_URI`.

A useful shorthand to paste in once — it prints the most recent events compactly:

```javascript
function events(n) {
  return db.webhookevents
    .find({}, { stripeEventId: 1, type: 1, status: 1, attempts: 1, _id: 0 })
    .sort({ receivedAt: -1 })
    .limit(n || 10)
    .toArray();
}
```

---

## 7. Validate a successful event end to end

This is the baseline. Everything else builds on it.

In a **fourth terminal (Terminal E)**:

```bash
stripe trigger payment_intent.succeeded
```

**Expected in Terminal A (the listener):**

```
2026-07-29 12:00:00  --> payment_intent.succeeded [evt_1AbCdEf...]
2026-07-29 12:00:00  <--  [200] POST https://localhost:5001/api/payments/webhook [evt_1AbCdEf...]
```

The `[200]` is the important part.

**Expected in Terminal D (MongoDB):**

```javascript
events(3);
```

You should see a document with `status: 'processed'` and `attempts: 1`.

Inspect it fully:

```javascript
db.webhookevents.findOne({ stripeEventId: "evt_1AbCdEf..." });
```

Confirm:

- `status` is `processed`
- `attempts` is `1`
- `receivedAt`, `processingStartedAt` and `processedAt` are all set
- `processedAt` is **after** `processingStartedAt`
- `payload` contains the full event
- `lastError` is empty

✅ **Pass criteria:** HTTP 200, one document, `processed`, three timestamps in order.

---

## 8. Validate duplicate delivery (idempotency)

This is the behaviour that was inverted before this work. It is the single most important check in
this guide.

Resend the **same event** you just triggered:

```bash
stripe events resend evt_1AbCdEf...
```

Use the real event ID from Section 7.

**Expected in Terminal A:** `[200]` again.

**Expected response body** — the CLI does not show it, so check Terminal B's logs for:

```
[WebhookStore] Event already recorded; skipping
```

**Expected in MongoDB:** the document is **unchanged**.

```javascript
db.webhookevents.findOne({ stripeEventId: "evt_1AbCdEf..." }, { attempts: 1, status: 1 });
```

- `attempts` is still `1` — **not 2**
- `status` is still `processed`

✅ **Pass criteria:** the redelivery is acknowledged with 200, no second document is created, the
attempt counter does **not** increase, and no business logic re-ran.

❌ **If `attempts` became 2**, the claim is not treating `processed` as terminal. That is a
regression in `claimWebhookEvent`.

---

## 9. Validate the events that matter most

Trigger each of these and confirm a `processed` record appears. These are the handlers that touch
money.

```bash
stripe trigger checkout.session.completed
stripe trigger charge.refunded
stripe trigger charge.dispute.created
stripe trigger charge.dispute.closed
stripe trigger payment_intent.payment_failed
```

After each, check:

```javascript
events(5);
```

**Expected:** every one reaches a terminal state.

> Some will land as `permanent_failure` with an error like "order not found". **That is correct.**
> `stripe trigger` creates synthetic objects with no matching order in your database, and a missing
> order is a permanent condition — retrying cannot conjure it up. Confirm the reason:
>
> ```javascript
> db.webhookevents.findOne({ type: "checkout.session.completed" }).lastError;
> ```
>
> You should see `classification: 'permanent'` and `httpStatus: 200`.

Pay particular attention to `charge.dispute.created`. Before this work, the **first** delivery of
this event was silently skipped. Confirm it now produces a record with `attempts: 1` and a terminal
status.

✅ **Pass criteria:** every triggered event produces exactly one document, none are silently
missing.

---

## 10. Validate retry behaviour (failure simulation)

You need the handler to fail in a way that is classified as _retryable_. The cleanest way is to make
MongoDB briefly unavailable mid-processing.

**Method A — stop MongoDB briefly (most realistic):**

```bash
# Terminal E
sudo systemctl stop mongod          # or: brew services stop mongodb-community
stripe trigger charge.refunded
sleep 5
sudo systemctl start mongod         # or: brew services start mongodb-community
```

**Expected in Terminal A:** `[500]`.

**Expected in Terminal B:** an error log, and — because the store itself was down — a
`[WebhookStore] Claim failed; processing without a durable record` warning. This is the **fail
open** path working as designed.

**Method B — force a retryable failure with the store still up.** Temporarily add a throw at the top
of one handler in `backend/src/controllers/payment.controller.js`:

```javascript
if (event.type === "charge.refunded") {
  throw new Error("simulated transient failure"); // ← REMOVE AFTER TESTING
}
```

Save (nodemon restarts), then:

```bash
stripe trigger charge.refunded
```

**Expected in Terminal A:** `[500]`.

**Expected in MongoDB:**

```javascript
db.webhookevents.findOne({ type: "charge.refunded" }, { status: 1, attempts: 1, lastError: 1 });
```

- `status` is `failed` — **not** `permanent_failure`
- `lastError.classification` is `transient`
- `lastError.httpStatus` is `500`

Now confirm the retry actually re-claims it:

```bash
stripe events resend evt_...     # the failed event's ID
```

- `attempts` increments to `2`
- `status` returns to `processing`, then settles

**Remove the simulated throw before continuing.**

✅ **Pass criteria:** a retryable failure returns 500, leaves the record re-claimable, and a
redelivery increments `attempts` rather than being deduplicated away.

---

## 11. Validate stale-processing recovery

This proves a worker that dies mid-event cannot wedge it forever.

Manually put a record into a stuck state:

```javascript
// Terminal D
db.webhookevents.updateOne(
  { stripeEventId: "evt_1AbCdEf..." },
  {
    $set: {
      status: "processing",
      processingStartedAt: new Date(Date.now() - 60 * 60 * 1000), // one hour ago
    },
  }
);
```

Confirm it now counts as stuck:

```javascript
db.webhookevents.countDocuments({
  status: "processing",
  processingStartedAt: { $lt: new Date(Date.now() - 5 * 60 * 1000) },
});
// expect: 1
```

Now resend that event:

```bash
stripe events resend evt_1AbCdEf...
```

**Expected:** the claim **succeeds** (the stale record is re-claimable), `attempts` increments, and
the record settles.

The staleness window is `WEBHOOK_STALE_PROCESSING_MS`, default 5 minutes.

✅ **Pass criteria:** a record stuck in `processing` past the window is reclaimed rather than
permanently skipped.

---

## 12. Validate the admin UI

Start the frontend (Section 5) and go to **Admin → System → Webhook Events**.

**Health counters** — verify against the database:

```javascript
db.webhookevents.aggregate([{ $group: { _id: "$status", n: { $sum: 1 } } }]);
```

The panel's tiles should match these counts.

**Filtering** — check each of these:

- Set **Status** to `Needs review` → only `permanent_failure` rows
- Set **Event type** to `charge.refunded` → only that type
- Enter an event ID in **Search** → that single row
- Enter a payment intent ID (`pi_...`) from a payload → matching rows
- Set a **From** date in the future → empty state, not a blank table

**Sorting** — click the Attempts and Received headers; the arrow flips and order reverses.

**Pagination** — set Per page to 25 and confirm the "Showing X–Y of Z" line is accurate.

**Detail** — click **Event detail** on any row. Confirm you see all four timestamps, attempt count,
replay count, the structured error (message, classification, HTTP status, stack), and the stored
payload.

**Themes** — toggle light/dark. No unreadable text, no invisible badges.

**RTL** — switch language to العربية. The layout must mirror; check that the table, filters and
modals do not overflow horizontally.

**Localisation** — switch through Español and Français. No raw keys like
`adminWebhookEvents.replay.action` should appear anywhere.

✅ **Pass criteria:** counts match the database, every filter narrows correctly, all four languages
render, both themes are legible.

---

## 13. Validate replay

**Single replay.**

Find a `permanent_failure` row whose payload is still stored, click **Replay**.

**Expected:** a confirmation dialog appears that explains _what replay does_, _why it is safe_, and
_when not to use it_. Nothing is sent until you confirm.

Click **Cancel** first and confirm in Terminal B that no request was made. Then click **Replay**
again and confirm.

**Expected in MongoDB:**

```javascript
db.webhookevents.findOne(
  { stripeEventId: "evt_..." },
  { replayCount: 1, replayedBy: 1, status: 1 }
);
```

- `replayCount` is `1`
- `replayedBy` is your admin user's ObjectId
- `status` has moved on from `permanent_failure`

**Expected in Terminal B:** `[AdminWebhookReplay] Replaying webhook event`.

**Replay of a discarded payload.**

Simulate retention having dropped the payload:

```javascript
db.webhookevents.updateOne(
  { stripeEventId: "evt_..." },
  { $set: { payload: null, payloadDiscardedAt: new Date() } }
);
```

Refresh the panel. **Expected:** the Replay button for that row is **disabled**, and hovering it
explains that the payload was discarded. The checkbox is also disabled, so it cannot be included in
a bulk replay.

**Bulk replay.**

Select two or three replayable rows, click **Replay selected (N)**, confirm.

**Expected:** a summary of how many succeeded, and `replayCount` incremented on each.

✅ **Pass criteria:** replay always requires confirmation, ineligible events cannot be replayed and
say why, and replay is recorded with the acting admin.

---

## 14. Validate monitoring, alerting and the heartbeat

**Run the maintenance job immediately** rather than waiting for the hour:

```bash
# Terminal B — stop the server first (Ctrl+C), then:
WEBHOOK_EVENT_MAINTENANCE_RUN_ON_BOOT=true npm run dev
```

**Expected in Terminal B:**

```
[WebhookMaintenance] Cycle complete { counts: {...}, stuck: 0, recentFailures: N, ... }
```

**Trigger the alerts.** Create a stuck record (Section 11) and a permanent failure, then restart
with the boot flag again.

**Expected:**

```
[WebhookMaintenance] Events stuck mid-processing { stuck: 1, alert: true }
[WebhookMaintenance] Webhook events in permanent_failure need review { permanentFailures: N, alert: true }
```

Any log line carrying `alert: true` is also forwarded to Sentry when `SENTRY_DSN` is configured.

**Verify heartbeat integration:**

```javascript
// Terminal D
db.jobhealths.findOne({ jobName: "webhook_event_maintenance" });
```

Confirm `lastStatus: 'success'`, and that `lastCompletedAt` is recent. This is what the heartbeat
watchdog reads — if the maintenance job stops running, the watchdog notices.

**Check the admin stats endpoint** matches:

```bash
curl -k https://localhost:5001/api/admin/webhook-events/stats \
  -H "Cookie: <your admin session cookie>"
```

Get the cookie from your browser's DevTools → Application → Cookies while logged in as admin.

**Inspect Redis** — the webhook path no longer uses it for idempotency, so confirm nothing lingers:

```bash
redis-cli KEYS 'stripe:webhook:*'      # expect: (empty array)
redis-cli KEYS 'processed_refund*'     # expect: (empty array)
redis-cli KEYS 'stock_reservation:*'   # may have entries — that is stock, not webhooks
```

✅ **Pass criteria:** the job runs, alerts fire above thresholds, `JobHealth` records success, and
no webhook idempotency keys remain in Redis.

---

## 15. Common mistakes

| Symptom                                     | Cause and fix                                                                                                                                       |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[400]` on every event                      | `STRIPE_WEBHOOK_SECRET` does not match the running `stripe listen`. The secret changes on every restart — copy the new one and restart the backend. |
| CLI cannot connect, TLS error               | Missing `--skip-verify`. The dev server uses a self-signed certificate.                                                                             |
| `[404]` on every event                      | Wrong path. It is `/api/payments/webhook`, not `/webhooks/stripe`.                                                                                  |
| Events processed but no `WebhookEvent` docs | Wrong database in `mongosh`. Check `MONGO_URI` in `backend/.env`.                                                                                   |
| `permanent_failure` on every trigger        | Expected for synthetic events with no matching order. Not a bug.                                                                                    |
| `attempts` climbing on redelivery           | The record is not reaching a terminal state — check the handler's status.                                                                           |
| Admin panel 403                             | Logged in as a non-admin. Replay endpoints are admin-only.                                                                                          |
| Raw translation keys in the UI              | Locale file not rebuilt. Restart the Vite dev server.                                                                                               |
| Redis errors on startup                     | `redis-server --daemonize yes`.                                                                                                                     |

---

## 16. Cleanup

**Stop the listener** in Terminal A with `Ctrl+C`.

**Remove test webhook events** from your dev database:

```javascript
// Terminal D
db.webhookevents.deleteMany({ "payload.livemode": false });
// or, to clear everything in dev:
db.webhookevents.deleteMany({});
```

**Remove any simulated failure code** you added in Section 10. Confirm with:

```bash
git diff backend/src/controllers/payment.controller.js
```

This must come back empty.

**Clear the job health record** if you want a clean slate:

```javascript
db.jobhealths.deleteOne({ jobName: "webhook_event_maintenance" });
```

**Reset the webhook secret** in `backend/.env` to whatever your normal dev value is, or remove the
CLI's temporary one.

**Remove the boot flag** if you exported it: `unset WEBHOOK_EVENT_MAINTENANCE_RUN_ON_BOOT`.

---

## 17. Troubleshooting

**Nothing arrives at the backend at all.** Confirm the listener is running and shows `Ready!`.
Confirm the backend is listening: `curl -k https://localhost:5001/api/health` (or any known route).
Check no firewall sits between them — in WSL, make sure both the CLI and the backend are inside WSL,
not split across it and Windows.

**Events show `[200]` but nothing happens.** Check Terminal B for
`Event already recorded; skipping`. You are probably resending an event that already reached a
terminal state — that is correct behaviour. Trigger a fresh one.

**A record is stuck in `processing` and never recovers.** It becomes re-claimable after
`WEBHOOK_STALE_PROCESSING_MS` (default 5 minutes). If it is still stuck after that, the next
delivery or replay will reclaim it. To force it manually:

```javascript
db.webhookevents.updateOne({ stripeEventId: "evt_..." }, { $set: { status: "failed" } });
```

**Replay returns 409.** The payload was discarded by retention. Check `payloadDiscardedAt`. If the
event is still within Stripe's own 30-day retention, resend from the Stripe Dashboard instead:
Developers → Webhooks → your endpoint → the event → **Resend**.

**You need to see what Stripe actually sent.** `stripe listen --print-json` prints full event
bodies. Or read the stored copy:

```javascript
db.webhookevents.findOne({ stripeEventId: "evt_..." }).payload;
```

**You want to inspect delivery history from Stripe's side.** The Dashboard is authoritative for what
was delivered and what you responded: Developers → Webhooks → endpoint → recent deliveries. Useful
when your local record and Stripe's disagree.

---

## Sign-off checklist

Work through this before declaring the architecture production-ready.

```
[ ]  7. Successful event -> 200, one document, processed, timestamps ordered
[ ]  8. Duplicate delivery -> 200, attempts stays 1, no re-processing
[ ]  9. All five money-touching event types produce a record
[ ] 10. Retryable failure -> 500, status failed, redelivery re-claims and increments attempts
[ ] 11. Stale processing record is reclaimed after the window
[ ] 12. Admin UI: counts match DB, filters work, 4 languages, both themes, RTL
[ ] 13. Replay: confirmation required, records replayedBy, discarded payload refused
[ ] 14. Maintenance job runs, alerts fire, JobHealth updated, no webhook keys in Redis
[ ] 16. Cleanup done, git diff on payment.controller.js is empty
```
