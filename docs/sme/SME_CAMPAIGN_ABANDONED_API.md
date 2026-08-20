# SME — Campaign Abandonment API

**Endpoint:** `GET /api/v1/sme/offers/:id/abandoned`
**Auth:** `x-api-key: <API_KEY_SECRET>` (same key as every other `/sme/*` route)
**Response:** raw JSON. **There is no `{success, data}` envelope** — `ResponseInterceptor` exists in
the codebase but is never registered. Parse the shape below literally.

Answers one question: **who reached for this campaign and did not end up paying?**

> **Building a screen from this? You MUST invoke the `frontend-design` skill first.**
> This endpoint produces a work queue a human acts on, and the ORDER/TAP distinction below is
> load-bearing — a UI that blends the two is actively misleading.

---

## 1. The one thing to understand before you build anything

Every row carries a **`signal`** field. It is not a category — it is a **confidence level**, and the
two levels come from completely different places.

| `signal` | Platform | Confidence | Where it comes from |
|---|---|---|---|
| `VIEW` | **all** | **Certain** | The user opened the **campaign paywall** and never created an order — "saw the offer and left". The server resolved the campaign for them, so this is an observation, not a client claim. **Added 2026-08-20.** |
| `ORDER` | Android, web | **Certain** | An `orders` row stamped with the campaign code, never PAID. The Razorpay/web path writes this row *before* the checkout sheet opens, so the row **is** the Join tap. |
| `TAP` | iOS only | **Inferred** | An `upgrade_tapped` / `checkout_opened` client event with `gateway = apple`. |
| `BOTH` | — | Certain | The same user did both. |

A user is counted at their **furthest** stage: someone who viewed *and* ordered is `ORDER`,
not `VIEW`. So the four signals partition the audience — they never double-count.

⚠️ **`VIEW` is deploy-forward.** Capture began 2026-08-20, when the paywall resolver started
stamping the campaign code onto the usage row. There is **no history before that date** — a
window that opens earlier under-reports views, and only views. Every other signal is complete
for the full campaign history.

**Why iOS is different, and why we cannot fix it.** Apple writes no order until a transaction
*completes*. An iOS user who taps Join, sees the StoreKit sheet and backs out leaves **no order
anywhere** — the client event is the only trace that exists. And that event **does not carry the
offer code**, because the mobile funnel emitter does not thread it through.

**The consequence you must not paper over:** an iOS user who tapped Buy on the *standard* paywall
during the campaign window is **indistinguishable** from one who tapped Join on the campaign screen.
`TAP` rows are upgrade intent inside the window, not proven campaign intent.

Do not sum `ORDER + TAP` into a single "abandoned carts" headline. Show them separately, or label
the total as approximate.

---

## 2. Request

```
GET /api/v1/sme/offers/62b20aac-eb13-42af-89bb-e36591819e7b/abandoned?signal=all&page=1&limit=20
x-api-key: <API_KEY_SECRET>
```

`:id` is the campaign **UUID** from `GET /sme/offers` — not the code.

| Param | Default | Notes |
|---|---|---|
| `signal` | `all` | `all` · `view` · `order` (certain only) · `tap` (inferred only) |
| `from` | campaign start | ISO lower bound. **Clamped to the campaign window — a wider value cannot widen it**, so a number from this endpoint always describes this campaign. |
| `to` | campaign end | ISO upper bound. Clamped the same way. |
| `minAppVersion` | `1.8` | Dotted-numeric floor applied to the **iOS TAP signal only**. Pass `0` to disable. |
| `page` | `1` | |
| `limit` | `20` | max `100` |

**`from`/`to` narrow, they never widen.** The code and the campaign window are read off the campaign
row; a caller may scope *inside* that window but cannot reach outside it, so a wrong answer stays
distinguishable from a right one.

🔑 **Why you usually want `from`.** A campaign goes LIVE days before marketing actually sends the
link — FREEDOM15 opened **10 Aug** but was not promoted until **14 Aug**. Nothing in the system
records "when we launched", so the untrimmed window reports four days of internal setup traffic as
demand. Pass `from` = the real promotion date.

### Why `minAppVersion` defaults to 1.8

`OfferScreen` ships in mobile **1.8**. A 1.7 build cannot send an offer code at all, and `FREEDOM15`
is `requiresCode: true` — so a 1.7 tap can *never* have been campaign intent. Counting it would be a
pure false positive. Events whose `app_version` is absent or non-numeric are always excluded: a
version we cannot attribute cannot be shown to be campaign-capable.

---

## 3. Response

