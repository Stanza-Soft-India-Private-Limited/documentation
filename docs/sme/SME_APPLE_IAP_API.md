# SME — Apple IAP Payments & Events API Guide

> Changed 2026-09-21 — exam dimension on users, orders, offers, banners; two BREAKING calls (see [WHAT_CHANGED_2026-09-21.md](./WHAT_CHANGED_2026-09-21.md)).

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

**Apple IAP is auto-renewable *subscriptions*, not one-time orders.**

**Products are per exam, and nothing is hard-coded.** Each exam owns an App Store Connect
**subscription group** (`exams.apple_subscription_group_id`) and each of its plans owns a
product (`exam_plans.apple_product_id`, **globally unique** — two exams can never share
one). The set of live product ids is therefore whatever `exam_plans` currently holds; read
it from `GET /sme/exams` rather than from a list in a doc.

That uniqueness is load-bearing: Apple's verify and webhook payloads **carry no exam**, so
the *only* way to know what was bought is the reverse lookup from the signed `productId`
to its `exam_plans` row. When that lookup misses — a product added in App Store Connect
but never seeded here — the order is recorded with **`examId: null` and premium is
deliberately NOT granted**, because granting a guessed exam for real money is worse than
granting nothing. Those orders surface as `reconcileHint: "UNRESOLVED_EXAM"` (§2).

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
GET /sme/transactions?source=APPLE&page=1&limit=20&userId=&paymentStatus=&exam=&startDate=&endDate=&search=&includeTest=
```

| Query | Description |
|---|---|
| **source** | Set to `APPLE` for Apple only. Omit for all sources; `RAZORPAY` / `MANUAL` for the others. Filters on the parent order's `paymentSource`. |
| paymentStatus | `PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED`. Apple captures land as **`CAPTURED`**. |
| userId | payments for one user (UserAuth id) |
| **exam** | Filters on the **parent order's** `exam_id` — same mechanism as `userId`/`source`. Pass a slug, or the literal **`none`** for orders with no exam (the unresolved Apple products, §2). **400 on an unknown slug.** |
| startDate / endDate | ISO date bounds on `createdAt` |
| search | payer/user email (note: Apple rows have no `razorpayPaymentId`; search by email) |
| **includeTest** | `true` to include App Store **sandbox** purchases. **Default `false`** — sandbox hits production, so leaving them in inflates every revenue figure. |

**What staging returns today**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/transactions?source=APPLE&limit=2` → **200**, verbatim:

```json
{ "data": [], "total": 0, "page": 1, "limit": 2, "hasMore": false }
```

> 🔎 **There has never been an Apple purchase on staging** — App Store sandbox
> purchases go to the *production* backend (Apple has no staging endpoint; see §2.1
> `isTest`), so `source=APPLE` is structurally empty here and no Apple row can be
> captured. The envelope above *is* live and is the part the portal codes against
> (`data` · `total` · `page` · `limit` · `hasMore`); **the row shape below is verified
> against the DTO + the Razorpay row captured on the same call shape in
> [`SME_PORTAL_API.md` §2.1](./SME_PORTAL_API.md), not captured from an Apple row.**

