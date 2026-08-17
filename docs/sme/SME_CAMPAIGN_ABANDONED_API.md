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
| `ORDER` | Android, web | **Certain** | An `orders` row stamped with the campaign code, never PAID. The Razorpay/web path writes this row *before* the checkout sheet opens, so the row **is** the Join tap. |
| `TAP` | iOS only | **Inferred** | An `upgrade_tapped` / `checkout_opened` client event with `gateway = apple`. |
| `BOTH` | — | Certain | The same user did both. |

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
| `signal` | `all` | `all` · `order` (certain only) · `tap` (inferred only) |
| `minAppVersion` | `1.8` | Dotted-numeric floor applied to the **iOS TAP signal only**. Pass `0` to disable. |
| `page` | `1` | |
| `limit` | `20` | max `100` |

**There is deliberately no date parameter.** The code and the window are read off the campaign row.
Letting the portal pass its own window would make a wrong answer indistinguishable from a right one.

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
  "pageCounts": { "order": 1, "tap": 10, "both": 1 },   // THIS PAGE only, not the corpus
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

1. **"Viewed the offer screen but never tapped" does not exist on any platform.** `GET /paywall` is
   not persisted per user, and `paywall_viewed` fires from the *standard* paywall screen, never from
   `OfferScreen`. There is no view-stage number to chart.
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