```jsonc
{
  "campaign": {
    "id": "62b20aac-eb13-42af-89bb-e36591819e7b",
    "code": "FREEDOM15",
    "name": "Freedom Sale 2026",
    "status": "LIVE",
    "startsAt": "2026-08-09T18:30:00.000Z",
    "endsAt":   "2026-08-23T18:29:00.000Z"
  },
  "window": { "from": "...", "to": "..." },   // what was actually queried, after clamping
  "funnel": {                                 // CAMPAIGN-WIDE distinct users, not page-local
    "viewed": 34,            // opened the campaign paywall (deploy-forward)
    "joined": 9,             // created a campaign-priced order
    "paid": 0,               // completed payment
    "viewedNotJoined": 25,   // saw the offer and left  ← the "just visited" number
    "paidByPlatform": { "apple": 0, "razorpay": 0 },
    "revenuePaise": 0,
    "discountGivenPaise": 0,        // actually handed over (only on PAID orders)
    "discountUnclaimedPaise": 1531700,  // offered and never claimed
    "orderCount": 17,        // ORDERS, not users — 9 users made 17 of them
    "viewedIsDeployForward": true
  },
  "pageCounts": { "view": 0, "order": 1, "tap": 10, "both": 1 },   // THIS PAGE only
  "data": [
    {
      "userId": "45c3c221-e811-472b-912b-9fe4e3993b01",
      "name": "Pranay tej",
      "email": "titangaming1109@gmail.com",
      "phoneNumber": "+916369842370",
      "userStatus": "UNSUBSCRIBED",
      "signal": "ORDER",
      "attempts": 2,
      "firstAttemptAt": "2026-08-14T11:46:04.839Z",
      "lastAttemptAt":  "2026-08-14T13:27:51.702Z",
      "lastOrderId": "…",
      "lastOrderStatus": "CREATED",
      "amountPaise": 499900,        // null on TAP rows
      "listAmountPaise": 590000,    // null on TAP rows
      "platform": "android"         // may be null — see §5
    }
  ],
  "total": 12,
  "page": 1,
  "limit": 20,
  "hasMore": false
}
```

**`pageCounts` is page-local.** A corpus-wide breakdown would need a second full scan, and the portal
renders these next to the rows it just received. Use `total` for the headline count.

`amountPaise` / `listAmountPaise` are **paise** (`499900` = ₹4,999). They are `null` on `TAP` rows —
there is no order, so there is no amount. Render a dash, never ₹0.

---

## 4. Who is deliberately excluded

A row is dropped if **any** of these hold. All of them are intentional; do not ask for them to be
relaxed without reading why.

- **The user converted.** Premium is currently active, *or* they have a PAID order any time after the
  campaign started — by any route, including a later standard-price purchase. Chasing someone who
  already paid is worse than having no list.
- **The order was already granted** (`premium_granted_at` set) or is `PAID`.
- **It is an iOS sandbox order** (`is_test = true`). TestFlight/StoreKit tests hit the production
  backend by design and manufacture real-looking orders; they are not real intent.
- **The account is `SUSPENDED` / `LOCKED` / `INACTIVE`.**

**Verified on production:** three users held unpaid `FREEDOM15` orders. One is premium through
2026-09-09 and was correctly excluded as converted; the other two appear as `ORDER` / `BOTH`.

---

## 5. What this endpoint cannot tell you

Read this section before promising a funnel chart to anyone.

1. ~~"Viewed the offer screen but never tapped" does not exist.~~ **Solved 2026-08-20** — this is now
   the `VIEW` signal and `funnel.viewedNotJoined`. The resolver stamps the campaign it matched onto
   the usage row, so a view is server-observed on every platform. ⚠️ **Deploy-forward: there is no
   history before 2026-08-20**, and it cannot be backfilled — the code was never stored. For anything
   earlier, Mailchimp's click report is the only surviving record.
2. **`platform` is best-effort and partly retroactive.** It is the user's most recent
   `api_usage.platform`, which comes from the `x-platform` header. **prepmonkey-web did not send that
   header until 2026-08-16**, so web activity before then reads as `null`, not `"web"`. For the
   Aug 14–23 FREEDOM15 window, web-vs-Android attribution is therefore **partial**. Treat `null` as
   "unknown", never as a platform.
3. **Two web taps leave no trace at all**: if the Razorpay script fails to load, or the
   `POST /payments/orders` call itself 429s (that endpoint is rate-limited), the user tapped Join and
   nothing was written. Rare, but the list is a floor, not a census.
4. **Upstream upgrade taps are excluded on purpose.** The quota `UpgradeModal` emits
   `upgrade_tapped` without a `gateway` key. On production for this window that is 4 more iOS users
   and 4 Android users. They are *not* counted, because that button is not the campaign's Join button
   — it is one screen upstream. If you want "upgrade intent anywhere", that is a different endpoint.
5. **No campaign-screen view count, no per-step drop-off.** Closing gaps 1 and 5 requires threading
   the offer code into the mobile funnel emitter and firing `paywall_viewed` from `OfferScreen` —
   a mobile release, and it would only collect data from that release forward. Nothing retroactive.

---

## 5b. The four denominators, and why they differ

The single biggest source of confusion on this screen is that four true numbers describe the same
campaign and none of them match. From the live FREEDOM15 window:

| Number | Value | What it counts |
|---|---|---|
| `funnel.orderCount` | 17 | **orders** — one user can create several |
| `funnel.joined` | 9 | **distinct users** behind those orders |
| certain abandoners | 7 | those 9, minus users excluded by §4 (converted / granted) |
| `funnel.viewed` | — | distinct users who opened the campaign paywall |