**Response 200** `data[]` (Apple example — shape from code):
<!-- shape verified against code @4f6622e — Apple rows do not exist on staging, see note above -->
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
    "examId": "upsc-cse",
    "isTest": false,
    "user": { "id": "…", "email": "…", "name": "…", "phoneNumber": "…" }
  }
}
```

**Apple-specific fields / values to surface:**
- `paymentSource: "APPLE"`, `method: "apple_iap"`.
- `appleTransactionId` — the per-period Apple transaction id (unique per row).
- `razorpayPaymentId`, `email`, `contact`, `bank`, `wallet`, `vpa` are **null** for Apple
  (those are Razorpay/card fields).
- **`order.examId`** — which exam the money bought, **nullable** (see §0 and §2).
  A payment has no exam of its own; a payment's exam is always its order's.
- **`order.isTest`** — sandbox purchase. Present on the nested order even though the
  list already excludes them by default, so a detail read is unambiguous.

> **The money fields that are *not* on a payment row.** `offerCode`, `listAmount`,
> `storefrontCurrency`, `chargedAmount` and `reconcileHint` live on the **order**, not on
> the payment. For anything about what was actually charged, read §2.

### 1.1 Payment detail
```
GET /sme/transactions/:id
```
Same shape + `notes` (Apple stores `{ rawJws, storefront }` on verify, or
`{ source: "webhook", storefront }` on a renewal). `404` if not found. Detail reads are
**never** filtered by `includeTest` — a sandbox payment is still returned, flagged.

---

## 2. Apple orders — `GET /sme/orders?source=APPLE`

One `Order` per Apple billing period (initial + each renewal). This is the best surface
for "what did this user buy and is their premium active".

```
GET /sme/orders?source=APPLE&page=&limit=&userId=&orderStatus=&exam=&startDate=&endDate=&search=&includeTest=
GET /sme/orders/:id
```

| Query | Description |
|---|---|
| **source** | `APPLE` for Apple only |
| orderStatus | `CREATED, PAID, FAILED, CANCELLED`. Apple orders are created as **`PAID`**. |
| **exam** | Filters `orders.exam_id`. Slug, or the literal **`none`** to select exactly the unresolved-product orders. **400 on an unknown slug.** |
| search | `receipt` (Apple receipts look like `apple_…` / `apple_wh_…`) or user email |
| **includeTest** | `true` to include sandbox orders. Default `false`. |

**What staging returns today**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/orders?source=APPLE&limit=2` → **200**, verbatim:

```json
{ "data": [], "total": 0, "page": 1, "limit": 2, "hasMore": false }
```

Same reason as §1 — no Apple purchase has ever reached staging. The envelope is live;
the row below is **shape from code**. A fully-captured *Razorpay* order of the identical
shape (every shared key, `reconcileHint` included) is in
[`SME_PORTAL_API.md` §2.3](./SME_PORTAL_API.md); the Apple-only keys —
`appleTransactionId`, `storefrontCurrency`, `chargedAmount` — are `null`/absent there,
which is itself the captured proof that they are per-source.

**Response 200** `data[]` (Apple example — an offer-code redemption on a US storefront,
shape from code):
<!-- shape verified against code @4f6622e — Apple rows do not exist on staging, see note above -->
```json
{
  "id": "…",
  "userId": "…",
  "razorpayOrderId": null,
  "appleTransactionId": "2000000812345678",
  "paymentSource": "APPLE",
  "planType": "ANNUAL",
  "examId": "upsc-cse",
  "amount": 3999,
  "currency": "INR",
  "offerCode": "INDE50",
  "listAmount": 5900,
  "storefrontCurrency": "USD",
  "chargedAmount": 47.99,
  "isTest": false,
  "receipt": "apple_2000000812345678",
  "status": "PAID",
  "premiumGrantedAt": "2026-06-28T10:15:02.000Z",
  "reconcileHint": "OK",
  "createdAt": "2026-06-28T10:15:02.000Z",
  "updatedAt": "2026-06-28T10:15:02.000Z",
  "user": { "id": "…", "email": "…", "name": "…", "phoneNumber": "…" },
  "payments": [ { "…": "see §1 shape" } ]
}
```

### 2.1 The six fields that decide what this order means

| Field | Meaning on an Apple order |
|---|---|
| `examId` | Which exam was bought, resolved from the **signed** `productId` via `exam_plans.apple_product_id`. **`null` when that lookup missed** — see `reconcileHint` below. |
| `isTest` | Sandbox purchase. iOS sandbox reaches the **production** backend (Apple has no separate endpoint), so every TestFlight/StoreKit test writes a genuine `PAID` order at a real price — one evening produced ₹59,000 of phantom revenue. Excluded from the lists by default; **never** counted as revenue. |
| `offerCode` | The campaign this redemption maps to, matching the Razorpay convention. `null` for a standard-price purchase — which is exactly why a renewal at list price inside a campaign window can never be mis-attributed to the campaign. |
| `listAmount` | ₹, the undiscounted price, set **only** when `amount` is a real charged amount — so "discounted" is never inferred from a guess. |
| `storefrontCurrency` | Apple's **true** storefront currency (`"USD"`, `"AED"`, …). `null` for Razorpay/MANUAL, and for any Apple transaction where Apple supplied no price (pre-2023 JWS). |
| `chargedAmount` | What Apple **actually charged**, in that storefront's currency, as a major unit. |

