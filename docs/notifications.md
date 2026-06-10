# In-App Notification Centre

Reference documentation for the notification system introduced in C6. For the high-level overview
and environment flag see the [main README](../README.md#in-app-notification-centre).

---

## 1. Feature flag

```env
FEATURE_IN_APP_NOTIFICATIONS=true
```

When unset or `false`, all `/api/notifications/*` routes return 404 and the bell UI is hidden on web
and mobile.

---

## 2. Data model

**Collection:** `notifications` — `backend/src/models/notification.model.js`

```
Field        Type       Notes
-----------  ---------  -------------------------------------------------------
userId       ObjectId   Recipient; sparse index for fast per-user queries
type         String     Enum — 22 values (see catalogue below)
titleKey     String     i18n key, e.g. "notifications.order_shipped.title"
bodyKey      String     i18n key, e.g. "notifications.order_shipped.body"
variables    Object     Interpolation map passed to t(key, variables)
read         Boolean    false by default; indexed for unread-count queries
createdAt    Date       Indexed; used for pagination (newest first)
```

Rendering is intentionally client-side: the server stores keys and variables, not translated
strings. Switching UI language instantly updates all notification copy without a round-trip.

### Notification type catalogue

```
order_shipped              order_delivered
refund_approved            refund_rejected
payout_completed           payout_failed
product_approved           product_rejected           product_action_required
kyc_approved               kyc_rejected
store_closed               store_closure_cancelled    store_closure_completed
new_review
dispute_opened             dispute_message            dispute_resolved          dispute_escalated
order_cancelled
new_order                  new_dispute
```

---

## 3. REST API

All routes require a valid session cookie (authenticated user). Unread count responses are
Redis-cached and invalidated on any write to the user's notification set.

```
GET    /api/notifications
       Query: page (default 1), limit (default 20)
       Response: { notifications: [...], total, page, pages, unreadCount }
       Unread notifications are sorted to the top of each page.

GET    /api/notifications/unread-count
       Response: { count: N }
       Result is Redis-cached (TTL: ~60 s). Invalidated on mark-read and delete.

PATCH  /api/notifications/:id/read
       Marks a single notification as read. Returns 404 if not found or not owned.

PATCH  /api/notifications/read-all
       Marks all notifications for the authenticated user as read.
       Returns { modifiedCount: N }.

DELETE /api/notifications/:id
       Deletes a single notification. Returns 404 if not found or not owned.
```

---

## 4. Web bell UI

**Component tree (web — `frontend/src/`):**

```
Navbar
  └─ NotificationBell          — bell icon + badge + polling
       ├─ NotificationDropdown — dropdown panel (most recent N)
       │    └─ NotificationRow — single row; click opens modal
       └─ NotificationModal    — portal-rendered detail view
```

**Behaviour:**

- Badge shows unread count; hidden when count is 0.
- Dropdown renders at most 10 recent notifications (configurable).
- Unread rows have a distinct background tint and left accent border using DaisyUI theme tokens — no
  hardcoded colours.
- Clicking a row marks it read and opens the detail modal; the dropdown stays open.
- The modal is portal-rendered to `document.body` to avoid z-index conflicts.
- Polling: `setInterval` at 60 000 ms; cleared on component unmount.
- "View all" navigates to `/notifications` (full paginated list page).

---

## 5. Mobile bell UI

**Component tree (mobile — `mobile/src/`):**

```
AppHeader
  └─ NotificationBell   — bell icon + badge
       └─ BottomSheet   — slide-up sheet with notification list
            └─ NotificationDetailModal — full detail overlay
```

**Behaviour:**

- Red dot badge on the bell mirrors web (count shown when > 0).
- Tapping the bell opens the bottom sheet listing recent notifications.
- Tapping a row opens the `NotificationDetailModal`.
- Polling is AppState-aware: resumes on `active`, pauses on `background` / `inactive`.
- Theme colours come from the generated mobile palette (`mobile/src/theme/generated/`).

---

## 6. i18n

All notification copy lives under `notifications.*` in `shared/locales/{en,ar,es,fr}.json`.

Key structure per type:

```json
"notifications": {
  "order_shipped": {
    "title": "Your order has shipped",
    "body": "Order #{{orderId}} is on its way."
  },
  ...
}
```

Variables available in interpolation are documented alongside each `fireNotification` call in the
service layer.

---

## 7. Dispatched (`order_shipped`) notification

The `order_shipped` notification is fired by `onOrderStatusChanged`
(`backend/src/services/orders/orderFinancials.hook.js`) whenever an order transitions to
`dispatched`. This happens in two paths:

- **Label-generated shipment:** seller purchases a Shippo carrier label via
  `POST /seller/orders/:id/shipments` (`source: 'api'`). The first label creation auto-transitions
  the order from `processing` → `dispatched`.
- **Manual tracking entry:** seller fills in a tracking number (and optionally a tracking URL and
  custom carrier name) in the ShipmentForm. Same `onOrderStatusChanged` hook fires.

### Variables map

| Variable      | Source                                                            | Notes                                             |
| ------------- | ----------------------------------------------------------------- | ------------------------------------------------- |
| `orderId`     | `order._id`                                                       | Always present                                    |
| `carrier`     | `shipment.carrier` (e.g. `"usps"`, `"dhl"`, `"My Local Courier"`) | Custom carrier name when "Other" selected in form |
| `trackingUrl` | `shipment.trackingUrl` — public carrier tracking page URL         | May be absent for manual shipments with no URL    |

### Body-key selection

The notification body key depends on whether a tracking URL is present:

- `notifications.types.order_shipped.body` — when `trackingUrl` is set; the client renders the URL
  as a clickable link.
- `notifications.types.order_shipped.bodyNoTracking` — when no tracking URL is available (manual
  shipments that have a tracking number but no public tracking page URL).

The `link` field on the notification document is set to `trackingUrl` when present, otherwise to
`/orders/<orderId>` so tapping the notification still opens the correct order detail page.

### Companion shipped email

`sendOrderShippedEmail` uses a **three-way template** keyed by tracking state:

| State                     | Template keys used                         |
| ------------------------- | ------------------------------------------ |
| No tracking number at all | `htmlNoTracking` / `plainNoTracking`       |
| Tracking number + URL     | `html` / `plain`                           |
| Tracking number, no URL   | `htmlTrackingNoUrl` / `plainTrackingNoUrl` |

The three-way selection prevents a broken `<a href="">` link in cases where a seller has a tracking
number (e.g. from a local carrier) but no publicly accessible tracking URL.

### Custom carrier name

When a seller selects **"Other"** in the carrier dropdown on the ShipmentForm, a free-text input is
revealed and its value is stored as `shipment.carrier`. This custom string is passed directly in the
notification `variables.carrier` so the customer sees the actual carrier name (e.g. "Aramex Local
Office", "Hand delivery") rather than a generic placeholder.

### Manual tracking URL

An optional **Tracking URL** field lets the seller paste the carrier's tracking page URL. When
provided it is stored on the `Shipment` document, set as the notification `link`, and embedded as a
clickable hyperlink in the shipped email. When absent the system gracefully falls back to the
`bodyNoTracking` key and the order page link — no broken links are ever rendered.

---

## 8. Wiring new notification types

Follow these four steps to add a notification type end-to-end:

**Step 1 — Model**

Add the new type string to the `type` enum in `backend/src/models/notification.model.js`.

**Step 2 — i18n**

Add `titleKey` and `bodyKey` entries in all four locale files:

```
shared/locales/en.json
shared/locales/ar.json
shared/locales/es.json
shared/locales/fr.json
```

Keys must be present in all four files before merge (locale key parity is required).

**Step 3 — Service call**

At the relevant callsite (controller or service) add:

```js
await fireNotification(userId, "your_new_type", { key: value });
```

Import `fireNotification` from `backend/src/services/notification/notification.service.js`.

The call is fire-and-forget: wrap in a try/catch if needed, but do not let a notification failure
propagate to the caller.

**Step 4 — (Optional) badge / UI**

If the new type needs special routing on tap (e.g., navigate to a specific screen), add a case to
the `onNotificationPress` handler in the web `NotificationRow` and mobile `BottomSheet` components.

---

## 9. Redis keys used

```
notification:unread:<userId>    — cached unread count (TTL ~60 s)
```

Invalidated by `PATCH /read`, `PATCH /read-all`, and `DELETE /:id`.
