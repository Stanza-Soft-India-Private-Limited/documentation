# SME — Apple IAP Payments & Events API Guide

Reference for the **SME portal** to display **Apple In-App Purchase** data — both the
**payments/orders** (who paid, how much, which plan, current status) and the **Apple
events** (the App Store Server Notification lifecycle: subscribed, renewed, expired,
refunded, …). Same server-to-server model as the rest of the SME surface: we expose the
endpoints, the SME team builds the UI.

This is the Apple sibling of what the portal already shows for Razorpay. **No new
endpoints were added** — Apple data flows through the *same* `/sme/transactions`,
`/sme/orders`, and `/sme/webhooks` endpoints you already use for Razorpay, via a
`source=APPLE` filter. This doc documents the Apple-specific values and one behaviour
change to `/sme/webhooks` (see §0).

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** request.
**Swagger:** `/api/docs` (group **SME**) documents these live with the `x-api-key` control.

- Missing/invalid key → **401** `{ "message": "Invalid or missing API key" }`.
- All routes use the `/api/v1/` global prefix.
- Amounts in **responses are in rupees** (`amount / 100`).
- List responses are paginated: `{ data[], total, page, limit, hasMore }`.

---

## 0. The mental model (read this first)

**Apple IAP is auto-renewable *subscriptions*, not one-time orders.** There are exactly
two products:

| Apple product id | Internal `planType` |
|---|---|
| `com.stanzasoft.upscbuddy.premium.monthly` | `MONTHLY` |
| `com.stanzasoft.upscbuddy.premium.annual`  | `ANNUAL` |

This is the key difference from Razorpay/MANUAL, which are **one-time** purchases. An
Apple subscription produces a **new `Order` + `Payment` row on every billing period**
(initial buy *and* each renewal), and a stream of **events** as Apple notifies us of
renewals, failures, expiries, and refunds.

Two surfaces give you the full Apple picture:

1. **Payments & Orders** — `GET /sme/transactions?source=APPLE` and
   `GET /sme/orders?source=APPLE`. These already returned Apple rows before this doc;
   nothing changed. §1–§2.
2. **Events** — `GET /sme/webhooks?source=APPLE`. **This is the new capability.** §3.

> ### ⚠️ Behaviour change on `/sme/webhooks` — read before you ship
>
> The `/sme/webhooks` endpoint previously read a legacy `webhook_events` table that **no
> code ever wrote to** — so for everyone it returned an **empty list**. It now reads the
> real provider event ledger (`payment_events`), which holds **both Razorpay webhooks and
> Apple notifications**.
>
> **Razorpay screens will not break.** The response **keeps every original field**
> (`id`, `eventId`, `eventType`, `source`, `payload`, `processedAt`) with the same key
> names — new fields are only **added** alongside them. The only difference is that the
> endpoint now returns *actual rows* instead of an empty array. If your Razorpay UI was
> reading this endpoint (and getting nothing), it will simply start receiving data in a
> shape it already understands. If your Razorpay UI gets its events from Razorpay's own
> dashboard, it is unaffected. Either way: **additive, non-breaking.**
>
> If you want this endpoint to stay Razorpay-only on an existing screen, pass
> `source=RAZORPAY` and you get exactly the Razorpay subset. Pass `source=APPLE` for the
> new Apple events.

---

## 1. Apple payments — `GET /sme/transactions?source=APPLE`

Read-only list of Apple `Payment` rows (one per captured billing period), newest first.

```
GET /sme/transactions?source=APPLE&page=1&limit=20&userId=&paymentStatus=&startDate=&endDate=&search=
```

| Query | Description |
|---|---|
| **source** | Set to `APPLE` for Apple only. Omit for all sources; `RAZORPAY` / `MANUAL` for the others. Filters on the parent order's `paymentSource`. |
| paymentStatus | `PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED`. Apple captures land as **`CAPTURED`**. |
| userId | payments for one user (UserAuth id) |
| startDate / endDate | ISO date bounds on `createdAt` |
| search | payer/user email (note: Apple rows have no `razorpayPaymentId`; search by email) |