> ### 🔴 `amount` / `currency` are INR-only — do not read them as "what Apple charged"
>
> For an **INR** storefront, `amount` *is* the charged amount. For **any other**
> storefront, `amount` falls back to the configured **INR list price** and `currency`
> stays `"INR"`, while the real charge lives in `storefrontCurrency` + `chargedAmount`.
>
> This is deliberate, not a bug to route around: every existing revenue sum (the SME
> campaign funnel, the transaction views) already reads `amount` as INR paise, so
> redefining it would corrupt history. The consequence for the portal is concrete —
> **never `SUM(amount)` across a mixed-storefront set and call it revenue**, and never
> add `amount` to `chargedAmount`. Show `chargedAmount storefrontCurrency` when it is
> present, and fall back to `amount` only when it is not.
>
> Known gap, documented rather than silent: `chargedAmount` assumes a 2-decimal currency
> (INR, USD, EUR, GBP…). A 0-decimal (JPY) or 3-decimal (KWD) storefront would need a
> currency-aware divisor; none is configured for this app today.

### 2.2 Reconcile — `reconcileHint`

Computed server-side. The old rule ("`PAID` + `premiumGrantedAt=null` → grant it") is
**no longer safe on its own**, because the two causes need opposite fixes:

| `reconcileHint` | Means | Fix |
|---|---|---|
| `OK` | Granted, or never became money. | Nothing. |
| `GRANT_PENDING` | `PAID`, **exam known**, not granted. Verify failed or a webhook never landed. | Re-run the grant with **this order's `examId`**. |
| `UNRESOLVED_EXAM` | `PAID`, not granted, **`examId` null**. Apple product with no `exam_plans` row — real money, unknown product. | **Seed the `exam_plans` row first** (map the `appleProductId` to its exam + plan), then grant with the right `examId`. **Never grant blind.** |

Find the whole bucket with `GET /sme/orders?source=APPLE&exam=none`. Sandbox orders are
**not** special-cased: a test purchase that failed to grant is still a broken grant path,
and it is already labelled `isTest`.

**Other notes:**
- `premiumGrantedAt` is stamped on healthy Apple orders. It is also the **idempotency
  marker** — it is what stops a redelivered webhook extending premium twice.
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
| ~~exam~~ | **Accepted by the shared query DTO but deliberately ignored here.** See the note under the field reference. |

**What staging returns today**
<!-- captured from staging 2026-09-21, backend f6329e6 -->
`GET /sme/webhooks?source=APPLE&limit=2` → **200**, verbatim:

```json
{ "data": [], "total": 0, "page": 1, "limit": 2, "hasMore": false }
```

Apple sends its server notifications to production only, so this list is empty on
staging. **The row shape itself was captured** — dropping `source=APPLE`
(`GET /sme/webhooks?limit=1`, `total: 12`) returns a real Razorpay row with **every key
listed in the field reference below**, which is the point: the two providers share one
row shape and differ only in `provider`/`source`, `eventType` and how `payload` is
encoded.

```json
{
  "data": [
    {
      "id": "7d93fb2a-5e64-4182-b2a9-35fbe7a04d86",
      "eventId": "XXXXXXXXXXXXXX",
      "eventType": "settlement.processed",
      "source": "razorpay",
      "payload": "{\"settlement\":{\"entity\":{\"id\":\"setl_XXXXXXXXXXXXXX\",\"entity\":\"settlement\",\"amount\":48452,\"status\":\"processed\",\"fees\":0,\"tax\":0,\"utr\":\"XXXXXXXXXXXXXXXXXXXX\",\"created_at\":1789717392}}}",
      "processedAt": "2026-09-18T07:46:24.836Z",
      "provider": "razorpay",
      "kind": "webhook",
      "result": "processed",
      "error": null,
      "userId": null,
      "orderId": null,
      "examId": null,
      "receivedAt": "2026-09-18T07:46:24.833Z"
    }
  ],
  "total": 12,
  "page": 1,
  "limit": 1,
  "hasMore": true
}
```