**Always say which unit you are showing.** "17 orders from 9 users, 7 still worth chasing" is
honest; "17" beside "7" with no bridge reads as a bug. The 9 → 7 gap is §4 doing its job — surface
it as "2 excluded (already converted)", never as an unexplained drop.

---

## 6. How to use this data

- **`ORDER` and `BOTH` rows are a genuine recovery list.** These users chose a plan and reached the
  payment sheet at the campaign price. They have an email and usually a phone number. This is the
  highest-intent, lowest-volume audience the product has.
- **`TAP` rows are a signal, not a list.** Good for "iOS interest is 10× Android interest this
  window" — which is what production currently shows. Bad for a per-person outreach campaign, since
  some fraction were never looking at the campaign at all.
- **Pair with the existing notify endpoint.** `POST /sme/users/:id/notify` already exists; fan out
  over the returned `userId`s. This endpoint deliberately has no send action of its own — a read
  surface that can also blast a push is one misclick from a mistake.
- **Watch `attempts > 1`.** A user with repeated `CREATED` orders hit something that stopped them.
  Cross-reference `GET /sme/users/:id/incidents` before treating them as "not interested".

---

## 7. Production numbers at time of writing

Run against production on **2026-08-16**, campaign `FREEDOM15` (LIVE, `requiresCode: true`,
window 2026-08-09 18:30Z → 2026-08-23 18:29Z):

| `signal` | Users | Span |
|---|---|---|
| `BOTH` | 1 | 2026-08-14 |
| `ORDER` | 1 | 2026-08-14 |
| `TAP` | 10 | 2026-08-12 → 2026-08-16 |
| **total** | **12** | |

Sanity anchors from the same run: 5 unpaid `FREEDOM15` orders across 3 users (one excluded as
converted); 16 `checkout_opened` + 16 `upgrade_tapped` events with `gateway = apple` across 12
distinct users, all on app 1.8.

**Read the mix honestly: essentially all measurable abandonment in this campaign is iOS, and the iOS
number is the inferred one.** The Android/web side is 2 users — small enough that it is a support
list, not a statistic.

---

## 7b. Companion endpoint — who actually bought

**`GET /api/v1/sme/offers/:id/purchased`** — the other end of the funnel. Same auth, same
`from`/`to`/`page`/`limit`, same raw-response rule.

🔑 **Attribution is `orders.offer_code = <campaign code>`, and nothing else.** That single rule is
what stops general payment records bleeding into a campaign number:

- an **Apple renewal at standard price** inside the campaign window carries no offer code → excluded
- a **Razorpay subscription** taken at list price → excluded
- a **MANUAL SME grant** → excluded (a comp is an adjustment, not a conversion)
- **App Store sandbox** orders → excluded (`is_test`)

This is deliberately *not* "Apple revenue between these dates", which is the shape that produces
inflated campaign numbers. If a purchase is not stamped with the code, this campaign did not cause it.

```jsonc
{
  "campaign": { … }, "window": { … },
  "pageTotals": { "revenuePaise": 0, "savedPaise": 0 },   // page-local
  "data": [
    {
      "userId": "…", "name": "…", "email": "…", "phoneNumber": "…",
      "orderId": "…",
      "source": "APPLE",              // or RAZORPAY — the till that charged
      "amountPaise": 499900,          // charged
      "listAmountPaise": 590000,      // pre-discount
      "savedPaise": 90100,            // what the campaign actually gave away
      "planType": "ANNUAL",
      "orderedAt": "…", "premiumGrantedAt": "…", "premiumExpiresAt": "…",
      "capturedPaise": 499900         // from the payments ledger; see below
    }
  ],
  "total": 0, "page": 1, "limit": 20, "hasMore": false
}
```

⚠️ **`capturedPaise: null` on a PAID row is a real signal, not a rendering gap** — it means the order
was marked PAID without a captured payment row behind it. Surface it (a warning chip) rather than
defaulting it to the order amount; that mismatch is exactly the class of problem this list should
expose. Related: the known Razorpay payment-row gap, where a PAID order can legitimately lack a
Payment row.

---

## 8. Implementation notes (backend, for reference)

- **No new table, no new column, no migration.** Pure read over `orders`, `auth_events`, `user_auth`
  and `api_usage`.
- One query: two CTEs (`ord`, `tap`) → `FULL OUTER JOIN` on user id → exclusions → window-function
  `totalCount` so pagination costs one scan, not two.
- Code: `src/modules/sme/services/sme-offer.service.ts` (`abandoned`),
  `src/modules/sme/dto/sme-offer.dto.ts`, `src/modules/sme/controllers/sme-offer.controller.ts`.
- The web `X-Platform: web` header was added in `prepmonkey-web/src/lib/api.ts` at the same time and
  is what makes gap §5.2 shrink going forward.

Related: [SME_OFFERS_API.md](./SME_OFFERS_API.md) · [SME_ACTIVITY_TRAIL_API.md](./SME_ACTIVITY_TRAIL_API.md) ·
[SME_METRICS_DECODER.md](./SME_METRICS_DECODER.md)