**Response 200** `data[]` (Apple example):
```json
{
  "id": "…",
  "orderId": "…",
  "razorpayPaymentId": null,
  "appleTransactionId": "2000000812345678",
  "amount": 499,
  "currency": "INR",
  "status": "CAPTURED",
  "method": "apple_iap",
  "email": null,
  "capturedAt": "2026-06-28T10:15:00.000Z",
  "createdAt": "2026-06-28T10:15:02.000Z",
  "order": {
    "id": "…",
    "userId": "…",
    "planType": "MONTHLY",
    "paymentSource": "APPLE",
    "user": { "id": "…", "email": "…", "name": "…", "phoneNumber": "…" }
  }
}
```

**Apple-specific fields / values to surface:**
- `paymentSource: "APPLE"`, `method: "apple_iap"`.
- `appleTransactionId` — the per-period Apple transaction id (unique per row).
- `razorpayPaymentId`, `email`, `contact`, `bank`, `wallet`, `vpa` are **null** for Apple
  (those are Razorpay/card fields).

### 1.1 Payment detail
```
GET /sme/transactions/:id
```
Same shape + `notes` (Apple stores `{ rawJws, storefront }` on verify, or
`{ source: "webhook", storefront }` on a renewal). `404` if not found.

---

## 2. Apple orders — `GET /sme/orders?source=APPLE`

One `Order` per Apple billing period (initial + each renewal). This is the best surface
for "what did this user buy and is their premium active".

```
GET /sme/orders?source=APPLE&page=&limit=&userId=&orderStatus=&startDate=&endDate=&search=
GET /sme/orders/:id
```

| Query | Description |
|---|---|
| **source** | `APPLE` for Apple only |
| orderStatus | `CREATED, PAID, FAILED, CANCELLED`. Apple orders are created as **`PAID`**. |
| search | `receipt` (Apple receipts look like `apple_…` / `apple_wh_…`) or user email |

**Response 200** `data[]` (Apple example):
```json
{
  "id": "…",
  "userId": "…",
  "razorpayOrderId": null,
  "appleTransactionId": "2000000812345678",
  "paymentSource": "APPLE",
  "planType": "ANNUAL",
  "amount": 3999,
  "currency": "INR",
  "receipt": "apple_2000000812345678",
  "status": "PAID",
  "premiumGrantedAt": "2026-06-28T10:15:02.000Z",
  "createdAt": "2026-06-28T10:15:02.000Z",
  "updatedAt": "2026-06-28T10:15:02.000Z",
  "user": { "id": "…", "email": "…", "name": "…", "phoneNumber": "…" },
  "payments": [ { "…": "see §1 shape" } ]
}
```

**Notes:**
- `premiumGrantedAt` is stamped on Apple orders, so the reconcile rule "`status=PAID` but
  `premiumGrantedAt=null` = money taken, premium not granted" works the same as Razorpay.
  Healthy Apple orders are always granted.
- `razorpayOrderId` is null; the Apple identity is `appleTransactionId` +
  `appleOriginalTransactionId` (the stable id that survives renewals — see §3).

---

## 3. Apple events — `GET /sme/webhooks?source=APPLE`

The App Store Server Notification (ASSN v2) lifecycle, as Apple sent it to us. This is the
Apple equivalent of the Razorpay webhook feed. Backed by the `payment_events` ledger
(append-only; written *before* processing so nothing is ever silently lost).

```
GET /sme/webhooks?source=APPLE&page=&limit=&startDate=&endDate=&search=
```

| Query | Description |
|---|---|
| **source** | `APPLE` = Apple notifications only · `RAZORPAY` = Razorpay webhooks only · omit = both · `MANUAL` = always empty (manual grants emit no provider event) |
| startDate / endDate | ISO bounds on `receivedAt` |
| search | matches `eventId` (Apple `notificationUUID`) or `eventType` (e.g. `apple.DID_RENEW`) |

**Response 200** `data[]` (Apple example):
```json
{
  "id": "…",
  "eventId": "9f1d2c3a-…-notificationUUID",
  "eventType": "apple.DID_RENEW",
  "source": "apple",
  "payload": "<signedPayload JWS string>",
  "processedAt": "2026-06-28T10:15:03.000Z",

  "provider": "apple",
  "kind": "webhook",
  "result": "processed",
  "error": null,
  "userId": "…",
  "orderId": "…",
  "receivedAt": "2026-06-28T10:15:02.000Z"
}
```

### Field reference