> **That captured row is a `settlement.processed` event: `userId`, `orderId` and
> `examId` are all `null` and `result` is still `"processed"`.** Account-level provider
> events legitimately resolve to nobody. Do not render a null `userId` as an error, and
> do not read `result: "processed"` as "a user got premium".

**Response 200** `data[]` (Apple example — shape from code):
<!-- shape verified against code @4f6622e — Apple rows do not exist on staging, see note above -->
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
  "examId": "upsc-cse",
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
| **`examId`** | Read off the **order the event was matched to** — `payment_events` carries no exam of its own, it is the raw provider envelope. `null` when the event matched no order (unknown Apple product, a webhook for a foreign account) or when the order it matched has no resolved exam. One extra query per page, not per row. |
| `receivedAt` | When we received it (always set; default sort key, newest first). |
| `processedAt` | When processing finished (may be null if still pending/errored). |

> **`?exam=` does not filter this list.** Hiding events by an exam they were never
> stamped with would hide exactly the unmatched Apple notifications a reconcile pass is
> hunting for. Filter client-side on the returned `examId` if you need to, and treat
> `examId: null` here as "this event never reached an order", not as missing data.

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

> **`apple_subscriptions` is still not exposed — re-verified against the code @4f6622e.**
> The current subscription **state** (active / expired / grace / revoked, expiry date,
> auto-renew flag) lives in `apple_subscriptions`, keyed on the stable
> `originalTransactionId`. Nothing under `src/modules/sme/` reads that table: the only
> Apple-shaped thing the SME surface owns is `exams.appleSubscriptionGroupId` (create/
> update on `/sme/exams`), which is configuration, not subscription state. If the portal
> needs a live "is this subscription active and when does it renew" view — beyond
> reconstructing it from the event stream + order expiry — ask and we'll expose it.
>
> The nearest thing that **is** exposed is `entitlements[]` on the user row
> (`SME_PORTAL_API.md` §1.1): per exam, it gives `status`, `source` and `expiresAt` from
> `user_exam_entitlements`. That is the *entitlement* we granted, not Apple's own
> renewal state, so it will not tell you that auto-renew was switched off.

---

## 4. Caveats checklist

- **`/sme/webhooks` is now populated.** Previously empty for everyone. Non-breaking
  (additive fields), but confirm any code that *asserted* it was empty.
- **`payload` for Apple is a JWS string, not JSON.** Don't blindly `JSON.parse` it.
- **`MANUAL` source has no events** — `source=MANUAL` on `/sme/webhooks` returns `[]`.
- **Apple amounts are two different things.** `amount`/`currency` are **INR-only** (the
  charged amount on an INR storefront, the INR list price otherwise);
  `chargedAmount`/`storefrontCurrency` are the real charge in the buyer's own currency.
  See the red box in §2.1 before building any total.
- **Sandbox orders are hidden by default** on `/sme/orders` and `/sme/transactions`
  (`isTest`, `includeTest=true` to opt back in). Detail reads and a user's
  `recentOrders` are never filtered — they show `isTest` instead.
- **A `PAID` Apple order can legitimately have no exam** (`examId: null`,
  `reconcileHint: "UNRESOLVED_EXAM"`). Seed `exam_plans`, then grant — never guess.
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
| Apple purchases for **one exam** | `GET /sme/orders?source=APPLE&exam=appsc-group-1` |
| Apple money taken for an **unknown product** | `GET /sme/orders?source=APPLE&exam=none` |
| Which product ids / subscription groups exist | `GET /sme/exams` (`exam_plans.appleProductId`, `exams.appleSubscriptionGroupId`) |
| Both providers' events | `GET /sme/webhooks` |