| Field | Meaning |
|---|---|
| `eventId` / `externalId` | Apple `notificationUUID` (dedup key). For Razorpay rows this is the Razorpay event id. |
| `eventType` / `type` | `apple.<NOTIFICATION_TYPE>[.<SUBTYPE>]`, e.g. `apple.SUBSCRIBED.INITIAL_BUY`, `apple.DID_RENEW`, `apple.EXPIRED.VOLUNTARY`. Razorpay rows are `payment.captured`, `payment.failed`, etc. |
| `source` / `provider` | `"apple"` or `"razorpay"` (lowercase). **Back-compat:** `source` mirrors `provider`. |
| `payload` / `rawBody` | The raw event body. For Apple this is the **signed JWS string** (not JSON) — display as-is or decode client-side; for Razorpay it is the raw JSON string. |
| `kind` | `"webhook"` (this endpoint only returns provider webhooks/notifications). |
| `result` | Outcome: `processed` (applied), `orphan` (no user resolvable yet), `error` (threw — see `error`). |
| `error` | Error message when `result = "error"`, else null. |
| `userId` / `orderId` | Resolved app ids when known (may be null for orphan/early events). |
| `receivedAt` | When we received it (always set; default sort key, newest first). |
| `processedAt` | When processing finished (may be null if still pending/errored). |

### Apple notification types you'll see

| `notificationType` | What it means | Entitlement effect |
|---|---|---|
| `SUBSCRIBED` (`INITIAL_BUY` / `RESUBSCRIBE`) | First purchase or resubscribe | Grant + set expiry |
| `DID_RENEW` | Auto-renewal succeeded | Extend expiry |
| `OFFER_REDEEMED` | Promo/offer redeemed | Grant + extend |
| `RENEWAL_EXTENDED` | Apple extended the renewal date | Extend expiry |
| `DID_FAIL_TO_RENEW` (`GRACE_PERIOD`) | Billing failed | Keep access while in grace; else sweep downgrades |
| `EXPIRED` | Subscription lapsed | Downgrade (unless another source still entitles) |
| `GRACE_PERIOD_EXPIRED` | Grace ended unpaid | Downgrade |
| `REVOKE` / `REFUND` | Apple revoked / refunded | Records a refund + downgrades |
| `REFUND_REVERSED` | Refund reversed | Re-grant |
| `DID_CHANGE_RENEWAL_STATUS` | User toggled auto-renew on/off | Logged only — **no** entitlement change |
| `PRICE_INCREASE`, `METADATA_UPDATE`, `CONSUMPTION_REQUEST`, `MIGRATION`, `TEST` | Informational | No-op (still recorded) |

> The current subscription **state** (active / expired / grace / revoked, expiry date,
> auto-renew flag) lives in `apple_subscriptions`, keyed on the stable
> `originalTransactionId`. That table is **not yet exposed** via an SME endpoint — if the
> portal needs a live "is this subscription active and when does it renew" view (beyond
> reconstructing it from the event stream + order expiry), ask and we'll expose it.

---

## 4. Caveats checklist

- **`/sme/webhooks` is now populated.** Previously empty for everyone. Non-breaking
  (additive fields), but confirm any code that *asserted* it was empty.
- **`payload` for Apple is a JWS string, not JSON.** Don't blindly `JSON.parse` it.
- **`MANUAL` source has no events** — `source=MANUAL` on `/sme/webhooks` returns `[]`.
- **Apple amounts** come from the App Store transaction in the app's display currency,
  stored in paise and returned in rupees like everything else.
- **Renewals create new orders.** A long-lived subscriber will have many Apple `Order`
  rows (one per period). Group by `appleOriginalTransactionId` (visible in detail) or by
  `userId` if you want a per-subscription view.
- **Live Swagger is the source of truth** for exact params: `/api/docs`, group **SME**.

---

## 5. Quick reference

| Need | Call |
|---|---|
| Apple payments (per billing period) | `GET /sme/transactions?source=APPLE` |
| One Apple payment + notes | `GET /sme/transactions/:id` |
| Apple orders (buy + renewals) | `GET /sme/orders?source=APPLE` |
| One Apple order + its payments | `GET /sme/orders/:id` |
| Apple notification/event stream | `GET /sme/webhooks?source=APPLE` |
| Razorpay event stream (unchanged) | `GET /sme/webhooks?source=RAZORPAY` |
| Both providers' events | `GET /sme/webhooks` |
